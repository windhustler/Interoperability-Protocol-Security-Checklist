# Arbitrum Security Checklist

## General

### Block number

Invoking `block.number` in a smart contract on Arbitrum will return the **L1** block number at which the sequencer received the transaction, not the **L2** block number. Also worth noting, the block number being returned is not necessarily the latest L1 block number. It is the latest synced block number and the syncing process occurs approximately every minute. In that period there can be ~5 new L1 blocks. So values returned by `block.number` on Arbitrum do not increase continuously but in "jumps" of ~5 blocks. If contract's logic requires tracking Arbitrum's L2 block numbers, that's possible as well using the precompile call `ArbSys(100).arbBlockNumber()`.

> Always check for incorrect use of `block.number` on Arbitrum chains, especially if it is used to track time over short periods.

Bug examples: [1](https://code4rena.com/reports/2022-12-tigris#m-15-_checkdelay-will-not-work-properly-for-arbitrum-or-optimism-due-to-blocknumber-), [2](https://solodit.cyfrin.io/issues/incorrect-use-of-l1-blocknumber-on-arbitrum-cantina-none-uniswap-pdf)

## L1 to L2 messaging

Retryable tickets are the canonical way to execute L1 to L2 messages in Arbitrum. User submits the message in the L1 Inbox (the main entrypoint for this is function [`createRetryableTicket`](https://github.com/OffchainLabs/nitro-contracts/blob/780366a0c40caf694ed544a6a1d52c0de56573ba/src/bridge/Inbox.sol#L261)) and once submission is finalized message will be (asynchronously) executed on L2. If execution fails for any reason, like TX running out of gas or smart contract reverts, the ticket stays in the buffer and can be manually redeemed (thus it is "retryable"). We will describe some common pitfalls related to the usage of retryable tickets.

### Out-of-order retryable ticket execution

One of the common issues is assuming that retryable tickets are guaranteed to be executed in the same order as they were created. This doesn't have to be the case. Let's say the L1 contract creates 2 retryables, A and B, in a single transaction. When it's time to execute A and B, gas price spikes on L2 so auto-redemption fails. At that point, anyone can manually redeem tickets by providing a new custom gas price. It is possible that user executes ticket B and only then ticket A. If protocol developers haven't anticipated such a situation and haven't built in guard rails, the protocol can be left in a bad state.

> Look for vulnerabilities in protocol that can come out of retryable tickets executing in different order than they were submitted

Bug examples: [1](https://docs.arbitrum.io/assets/files/2024_08_01_trail_of_bits_security_audit_custom_fee_token-7ce514634632f4735a710c81b55f2d27.pdf#page=12)

### Address aliasing and cross-chain authentication

When processing unsigned L1 to L2 messages, including retryable tickets, Arbitrum node will apply [address aliasing](https://docs.arbitrum.io/how-arbitrum-works/l1-to-l2-messaging#address-aliasing). That means the `msg.sender` on the L2 side will be modified in a deterministic way from the L1 address that issued the message. Why is this necessary? Let's say there is a contract deployed at address `0xabc` at both L1 and L2 chains. It is possible that contract `0xabc` on L1 is controlled by a different party than contract `0xabc` on L2 (remember [Wintermute](https://inspexco.medium.com/how-20-million-op-was-stolen-from-the-multisig-wallet-not-yet-owned-by-wintermute-3f6c75db740a)?). So, imagine a situation where a developer deploys a contract on Arbitrum which has critical functions gated by check `msg.sender == 0xabc` because developer controls `0xabc` multisig on Arbitrum. If there was no aliasing feature in place, a malicious owner of `0xabc` on Ethereum could send a retryable ticket and authenticate themselves to access the critical function! With aliasing, this is not possible because the address comparison check will fail.

On the other hand, there are cross-chain app usecases where the L2 contract has a function that shall be only callable by the L1 counterpart contract. In such cases, it is important to apply alias when checking the sender. If the sender is not aliased before the check, the function won't be callable at all. The check should look something like this:

```solidity
require(msg.sender == AddressAliasHelper.applyL1ToL2Alias(l1Counterpart), "only counterpart");
```

> When retryable ticket's sender address is checked make sure alias is applied where needed

Bug examples: [1](https://solodit.cyfrin.io/issues/msgsender-has-to-be-un-aliased-in-l2blastbridgefinalizebridgeethdirect-spearbit-none-draft-pdf)

### Spending the nonce

There are different ways in which retryable tickets can fail on L2. If the L2 gas price provided is too low, ie. due to a gas price spike, retryable ticket will not be scheduled for auto-redemption. On the other hand, auto-redemption can be scheduled, but fail due to too low gas limit provided (TX runs out of gas). In the latter case (aliased) sender's nonce will be spent. This can have important implications if for example L1 contract is used to deploy a contract on L2 and predict the address of the new contract. If the gas limit provided for deployment is too low, nonce 0 will be burned. Deployment can still be done by doing the manual redemption of retryable ticket, however, address of the deployed contract will be different than the one initially predicted by the L1 contract because nonce 1 was used for deployment instead of nonce 0.

Implementation example:  
 [`L1AtomicTokenBridgeCreator`](https://github.com/OffchainLabs/token-bridge-contracts/blob/5bdf33259d2d9ae52ddc69bc5a9cbc558c4c40c7/contracts/tokenbridge/ethereum/L1AtomicTokenBridgeCreator.sol#L52) is used to deploy and set up token bridge contracts on both L1 and L2 side of Arbitrum chains. It issues 2 retryable tickets to configure the L2 side:

- 1st ticket will create "factory" on L2
- 2nd ticket will call factory's [function](https://github.com/OffchainLabs/token-bridge-contracts/blob/5bdf33259d2d9ae52ddc69bc5a9cbc558c4c40c7/contracts/tokenbridge/arbitrum/L2AtomicTokenBridgeFactory.sol#L35) to deploy token bridge's L2 contracts

2nd ticket [assumes](https://github.com/OffchainLabs/token-bridge-contracts/blob/5bdf33259d2d9ae52ddc69bc5a9cbc558c4c40c7/contracts/tokenbridge/ethereum/L1AtomicTokenBridgeCreator.sol#L131..L132) that factory is deployed at an address from nonce 0. So it is critical to make sure that 1st ticket does not fail due to out-of-gas error, since that would burn the deployer's nonce 0. This is achieved by [hard-coding gas limit](https://github.com/OffchainLabs/token-bridge-contracts/blob/main/contracts/tokenbridge/ethereum/L1AtomicTokenBridgeCreator.sol#L90..L93) which is guaranteed to be big enough for factory deployment to succeed.

> When L1 contract is used to deploy contracts to L2 look for any hidden assumptions about the deployment nonce

### Permission to cancel the retryable ticket

When creating a new retryable ticket one of the parameters that are provided to the [`Inbox::createRetryableTicket`](https://github.com/OffchainLabs/nitro-contracts/blob/780366a0c40caf694ed544a6a1d52c0de56573ba/src/bridge/Inbox.sol#L261) is [`callValueRefundAddress`](https://github.com/OffchainLabs/nitro-contracts/blob/780366a0c40caf694ed544a6a1d52c0de56573ba/src/bridge/IInbox.sol#L81). This is the address where `l2CallValue` is refunded to in the case that retryable ticket execution fails on L2. But additionally `callValueRefundAddress` has the permission to cancel the ticket, making it permanently unclaimable. It is important to keep this in mind in case protocol is designed in a way where this can be exploited. Here's an example of such a scenario:

- protocol has a permissionless function on L1 that lets anyone create retryable ticket to execute action X on the L2
- protocol assumes once retryable ticket is submitted action X will surely be executed on L2
  - there is no way to re-trigger the action X
- attacker calls the L1 function to submit a retryable ticket but intentionally provides too small gas limit to make sure ticket can't be auto-redeemed on L2
- attacker also provides their own address for `callValueRefundAddress`
- retryable ticket auto-redemption fails and the attacker cancels the ticket before anyone else can redeem it
- as a consequence, action X can never be executed

> Make sure the account provided as `callValueRefundAddress` cannot misuse its capability to cancel the ticket in a way that hurts the protocol

Bug examples: [1](https://github.com/code-423n4/2024-05-olas-findings/issues/29)

### Safe vs unsafe retryable ticket creation

Arbitrum's `Inbox` contract offers 2 different entrypoints for creating retryables - [createRetryableTicket](https://github.com/OffchainLabs/nitro-contracts/blob/v3.0.0/src/bridge/Inbox.sol#L261) and [unsafeCreateRetryableTicket](https://github.com/OffchainLabs/nitro-contracts/blob/v3.0.0/src/bridge/Inbox.sol#L285). There are couple of important differences between the two.

Unsafe version **will not** check that value provided is enough to cover the L2 execution cost of `gasLimit * gasPrice`. Creator of retryable ticket can even provide 0 value for gas limit and gas price and submission will succeed (value provided still must cover `maxSubmissionCost`), but auto-redemption will not be scheduled and ticket will need to be executed manually by providing L2 funds.

There is also an important difference in how `excessFeeRefundAddress` (receives execution refund) and `callValueRefundAddress` (receives L2 call value if ticket is cancelled) are handled. Safe version will apply alias to both addresses in case they are a contract on L1 side, while unsafe version does not apply alias. So it is critical to ensure that the contract address used as the L2 refund address can control the refunded funds.

> When the unsafe function for creating retryable tickets is used make sure the L2 contract is able to control the refunded funds

## Oracle integration

### Sequencer uptime

Typically users interact with Arbitrum through sequencer. The sequencer is responsible for including and processing transactions. If the sequencer goes down users can still submit transactions using [Inbox](https://etherscan.io/address/0x4Dbd4fc535Ac27206064B68FfCf827b0A60BAB3f) contract on L1. However, such transactions will be included with potentially significant delay. In the worst case, if the sequencer is down for a prolonged time, transactions submitted to L1 can be force included only after 24h. In the context of price feeds, it means that reported prices can get stale and significantlly drift from the actual price. Thus protocols should always check the sequencer's uptime and revert the price request if the sequencer is down for more than a predefined period of time.

Chainlink's sequencer uptime feeds can be used for this purpose. An integration example can be found [here](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code).

> Integrations with Chainlink's price feeds on Arbitrum should always check the sequencer uptime in order to avoid stale prices. If missing, think in what ways can the protocol be negatively impacted

Bug examples: [1](https://code4rena.com/reports/2024-07-benddao#m-12--no-check-if-arbitrumoptimism-l2-sequencer-is-down-in-chainlink-feeds-priceoraclesol-), [2](https://github.com/shieldify-security/audits-portfolio-md/blob/main/Pear-v2-Security-Review.md#m-02-missing-check-for-active-l2-sequencer-in-calculatearbamount), [3](https://github.com/sherlock-audit/2023-05-perennial-judging/issues/37)

### Incorrect staleness threshold or feed decimals used

The staleness threshold is used to determine if the price fetched from the price feed is fresh enough. Most often staleness threshold should match the feed's heartbeat - the longest amount of time that can pass between consequent price updates. Also, feeds use different number of decimals (not necessarily related to the number of decimals used by underlying tokens). Using incorrect values for staleness or feed decimals can lead to critical outcomes. Different feeds on Arbitrum require different values for the mentioned parameters. For example:

- [LINK / ETH](https://arbiscan.io/address/0xb7c8Fb1dB45007F98A68Da0588e1AA524C317f27) has 24h heartbeat and uses 18 decimals
- [LINK / USD](https://arbiscan.io/address/0x86E53CF1B870786351Da77A57575e79CB55812CB) has 1h heartbeat and uses 8 decimals

> Make sure that the contract uses the correct staleness threshold and decimal precision for every price feed being integrated

Bug examples: [1](https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/166), [2](https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/9),
[3](https://github.com/solodit/solodit_content/blob/d4d5dbc/reports/Zokyo/2024-06-23-Copra.md#incorrect-staleness-threshold-for-chainlink-price-feeds)

### Price going out of aggregator's acceptable price range

Some Chainlink price feeds use `minAnswer` and `maxAnswer` variables to limit the price range that can be reported by the feed. If the price goes below the min price during flash crash, or goes above the max price, the feed will answer with an incorrect price.

Even though min/max answers have been deprecated on newer feeds, some older feeds still use it. Here are couple of examples on Arbitrum:

- [ETH / USD](https://arbiscan.io/address/0x639Fe6ab55C921f74e7fac1ee960C0B6293ba612) feed is limited to report price in range [\[$10, $1000000\]](https://arbiscan.io/address/0x3607e46698d218B3a5Cae44bF381475C0a5e2ca7#readContract)
- [USDC / USD](https://arbiscan.io/address/0x50834F3163758fcC1Df9973b6e91f0F0F0434aD3) feed is limited to report price in range [\[$0.01, $1000\]](https://arbiscan.io/address/0x2946220288DbBF77dF0030fCecc2a8348CbBE32C#readContract)
- [USDT / USD](https://arbiscan.io/address/0x3f3f5dF88dC9F13eac63DF89EC16ef6e7E25DdE7) feed is limited to report price in range [\[$0.01, $1000\]](https://arbiscan.io/address/0xCb35fE6E53e71b30301Ec4a3948Da4Ad3c65ACe4#readContract)

> Check if the price feed used by the protocol has `minAnswer` and `maxAnswer` configured, and analyze the implications of the unlikely case that the actual price goes out of the range

Bug examples: [1](https://github.com/pashov/audits/blob/master/team/md/Cryptex-security-review.md#m-02-circuit-breakers-are-not-considered-when-processing-chainlinks-answer), [2](https://code4rena.com/reports/2024-05-bakerfi#m-06-min-and-maxanswer-never-checked-for-oracle-price-feed)

## Orbit chains

Orbit chains are custom chains deployed using Arbitrum's Nitro software stack. They are mostly deployed as L2s on top of Ethereum or as L3s on top of Arbitrum.

### Using custom fee token

Transaction fees on Arbitrum are paid in ETH. Where does the ETH come from? Users lock their ETH in the bridge contract on the parent chain (Ethereum) and Arbitrum node mints the same amount of native currency on the child chain (Arbitrum) in user's account.

But Orbit chains don't need to necessarily use ETH to pay for gas. Orbit chain owner can select, at deployment time, any ERC20 to be the fee token for new chain. Once set, fee token for the chain cannot be changed. In this case user will lock the ERC20 fee token on the parent chain and Arbitrum node will mint the same amount of native currency on the Orbit chain. Customized bridge contracts are used on the parent chain for this purpose - `ERC20Bridge`, `ERC20Inbox` and `ERC20Outbox` are used instead of `Bridge`, `Inbox` and `Outbox`.

> Check if Orbit chain uses custom fee token by calling [nativeToken()](https://github.com/OffchainLabs/nitro-contracts/blob/780366a0c40caf694ed544a6a1d52c0de56573ba/src/bridge/ERC20Bridge.sol#L38) function on the chain's bridge contract. If chain is ETH based call will revert, otherwise it will return the address of the fee token on the parent chain

### Custom fee token with non-18 decimals

Let's say a team creates a new Orbit chain which uses USDC as the custom fee token. There is an inherent mismatch - USDC uses 6 decimals, while native currency is assumed to have 18 decimals. Orbit's bridge contracts deal with this in following way - when the native fee tokens are deposited from the parent chain to the Orbit chain, the deposited amount is scaled to 18 decimals. When native currency is withdrawn from the Orbit chain to the parent chain the amount is scaled from 18 back to the fee token's decimals. It's important to notice that scaling process can round-down amounts and cause user to lose some dust.

As an example - Orbit chain is deployed as an L3 on top of Arbitrum and chain uses USDC as custom fee token. User bridges 10 USDC from Arbitrum to Orbit chain.

```
Deposited on Arbitrum: 10000000 (10 USDC)
Minted on Orbit (scale to 18 decimals): 10000000000000000000 (10 "ETH" of native currency)
```

On Orbit chain user earns some yield, let's say `2*10^18 + 300`. User's total balance now is `12000000000000000300`. User decides to bridge it back to Arbitrum.

```
Withdrawn from Orbit: 12000000000000000300 (12.0000000000000003 "ETH" of native currency)
Unlocked on Arbitrum: 12000000000000000300 / 10^12 = 12000000 (12 USDC)
```

User gets back 12 USDC, while the dust is lost in conversion process and it stays locked in the `ERC20Bridge` contract.

> Look for any issues that can be caused by scaling or rounding logic when Orbit chain's fee token uses non-18 decimals

### Retryable tickets in Orbit chains using non-18 decimals fee tokens

When creating a retryable ticket to send L1 to L2 message to Orbit chain user needs to provide [4 numerical values](https://github.com/OffchainLabs/nitro-contracts/blob/v3.0.0/src/bridge/IERC20Inbox.sol#L21..L36):

- l2CallValue (call value for execution on L2)
- maxSubmissionCost (value to pay for storing the retryable ticket)
- maxFeePerGas (gas price bid on L2)
- tokenTotalFeeAmount (amount of fee tokens to be transferred from user to cover all the costs)

Let's again assume Orbit chain's fee token is USDC. In that case it is not obvious whether the retryable ticket's parameters are denominated in USDC (6 decimals) or in the native currency (18 decimals). Answer is mixed - `tokenTotalFeeAmount` is denominated in USDC's decimals, while other 3 params are denominated in native currency's decimals. This is because `tokenTotalFeeAmount` signals how many tokens user needs to spend on **parent chain**, while other 3 params signal how execution is to be performed on **Orbit chain**.

> When contract is integrating the Orbit retryable tickets, check that all the ticket input parameters are properly denominated

## Useful resources

- [Arbitrum official docs](https://docs.arbitrum.io/intro/)
- [Out-of-order retryable tickets execution](https://blog.trailofbits.com/2024/03/01/when-try-try-try-again-leads-to-out-of-order-execution-bugs/)
