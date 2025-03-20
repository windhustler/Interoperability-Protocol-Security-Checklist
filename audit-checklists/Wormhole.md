# Wormhole Security Checklist

## Overview

Wormhole is a cross-chain messaging protocol that provides several services:
- Message passing
- Fungible token bridging
- NFT bridging
- Native token transfers (NTT)
- Cross-chain governance

The protocol operates as a multisig bridge system with the following key components:
- Guardian nodes: A minimum of 13 guardian nodes must attest to messages (VAAs) for them to be valid
- Relayers: Trustless parties responsible for delivering VAAs to destination chains
- Wormhole-Core contract: Verifies VAAs on destination chains

## Core Security Assumptions

1. Bridge security relies on the guardian node committee
2. VAAs are broadcast publicly and can be verified on any chain with Wormhole-Core
3. Relayers are untrusted and may drop or skip VAAs
   - Consider running a custom relayer or subscribing to a known relayer operator
4. For protocols deployed on multiple chains, implement replay protection

## Service-Specific Security Considerations

Services provided by Wormhole and their security issues you should consider. 

1. Message Passing 
2. Token Bridge
3. Native Token Transfers
4. CCTP
5. Relayers


### Direct `publishMessage` Usage

Protocols can publish messages directly through the Wormhole-Core contract's `publishMessage` function:

```solidity
function publishMessage(
    uint32 nonce,
    bytes memory payload,
    uint8 consistencyLevel
) external payable returns (uint64 sequence);
```

Key parameters and considerations:

1. `nonce`:
   - Must be unique for each bridge message and caller
   - Protocols must implement proper nonce generation

2. `payload`:
   - Contains the data being sent to the destination chain
   - Must be properly encoded/decoded between chains
   - Use matching encoding/decoding methods (e.g., `abi.encode`/`abi.decode`)
   - Incorrect encoding can lead to undefined behavior

3. `consistencyLevel`:
   - Determines the security level and processing delay
   - Higher levels = better security but longer delays
   - Lower levels = faster processing but vulnerable to block reorgs

### Message Fee Handling

The `publishMessage()` function is payable and may charge fees. This is a critical consideration when implementing cross-chain operations, especially with token transfers.

#### Key Considerations

1. **Fee Management**
   - Never hardcode `msg.value` for the function call
   - Always fetch the current fee using `messageFee()` on the Wormhole-Core contract
   - Use the returned value as `msg.value` in the `publishMessage()` call
   - Hardcoded values may cause transaction failures if Wormhole updates fees

2. **Common Implementation Bug**
   ```solidity
   // Problematic Implementation
   ICircleIntegration(wormholeCircleBridge).transferTokensWithPayload(
       transferParameters, 
       0,  // No fee value passed
       abi.encode(msg.sender)
   );
   ```

   This implementation will fail when:
   - The destination chain has message fees enabled
   - The `msg.value` is not sufficient to cover the required fee
   - The transaction will revert due to insufficient funds

3. **Impact**
   - Token transfers will fail on chains with enabled message fees
   - Users may lose gas fees on failed transactions
   - Protocol functionality breaks on specific chains

4. **Correct Implementation**
   ```solidity
   uint256 messageFee = wormholeCore.messageFee();
   ICircleIntegration(wormholeCircleBridge).transferTokensWithPayload{value: messageFee}(
       transferParameters,
       messageFee,
       abi.encode(msg.sender)
   );
   ```

5. **Best Practices**
   - Always check and include message fees for cross-chain operations
   - Handle fee requirements for all supported chains
   - Consider implementing fee estimation for better UX
   - Test fee scenarios across all supported chains

Reference report: [1](https://0xmacro.com/library/audits/infinex-1.html#M-1), [2](https://iosiro.com/audits/infinex-accounts-smart-account-smart-contract-audit#IO-IFX-ACC-007), [3](https://certificate.quantstamp.com/full/hashflow-hashverse/1af3e150-d612-4b24-bc74-185624a863f8/index.html#findings-qs5)

### Relayer Authorization

When implementing cross-chain message reception, proper sender validation is crucial for security. The following security checks should be implemented:

1. **Relayer Authentication**
   - Verify that messages originate from the authorized Wormhole relayer
   - Prevent unauthorized contracts from sending messages
   - Use the `isRegisteredSender` modifier to validate source chain and address

2. **Implementation Example**
~
```solidity
function receiveWormholeMessages(
	bytes memory payload,
	bytes[] memory,
	bytes32 sourceAddress,
	uint16 sourceChain,
	bytes32
) public payable override 
isRegisteredSender(sourceChain, sourceAddress) 
{
 // Verify message sender is the authorized Wormhole relayer
	require(
	msg.sender == address(wormholeRelayer),
	"Only the Wormhole relayer can call this function"
	);

	emit MessageReceived(message);
}
```

3. **Security Considerations**
   - Always implement both the `isRegisteredSender` modifier and relayer check
   - Missing either check could allow unauthorized message processing
   - Consider implementing additional validation for message content
   - Log all received messages for audit purposes

### Chain ID vs Domain Mislabeling

A common issue occurs when dealing with Wormhole's chain identifiers. The code incorrectly labels chain IDs as "domains", which can lead to confusion and potential implementation errors.

**Problematic Implementation:**
```solidity
// Incorrectly named parameter as domain when it's actually a chain ID
function _bridgeUSDCWithWormhole(
    uint256 _amount, 
    bytes32 _destinationAddress, 
    uint16 _destinationDomain  // Mislabeled - should be _destinationChainId
) internal {
    _checkExceedsMinBridgeAmount(_amount);
    if (!_validateWormholeDestinationDomain(_destinationDomain)) revert Error.InvalidDestinationDomain();
    // ...
}
```

**Impact:**
- Confusion between chain IDs and domains
- Potential incorrect value assignments during initialization
- May require account upgrades to fix

**Correct Implementation:**
```solidity
function _bridgeUSDCWithWormhole(
    uint256 _amount, 
    bytes32 _destinationAddress, 
    uint16 _destinationChainId  // Correctly labeled as chain ID
) internal {
    _checkExceedsMinBridgeAmount(_amount);
    if (!_validateWormholeDestinationChainId(_destinationChainId)) revert Error.InvalidChainId();
    // ...
}
```

**Key Points:**
- Always use clear terminology: chain IDs for Wormhole chain identifiers
- Validate chain IDs against the correct mapping
- Consider renaming variables and functions to reflect correct terminology

Reference reports: [1](https://0xmacro.com/library/audits/infinex-1.html#L-1)

### Decimal Scaling Issues in Token Bridging

When implementing token bridging, it's crucial to handle different token decimals correctly. A common issue occurs when using a hardcoded maximum bridge amount without considering token-specific decimal places.

**Problematic Implementation:**
```solidity
function _computeScaledAmount(uint256 _amount, address _tokenAddress) internal returns (uint256) {
    uint256 scaledAmount = uint256(DecimalScaling.scaleTo(
        int256(_amount), 
        IERC20Metadata(_tokenAddress).decimals()
    ));

    // Incorrectly using USDC's max amount for all tokens
    if (scaledAmount > BridgeIntegrations._getBridgeMaxAmount()) 
        revert Error.BridgeMaxAmountExceeded();

    if (scaledAmount > IERC20(_tokenAddress).balanceOf(address(this))) 
        revert Error.InsufficientBalance();

    return scaledAmount;
}
```

**Impact:**
- Incorrect maximum bridge amount validation for non-USDC tokens
- Potential transaction failures for valid amounts
- Example: For 18-decimal tokens, USDC's max amount (1,000,000) would incorrectly limit to 0.000001 tokens

**Correct Implementation:**
```solidity
function _computeScaledAmount(uint256 _amount, address _tokenAddress) internal returns (uint256) {
    uint256 scaledAmount = uint256(DecimalScaling.scaleTo(
        int256(_amount), 
        IERC20Metadata(_tokenAddress).decimals()
    ));

    // Token-specific max amount validation
    if (_tokenAddress == Bridge._USDC()) {
        if (scaledAmount > BridgeIntegrations._getBridgeMaxAmount()) 
            revert Error.BridgeMaxAmountExceeded();
    } else {
        // Implement token-specific max amount logic
        if (scaledAmount > getTokenSpecificMaxAmount(_tokenAddress)) 
            revert Error.BridgeMaxAmountExceeded();
    }

    if (scaledAmount > IERC20(_tokenAddress).balanceOf(address(this))) 
        revert Error.InsufficientBalance();

    return scaledAmount;
}
```

**Key Points:**
- Always consider token-specific decimal places when validating amounts
- Implement separate max amount checks for different tokens
- Test bridging with tokens of different decimal places
- Consider moving max amount validation outside of scaling functions

Reference reports: [1](https://0xmacro.com/library/audits/infinex-1.html#L-2)


