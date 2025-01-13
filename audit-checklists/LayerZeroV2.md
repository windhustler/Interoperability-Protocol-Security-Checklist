# LayerZeroV2 security checklist

## Message Execution Options
LayerZeroV2 provides message execution options, where you can specify gas amount, `msg.value` and other options for the destination transaction. This info gets picked up by the application defined [Executor](https://docs.layerzero.network/v2/home/permissionless-execution/executors) contract. 

### Native airdrop option cap
The default LayerZero [Executor](https://etherscan.io/address/0x173272739Bd7Aa6e4e214714048a9fE699453059) contract has a [configuration](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/messagelib/contracts/Executor.sol#L63) for every chain. If you're sending a message from Ethereum -> Polygon it will use the Polygon configuration to estimate the fee while doing a few sanity checks. The structure of the configuration is as follows:
```solidity
    struct DstConfig {
        uint64 lzReceiveBaseGas;
        uint16 multiplierBps;
        uint128 floorMarginUSD; // uses priceFeed PRICE_RATIO_DENOMINATOR
        uint128 nativeCap;
        uint64 lzComposeBaseGas;
    }
````

One of the values in the configuration is `nativeCap`, which is the maximum amount of native tokens that can be sent to the destination chain. 

Here is an example of configuration for Polygon:

```bash
cast call 0x173272739Bd7Aa6e4e214714048a9fE699453059 "dstConfig(uint32)(uint64,uint16,uint128,uint128,uint64)" 30109 --rpc-url https://eth.llamarpc.com // nativeCap is 1500000000000000000000 ~ 1500e18 MATIC
```

The maximum amount of native tokens to airdrop from Ethereum -> Polygon is 1500e18 MATIC.

## OFT standard

### Dust removal
The OFT standard was created to allow transferring tokens across different blockchain VMs. WHile `EVM` chains support `uint256` for token balances, many non-EVM chains use `uint64`. Because of this, the default OFT standard has a max token supply of `(2^64 - 1)/(10^6)`, or `18,446,744,073,709.551615`. 
This property is defined by [`sharedDecimals` and `localDecimals`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/oapp/contracts/oft/OFTCore.sol#L54-L57) for the token. 

In practice, this means that you can only transfer the amount that can be represented in the shared system. An, example for the default OFT standard that used local decimals of `18` and shared decimals of `6`, which means a conversion rate of `10^12`.

1. User specifices the amount to `send` that equals `1234567890123456789`
2. The OFT standard will first divide this amount by `10^12` to get the amount in the shared system, which equals `1234567`.
3. On the receiving chain it will be multiplied by `10^12` to get the amount in the local system, which equals `1234567000000000000`.

This process removes the last 12 digits from the original amount, effectively "cleaning" the amount from any "dust" that cannot be represented in a system with 6 decimal places.

```solidity
amountToSend   = 1234567890123456789;
amountReceived = 1234567000000000000;
```

It's important to highlight that the dust removed is not lost, it's just cleaned from the input amount. 

> Look for custom fees added to the OFT standard, `_removeDust` should be called after determining the actual transfer amount. 

Bug examples: [1](https://github.com/windhustler/audits/blob/21bf9a1/solo/PING-Security-Review.pdf)

## Useful resources

- [LayerZeroV2 developer docs](https://docs.layerzero.network/v2)
- [Decode LayerZero V2](https://senn.fun/decode-layerzero-v2)