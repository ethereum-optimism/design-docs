# Purpose

*The document presented below is part of the [Interoperability project](https://github.com/ethereum-optimism/optimism/issues/10899).*

This document discusses possible solutions to address the constraints on ETH withdrawals resulting from the introduction of interop and `SuperchainWETH` that share ETH liquidity across a set of OP Chains.

# Summary

With interoperable ETH, withdrawals may fail if the referenced `OptimismPortal`lacks sufficient ETH—especially for large amounts or on OP Chains with low liquidity—since ETH liquidity isn't shared at the L1 level. Several solutions have been explored to prevent users from getting stuck. Currently, the Superchain design favors the `SharedLockbox` as the most effective solution. Key considerations are detailed at the end of the document.

# Problem Statement + Context

With the introduction of interop, `SuperchainWETH` is created as a medium to move ETH across a set of interoperable chains, instead of using native ETH. To enable users to convert `SuperchainWETH` into native ETH, `ETHLiquidity` is also introduced, containing a sufficiently large virtual pool of native ETH, which can only be used for this purpose. As a result, ETH can be shared through all interconnected chains.

However, one remaining problem to solve relates to [ETH withdrawals](https://github.com/ethereum-optimism/specs/issues/362). Currently, with Interop, the amount of ETH on an L2 can differ from what is deposited in the respective `OptimismPortal`. This mismatch can interfere with the finalization of withdrawals, especially if a request requires more ETH than is actually deposited at a given time.

As a side note, this problem is also closely related to the process of how an OP Chain enters and leaves an interoperable set of chains. Not defining this scenario could create a problem in the future.

# Prior considerations

Before explaining the proposed solution, it is important to note that there are essential assumptions that underpin the whole design.

### Chain Management

This refers to how OP Chains will be governed within an interoperable set of chains (also called a "cluster"). Once governance principles are well-established from the perspective of the technological stack, state transitions can be secured, enabling ETH to be uniformly integrated and secured across the entire cluster. Currently, this is achieved by addressing the following:

- The interoperable set of chains will be governed by an entity responsible for:
    - Approving chains to join the cluster. This is done either through trusted deployment methods (e.g., OP Stack Manager) or by approval after security checks are performed.
    - Managing (adding and removing) chains within the interoperable graph.
    - Approving protocol upgrades for all involved.
    - Taking action if an OP Chain decides not to follow the established standard (e.g. Expresses a desire to leave the cluster.).
- Each chain is capable of defending against any maliciously claimed state transition. Ideally, this is accomplished through a set of security features at the proof level, such as permissionless proofs, capability of proving executions that are related to interop executions, and the presence of the Guardian role for the whole system. This is indifferent to what proof system is used (ZK, fault proofs, etc.).
- Joining a cluster will automatically enable the use of `SuperchainWETH` for the specific set of chains defined by the `dependencyManager`.

### SuperchainWETH usage

As mentioned above, in any OP Chain within the cluster where interop is enabled, the use of `SuperchainWETH` will be activated. As a result, equivalence between ETH deposits and the actual ETH supply will vary from the outset. In a world with freedom of movement, all ETH liquidity is theoretically shared across the entire cluster eventually, regardless of how deposits are handled. This is an unavoidable scenario that deserves careful consideration and is crucial for understanding the following sections.

# Solution

We propose an L1 shared liquidity design, achieved by introducing a new `SharedLockbox` in L1 that serves as a singleton ETH contract for the entire interoperable set of chains. New ETH deposits will be forwarded to the lockbox, and the same applies to ETH withdrawals. This could also serve as the singleton contract for gas tokens in the case of CGT chains.

### Spec changes

The core changes proposed are the following:

- Introduce `SharedLockbox` which accepts deposits and withdrawals of ETH and Gas Tokens as well.
- Modify `OptimismPortal` to forward ETH deposits into `SharedLockbox` and allow extraction of ETH when a withdrawal is finalized.
- Maintain an on-chain list (through `SystemConfigInterop` or another contract if already defined) that registers the chains that are part of the cluster. This allows `SharedLockbox` to validate a withdrawal from a chain that is part of the cluster.

This would require a one-time L1 liquidity migration process for each chain. Newly deployed chains via `OPSM` should use the `SharedLockbox` from the beginning.

### Joining the Cluster

From the point of view of a chain deployed via `OPSM`, we suggest adopting `SharedLockbox` even when there is no interoperable set established for it. Since the security is guarded by the security council and governance, it should not represent a risk.

For existing chains, the `OptimismPortal` should incorporate a new gated function that hardcodes the sending of ETH to the `SharedLockbox` contract and that can only be called by a trusted entity, e.g., the Security Council. This function should also emit an event which the off-chain components can use to track how much ETH was initially moved from a chain to the `SharedLockbox`, which may be required in case the chain wishes to opt out of the cluster in the future.

```solidity
function joinInteropCluster() external onlySecurityCouncil {
	uint256 balance = address(this).balance;
	address(SHARED_LOCKBOX).call{value: balance}("");
	emit JoinedInteropCluster(chainId, balance);
}

```

### Opt-out the cluster

A process to opt out of the cluster should be possible. For this to work properly, it is first necessary to track the actual amount of ETH that pertains to the L2 at a given time. As an open system, we need to monitor the net flows between L1-L2 and L2-L2. For L1-L2, this involves tracking deposits and withdrawals through an L1 deposit mapping. For L2-L2, an `InteropETHTransferred` mapping is introduced for the same purpose. When interop is enabled, the following elements need to be present:

- The initial migrated ETH, serves as the starting amount.
- A deposit mapping that tracks ETH successfully deposited and finalized withdrawals.
- The `InteropETHTransferred` mapping is implemented when `SuperchainWETH` interoperability transfers are enabled.

Then, the opt-out process would be performed as follows:

1. Disconnect the chain from the interop set.
2. Pause new deposits.
3. Waiting time for stabilizing pending interop messages and deposits.
4. Trigger the migration by porting `InteropETHTransferred` back to L1.

One path could be having two functions to initiate and finalize the migration, or directly via a protocol upgrade.

The resultant math to calculate the exact ETH amount to migrate will be:

ETH to Migrate = (1)initial balance + (2)deposit mapping + (3)Netflow ETH interop transferred.
Note that (1) should be zero if the chain is deployed using `SharedLockbox` by default. (2) and (3) are integers as their number could become negative. For (3) to be an accurate value, there should be no pending messages waiting to expire. 

### Impact

The whole work can be split into two big phases:

1. **First phase**: Modify the necessary core L1 contracts, adding the `InteropETHTransferred` counter for `SuperchainWETH` total transfer history and the standardized process for new and existing chains to adopt the `SharedLockbox`.
2. **Second phase**: Develop the needed pieces for the opt-out process.

All phases require an audit of the new and modified contracts: `OptimismPortal`, `SharedLockbox`, `SystemConfigInterop`, `SuperchainWETH` transfer mapping in L2, and possibly others. This also includes the scripts to perform the migration.

# Alternatives Considered

### Reverts on L2

One alternative is not to share liquidity at the L1 level and instead prevent withdrawals from being stuck by reverting them on L2. This approach requires tracking the exact ETH balance in L2 via deposits and withdrawals. An `ETHBalance` counter would increment with new mints from deposits and decrease with successful `initiateWithdrawal` calls.

This method would require minimal changes to `L1Block` and adjustments to how `TransactionDeposited` is processed to validate new ETH counts. Additionally, it necessitates porting the initial balance as part of an upgrade transaction.

The problem with L2 reverts is that it breaks the ETH withdrawal guarantee invariant and still exposes the system to solvency issues. Based on previous feedback, this would affect existing applications and pre-deployed contracts such as the current `FeeVault` withdrawal contract logic.

### **Permission to withdraw ETH from a different Portal**

[Another solution](https://github.com/ethereum-optimism/specs/issues/362#issuecomment-2332481041) involves allowing ETH withdrawals to be finalized by taking ETH from one or more `OptimismPortal` contracts. In a cluster, this is done by authorizing withdrawals across the set of chains. For example, if Chain A connects to Chain B and Chain B connects to Chain C (Chain A ⮀ Chain B ⮀ Chain C), but Chain A does not connect directly to Chain C (Chain A ↮ Chain C), a withdrawal initiated from A could be finalized by taking funds from C if needed.

The implementation would require iterating through the dependency set, determined on L1 to find the next `OptimismPortal` with available funds. This approach incurs more modifications to the `OptimismPortal`, increasing complexity.

# Risks & Uncertainties

- **Scalable security**: With interop, withdrawals are a critical flow to protect, especially for ETH, since it tentatively becomes the most bridged asset across the Superchain. This means proof systems, dedicated monitoring services, the Security Council, and the Guardian need to be proven to tolerate the growing number of chains.
- **Unfinalized interop messages**: For `InteropETHTransferred` to work properly, we need to figure out what it means for the current rollback message approach, especially relevant at the moment of the opt-out.
    - For example, one main scenario to avoid is when the `InteropETHTransferred` value is pulled back to L1, but there are still expired messages ready to be claimed, resulting in the L2 ETH supply being greater than the amount reported at the L1 level.
- **Interoperable CGT**: how to make ETH interoperable with Custom Gas Token chains, as well as gas tokens for normal OP Chains. If the Shared Liquidity approach is accepted, this will be prioritized.