# Polygon Security Checklist

This checklist explains security considerations when developing or integrating with Polygon PoS.


## General Considerations

### `block.timestamp` over `block.number` on Polygon PoS:

Polygon PoS uses a Heimdall+Bor consensus layer with variable block production intervals.`block.number` increases as expected with each block. However, because Polygon PoS relies on periodic checkpoints to Ethereum for security, finality is delayed, which means that using `block.number` for precise timing is unreliable. These checkpoints can introduce variations in block production times, making `block.number` an inconsistent measure for time-dependent logic. Relying on `block.timestamp` is recommended for more accurate time calculations.​

> **Avoid** using `block.number` as a proxy for time on Polygon PoS, as block production intervals and checkpoint submissions vary due to network congestion, Ethereum gas fees, and validator dynamics.

- Example Bug: A staking contract with a 24-hour cooldown enforced using block.number may under- or overestimate the delay based on block time variance, potentially enabling premature withdrawals.

## L1 -> L2 Messaging (State Sync)

Polygon uses a state sync mechanism for sending messages from Ethereum (L1) to Polygon (L2). These are processed through the `onStateReceive` function.

### `onStateReceive` Permissions

```
function onStateReceive(uint256 id, bytes calldata message) external only(STATE_SYNCER_ROLE) {
    _processMessageFromRoot(message);
}
```
Only the address `0x000...1001` (Polygon system address) can call `onStateReceive`. This is enforced via `only(STATE_SYNCER_ROLE)`:

```
## BaseChildTunnel.sol

function onStateReceive(uint256, bytes calldata message) external only(STATE_SYNCER_ROLE) {
        _processMessageFromRoot(message);
    }

    /**
     * @notice Emit message that can be received on Root Tunnel
     * @dev Call the internal function when need to emit message
     * @param message bytes message that will be sent to Root Tunnel
     * some message examples -
     *   abi.encode(tokenId);
     *   abi.encode(tokenId, tokenMetadata);
     *   abi.encode(messageType, messageData);
     */
    function _sendMessageToRoot(bytes memory message) internal {
        emit MessageSent(message);
    }

    /**
     * @notice Process message received from Root Tunnel
     * @dev function needs to be implemented to handle message as per requirement
     * This is called by onStateReceive function.
     * Since it is called via a system call, any event will not be emitted during its execution.
     * @param message bytes message that was sent from Root Tunnel
     */
    function _processMessageFromRoot(bytes memory message) virtual internal;
```
- The `_processMessageFromRoot` function is virtual and must be implemented in inheriting contracts. Avoid unbounded loops or expensive external calls inside _processMessageFromRoot.

Example bugs:

1.  **Spoofed Data**: If `_processMessageFromRoot` doesn’t validate message contents, malicious inputs could be passed my attackers.  
    Example: A protocol decoded `message` as `(address user, uint amount)` but forgot to check `amount > 0`, allowing free mints.
    
2.  **Replay Attacks**: Without state IDs/nonces, the same message could be processed twice.  

3.  **Gas Limits**: Complex logic in `_processMessageFromRoot` might fail during gas spikes.


## L2 → L1 Withdrawal Risks on Polygon PoS

Polygons withdrawal system relies on checkpoints submitted to Ethereum. When users withdraw funds, validators batch the transactions into Merkle trees and submit the roots to Ethereum’s `RootChainManager`.

The withdrawal path includes:

-   Event emission on Polygon (for example, `Withdraw()`)
    
-   Inclusion of that block in a checkpoint
    
-   Finalization on Ethereum
    
-   Merkle proof verification on `RootChainManager`

The key part of the withdrawal flow is validating Merkle proofs of inclusion:

```
## RootChainManager.sol
_checkBlockMembershipInCheckpoint(...);
```
> Always use Polygon’s audited [MerklePatriciaProof](https://github.com/0xPolygon/pos-contracts/tree/main/contracts/common/lib) library and validate all proof components.

Reorganizations (reorgs) on Polygon’s Heimdall chain add another layer of complexity. While Ethereum provides strong finality, Polygon’s L2 can reorg up to 16 blocks. This means withdrawals should only be considered final after:

1.  Being included in a checkpoint on Ethereum
    
2.  16 blocks have passed on Heimdall

### Predicate Contract Enforcement

The [RootChainManager.sol](https://github.com/maticnetwork/pos-portal/blob/master/flat/RootChainManager.sol) contract is the canonical production contract deployed by Polygon. It acts as the entry point for all Polygon PoS withdrawals and is responsible for validating Merkle proofs and triggering corresponding state changes on L1. This contract should not be directly modified. Instead building an own costum contract that allows the RootChainManager to trigger those via designated predicate contracts. 

If an application involves custom logic, such as updating application state on Ethereum after an event on Polygon, creating a root contract that guards all state-mutating functions with the `onlyPredicate` modifier is the way to go. This ensures only the trusted `RootChainManager` (through its designated predicate) can trigger updates, following successful checkpoint validation.

Here's an example of a secure root contract:

```
contract Root {
    address public predicate;
    constructor(address _predicate) public {
        predicate = _predicate;
    }

    modifier onlyPredicate() {
        require(msg.sender == predicate, "Only predicate can call");
        _;
    }

    uint256 public data;

    function setData(bytes memory bytes_data) public onlyPredicate {
        data = abi.decode(bytes_data, (uint256));
    }
}
```

The Polygon PoS bridge flow guarantees that only verified Polygon-side transactions can invoke the predicate, and so change Ethereum-side state. This is achieved by checkpoint inclusion, Merkle proof verification, and strict access controls in the root logic.

Bug Example:

- If `onlyPredicate` is forgot, any user can directly call `setData()` on the root contract, bypassing proof verification entirely, allowing unauthorized state changes.

## Checkpointing & Withdrawals

Polygon PoS withdrawals rely on periodic checkpoint submissions from validators to Ethereum. These checkpoints commit Merkle roots of Polygon blocks to the RootChainManager contract on Ethereum.

### Withdrawal Finality Conditions

- A transaction is executed on the Polygon PoS child contract.

- The transaction must emit an event whose parameters include the data intended for transfer.

- Within ~10–30 minutes, validators include this transaction in a checkpoint and commit it to Ethereum.

- Once finalized, the hash of the Polygon transaction can be submitted to the RootChainManager on Ethereum.

- RootChainManager verifies the transaction inclusion and decodes the event log.

- The event data is forwarded to a predicate contract, which updates state on the Ethereum root contract.

Only then can the exit function on RootChainManager be invoked to process the withdrawal.

```
function exit(bytes calldata inputData) external override {
    ExitPayloadReader.ExitPayload memory payload = inputData.toExitPayload();
    _checkBlockMembershipInCheckpoint(...);
    // parse receipt and call predicate contract
}
```
> Risk: If checkpoints are delayed due to high Ethereum gas fees, withdrawal finality can take hours or even longer. This introduces significant liquidity risks.

### Exit Uniqueness via Branch Mask

Each exit is uniquely identified using a hash of (blockNumber, branchMask, receiptLogIndex). Improper mask parsing can bypass exit deduplication checks:

```
## RootChainManager.sol 

bytes32 exitHash = keccak256(
    abi.encodePacked(
        payload.getBlockNumber(),
        MerklePatriciaProof._getNibbleArray(branchMaskBytes),
        payload.getReceiptLogIndex()
    )
);
```
> Known Bug Class: Weak uniqueness check due to prefix collisions in branch masks.

### Emergency Withdrawal
    
The emergency withdrawal pattern is crucial for handling edge cases. A well-designed system should include:

```
function emergencyWithdraw() external {
    require(
        block.timestamp > lastCheckpointTime + 24 hours, 
        "Standard withdrawal still possible"
    );
    _burn(msg.sender, amount);
    emit EmergencyWithdraw(msg.sender, amount);
}
```

## Validator & Staking Risks

The `ValidatorShare` contract handles delegator stakes via share-based accounting. Some key functions include the `buyVoucher`function which converts staked POL to shares using `exchangeRate`.

```
## ValidatorShare.sol

function buyVoucher(uint256 _amount, uint256 _minSharesToMint) public returns (uint256) {
    uint256 shares = _amount.mul(_getRatePrecision()).div(exchangeRate());
    require(shares >= _minSharesToMint);
    _mint(msg.sender, shares);
}

// Exchange rate calculation
function exchangeRate() public view returns (uint256) {
    uint256 totalShares = totalSupply();
    return totalShares == 0 ? _getRatePrecision() 
         : stakeManager.delegatedAmount(validatorId).mul(_getRatePrecision()).div(totalShares);
}
```
### Share Calculation Bugs

1.  **Rounding Errors** Division truncates in `exchangeRate()`, potentially giving fewer shares than expected.
    
    > Example Bug: User stakes 100 POL and receives 99.99 shares due to precision loss.
    
2.  **Precision Mismatch** Older validators use `EXCHANGE_RATE_PRECISION = 100`, newer ones use `1e29`. Mixing them leads to misaligned share calculations.
    
    > Standardize precision and enforce consistent validator config.

## Recommended resources

- [Polygon Knowledge Layer](https://docs.polygon.technology/pos/)
- [Polygon PoS Audit](https://www.chainsecurity.com/security-audit/polygon-pos-portal-smart-contracts)
