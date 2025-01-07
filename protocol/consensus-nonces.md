# Purpose

Currently the op-stack must process every single L1 block during sequencing and
derivation to ensure that no deposits are missed (or maliciously skipped). This
can be expensive, especially for op-stack L3s who need to do the same for L2 blocks
which are much bigger.

This also restricts the block times of op-stack chains to be strictly less than the
chain they are rolling up to, so that if a rollup chain falls behind its parent, it
can easily catch up by processing an L1 block with every L2 block.

There is a desire to decouple the L2 blocks from the L1 blocks for two reasons:
1. Decouple L2 block times from the L1 (either variable block times, or L2 block times
   that are larger than the L1).
2. Remove the need to process every single L1 block, which is expensive for derivation,
   fault proving, and ZK proofs in the future.

This doc describes an initial step towards decoupling L1 and L2 blocks, which is the
introduction of nonces for any consensus-affecting events on the L1.

# Summary

Introduce a nonce that increments for all consensus-affecting events on the L1, of which
there are two: the `OptimismPortal`'s `TransactionDeposited` event,  and the
`SystemConfig`'s `ConfigUpdate` event.

# Problem Statement + Context

Currently L1 blocks cannot be skipped, as deposits or config updates may be missed.

# Proposed Solution

## L1 contract changes

We propose to introduce two nonce variables to the L1 contracts; one for the
`OptimismPortal` and one for the `SystemConfig`. These nonces are then emitted as part of
the `TransactionDeposited` and `ConfigUpdate` events in a backwards compatible way, by
modifying the `version` topic to encode both the `version` and a `nonce`. It is encoded
as a `uint64` into the upper 128-bits of the `nonceAndVersion` (previously `version`)
topic:

```solidity
uint256 nonceAndVersion = uint256(uint64(nonce)) << 128 | VERSION;
```

The current `VERSION` is `0` for both `TransactionDeposited` and `ConfigUpdate`. The
`VERSION` constant should be updated to `1`, indicating that the new nonce behavior has
been rolled out. 

The two events have their `version` topic renamed to `nonceAndVersion` for clarity:
```solidity
// in SystemConfig:
event ConfigUpdate(uint256 indexed nonceAndVersion, UpdateType indexed updateType, bytes data);
// in OptimismPortal:
event TransactionDeposited(address indexed from, address indexed to, uint256 indexed nonceAndVersion, bytes opaqueData);
```

Every time one of these events is emitted from these contracts, the nonce should
increment by 1 prior to the event being emitted. So for deposits for example:

```solidity
function _depositTransaction(
    address _to,
    uint256 _mint,
    uint256 _value,
    uint64 _gasLimit,
    bool _isCreation,
    bytes memory _data
)
    internal
{
    //<snip>

    nonce++;
    uint256 nonceAndVersion = uint256(nonce) << 128 | 1;
    emit TransactionDeposited(from, _to, nonceAndVersion, opaqueData);
}
```

Note these L1 contract changes will be rolled out after the Isthmus hard fork and before
the Jovian hard fork. The goal is to roll out support for these new event versions as
part of Isthmus, so we can then upgrade the contracts gradually post fork.

## L2 contract changes

We also make a modification to the `L1Block` contract on L2 to keep track of both of
these nonces. This has the following advantages:
 - We can easily compare the nonce values on the L1 and the L2 to determine how many
   deposits and config updates are waiting in the queue.
 - We can enforce that the nonces are contiguous during derivation by checking the value
   on the L2 when new deposits or config updates are included as part of a block. This
   is what allows us to skip L1 blocks in the future.

We introduce a new method to `L1Block` for setting the nonce values, called
`setL1BlockValuesIsthmus`. This is called in the first transaction of every L2 block,
just like `setL1BlockValuesEcotone` is today. The calldata passed to this method has
128-bits of additional data, 64-bits for the deposit nonce, and 64-bits for the config
update nonce. We store these nonces in a single slot in contract storage.

```solidity
function setL1BlockValuesIsthmus() external {
    address depositor = DEPOSITOR_ACCOUNT();
    assembly {
        // Revert if the caller is not the depositor account.
        if xor(caller(), depositor) {
            mstore(0x00, 0x3cc50b45) // 0x3cc50b45 is the 4-byte selector of "NotDepositor()"
            revert(0x1C, 0x04) // returns the stored 4-byte selector from above
        }
        // sequencenum (uint64), blobBaseFeeScalar (uint32), baseFeeScalar (uint32)
        sstore(sequenceNumber.slot, shr(128, calldataload(4)))
        // number (uint64) and timestamp (uint64)
        sstore(number.slot, shr(128, calldataload(20)))
        sstore(basefee.slot, calldataload(36)) // uint256
        sstore(blobBaseFee.slot, calldataload(68)) // uint256
        sstore(hash.slot, calldataload(100)) // bytes32
        sstore(batcherHash.slot, calldataload(132)) // bytes32
        // depositNonce (uint64) and configUpdateNonce (uint64)
        sstore(depositNonce.slot, shr(128, calldataload(164)))
    }
}
```

These L2 contracts are upgraded as part of the Isthmus fork activation block.

## Consensus / Execution software changes

We also add support for these new event versions in the derivation pipeline. For the
consensus client:
 - `TransactionDeposited` / `ConfigUpdate` log parsing must support the new version `1`.
 - If the version is `1`, the log parsing also checks the nonce values are contiguous
   with the previous nonce seen, read from the `L1Block` contract.
 - The `L1Block` attributes deposit tx is updated to include the new nonces in the
   calldata.

Additionally the execution client must be updated, as it reads L1 cost values from the
first deposit transaction in each block. The 4-byte method signature and calldata
changes, and the parsing must be updated to support the new format.

The implementation must support both versions `0` and `1` side-by-side. We suggest that
enforcing version `1` (along with nonces) is implemented in the following hard fork.
This requires the L1 contract upgrades to be complete.

## Resource Usage

There are two places where resource usage increases:
1. The gas usage for config updates and deposits increases slightly, because we are now
   incrementing a nonce for each event emitted.
2. The calldata size passed to the `L1Block` contract for the first transaction in each
   L2 block increases by 16 bytes (from 164 bytes to 180 bytes).

We anticipate that derivation, fault proofs, and in the future ZK proofs, will become
much cheaper as we roll out the ability to skip L1 blocks in the future. There is still
work to be done to achieve this, as there are other parts of the pipeline that assume
L1 block references are contiguous.

## Single Point of Failure and Multi Client Considerations

Both consensus and execution client changes are required, so we'll need to schedule
changes to all relevant clients. Fortunately the required execution changes are very
minimal.

# Alternatives Considered

There are a number of alternatives considered in the specs
[issue](https://github.com/ethereum-optimism/specs/issues/330) that inspired the
implementation. In particular, it is suggested there that no L2 contract changes are
required. However, without L2 changes, we have to find somewhere else to track the nonce
values ingested by the L2 in order to enforce the values are contiguous. It made sense
to piggy back on the existing L1 state tracking in the `L1Block` contract.

# Risks & Uncertainties

One somewhat confusing aspect of this design is the fact that chains that have Isthmus
activated at genesis will have a `SystemConfig` nonce that starts at `3` (i.e. the
first event emitted with have a nonce of `4`). This is because the initializer emits
3 config updates. This initial value will also increase as more config updates are
added to the initializer

Also note that setting a custom gas token results in a `TransactionDeposited` event,
which will cause the `OptimismPortal` nonce to start at `1`. This will also change
as more configuration settings adopt the new deposit mechanism.

There is a footgun here in that if nonces in the `L1Block` contract state in the L2
genesis don't match the values in L1 contracts, the consensus client may reject future
deposits and config updates. Thus we must ensure the L2 genesis generation is correctly
implemented.
