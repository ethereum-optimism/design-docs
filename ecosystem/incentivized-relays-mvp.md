# [Incentivized Relays MVP]: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | _Hamdi Allam_                                      |
| Created at         | _2025-04-21_                                       |
| Initial Reviewers  | _Reviewer Name 1, Reviewer Name 2_                 |
| Need Approval From | _Reviewer Name_                                    |
| Status             | _Draft / In Review / Implementing Actions / Final_ |

## Purpose

<!-- This section is also sometimes called “Motivations” or “Goals”. -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

We want to preserve a single transaction experience in the Superchain event when a transaction spawns asynchronous cross chain invocations.

## Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

With an incentivation framework, we can ensure permissionless delivery of cross domain messages by any relayer. Very similar to solvers fulfilling cross chain [intents](https://www.erc7683.org/) that are retroactively paid by the settlement system.

This settlement system is built outside of the core protocol contracts, with this iteration serving as a functional MVP to start from. Any cut scope significantly simplifies implementation and can be re-added as improvements in further versions.

## Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

In order to pay relayers for delivering cross chain messages, relayers need to reimburse themselves for gas used during delivery. Since cross domain messages can span many hops, i.e A->B->C or A->B->A, this settlement system must ensure all costs are paid by the same fee payer. Thus all callbacks and transitive messages are implicitly incentivized by the originating transaction.

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

An initial invariant that works well in establishing the fee payer is the `tx.origin` of the originating transaction. This is the same invariant that holds for 4337 and 7702 sponsored transaction.

The changes proposed in [#266](https://github.com/ethereum-optimism/design-docs/pull/266) & [#282](https://github.com/ethereum-optimism/design-docs/pull/282) provide the foundation for creating the first iteration of this settlement system -- without enshrinment in the core protocol contracts. The `RelayedMessageGasReceipt` includes contextual information on the gas consumed, and the propogated (`tx.origin`, `rootMessageHash`) upon nested cross domain messages that can be used to appropriately charge `tx.origin`.

This settlement system is a permissionless CREATE2 deployment, `L2ToL2CrossDomainGasTank`, where tx senders can hold an ETH deposit, used to asynchronously pay relayers for delivered messages.

```solidity
contract L2ToL2CrossDomainGasTank {
    uint256 constant MAX_DEPOSIT = 0.01 ether;

    event Deposit(address depositor, uint256 amount);

    mapping(address => uint256) public balanceOf;

    function deposit() nonReentrant external payable {
        uint256 amount = msg.value;
        require(amount > 0);

        uint256 newBalance = balanceOf[msg.sender] + amount;
        require(newBalance < MAX_DEPOSIT);

        balanceOf[msg.sender] = newBalance;
        emit Deposit(msg.sender, amount)
    }
}
```

We cap the deposits in order to cut scope in supporting withdrawals. This makes our first iteration super simple since as withdrawal support introduces a race between deposited funds and a relayer compensating themselves for message delivery. With a relatively low max deposit, we eliminate the risk of a large amount of stuck funds for a given account whilst the amount being sufficient enough to cover hundreds of sub-cent transactions.

Relayers that deliver messages can compensate themselves by pushing through the `RelayedMessageGasReceipt` event, validated with Superchain interop. A tx sender with no deposit makes this feature a no-op as incentive exists. The user must either relay messages themselves or rely on a 3rdparty provider.

```solidity
contract L2ToL2CrossDomainGasTank {

    event Claimed(bytes32 msgHash, address relayer, uint256 amount);

    mapping(bytes32 => bool) claimed;

    function claim(Identifier calldata id, bytes calldata payload) nonReentrant external {
        require(id.origin == address(messenger));
        ICrossL2Inbox(Predeploys.CROSS_L2_INBOX).validateMessage(id, keccak256(payload));

        // parse the receipt
        require(payload[:32] == RelayedMessageGasReceipt.selector);
        (bytes32 msgHash, bytes32 rootMsgHash, address relayer, address txOrigin, uint256 relayCost) = _decode(payload);

        // ensure the original outbound message was sent from this chain
        require(messenger.sentMessages[rootMsgHash]);

        // ensure unclaimed, and mark the claim
        require(!claimed[msgHash]);
        claimed[msgHash] = true

        // compute total cost (adding the overhead of this claim)
        uint256 claimCost = CLAIM_OVERHEAD * block.basefee;
        uint256 cost = relayCost + claimCost;
        require(balanceOf[txOrigin] >= cost);

        // compensate the relayer (whatever is left if there is not enough)
        uint256 amount = SafeMath.min(balanceOf[txOrigin], cost)
        balanceOf[txOrigin] -= amount;

        new SafeSend{ value: amount }(payable(relayer));

        emit Claimed(msgHash, relayer, cost)
    }
}
```

As relayers deliver messages, they can claim the cost against the original tx sender's deposit. This design introduces some off-chain complexity for the relayer.

1. The relayer must simulate the call and ensure the emitted tx origin has a deposit that will cover the cost on the originating chain.
2. The relayer should also track the pending claimable `RelayedMessageGasReceipt` of the `rootMsgHash` callstack to maximize the likelihood that the held deposit is sufficient.
3. The relayer should claim the receipt within a reasonable time frame to ensure they are compensated.
   - An unrelated relayer not checking (2) might claim newer transitive message that depletes the funds. A version of the gas tank where claims are ordered would fix this.
   - There are restrictions for the max age of any event pulled in with interop -- applying to claim flow with `RelayedMessageGasReceipt`

### Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

As a CREATE2 deployment, this settlement framework does not affect the core protocol contracts. All contract operations are on fixed-sized fields bounding the gas consumption of this contract.

### Single Point of Failure and Multi Client Considerations

<!-- Details on how this change will impact multiple clients. Do we need to plan for changes to both op-geth and op-reth? -->

No external contract calls are made during the deposit and claim pathways, with the `nonReentrant` modifier also applied to protect against re-entrancy attacks as ETH is transferred between the gas tank and the caller.

## Failure Mode Analysis

<!-- Link to the failure mode analysis document, created from the fma-template.md file. -->

_pending_.

Important to note that holding no deposit is makes this feature a no-op. Users leveraging 3rdparty relayers or different infrastructure can continue to do so without affect. The failure mode here is simply an unused gas tank. The `RelayedMessageGasReceipt` emmitted by the messenger can be ignored or consumed by a different party for their own purposes.

## Impact on Developer Experience

<!-- Does this proposed design change the way application developers interact with the protocol?
Will any Superchain developer tools (like Supersim, templates, etc.) break as a result of this change? -->

When sending a transaction that involves cross chain interactions, the frontend should simulate these interactions and ensure the gas tank has appropriate funds to pay. With `multicall`, a single transaction can bundle together the funding operation with the transaction if neededed.

The infrastructure required for make cross chain tx simulation as simple as possible must also be taken into consideration. Since a cross chain tx requires a valid `CrossL2Inbox.Identifier`, there's already added complexity here in simulating side effects where dependencies have not yet been executed. However, frontends can liberally make deposits without full simulation as the settlement system only charges what was used.

## Alternatives Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

1. Rely on 3rdparty Bridge/Relay providers. See the problem statement in [#266](https://github.com/ethereum-optimism/design-docs/pull/266).
2. Offchain attribution. Schemes can be derived with web2-esque approaches such as API keys. With a special tx-submission endpoint, being able to tag cross chain messages with off-chain accounts to thus charge for gas used. This might a good fallback mechanism to have in place.

### Permissioned Gas Tank

We can create an intermediate version of the gas tank to build the entire flow end to end. The deployment of this gas tank is scoped to specific relayers which can unliterally charge depositors without needing a validated receipt. By having the same user-facing deposit api, we can get to a faster implementation to test any UX flows ahead of time.

```solidity
contract PermissionedL2ToL2CrossDomainGasTank {
    constructor(address[] relayers) {}

    function deposit() payable;
    function claim(address txSender, uint256 amount) onlyRelayers {}
}
```

## Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

1. This incentive framework is insufficient and unused. This is not a problem as this settlement framework is not enshrined in the protocol and is a simple CREATE2 deployment. Entirely new frameworks can be derived to replace this, leaving this unused.

2. Also important to note that holding no deposit is makes this feature a no-op. Users leveraging 3rdparty relayers or different infrastructure can continue to do so without affect.
