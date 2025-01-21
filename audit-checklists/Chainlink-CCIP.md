# Chainlink CCIP Security Checklist

## Message Execution Options

### Fees
CCIP supports fee payments in LINK and in alternative assets, including blockchain-native gas tokens and their ERC-20 wrapped versions.
The fee is calculated by the following [formula](https://docs.chain.link/ccip/billing#billing-mechanism):
```
fee = blockchain fee + network fee
```
- Blockchain fee: An estimation of the gas cost the node operators will pay to deliver the CCIP message to the destination blockchain.
- Network fee: A fee paid to CCIP service providers, which varies based on the use case, the chosen lanes, and the fee token (See the [network fee table](https://docs.chain.link/ccip/billing#network-fee-table)).

> **Note**: It is recommended to use the `getFee` function to estimate the fee accurately.


### Additional Arguments
There are two additional arguments that can be included when sending a message:
```solidity
extraArgs: Client._argsToBytes(
    Client.EVMExtraArgsV2({
        gasLimit: 200_000,
        allowOutOfOrderExecution: true
    })
)
```

- **`gasLimit`**: Specifies the maximum amount of gas that CCIP can consume to execute `ccipReceive()` on the contract located on the destination blockchain. It is the main factor in determining the fee to send a message.   
  - If `gasLimit` is not provided, the default value is `200,000`. Ensure that `ccipReceive()` does not require more gas than this default.
  - Unspent gas is **not refunded**.
- **`allowOutOfOrderExecution`**: Controls the execution order of your messages on the destination blockchain. This parameter is available only on lanes where the `Out of Order Execution` property is set to *Optional* or *Required*.  
  - When `allowOutOfOrderExecution` is **Optional**: You can set it to either `true` or `false`.  
  - When `allowOutOfOrderExecution` is **Required**: You must set it to `true`. This acknowledges that messages may be executed out of order. If set to `false`, the message will revert and will not be processed.

> **Note**: It is recommended to use mutable `extraArgs` (not hardcoded) in production deployments.


### CCIP Rate Limits
Rate limits consist of a maximum capacity and a refill rate, which determines how quickly the maximum capacity is restored after a token transfer consumes some or all of the available capacity.

- **[Token Pool Rate Limit](https://docs.chain.link/ccip/architecture#token-pool-rate-limit)**: For each supported token on every individual [lane](https://docs.chain.link/ccip/concepts#lane), this rate limit manages the total number of tokens that can be transferred within a specified time frame. This limit is independent of the token's USD value.  
- **[Aggregate Rate Limit](https://docs.chain.link/ccip/architecture#aggregate-rate-limit)**: Each lane also has an aggregate rate limit that caps the total USD value of transfers across all supported tokens on that lane.


## Token Decimal Handling
When tokens move between blockchains with different decimal places, rounding may occur. This rounding can impact small amounts of tokens during cross-chain transfers.  

**Example**:  
Token precision on Chain A is 18 decimals, and on Chain B is 9 decimals.  
```
• Sent from A: 1.123456789123456789  
• Received on B: 1.123456789  

• Lost: 0.000000000123456789  
```


## Manual Execution
In certain exceptional conditions, users might need to manually execute the transaction on the destination blockchain. 
These conditions include:

- **Unhandled exceptions**: Logical errors in the receiver contract.  
- **Gas limit exceeded for token pools**: If the combined execution of the required functions (`balanceOf` checks and [releaseOrMint](https://github.com/smartcontractkit/ccip/blob/bca2fe0/contracts/src/v0.8/ccip/pools/BurnMintTokenPoolAbstract.sol#L36)) surpasses the default gas limit of 90,000 on the destination blockchain.
- **Insufficient gas**: If the gas limit provided in the [extraArgs](https://github.com/smartcontractkit/ccip/blob/5e7b209/contracts/src/v0.8/ccip/libraries/Client.sol#L49) parameter of the message is insufficient to execute the `ccipReceive()` function. 
- **Smart Execution time window exceeded**: If the message cannot be executed on the destination chain within CCIP’s Smart Execution time window (currently set to 8 hours). 
  - This could occur during extreme network congestion or gas price spikes.  

> **Note**: After the Smart Execution time window expires, all subsequent messages will fail until the **failing message** is successfully executed.

Bug examples: [1](https://code4rena.com/reports/2024-04-renzo#m-04-price-updating-mechanism-can-break)


## Requirements for Token Pools

### Gas Requirements
On the destination blockchain, the CCIP OffRamp contract performs three key operations when releasing or minting tokens:

1. **`balanceOf` before minting/releasing tokens**
2. **[`releaseOrMint`](https://github.com/smartcontractkit/ccip/blob/bca2fe0/contracts/src/v0.8/ccip/pools/LockReleaseTokenPool.sol#L64) to mint or release tokens**
3. **`balanceOf` after minting/releasing tokens**

> **Note**: If the combined gas consumption of these three operations exceeds the default gas limit of 90,000 on the destination blockchain, the CCIP execution will fail.

### Custom Token Pools
If custom `TokenPool` is build, it is crucial to follow these guidelines:

- **For Burn and Mint mechanisms**:  
  Custom token pool should inherit from [BurnMintTokenPoolAbstract](https://github.com/smartcontractkit/ccip/blob/bca2fe0/contracts/src/v0.8/ccip/pools/BurnMintTokenPoolAbstract.sol).
- **For Lock and Release mechanisms**:  
  Custom token pool can:  
  - Inherit from [TokenPool](https://github.com/smartcontractkit/ccip/blob/478f0e5/contracts/src/v0.8/ccip/pools/TokenPool.sol) and implement the [ILiquidityContainer](https://github.com/smartcontractkit/ccip/blob/19dafcc/contracts/src/v0.8/liquiditymanager/interfaces/ILiquidityContainer.sol) interface.  
  - Or directly inherit from [LockReleaseTokenPool](https://github.com/smartcontractkit/ccip/blob/bca2fe0/contracts/src/v0.8/ccip/pools/LockReleaseTokenPool.sol) and reimplement the `lockOrBurn` and `releaseOrMint` functions as needed.


## Validate CCIP Inputs
If users can call the `ccipSend()` function, it's important to check the CCIP inputs before sending the message. If the state changes before the message is sent and an attacker provides wrong inputs, funds might get stuck in the contract.

Bug examples: [1](https://github.com/sherlock-audit/2024-08-winnables-raffles-judging/issues/50)


## Useful resources

- [Chainlink CCIP developer docs](https://docs.chain.link/ccip)
- [CCIP local simulator in Foundry](https://docs.chain.link/chainlink-local/build/ccip/foundry/local-simulator)
