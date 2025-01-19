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
The OFT standard was created to allow transferring tokens across different blockchain VMs. While `EVM` chains support `uint256` for token balances, many non-EVM chains use `uint64`. Because of this, the default OFT standard has a max token supply of `(2^64 - 1)/(10^6)`, or `18,446,744,073,709.551615`. 
This property is defined by [`sharedDecimals` and `localDecimals`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/oapp/contracts/oft/OFTCore.sol#L54-L57) for the token. 

In practice, this means that you can only transfer the amount that can be represented in the shared system. The default OFT standard uses local decimals equal to `18` and shared decimals of `6`, which means a conversion rate of `10^12`.

Take the example: 

1. User specifies the amount to `send` that equals `1234567890123456789`. 
2. The OFT standard will first divide this amount by `10^12` to get the amount in the shared system, which equals `1234567`.
3. On the receiving chain it will be multiplied by `10^12` to get the amount in the local system, which equals `1234567000000000000`.

This process removes the last `12` digits from the original amount, effectively "cleaning" the amount from any "dust" that cannot be represented in a system with `6` decimal places.

```solidity
amountToSend   = 1234567890123456789;
amountReceived = 1234567000000000000;
```

It's important to highlight that the dust removed is not lost, it's just cleaned from the input amount. 

> Look for custom fees added to the OFT standard, `_removeDust` should be called after determining the actual transfer amount. 

Bug examples: [1](https://github.com/windhustler/audits/blob/21bf9a1/solo/PING-Security-Review.pdf)

## LayerZero Read
[LayerZero Read](https://docs.layerzero.network/v2/developers/evm/lzread/overview) enables requesting data from a remote chain without executing a transaction there. It works with a request-response pattern, where you request a certain data from the remote chain and the DVNs will respond by directly reading the data from the node on the remote chain. 

### Reverts while reading data blocks subsequent messages
The request can contain multiple read commands and compute operations. Here is an example of how to specify those commands with [EVMCallRequestV1 and EVMCallComputeV1](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/oapp/contracts/oapp/examples/LzReadCounter.sol#L73-L105) structs, and corresponding functions that get called by the DVNs on the remote chain -- [`readCount`, `lzMap` and `lzReduce`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/oapp/contracts/oapp/examples/LzReadCounter.sol#L108-L133). 

If any of these functions calls revert(`readCount`, `lzMap` and `lzReduce`), the DVNs are not able to create a response and verify the message. Let's look at what happens if the message with certain nonce can't be verified. An example covers sending a message on Ethereum to read the data from Polygon. 

- Sending a message on Ethereum to read the data on Polygon assigns a [monotonically increasing nonce](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L116-L127) to the message per `sender`, `dstEid` and `receiver`.  
```solidity
function _send(
    address _sender,
    MessagingParams calldata _params
) internal returns (MessagingReceipt memory, address) {
    // get the correct outbound nonce
>>>        uint64 latestNonce = _outbound(_sender, _params.dstEid, _params.receiver);

    // construct the packet with a GUID
    Packet memory packet = Packet({
        nonce: latestNonce,
        srcEid: eid,
        sender: _sender,
        dstEid: _params.dstEid,
        receiver: _params.receiver,
        guid: GUID.generate(latestNonce, eid, _sender, _params.dstEid, _params.receiver),
        message: _params.message
    });

/// @dev increase and return the next outbound nonce
function _outbound(address _sender, uint32 _dstEid, bytes32 _receiver) internal returns (uint64 nonce) {
    unchecked {
        nonce = ++outboundNonce[_sender][_dstEid][_receiver];
    }
}
````
- In case of lzRead, `dstEid` is the `channelId` equal to `4294967295`. Read paths information can be found in the [Read Paths](https://docs.layerzero.network/v2/developers/evm/lzread/read-paths) section in the LayerZero docs.
- This `Packet` gets processed in the [`ReadLib1002`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/messagelib/contracts/uln/readlib/ReadLib1002.sol#L97) contract. 
- Application configured DVNs needs to [verify the message and commit verification](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/messagelib/contracts/uln/readlib/ReadLib1002.sol#L138-L167) needs to be called.
- If the DVNs can't generate a response, they can't verify that specific message.
```solidity
// ============================ External ===================================
/// @dev The verification will be done in the same chain where the packet is sent.
/// @dev dont need to check endpoint verifiable here to save gas, as it will reverts if not verifiable.
/// @param _packetHeader - the srcEid should be the localEid and the dstEid should be the channel id.
///        The original packet header in PacketSent event should be processed to flip the srcEid and dstEid.
function commitVerification(bytes calldata _packetHeader, bytes32 _cmdHash, bytes32 _payloadHash) external {
    // assert packet header is of right size 81
    if (_packetHeader.length != 81) revert LZ_RL_InvalidPacketHeader();
    // assert packet header version is the same
    if (_packetHeader.version() != PacketV1Codec.PACKET_VERSION) revert LZ_RL_InvalidPacketVersion();
    // assert the packet is for this endpoint
    if (_packetHeader.dstEid() != localEid) revert LZ_RL_InvalidEid();

    // cache these values to save gas
    address receiver = _packetHeader.receiverB20();
    uint32 srcEid = _packetHeader.srcEid(); // channel id
    uint64 nonce = _packetHeader.nonce();

    // reorg protection. to allow reverification, the cmdHash cant be removed
    if (cmdHashLookup[receiver][srcEid][nonce] != _cmdHash) revert LZ_RL_InvalidCmdHash();

    ReadLibConfig memory config = getReadLibConfig(receiver, srcEid);
    _verifyAndReclaimStorage(config, keccak256(_packetHeader), _cmdHash, _payloadHash);

    // endpoint will revert if nonce <= lazyInboundNonce
    Origin memory origin = Origin(srcEid, _packetHeader.sender(), nonce);
>>>    ILayerZeroEndpointV2(endpoint).verify(origin, receiver, _payloadHash);
}
```
- srcEid is the `channelId`, while nonce is the `latestNonce` assigned while sending the message.
- [`EndpointV2::verify`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L151) function updates the `inboundPayloadHash` mapping with the `latestNonce`. 
```solidity
/// @dev MESSAGING STEP 2 - on the destination chain
/// @dev configured receive library verifies a message
/// @param _origin a struct holding the srcEid, nonce, and sender of the message
/// @param _receiver the receiver of the message
/// @param _payloadHash the payload hash of the message
function verify(Origin calldata _origin, address _receiver, bytes32 _payloadHash) external {
    if (!isValidReceiveLibrary(_receiver, _origin.srcEid, msg.sender)) revert Errors.LZ_InvalidReceiveLibrary();

    uint64 lazyNonce = lazyInboundNonce[_receiver][_origin.srcEid][_origin.sender];
    if (!_initializable(_origin, _receiver, lazyNonce)) revert Errors.LZ_PathNotInitializable();
    if (!_verifiable(_origin, _receiver, lazyNonce)) revert Errors.LZ_PathNotVerifiable();

    // insert the message into the message channel
>>    _inbound(_receiver, _origin.srcEid, _origin.sender, _origin.nonce, _payloadHash);
    emit PacketVerified(_origin, _receiver, _payloadHash);
}

/// @dev inbound won't update the nonce eagerly to allow unordered verification
/// @dev instead, it will update the nonce lazily when the message is received
/// @dev messages can only be cleared in order to preserve censorship-resistance
function _inbound(
    address _receiver,
    uint32 _srcEid,
    bytes32 _sender,
    uint64 _nonce,
    bytes32 _payloadHash
) internal {
    if (_payloadHash == EMPTY_PAYLOAD_HASH) revert Errors.LZ_InvalidPayloadHash();
>>>    inboundPayloadHash[_receiver][_srcEid][_sender][_nonce] = _payloadHash;
}
```

- During invocation of `lzReceive`, [`_clearPayload`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/MessagingChannel.sol#L138) internal function gets called. 
```solidity
function _clearPayload(
       address _receiver,
       uint32 _srcEid,
       bytes32 _sender,
       uint64 _nonce,
       bytes memory _payload
   ) internal returns (bytes32 actualHash) {
       uint64 currentNonce = lazyInboundNonce[_receiver][_srcEid][_sender];
       if (_nonce > currentNonce) {
           unchecked {
               // try to lazily update the inboundNonce till the _nonce
               for (uint64 i = currentNonce + 1; i <= _nonce; ++i) {
>>>                   if (!_hasPayloadHash(_receiver, _srcEid, _sender, i)) revert Errors.LZ_InvalidNonce(i);
               }
               lazyInboundNonce[_receiver][_srcEid][_sender] = _nonce;
           }
       }

function _hasPayloadHash(
    address _receiver,
    uint32 _srcEid,
    bytes32 _sender,
    uint64 _nonce
) internal view returns (bool) {
    return inboundPayloadHash[_receiver][_srcEid][_sender][_nonce] != EMPTY_PAYLOAD_HASH;
}       
```

- The key part of this function is updating the `lazyInboundNonce` to the latest nonce.
- In case a message with a certain nonce has been sent, but couldn't been verified `_hasPayloadHash` for that nonce will return `false` and the `lzReceive` function will revert. 

In summary, if a message with a certain nonce has been sent, but couldn't been verified, the `lzReceive` function will revert until that nonce is verified. 

> The `OAppRead` or its delegate can call [`EndpointV2::skip`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/MessagingChannel.sol#L76-L88) function to increment the `lazyInboundNonce` without having had that corresponding message be verified. This can be used to skip the verification but it's paramount to ensure that the message can be verified in the first place but not having reverts during reading data. 

## Useful resources

- [LayerZeroV2 developer docs](https://docs.layerzero.network/v2)
- [Decode LayerZero V2](https://senn.fun/decode-layerzero-v2)
- [LayerZero V2 Deep Dive Video](https://www.youtube.com/watch?v=ercyc98S7No)


