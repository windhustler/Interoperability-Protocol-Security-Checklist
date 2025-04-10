# LayerZeroV2 Security Checklist

## EndpointV2

### `lzReceive` function can revert with an "out of gas" (OOG) error 
Every time a new message is [verified](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L159) inside the `EndpointV2` contract a mapping for the `receiver`, `srcEid` and `nonce` combination is set to the corresponding `payloadHash` for later execution with `lzReceive`. Every new message verified for the pathway has a sequential, monotonically increasing nonce.

During the `lzReceive` function execution the [`_clearPayload`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b9e5967643853476445ffe0e777360b906/packages/layerzero-v2/evm/protocol/contracts/MessagingChannel.sol#L134-L142) function tries to lazily update the `lazyInboundNonce` to the nonce of the message being received. It loops through all the nonces to check if they have been verified and updates the `lazyInboundNonce` to the nonce of the message being received.

>In extreme cases, if there are a lot of messages verified, the looping can cause an "out of gas" (OOG) error. 
However, there's a straightforward solution -- instead of processing a message with a high nonce, you can first process messages with lower nonces. This allows the `lazyInboundNonce` to be updated in smaller steps.

## `lzCompose` implementation should always enforce `from` address and the `msg.sender`
If we look at the default `OFT` standard implementation and the usage of `lzCompose` within `lzReceive`:
```solidity
## OFTCore.sol

function _lzReceive(
    Origin calldata _origin,
    bytes32 _guid,
    bytes calldata _message,
    address /*_executor*/, // @dev unused in the default implementation.
    bytes calldata /*_extraData*/ // @dev unused in the default implementation.
) internal virtual override {
    // @dev The src sending chain doesnt know the address length on this chain (potentially non-evm)
    // Thus everything is bytes32() encoded in flight.
    address toAddress = _message.sendTo().bytes32ToAddress();
    // @dev Credit the amountLD to the recipient and return the ACTUAL amount the recipient received in local decimals
>>>        uint256 amountReceivedLD = _credit(toAddress, _toLD(_message.amountSD()), _origin.srcEid);

    if (_message.isComposed()) {
        // @dev Proprietary composeMsg format for the OFT.
        bytes memory composeMsg = OFTComposeMsgCodec.encode(
            _origin.nonce,
            _origin.srcEid,
            amountReceivedLD,
            _message.composeMsg()
        );

        // @dev Stores the lzCompose payload that will be executed in a separate tx.
        // Standardizes functionality for executing arbitrary contract invocation on some non-evm chains.
        // @dev The off-chain executor will listen and process the msg based on the src-chain-callers compose options passed.
        // @dev The index is used when a OApp needs to compose multiple msgs on lzReceive.
        // For default OFT implementation there is only 1 compose msg per lzReceive, thus its always 0.
>>>            endpoint.sendCompose(toAddress, _guid, 0 /* the index of the composed message*/, composeMsg);
    }

    emit OFTReceived(_guid, _origin.srcEid, toAddress, amountReceivedLD);
}
```

The key points here are:

- Tokens are first credited to the `toAddress` contract which should implement the `lzCompose` function.
- The `lzCompose` function gets executed in a separate transaction.
- The tokens remain in the `toAddress` contract until `lzCompose` is executed

Let's observe how the `sendCompose` and `lzCompose` inside the LayerZero contracts works:

```solidity
## MessagingComposer.sol

function sendCompose(address _to, bytes32 _guid, uint16 _index, bytes calldata _message) external {
    // must have not been sent before
    if (composeQueue[msg.sender][_to][_guid][_index] != NO_MESSAGE_HASH) revert Errors.LZ_ComposeExists();
>>>    composeQueue[msg.sender][_to][_guid][_index] = keccak256(_message);
    emit ComposeSent(msg.sender, _to, _guid, _index, _message);
}

function lzCompose(
    address _from,
    address _to,
    bytes32 _guid,
    uint16 _index,
    bytes calldata _message,
    bytes calldata _extraData
) external payable {
    // assert the validity
    bytes32 expectedHash = composeQueue[_from][_to][_guid][_index];
    bytes32 actualHash = keccak256(_message);
    if (expectedHash != actualHash) revert Errors.LZ_ComposeNotFound(expectedHash, actualHash);

    // marks the message as received to prevent reentrancy
    // cannot just delete the value, otherwise the message can be sent again and could result in some undefined behaviour
    // even though the sender(composing Oapp) is implicitly fully trusted by the composer.
    // eg. sender may not even realize it has such a bug
>>>    composeQueue[_from][_to][_guid][_index] = RECEIVED_MESSAGE_HASH;
    ILayerZeroComposer(_to).lzCompose{ value: msg.value }(_from, _guid, _message, msg.sender, _extraData);
    emit ComposeDelivered(_from, _to, _guid, _index);
}
```

When `sendCompose` is called from within `lzReceive`, the `msg.sender` (and therefore the `from` address stored in `composeQueue`) will be the OFT token contract. This is important because this same `from` address will be passed to `lzCompose` when executing the composed message.

> When implementing `lzCompose`, you must validate:
> 1. The `from` parameter matches your expected OFT token contract - this is the original sender that queued the composed message
> 2. The `msg.sender` is the EndpointV2 contract - only the official endpoint should be able to trigger composed message execution
>
> Failing to validate either of these could allow unauthorized contracts to execute malicious composed messages.

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

### Don't rely on the gas limit and `msg.value`
All the metadata passed as options to the `Endpoint::send` function is simply an off-chain agreement with the Executor. The `lzReceive` function can be executed by anyone with a different `msg.value` and gas limit compared to what was specified on the sending side. 

There are however various ways to enforce certain properties on the receiving side:
- Whitelisting address that can execute [`lzReceive`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L181), in the receiving app, e.g. only allowing the LayerZero executor or whitelisted addresses to execute it.
- Encoding the `msg.value` into the message payload and reverting if the value is different from what was specified on the sending side.
- Enforcing certain gas limit in the `lzReceive` function, e.g. [`SafeCallMinGas.sol`](https://github.com/liquity/V2-gov/blob/22bc82f/src/utils/SafeCallMinGas.sol) contract. 

Bug examples: [1](https://solodit.cyfrin.io/issues/bridgedgovernorlzreceive-can-be-executed-with-different-msgvalue-than-intended-cantina-none-drips-pdf)

### Execution Ordering
The default OApp implementation of `lzReceive` is un-ordered execution. This means if nonce 4,5,6 are verified, the Executor will try to execute the message with nonce 4 first, but if it fails (due to some gas or user logic related issue), it will try to execute the message with nonce 5 and so on.

The process the off-chain executor uses if you want to enforce ordered execution:
1. It checks if the [ordered execution option](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/oapp/contracts/oapp/libs/OptionsBuilder.sol#L107-L111) has been set.
2. If this is true then it queries the [nextNonce](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/oapp/contracts/oapp/OAppReceiver.sol#L78) function in the receiver contract.
3. Let's assume nextNonce returns nonce 4. It tries to execute nonce 4 and if this transaction fails for any reason, it will block all subsequent transactions with higher nonces from being executed until nonce 4 is resolved.

> If you want to enforce ordered execution, you need to ensure that the `nextNonce` function is implemented correctly and that it returns the correct nonce. Make sure to never have a reverting transaction due to the blocking nature of the system. 

Bug examples: [1](https://solodit.cyfrin.io/issues/non-executable-messages-in-bridgedgovernor-can-result-in-an-unrecoverable-state-cantina-none-drips-pdf)

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

### Overriding shared decimals
The `OFTCore.sol` contract uses a default `sharedDecimals` value of `6`. When overriding this value, be aware of a critical limitation, the `_toSD` function casts amounts to `uint64` when converting from local to shared decimals.

```solidity
    function _buildMsgAndOptions(
        SendParam calldata _sendParam,
        uint256 _amountLD
    ) internal view virtual returns (bytes memory message, bytes memory options) {
        bool hasCompose;
        // @dev This generated message has the msg.sender encoded into the payload so the remote knows who the caller is.
        (message, hasCompose) = OFTMsgCodec.encode(
            _sendParam.to,
 >>>           _toSD(_amountLD),
            // @dev Must be include a non empty bytes if you want to compose, EVEN if you dont need it on the remote.
            // EVEN if you dont require an arbitrary payload to be sent... eg. '0x01'
            _sendParam.composeMsg
        );

    function _toSD(uint256 _amountLD) internal view virtual returns (uint64 amountSD) {
        return uint64(_amountLD / decimalConversionRate);
    }
```

This becomes important when `localDecimals` and `sharedDecimals` are both set to 18. In this case:
- The `decimalConversionRate` becomes 1 (no decimal adjustment)
- Any amount larger than `uint64.max` will silently be truncated to `uint64`

> This truncation can lead to unexpected behavior where users might think they're transferring a larger amount, but the actual transfer will be cast into `uint64`, resulting in a loss of value. 

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

### lzRead can be used to read data from the same chain
As opposed to standard LayerZero messages where you can only send data to a different chain, `lzRead` allows to read data from the same chain. 

Here is an example of supported chains for Ethereum:
![Ethereum Read Paths](/resources/LayerZero-lzRead-Ethereum.png)

The targetEid is specified inside the `EVMCallRequestV1` and `EVMCallComputeV1` structs.
```solidity
struct EVMCallRequestV1 {
    uint16 appRequestLabel; // Label identifying the application or type of request (can be use in lzCompute)
>>>    uint32 targetEid; // Target endpoint ID (representing a target blockchain)
    bool isBlockNum; // True if the request = block number, false if timestamp
    uint64 blockNumOrTimestamp; // Block number or timestamp to use in the request
    uint16 confirmations; // Number of block confirmations on top of the requested block number or timestamp before the view function can be called
    address to; // Address of the target contract on the target chain
    bytes callData; // Calldata for the contract call
}

struct EVMCallComputeV1 {
    uint8 computeSetting; // Compute setting (0 = map only, 1 = reduce only, 2 = map reduce)
>>>    uint32 targetEid; // Target endpoint ID (representing a target blockchain)
    bool isBlockNum; // True if the request = block number, false if timestamp
    uint64 blockNumOrTimestamp; // Block number or timestamp to use in the request
    uint16 confirmations; // Number of block confirmations on top of the requested block number or timestamp before the view function can be called
    address to; // Address of the target contract on the target chain
}
```

> Make sure to check the `targetEid` for the `lzRead` request and assess if you need to read data from the same chain, or any other for that matter. As highlited in the [Reverts while reading data blocks subsequent messages](#reverts-while-reading-data-blocks-subsequent-messages) section, it's paramount that the `lzRead` request doesn't revert.

## LayerZero immutability

How immutable is LayerZero? 

Based on the [LayerZeroV2 docs](https://docs.layerzero.network/v2/developers/evm/overview):
> LayerZero is an immutable, censorship-resistant, and permissionless smart contract protocol that enables anyone on a blockchain to send, verify, and execute messages on a supported destination network.

Is this true? Continue reading if you want to learn why you should always configure your OApp. 

Let's examine the critical dependencies in the `EndpointV2` contract, which is the core contract of the system. Two key external dependencies are:

1. **Message Sending**: The [send library lookup](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L75) during message transmission
   ```solidity
   address _sendLibrary = getSendLibrary(_sender, _params.dstEid);
   ```
2. **Message Verification**: The [receive library validation](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L152) on the destination chain
   ```solidity
   if (!isValidReceiveLibrary(_receiver, _origin.srcEid, msg.sender)) revert Errors.LZ_InvalidReceiveLibrary();
   ```

The configuration of send and receive libraries is managed through the `MessageLibManager` contract, which `EndpointV2` extends.

> Only the LayerZero time can register libraries that can be used to send or receive messages.

#### Key Privileges of LayerZero Team
1. **Library Registration**: Only LayerZero can register new send/receive libraries via [`MessageLibManager.registerLibrary()`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/MessageLibManager.sol#L140)
2. **Default Library Control**: LayerZero can change default send/receive libraries via:
   - [`setDefaultSendLibrary()`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/MessageLibManager.sol#L157)
   - [`setDefaultReceiveLibrary()`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/MessageLibManager.sol#L171)


While only LayerZero can register new libraries, each protocol can select and configure their preferred send and receive libraries from the registered options.

What are the options? Let's check the [EndpointV2 on Ethereum](https://etherscan.io/address/0x1a44076050125825900e736c501f859c50fE728c) contract and call the `getRegisteredLibraries` variable. Here is what we get:

- [BlockLibrary](https://etherscan.io/address/0x1ccBf0db9C192d969de57E25B3fF09A25bb1D862) - dummy library that completely disables sending and receiving messages. 
- [SendUln302](https://etherscan.io/address/0xbb2ea70c9e858123480642cf96acbcce1372dce1)
- [ReceiveUln302](https://etherscan.io/address/0xc02ab410f0734efa3f14628780e6e695156024c2)
- [ReadLibrary1002](https://etherscan.io/address/0x74f55bc2a79a27a0bf1d1a35db5d0fc36b9fdb9d)

Currently, the default libraries are the only available options and are required for cross-chain communication. Protocols that don't explicitly configure their libraries will automatically use these defaults.

There are two security considerations here. The attack threat is LayerZero acting maliciously.

1. **Protocol hasn't configured a send/receive library**
    - Relies on system defaults
    - LayerZero can freely change these defaults
    - Risk of protocol functionality being bricked

2. **Protocol has explicitly configured their send/receive library to use the current LayerZero defaults**
    - While this may seem similar to not configuring at all (since currently the default libraries are the only option), there is a crucial distinction.
    - When you explicitly configure your send/receive library, that configuration is locked in for your protocol.
    - Even if LayerZero later adds new libraries or changes the defaults, your protocol will continue using your configured libraries
    - This gives you control over your security posture - you won't be affected by changes to system defaults

## Configuration Tips

### Pausing bidirectional messages

When deploying an OApp on multiple chains (e.g., Ethereum and Arbitrum), bidirectional communication is established by [setting peers](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/oapp/contracts/oapp/OAppCore.sol#L56) on both OApps. However, you might want to allow messages in only one direction (e.g., only Ethereum -> Arbitrum).

This cannot be achieved through the `setPeer` configuration alone since `_getPeerOrRevert` is called during both [sending](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/oapp/contracts/oapp/OAppSender.sol#L88) and [receiving](https://github.com/LayerZero-Labs/LayerZero-v2/blob/943ce4a/packages/layerzero-v2/evm/oapp/contracts/oapp/OAppReceiver.sol#L106) messages.

However, there is a workaround to disable communication in one direction without modifying the peer configuration. By setting the [Executor configuration](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/MessageLibManager.sol#L307) parameter [`maxMessageSize`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/messagelib/contracts/SendLibBase.sol#L24) to 1 byte, the [`send`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/messagelib/contracts/SendLibBase.sol#L162) function will always revert, effectively blocking messages from being sent from that chain.

## Non-standard implementations

### Message receiver should implement `allowInitializePath` function

When a DVN verifies a message for a pathway the first time, it calls [`allowInitializePath`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L340) on the receiver to check if messages from that sender and source chain are allowed. The default OApp implementation checks if the sender is a trusted peer:

```solidity
## OAppReceiver.sol

function allowInitializePath(Origin calldata origin) public view virtual returns (bool) {
    return peers[origin.srcEid] == origin.sender;
}
```

> If you're not using the default OApp implementation, make sure to implement the `allowInitializePath` function in your receiving contract.


## Useful resources

- [LayerZeroV2 developer docs](https://docs.layerzero.network/v2)
- [Decode LayerZero V2](https://senn.fun/decode-layerzero-v2)
- [LayerZero V2 Deep Dive Video](https://www.youtube.com/watch?v=ercyc98S7No)
- [Comparison between Hyperlane, Wormhole and LayerZero](https://lindgren.xyz/posts/how-interopability-work/)
