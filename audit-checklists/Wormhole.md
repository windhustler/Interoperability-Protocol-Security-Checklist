# Wormhole Security Checklist

## Overview

Wormhole is a cross-chain messaging protocol that provides several services:
- Message passing
- Fungible token bridging
- NFT bridging
- Native token transfers (NTT)
- Cross-chain governance

The protocol operates as a multisig bridge system with the following key components:
- Guardian nodes: A minimum of 13 guardian nodes must attest to messages (VAA's) for them to be valid
- Relayers: Trustless parties responsible for delivering VAAs to destination chains
- Wormhole-Core contract: Verifies VAAs on destination chains

## Core Security Assumptions

1. Bridge security relies on the guardian node committee
2. VAAs are broadcast publicly and can be verified on any chain with Wormhole-Core
3. Relayers are untrusted and may drop or skip VAAs
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
   - Missing nonce validation can lead to replay attacks

2. `payload`:
   - Contains the data being sent to the destination chain
   - Must be properly encoded/decoded between chains
   - Use matching encoding/decoding methods (e.g., `abi.encode`/`abi.decode`)
   - Incorrect encoding can lead to undefined behavior or data corruption

3. `consistencyLevel`:
   - Determines the security level and processing delay
   - Higher levels = better security but longer delays
   - Lower levels = faster processing but vulnerable to block reorgs
   - Must be set appropriately based on chain finality guarantees

Reference bugs : 
- [P1-H-01] (https://2301516674-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FNGiifMbrcug9ogAfisQY%2Fuploads%2FPxfV4xPmOCVKng8LcjiH%2FMayan_MCTP_Sec3.pdf?alt=media&token=62699afe-9e67-44fb-96fe-b593041365f4) - Example of incorrect decode on destination chain


### Message Fee Handling

The `publishMessage()` function is payable and may charge fees. This is a critical consideration when implementing cross-chain operations.

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

Reference bugs:
- [Infinex M-1](https://0xmacro.com/library/audits/infinex-1.html#M-1): Missing message fee handling in cross-chain operations
- [Infinex IO-IFX-ACC-007](https://iosiro.com/audits/infinex-accounts-smart-account-smart-contract-audit#IO-IFX-ACC-007): Incorrect fee handling in token bridging
- [Hashflow QS5](https://certificate.quantstamp.com/full/hashflow-hashverse/1af3e150-d612-4b24-bc74-185624a863f8/index.html#findings-qs5): Message fee validation bypass
- [P1-I-02] (https://2301516674-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FNGiifMbrcug9ogAfisQY%2Fuploads%2FPxfV4xPmOCVKng8LcjiH%2FMayan_MCTP_Sec3.pdf?alt=media&token=62699afe-9e67-44fb-96fe-b593041365f4) : Check if "msg.value" equals to "wormhole.messageFee()"
- [OS-SNM-SUG-02](https://2239978398-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FzjiJ8UzMPEfugKsLon59%2Fuploads%2FYr6wLCHl8r6uS6eHAYnb%2Fsynonym_audit_final.pdf?alt=media&token=3ad993f9-da68-496d-be06-d7eed5d305de) :  Wormhole Fee Calculation Discrepancy

### Relayer Authorization

When implementing cross-chain message reception, proper sender validation is crucial for security. The following security checks should be implemented:

1. **Relayer Authentication**
   - Verify that messages originate from the authorized Wormhole relayer
   - Prevent unauthorized contracts from sending messages
   - Use the `isRegisteredSender` modifier to validate source chain and address

2. **Implementation Example**
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

Reference bugs:
- [Infinex L-1](https://0xmacro.com/library/audits/infinex-1.html#L-1): Chain ID validation bypass due to incorrect domain labeling
- [Hashflow QS3](https://certificate.quantstamp.com/full/hashflow-hashverse/1af3e150-d612-4b24-bc74-185624a863f8/index.html#findings-qs3): Domain mislabeling leading to incorrect chain validation
- [P1-I-05](https://2301516674-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FNGiifMbrcug9ogAfisQY%2Fuploads%2Fz6Gq4wJprv7LomYsQ1LY%2FMayan_Swift_Sec3.pdf?alt=media&token=4a1b7f69-a626-4e34-9db2-4e931c3bc11f) : Unintended wormhole messages emitted 

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

Reference bugs:
- [Infinex L-2](https://0xmacro.com/library/audits/infinex-1.html#L-2): Decimal scaling issues in token bridging
- [Hashflow QS4](https://certificate.quantstamp.com/full/hashflow-hashverse/1af3e150-d612-4b24-bc74-185624a863f8/index.html#findings-qs4): Incorrect decimal handling in cross-chain token transfers
- [Magpie M-1](https://cdn.prod.website-files.com/621a140a057f392845dfaef3/65c9d04d3bc92bd2dfb6dc87_SmartContract_Audit_MagpieProtocol(v4)_v1.1.pdf)

### Normalize and Denormalize makes dust amount stuck in contracts

When implementing token bridging with Wormhole, the token bridge performs normalize and denormalize operations on amounts to remove dust. This can lead to tokens getting stuck in the contract if not handled properly.

**Problematic Implementation:**
```solidity
function swap(uint256 amountIn) external {
    // Transfer full amount from user
    IERC20(token).transferFrom(msg.sender, address(this), amountIn);
    
    // Token bridge normalizes and denormalizes the amount
    // This can result in dust amounts being left in the contract
    tokenBridge.transferTokens(
        token,
        amountIn,
        destinationChain,
        destinationAddress
    );
}
```

**Impact:**
- Dust amounts from token transfers get stuck in the contract
- Users lose small amounts of tokens that cannot be retrieved
- For ETH transfers, dust amounts from wrapping/unwrapping become stuck
- No way to recover these stuck amounts without additional functionality

**Proof of Concept:**
1. Call the swap function with tokens that have more than 8 decimals
2. Use an amountIn value with no trailing zeroes
3. Only the denormalize(normalize(amountIn)) value is transferred
4. The remaining dust amount is left in the contract balance

**Correct Implementation:**
```solidity
function swap(uint256 amountIn) external {
    // Calculate the normalized amount first
    uint256 normalizedAmount = normalize(amountIn);
    uint256 denormalizedAmount = denormalize(normalizedAmount);
    
    // Only transfer the exact amount that will be bridged
    IERC20(token).transferFrom(msg.sender, address(this), denormalizedAmount);
    
    tokenBridge.transferTokens(
        token,
        denormalizedAmount,
        destinationChain,
        destinationAddress
    );
}
```

**Remediation Options:**
1. Take only the required amount from users (without dust amounts)
2. Implement a guardian function to collect dust amounts
3. Add a dust amount threshold check before transfers
4. Consider implementing a dust collection mechanism for users

Reference bugs:
- [OS-MYN-ADV-05](https://2301516674-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FNGiifMbrcug9ogAfisQY%2Fuploads%2F9YSweDuRP1P28bMmjy63%2FMayan_Audit_OtterSec.pdf?alt=media&token=ffefae4d-367d-401f-bd16-2d799dd3a403)
- [OS-OPF-ADV-00](https://2239978398-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FzjiJ8UzMPEfugKsLon59%2Fuploads%2F19FGz84wBMigB9EBngTu%2Foptimistic_finality_audit_final.pdf?alt=media&token=c3a631b5-0cc0-4104-a781-0691e2491973) : Unaccounted Amount Trimming

### Double Normalization/Denormalization Issues

When implementing token bridging with Wormhole, multiple normalization/denormalization operations can lead to incorrect amount calculations and potential fund loss. This issue manifests in two forms:

1. **Double Normalization**
When amounts are normalized multiple times, leading to incorrect calculations:

**Problematic Implementation:**
```solidity
function getCurrentAccrualIndices(address assetAddress) public view returns(AccrualIndices memory) {
    uint256 lastActivityBlockTimestamp = getLastActivityBlockTimestamp(assetAddress);
    uint256 secondsElapsed = block.timestamp - lastActivityBlockTimestamp;
    uint256 deposited = getTotalAssetsDeposited(assetAddress); // Already normalized in storage
    AccrualIndices memory accrualIndices = getInterestAccrualIndices(assetAddress);
    
    if ((secondsElapsed != 0) && (deposited != 0)) {
        uint256 borrowed = getTotalAssetsBorrowed(assetAddress); // Already normalized in storage
        // Double normalization occurs here
        uint256 normalizedDeposited = normalizeAmount(deposited, accrualIndices.deposited, Round.DOWN);
        uint256 normalizedBorrowed = normalizeAmount(borrowed, accrualIndices.borrowed, Round.DOWN);
        // ...
    }
}
```

2. **Double Denormalization**
When amounts are denormalized multiple times, leading to incorrect calculations:

**Problematic Implementation:**
```solidity
function calculateNotionals(
    address asset,
    VaultAmount memory vaultAmount
) public view returns (VaultAmount memory) {
    AssetInfo memory assetInfo = getAssetInfo(asset);
    // Double denormalization occurs here when vaultAmount is already denormalized
    VaultAmount memory denormalized = denormalizeVault(asset, vaultAmount);
    (uint256 priceCollateral, uint256 priceDebt) = getPrices(asset);
    uint256 expVal = 10 ** (hub.getMaxDecimals() - assetInfo.decimals);
    
    return VaultAmount(
        denormalized.deposited * priceCollateral * expVal,
        denormalized.borrowed * priceDebt * expVal
    );
}
```

**Impact:**
- Incorrect interest calculations and accrual indices
- Wrong withdrawal limits and borrowing limits
- Inaccurate liquidation amounts
- Protocol instability due to faulty values
- Potential fund loss due to incorrect calculations

**Proof of Concept:**
1. For Double Normalization:
   - Fetch normalized amount from storage
   - Apply normalization again
   - Result is incorrectly normalized amount
   - Affects all calculations using this amount

2. For Double Denormalization:
   - Pass denormalized amount to function
   - Function denormalizes again
   - Result is incorrectly denormalized amount
   - Affects all calculations using this amount

**Correct Implementation:**
```solidity
// For Double Normalization
function getCurrentAccrualIndices(address assetAddress) public view returns(AccrualIndices memory) {
    uint256 deposited = getTotalAssetsDeposited(assetAddress); // Already normalized
    AccrualIndices memory accrualIndices = getInterestAccrualIndices(assetAddress);
    
    if ((secondsElapsed != 0) && (deposited != 0)) {
        uint256 borrowed = getTotalAssetsBorrowed(assetAddress); // Already normalized
        // Use values directly without additional normalization
        // ... rest of the function
    }
}

// For Double Denormalization
function calculateNotionals(
    address asset,
    VaultAmount memory vaultAmount
) public view returns (VaultAmount memory) {
    AssetInfo memory assetInfo = getAssetInfo(asset);
    // Check if amount needs denormalization
    VaultAmount memory processedAmount = isNormalized(vaultAmount) ? 
        denormalizeVault(asset, vaultAmount) : vaultAmount;
    
    (uint256 priceCollateral, uint256 priceDebt) = getPrices(asset);
    uint256 expVal = 10 ** (hub.getMaxDecimals() - assetInfo.decimals);
    
    return VaultAmount(
        processedAmount.deposited * priceCollateral * expVal,
        processedAmount.borrowed * priceDebt * expVal
    );
}
```

**Remediation Options:**
1. Track normalization state of amounts throughout the protocol
2. Add flags to indicate if amounts are normalized/denormalized
3. Implement validation checks before normalization/denormalization
4. Use consistent normalization/denormalization patterns across the protocol
5. Add clear documentation about amount states

Reference bugs:
- [OS-SNM-ADV-04](https://2239978398-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FzjiJ8UzMPEfugKsLon59%2Fuploads%2FYr6wLCHl8r6uS6eHAYnb%2Fsynonym_audit_final.pdf?alt=media&token=3ad993f9-da68-496d-be06-d7eed5d305de): Double Normalization in interest calculations
- [OS-SNM-ADV-05](https://2239978398-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FzjiJ8UzMPEfugKsLon59%2Fuploads%2FYr6wLCHl8r6uS6eHAYnb%2Fsynonym_audit_final.pdf?alt=media&token=3ad993f9-da68-496d-be06-d7eed5d305de): Double Denormalization in notional calculations



