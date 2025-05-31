# Across Security Checklist
Across Protocol is an intent-based cross-chain protocol. It enables fast, low-cost, and secure asset transfers across multiple blockchain networks by leveraging a decentralized network of relayers and an optimistic verification model.

Across Protocol’s architecture breaks down into three main parts:
1. *Request for Quote*: This is where users submit their intents/orders, specifying what they want not execution paths.
2. *Relayer Network*: Enabling a competitive network of relayers to bid, claim and fill those orders.
3. *Settlement Layer*: A settlement layer to verify intent fulfillment and repay relayers.

## Core Security Assumptions
- Bridge security relies on optimistic verification with challenge periods
- Relayers are economic actors incentivized by fees but are untrusted

### Relayers can spoof messages
According to Across V3's [docs](https://docs.across.to/quickstart/embedded-crosschain-actions/crosschain-actions-integration-guide/using-the-generic-multicaller-handler-contract#security-and-safety-considerations), the protocol does not guarantee the integrity of the parameters that `handleV3AcrossMessage()` is called with:

> Avoid making unvalidated assumptions about the message data supplied to `handleV3AcrossMessage()`. Across does not guarantee message integrity, only that a relayer who spoofs a message will not be repaid by Across. If integrity is required, integrators should consider including a depositor signature in the message for additional verification. Message data should otherwise be treated as spoofable and untrusted for use beyond directing the funds passed along with it.

Also [docs](https://docs.across.to/quickstart/embedded-crosschain-actions/crosschain-actions-integration-guide/using-the-generic-multicaller-handler-contract#message-constraints) states:
> Handler contracts only use the funds that are sent to it. That means that the message is assumed to only have authority over those funds and, critically, no outside funds. This is important because relayers can send invalid relays. They will not be repaid if they attempt this, but if an invalid message could unlock other funds, then a relayer could spoof messages maliciously. 

More specifically, a malicious relayer can call `handleV3AcrossMessage()` with any arguments, regardless
of what was sent by users in the cross-chain transaction on the source chain. As such, all parameters of
`handleV3AcrossMessage()` must be validated to ensure they are not manipulated by a malicious relayer. 
Important args like `tokenSent` and `intentAmount` etc, should be validated.

Bug example: [Finding 3.2.9](https://github.com/superform-xyz/v2-core-public-cantina/blob/main/audits/2025.04.19-cantinacode-superform-core.pdf)


### Depositor Address and Refund Handling
When a deposit expires or a fill fails, refunds are sent to the depositor address on the origin chain. Across does not support specifying a separate refund address like other protocols (e.g., LayerZero). It’s critical to ensure that the [depositor](https://github.com/across-protocol/contracts/blob/0fee0264009e662a17e2cd8c22c4c493f12b8a03/contracts/SpokePool.sol#L449) address provided in the deposit is correct and able to receive refunds. Failing to do so can result in lost funds.

Across Docs [states](https://docs.across.to/concepts/intent-lifecycle-in-across#slow-fill-or-expiration-if-no-fill):
> In cases where a slow fill can't or does not happen and a relayer does not fill the intent, the intent expires. Like a slow fill, this expiry must be optimistically verified, which takes a few hours. Once this verification is done, the user is then refunded their money on the origin chain.

Bug example: [Finding 6.1.3](https://github.com/superform-xyz/v2-core-public-cantina/blob/main/audits/2025.03.24-sujithsomraaj-superform-core.pdf)


### The `outputAmount` should not be outside the recommended range otherwise tokens can be locked temporarily
If the inputAmount is set too high, it can take a while for the deposit to be filled depending on available relayer liquidity. BUT if the outputAmount is set too high, it can be unprofitable to relay. The contracts will not revert if the outputAmount is set outside of the recommended range, but it will probably lock up funds for an unexpected length of time. The protocols should make sure that the user is not inputting an `outputAmount` which is outside the recommended range. user given `outputAmount` should be vetted against this:
```
recommended outputAmount = inputAmount * ( 1 - relayerFeePct - lpFeePct).
```
or using the API:
> If you are using the API to set the outputAmount then you should set it equal to inputAmount * (1 - fees.totalRelayFee.pct) where fees.totalRelayFee is returned by the /suggested-fees endpoint.

Ref: [Across Docs](https://docs.across.to/reference/selected-contract-functions#deposit-1)

### Across doesn't send the origin sender address, which makes spoofable attacks easier
The Across protocol doesn't provide origin sender information in bridged messages, making the msgs spoofable if not handled correctly. This absence requires developers to implement additional verification measures to prevent attackers from exploiting this gap
Bug Example: [Finding 5.2.2](https://github.com/meliopolis/chainhopper-protocol/blob/main/docs/Spearbit-audit.pdf)

### `handleV3AcrossMessage` should be callable by only the Spoke Contract
Restricting `handleV3AcrossMessage` to be callable only by the authorized SpokePool contract (or another designated trusted contract) ensures that arbitrary or malicious actors cannot invoke this function with crafted or spoofed data. Without such access control, any external party could call the function with unauthorized messages, potentially causing unintended state changes, fund manipulation, or denial of service. This would undermine the protocol’s security guarantees and expose user funds to risk.
```solidity
    function handleV3AcrossMessage(
        address tokenSent,
        uint256 amount,
        address relayer,
        bytes memory message
    ) external override {
        // 1. Validate Sender [MUST]
        if (msg.sender != acrossSpokePool) {
            revert INVALID_SENDER();
        }
```