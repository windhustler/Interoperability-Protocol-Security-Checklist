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

### Guardian Set Transition Issues

When implementing Wormhole guardian set updates, proper handling of the transition period is crucial. The protocol needs to maintain both old and new guardian sets active during the transition period to ensure continuous operation. This issue can affect any chain implementing Wormhole guardian sets.

**Example Implementation (from TON blockchain):**
```func
;; Example of incorrect strict equality check during guardian set transition
;; This issue can occur in any chain's implementation of Wormhole guardian sets
throw_unless(
    ERROR_INVALID_GUARDIAN_SET(
        current_guardian_set_index == vm_guardian_set_index(
            (expiration_time == 0) | (expiration_time > now())
        )
    )
);
```

**Impact:**
- Old guardian set cannot update messages during transition period
- Messages from old guardians are rejected
- Critical updates may be lost
- Protocol may experience data staleness during guardian set updates
- Potential disruption in cross-chain services
- Affects all chains implementing Wormhole guardian sets

**Exploit Scenario:**
1. New guardian set is announced
2. During the 24-hour propagation period:
   - New message arrives signed by old guardian set
   - Contract rejects the update due to strict equality check
   - Message information is lost
   - Protocol continues with stale data
   - Can affect any service using Wormhole guardian sets (e.g., price feeds, token bridges, governance)

**Correct Implementation:**
```solidity
// Allow both old and new guardian sets during transition
// This pattern should be implemented on all chains
require(
    currentGuardianSetIndex >= vmGuardianSetIndex,
    "Invalid guardian set"
);
```

**Key Considerations:**
1. Guardian set updates take up to 24 hours to propagate across all chains
2. Both old and new guardian sets must remain active during transition
3. Messages from either set should be accepted
4. Implement proper expiration time checks
5. Consider implementing a grace period for guardian set transitions
6. Pay attention to chain-specific message passing models

**Remediation Options:**
1. Change equality operator (==) to greater than or equal (>=)
2. Implement a transition period flag
3. Track active guardian sets during transition
4. Add validation for guardian set expiration times
5. Consider implementing a backup mechanism for critical updates
6. Ensure proper handling of chain-specific message passing during transitions

Reference bugs:
- [TOB-PYTHTON-1](https://github.com/pyth-network/audit-reports/blob/main/2024_11_26/pyth_ton_pull_oracle_audit_final.pdf): Guardian set transition period issues in Pyth TON Oracle integration

### Empty Guardian Set Validation

When implementing Wormhole guardian set updates, proper validation of the guardian set size is crucial. The protocol must ensure that guardian sets cannot be set to empty, as this would break all guardian verification functionality.

**Example Implementation (from TON blockchain):**
```func
;; Example of missing guardian length validation in TON
;; This issue can occur in any chain's implementation of Wormhole guardian sets
(int, int, int, cell, int) parse_encoded_upgrade(slice payload) impure {
    int module = payload~load_uint(256);
    throw_unless(ERROR_INVALID_MODULE, module == UPGRADE_MODULE);
    int action = payload~load_uint(8);
    throw_unless(ERROR_INVALID_GOVERNANCE_ACTION, action == 2);
    int chain = payload~load_uint(16);
    int new_guardian_set_index = payload~load_uint(32);
    throw_unless(ERROR_NEW_GUARDIAN_SET_INDEX_IS_INVALID, new_guardian_set_index == (current_guardian_set_index + 1));
    
    int guardian_length = payload~load_uint(8);  // Missing validation for non-zero length
    cell new_guardian_set_keys = new_dict();
    int key_count = 0;
    // ... rest of the function
}
```

**Impact:**
- Guardian set can be set to empty through upgrade messages
- All guardian verification functionality breaks
- No messages can be verified after empty guardian set
- Protocol becomes completely unusable
- Requires emergency fix or contract upgrade to recover
- Affects all chains implementing Wormhole guardian sets

**Exploit Scenario:**
1. Malicious or incorrect guardian set upgrade message arrives
2. Message contains zero guardian_length and no keys
3. Contract processes upgrade without validation
4. Guardian set becomes empty
5. All subsequent guardian verifications fail
6. Protocol functionality breaks completely

**Correct Implementation:**
```solidity
// Validate guardian set size before processing upgrade
require(guardianLength > 0, "Guardian set cannot be empty");
require(keyCount > 0, "Guardian set must contain at least one key");
```

**Key Considerations:**
1. Always validate guardian set size before processing upgrades
2. Ensure guardian set contains at least one key
3. Implement proper error handling for invalid guardian sets
4. Consider implementing minimum guardian set size requirements
5. Add validation at multiple levels (parsing and processing)
6. Include comprehensive test cases for edge cases

**Remediation Options:**
1. Add explicit validation for non-zero guardian length
2. Validate key count after processing guardian set
3. Implement minimum guardian set size requirements
4. Add emergency pause functionality for guardian set updates
5. Consider implementing guardian set size limits
6. Add comprehensive test coverage for guardian set edge cases

Reference bugs:
- [TOB-PYTHTON-2](https://github.com/pyth-network/audit-reports/blob/main/2024_11_26/pyth_ton_pull_oracle_audit_final.pdf): Empty guardian set validation in Pyth TON Oracle integration

### Signature Verification Issues in Guardian Sets

When implementing Wormhole guardian set signature verification, proper validation of unique signatures is crucial. The protocol must ensure that the same guardian signature cannot be used multiple times to achieve quorum.

**Example Implementation (from TON blockchain):**
```func
;; Example of missing unique signature validation in TON
;; This issue can occur in any chain's implementation of Wormhole guardian sets
() verify_signatures(int hash, cell signatures, int signers_length, cell guardian_set_keys, int guardian_set_size) impure {
    slice cs = signatures.begin_parse();
    int i = 0;
    int valid_signatures = 0;
    
    while (i < signers_length) {
        // ... signature parsing code ...
        
        int guardian_index = sig_slice~load_uint(8);
        int r = sig_slice~load_uint(256);
        int s = sig_slice~load_uint(256);
        int v = sig_slice~load_uint(8);
        
        // Missing validation for unique guardian indices
        (slice guardian_key, int found?) = guardian_set_keys.udict_get?(8, guardian_index);
        int guardian_address = guardian_key~load_uint(160);
        
        throw_unless(ERROR_INVALID_GUARDIAN_ADDRESS, parsed_address == guardian_address);
        valid_signatures += 1;
        i += 1;
    }
    
    ;; Check quorum (2/3 + 1)
    throw_unless(ERROR_NO_QUORUM, valid_signatures >= (((guardian_set_size * 10) / 3) * 2) / 10 + 1);
}
```

**Impact:**
- Single guardian can forge messages by repeating their signature
- Quorum can be achieved with duplicate signatures
- Protocol security is completely compromised
- Malicious guardian can create arbitrary messages
- Affects all chains implementing Wormhole guardian sets
- Breaks the fundamental security assumption of multi-signature verification

**Exploit Scenario:**
1. Current guardian set has 13 guardians
2. Rogue guardian discovers signature verification vulnerability
3. Guardian repeats their signature 13 times in a message
4. Message passes quorum check despite using same signature
5. Guardian can now create arbitrary messages without consensus
6. Protocol security is completely compromised

**Correct Implementation:**
```solidity
// Track used guardian indices to prevent duplicates
mapping(uint8 => bool) usedIndices;

function verifySignatures(
    bytes32 hash,
    bytes[] calldata signatures,
    address[] calldata guardianSet
) public {
    uint256 validSignatures = 0;
    
    for (uint256 i = 0; i < signatures.length; i++) {
        uint8 guardianIndex = uint8(signatures[i][0]);
        
        // Check for duplicate indices
        require(!usedIndices[guardianIndex], "Duplicate guardian signature");
        usedIndices[guardianIndex] = true;
        
        // Verify signature
        address signer = recoverSigner(hash, signatures[i]);
        require(signer == guardianSet[guardianIndex], "Invalid guardian signature");
        
        validSignatures++;
    }
    
    // Check quorum
    require(
        validSignatures >= ((guardianSet.length * 2) / 3) + 1,
        "Quorum not reached"
    );
}
```

**Key Considerations:**
1. Always validate unique guardian indices
2. Track used signatures during verification
3. Implement proper signature recovery
4. Consider implementing signature replay protection
5. Add validation for guardian set size
6. Include comprehensive test cases for edge cases

**Remediation Options:**
1. Add tracking of used guardian indices
2. Implement signature uniqueness checks
3. Add validation for minimum unique signatures
4. Consider implementing signature expiration
5. Add emergency pause functionality for signature verification
6. Implement comprehensive test coverage for signature verification

Reference bugs:
- [TOB-PYTHTON-3](https://github.com/trailofbits/publications/blob/master/reviews/PythTON.pdf): Single guardian signature quorum bypass in Pyth TON Oracle integration



