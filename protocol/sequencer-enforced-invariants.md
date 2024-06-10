# Purpose

We ([Firewall](https://usefirewall.com/)) are building the Safe Sequencer for OP Stack chains. Safe sequencing uses exploit detection during block construction to exclude transactions from blocks that are identified as smart contract exploits. However, forced L1 → L2 messages can bypass the exploit detection of a safe sequencer and force include an exploit into a block.

# Summary

This is a design doc to enable (optional) sequencer-enforced invariants. We propose a design that uses L1 → L2 message validation hooks to delegate inclusion and ordering invariants to the sequencer without requiring changes to the derivation pipeline for handling deposits.

# Problem Statement + Context

The OP Stack does not allow computation in the sequencer to determine deposit inclusion since L1 → L2 messages are always force executed on L2. This forced execution can bypass additional inclusion and ordering invariants offloaded to the sequencer role.

We believe delegating inclusion and ordering invariants to the sequencer can be a useful pattern, especially when heavy and complex computation is needed to enforce invariants, as seen in safe sequencing. This separation from the derivation pipeline and fault proof program of a rollup offers some nice benefits:

- Sequencer-enforced invariants can be added or removed from rollups through the SystemConfig.
- Sequencer-enforced invariants can be upgraded and managed at a pace appropriate for their own domain.
- Sequencer-enforced invariants cannot create bugs in the rollup’s fault proof program that effect withdrawals.

Sequencer-enforced invariants minimize complexity, yet still maintain rollup sovereignty. User sovereignty can also be maintained by allowing a fault or in(validity) proof of the delegated sequencer to [automatically trigger](https://x.com/sam___iamm/status/1798008895488827662) the SystemConfig to fall back to OP Stack defaults.

### Related Problems

MEV capture designs such as [Priority Is All You Need](https://www.paradigm.xyz/2024/06/priority-is-all-you-need) also use the sequencer role to enforce inclusion and ordering invariants and may require special handling of deposits. However, we note that sequencer-enforced invariants are more relevant for heavy and complex computation like exploit detection algorithms.

# Proposed Solution

## Hooks for configurable static analysis of L1 → L2 Messages

Message validation hooks can be used within the bridging contracts to ensure that deposit messages will not violate sequencer-enforced invariants. These hooks are optional, turned off by default, and managed through the rollup’s SystemConfig.

### Message Validation Hook Design

Reference implementation: https://github.com/ethereum-optimism/optimism/pull/10791

Two storage variables are added to the `SystemConfig` contract on L1 and, correspondingly, the `L1Block` contract on L2: `L1_MESSAGE_VALIDATOR_SLOT` and `L2_MESSAGE_VALIDATOR_SLOT`. The addresses at these storage slots, if they are non-zero, MUST adhere to the `IL1MessageValidator` and `IL2MessageValidator` interfaces respectively.

```solidity
interface IL1MessageValidator {
		// Returns a boolean indicating if the message passes additional validation.
    function validateMessage(
        address _from,
        address _to,
        uint256 _mint,
        uint256 _value,
        uint64 _gasLimit,
        bool _isCreation,
        bytes memory _data
    )
        external
        view
        returns (bool);
}
```

```solidity
interface IL2MessageValidator {
    // Returns a boolean indicating if the message passes additional validation.
    function validateMessage(
        uint256 _nonce,
        address _sender,
        address _target,
        uint256 _value,
        bytes calldata _message
    )
        external
        view
        returns (bool);
}
```

The `IL1MessageValidator` is static called within the `OptimismPortal` during the `_depositTransaction` function before the `TransactionDeposited` event is emitted. If the message validation fails, then the call reverts.

The `IL2MessageValidator` is called within the `L2CrossDomainMessenger` during the `relayMessage` function before the external call of a given message. If message validation fails, then the message is added to the `failedMessages` mapping and the function returns. 

A failed message can be replayed through a transaction submitted to the sequencer, or through forced inclusion if the L2 hook is later removed by the SystemConfig. This allows sequencer-level invariants to be enforced ubiquitously and deposits to be recovered in case of a censoring sequencer.

The `IL2MessageValidator` hook can only be enforced if the `IL1MessageValidator` implementation requires L1 → L2 messages to go through the `CrossDomainMessenger` contracts. The `L2CrossDomainMessenger` is embedded with this assumption and uses an `_isOtherMessenger()` check to ensure message validation only occurs on forced deposits, not replays.

### Example Usage: Safe Sequencing

To enable safe sequencing, we add the following validation rules:

- On L1 in our `IL1MessageValidator` implementation:
    - `_from` == `L1CrossDomainMessenger`, `_to` == `L2CrossDomainMessenger`, OR
    - _`value` > 0, `_from`  is an EOA (account with no code), and `_from` == `_to`
- On L2 in our `IL2MessageValidator` implementation:
    - If `_isOtherMessenger()`, then we know that the transaction is a forced deposit and the sequencer cannot enforce additional invariants. Therefore, we apply conservative rules:
        - `_target` == `L2StandardBridge`, `finalizeBridgeETH` is the function, and `_to` is an EOA that contains no code OR
        - `_target` == `L2StandardBridge`, `finalizeBridgeERC20` is the function, and `_localToken` is in a trusted whitelist OR
        - `_target` == `L2ERC721Bridge`, `finalizeBridgeERC721` is the function, `_localToken` is in a trusted whitelist, and `_to` is an EOA that contains no code OR
        - `_value` > 0 and `_target`  is an EOA that contains no code OR
        - needs work but potentially: `_value` > 0 and `_target` is not an EOA, but we check there is no fallback function
    - Otherwise, we don’t enforce additional rules.

### UX Considerations

The `IL2MessageValidator` implementation will have strict rules to protect the sequencer’s invariants. It is recommended that the sequencer immediately tries to insert a transaction that replays a failed message due to `IL2MessageValidator` validation.

# Alternatives Considered

### Forced [no-op deposits](https://github.com/ethereum-optimism/design-docs/blob/1566367c32922ab0c64adb9865c68197d5563247/protocol/deposits-interop.md#proposed-solution) by the sequencer.

Unsafe payloads gossiped by the sequencer and data posted on L1 could be extended to include information that identifies certain transactions as failing the sequencer’s invariants. These transactions could be force-reverted, assuming a change to the execution layer. For deposit transactions, this could look a lot like the [no-op deposits specification](https://github.com/ethereum-optimism/design-docs/blob/1566367c32922ab0c64adb9865c68197d5563247/protocol/deposits-interop.md#proposed-solution). For normal transactions, this could charge a certain amount of gas used for built-in DDoS protection.

This requires modifying the derivation pipeline and does not allow deposit messages to be replayed unless transactions targeting the `L2CrossDomainMessenger` are handled specifically. These changes impact the fault proof program of a rollup and are more involved throughout the stack than the proposed solution. However, this solution still separates the actual computation for sequencer-enforced invariants from the rollup’s derivation and fault proof program. The result of the sequencer’s computation must be fed into the rollup’s derivation and fault proof program as an input.

### Embed “sequencer”-enforced invariants within the derivation pipeline and fault proof program.

In this case, we abandon the separation of concerns and move “sequencer”-enforced invariants and associated compute within the derivation and fault proof program of a rollup. In the case of safe sequencing, exploit detection algorithms must be run by each node on the network and embedded within the derivation and fault proof program of a rollup.

# Risks & Uncertainties

- L1 → L2 message validation hooks should only be added to an existing network after a delay to allow users to exit if they don’t agree with the change. Should we embed this delay into the specification?

- It is presumed that the sequencer’s additional computation can be proven on-chain by a fault or in(validity) proof that automatically revokes L1 → L2 message validation hooks. This allows the sequencer’s proof system to [quickly manage a soft fork](https://x.com/gakonst/status/1797836173269983335) if censorship occurs. Should we use a separate owner within the SystemConfig for managing message validation hooks to ensure a sequencer’s proof can’t control unrelated values?

- We optimistically accept results shared by the sequencer and do not reorg the chain if the sequencer violates its promised invariants. This can make the design unworkable for modifications like [Interop](https://github.com/ethereum-optimism/design-docs/blob/1566367c32922ab0c64adb9865c68197d5563247/protocol/deposits-interop.md) where invariants must be true within the rollup’s state-transition function.

- In the proposed solution, the sequencer must manage DDoS attack vectors on its own, as it cannot charge gas for transactions that are not included in a block. The “Forced no-op deposits by the sequencer” alternative naturally solves the problem by charging gas used for transactions identified as failing a sequencer-enforced invariant.