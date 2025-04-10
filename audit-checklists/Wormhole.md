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


### Direct `publishMessage` Usage

Protocols can publish messages directly through the Wormhole-Core contract's [`publishMessage`](https://github.com/wormhole-foundation/wormhole/blob/fd1ed1564e3a4047cca78ac539956c9932664d96/ethereum/contracts/Implementation.sol#L15) function:

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

### Message Fee Handling

The `publishMessage()` function is payable and requires the correct fee to be passed. Always fetch the current fee using `messageFee()` on the Wormhole-Core contract instead of hardcoding the value.

```solidity
// Problematic Implementation
ICircleIntegration(wormholeCircleBridge).transferTokensWithPayload(
    transferParameters, 
    0,  // No fee value passed
    abi.encode(msg.sender)
);

// Correct Implementation
uint256 messageFee = wormholeCore.messageFee();
ICircleIntegration(wormholeCircleBridge).transferTokensWithPayload{value: messageFee}(
    transferParameters,
    messageFee,
    abi.encode(msg.sender)
);
```

Bug examples: [1](https://0xmacro.com/library/audits/infinex-1.html#M-1), [2](https://cdn.prod.website-files.com/621a140a057f392845dfaef3/65c9d04d3bc92bd2dfb6dc87_SmartContract_Audit_MagpieProtocol(v4)_v1.1.pdf), [3](https://iosiro.com/audits/infinex-accounts-smart-account-smart-contract-audit#IO-IFX-ACC-007),  [QS-5](https://certificate.quantstamp.com/full/hashflow-hashverse/1af3e150-d612-4b24-bc74-185624a863f8/index.html#findings-qs5), [P1-I-02](https://2301516674-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FNGiifMbrcug9ogAfisQY%2Fuploads%2FPxfV4xPmOCVKng8LcjiH%2FMayan_MCTP_Sec3.pdf?alt=media&token=62699afe-9e67-44fb-96fe-b593041365f4), [OS-SNM-SUG-02](https://2239978398-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FzjiJ8UzMPEfugKsLon59%2Fuploads%2FYr6wLCHl8r6uS6eHAYnb%2Fsynonym_audit_final.pdf?alt=media&token=3ad993f9-da68-496d-be06-d7eed5d305de)

### Relayer Authorization

Verify messages originate from the authorized Wormhole relayer using both `isRegisteredSender` modifier and relayer address check.

```solidity
// Problematic Implementation
function receiveWormholeMessages(bytes memory payload) public {
    // Missing relayer validation
}

// Correct Implementation
function receiveWormholeMessages(
    bytes memory payload,
    bytes[] memory,
    bytes32 sourceAddress,
    uint16 sourceChain,
    bytes32
) public payable override 
isRegisteredSender(sourceChain, sourceAddress) 
{
    require(msg.sender == address(wormholeRelayer), "Only Wormhole relayer");
}
```

### Chain ID vs Domain Mislabeling

Check for incorrect labeling of chain IDs as "domains" in function parameters and variable names.

```solidity
// Problematic Implementation
function _bridgeUSDCWithWormhole(
    uint256 _amount, 
    bytes32 _destinationAddress, 
    uint16 _destinationDomain  // Mislabeled - should be _destinationChainId
) internal {
    if (!_validateWormholeDestinationDomain(_destinationDomain)) revert Error.InvalidDestinationDomain();
}

// Correct Implementation
function _bridgeUSDCWithWormhole(
    uint256 _amount, 
    bytes32 _destinationAddress, 
    uint16 _destinationChainId
) internal {
    if (!_validateWormholeDestinationChainId(_destinationChainId)) revert Error.InvalidChainId();
}
```

Bug examples: [L1](https://0xmacro.com/library/audits/infinex-1.html#L-1), [QS-3](https://certificate.quantstamp.com/full/hashflow-hashverse/1af3e150-d612-4b24-bc74-185624a863f8/index.html#findings-qs3), [P1-I-05](https://2301516674-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FNGiifMbrcug9ogAfisQY%2Fuploads%2Fz6Gq4wJprv7LomYsQ1LY%2FMayan_Swift_Sec3.pdf?alt=media&token=4a1b7f69-a626-4e34-9db2-4e931c3bc11f)

### Decimal Scaling Issues in Token Bridging

When implementing token bridging, it's crucial to handle different token decimals correctly. A common issue occurs when using a hardcoded maximum bridge amount without considering token-specific decimal places.

**Problematic Implementation:**
```solidity
// Problematic Implementation
function _computeScaledAmount(uint256 _amount, address _tokenAddress) internal returns (uint256) {
    uint256 scaledAmount = uint256(DecimalScaling.scaleTo(
        int256(_amount), 
        IERC20Metadata(_tokenAddress).decimals()
    ));
    // Incorrectly using USDC's max amount for all tokens
    if (scaledAmount > BridgeIntegrations._getBridgeMaxAmount()) 
        revert Error.BridgeMaxAmountExceeded();
}

// Correct Implementation
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
}
```

Bug examples: [1](https://0xmacro.com/library/audits/infinex-1.html#L-2)

### Normalize and Denormalize makes dust amount stuck in contracts

When implementing token bridging with Wormhole, the token bridge performs normalize and denormalize operations on amounts to remove dust. This can lead to tokens getting stuck in the contract if not handled properly.

**Problematic Implementation:**
```solidity
// Problematic Implementation
function swap(uint256 amountIn) external {
    IERC20(token).transferFrom(msg.sender, address(this), amountIn);
    tokenBridge.transferTokens(token, amountIn, destinationChain, destinationAddress);
}

// Correct Implementation
function swap(uint256 amountIn) external {
    uint256 normalizedAmount = normalize(amountIn);
    uint256 denormalizedAmount = denormalize(normalizedAmount);
    
    // Only transfer the exact amount that will be bridged
    IERC20(token).transferFrom(msg.sender, address(this), denormalizedAmount);
    tokenBridge.transferTokens(token, denormalizedAmount, destinationChain, destinationAddress);
}
```

Bug examples: [OS-MYN-ADV-05](https://2301516674-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FNGiifMbrcug9ogAfisQY%2Fuploads%2F9YSweDuRP1P28bMmjy63%2FMayan_Audit_OtterSec.pdf?alt=media&token=ffefae4d-367d-401f-bd16-2d799dd3a403), [OS-OPF-ADV-00](https://2239978398-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FzjiJ8UzMPEfugKsLon59%2Fuploads%2F19FGz84wBMigB9EBngTu%2Foptimistic_finality_audit_final.pdf?alt=media&token=c3a631b5-0cc0-4104-a781-0691e2491973)

### Double Normalization/Denormalization Issues

Check for multiple normalization/denormalization operations on the same amount.

```solidity
// Problematic Implementation
function getCurrentAccrualIndices(address assetAddress) public view returns(AccrualIndices memory) {
    uint256 deposited = getTotalAssetsDeposited(assetAddress); // Already normalized
    // Double normalization occurs here
    uint256 normalizedDeposited = normalizeAmount(deposited, accrualIndices.deposited, Round.DOWN);
}

// Correct Implementation
function getCurrentAccrualIndices(address assetAddress) public view returns(AccrualIndices memory) {
    uint256 deposited = getTotalAssetsDeposited(assetAddress); // Already normalized
    // Use values directly without additional normalization
}
```

Bug examples: [OS-SNM-ADV-04](https://2239978398-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FzjiJ8UzMPEfugKsLon59%2Fuploads%2FYr6wLCHl8r6uS6eHAYnb%2Fsynonym_audit_final.pdf?alt=media&token=3ad993f9-da68-496d-be06-d7eed5d305de), [OS-SNM-ADV-05](https://2239978398-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FzjiJ8UzMPEfugKsLon59%2Fuploads%2FYr6wLCHl8r6uS6eHAYnb%2Fsynonym_audit_final.pdf?alt=media&token=3ad993f9-da68-496d-be06-d7eed5d305de)

### Guardian Set Transition Issues

When implementing Wormhole guardian set updates, proper handling of the transition period is crucial. The protocol needs to maintain both old and new guardian sets active during the transition period to ensure continuous operation. This issue can affect any chain implementing Wormhole guardian sets.

The following is an example from TON blockchain smart contracts of incorrect handling of guardian node transitions.

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

;; Correct Implementation (change the operator == to >=)
throw_unless(
    ERROR_INVALID_GUARDIAN_SET(
        current_guardian_set_index >= vm_guardian_set_index(
            (expiration_time == 0) | (expiration_time > now())
        )
    )
);
```

Bug examples: [TOB-PYTHTON-1](https://github.com/pyth-network/audit-reports/blob/main/2024_11_26/pyth_ton_pull_oracle_audit_final.pdf)

### Empty Guardian Set Validation

Check for missing validation of guardian set size during upgrades.

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
    
    ;; Missing validation for non-zero length
    int guardian_length = payload~load_uint(8); 
    cell new_guardian_set_keys = new_dict();
    int key_count = 0;
    while (key_count < guardian_length) {
        builder key = begin_cell();
        int key_bits_loaded = 0;
        while (key_bits_loaded < 160) {
            int bits_to_load = min(payload.slice_bits(), 160 - key_bits_loaded);
            key = key.store_slice(payload~load_bits(bits_to_load));
            key_bits_loaded += bits_to_load;
            if (key_bits_loaded < 160) {
                throw_unless(ERROR_INVALID_GUARDIAN_SET_UPGRADE_LENGTH, ~ payload.slice_refs_empty?());
                payload = payload~load_ref().begin_parse();
            }
        }
        slice key_slice = key.end_cell().begin_parse();
        new_guardian_set_keys~udict_set(8, key_count, key_slice);
        key_count += 1;
    }
    throw_unless(ERROR_GUARDIAN_SET_KEYS_LENGTH_NOT_EQUAL, key_count == guardian_length);
    throw_unless(ERROR_INVALID_GUARDIAN_SET_UPGRADE_LENGTH, payload.slice_empty?());

    return (action, chain, module, new_guardian_set_keys, new_guardian_set_index);
}
```

Missing empty guardian set validation causes `parse_and_verify_wormhole_vm` to fail in the subsequent calls, requiring at least one guardian key. 

```diff
;; Correct Implementation:
;; Validate guardian set size before processing upgrade
;; Guardian set cannot be empty or Guardian set must contain at least one key after upgrade
    int guardian_length = payload~load_uint(8); 
+   throw_unless(ERROR_EMPTY_GUARDIAN_SET, guardian_length != 0);
    cell new_guardian_set_keys = new_dict();
    int key_count = 0;
    while (key_count < guardian_length) {
```

Bug examples: [TOB-PYTHON-2](https://github.com/pyth-network/audit-reports/blob/main/2024_11_26/pyth_ton_pull_oracle_audit_final.pdf)

### Signature Verification Issues in Guardian Sets

Check for missing validation of unique guardian signatures during verification.

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
The above TON smart contract implements verification of guardian node signatures with a quorum of super majority (`>2/3`). But the issue is that it is not tracking the verified guardian nodes. Meaning that it allows replay of one guardian node signature for `>2/3` times will pass the quorum. 

```func
;; A temporary example of correct implementation, but you should check all the edge cases of guardian node signing process. 
function verifySignatures(bytes[] calldata signatures) public {
    mapping(uint8 => bool) usedIndices;
    for (uint256 i = 0; i < signatures.length; i++) {
        uint8 guardianIndex = uint8(signatures[i][0]);
        require(!usedIndices[guardianIndex], "Duplicate guardian signature");
        ;; tracking of verified guardian node signatures.
        usedIndices[guardianIndex] = true;
        
        ;; Verify signature
        address signer = recoverSigner(hash, signatures[i]);
        require(signer == guardianSet[guardianIndex], "Invalid guardian signature");
        
        validSignatures++;
    }
}
```

Bug examples: [TOB-PYTHON-3](https://github.com/pyth-network/audit-reports/blob/main/2024_11_26/pyth_ton_pull_oracle_audit_final.pdf)



