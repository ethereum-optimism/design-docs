# TimelockGuard

## Context and Problem Statement

<!-- The Context and Problem Statement section is one of the most critical parts of the design
document process. Use this section to clearly highlight the background context for the problem,
the specific issues being faced by customers, and any constraints on a solution as defined either
by customers or by technical limitations. Context and Problem Statement is an opportunity to tell
the story that helps to motivate the rest of the design document. -->

[Safe Accounts](https://safe.global) are configured as a set of owner addresses where some
threshold of addresses can sign and ultimately execute a transaction through the account. One
common concern with Safe accounts is that, by default, transactions execute immediately once signed
without any type of delay.

A mandatory transaction delay is a significant improvement in the security model for Safe accounts
because it provides ample time for review and entirely mitigates the impact of compromised signing
machines. It would be immensely valuable to OP Labs and Optimism Foundation multisig operations if
all multisigs managed by these entities were to require mandatory delays. We want to build a
component that we can fit into our existing multisigs that provides this delay with minimal
configuration and minimal overhead for both signers and multisig leads.

## Customer Description

<!-- Provide a brief summary of the customers for this design document. -->

The customers for this design doc are any participants in the multisig accounts that would utilize
this guard, as well as any 3rd parties who rely on the security properties of these accounts.

### Customer Reviewers

<!-- Identify at least one customer who should be involved in the review of this document. -->

- Optimism Foundation Finance
- OP Labs Finance

## Requirements and Constraints

<!-- Identify the solution requirements and any additional design constraints from the Context and
Problem Statement section in a bulleted list. -->

- Mitigate the impact of a malicious majority of owners capable to execute a single transaction.
- Mitigate the impact of an honest majority approving a malicious or erroneous transaction.
- Keep code as minimal as possible to avoid complex logic in Safe modules/guards.
- Limit mental overhead for Safe owners and multisig leads, don't break workflows if possible.
- Apply cleanly for all of the major multisigs.
- Support nonceless execution, which we are planning to implement for operational reasons.
- The designed solution should be aplicable to single and arbitrarily nested Safe configurations.

## Proposed Solution

<!-- Explain the solution that you believe best addresses the problem described above. -->

We propose the introduction of a new Safe guard, the `TimelockGuard`. The `TimelockGuard` would
take advantage of functionality in the latest version of Safe accounts to enforce a mandatory
timelock with minimal effort on the part of signers and a mostly unchanged workflow for multisig
leads.

### Singleton

`TimelockGuard` is a singleton that exists at a known address on all chains and has no constructor
parameters. Any Safe on any network can choose to configure the Guard and then enable the Guard
within the Safe.

It would be convenient if we can have one singleton timelock + liveness contract that we install
as both a guard and module, and can install the exact same contract address on any safe and just
configure it during installation. To be explored during implementation.

### Configuration

A transaction from the Safe configures the Guard by executing a transaction that optionally sets
a delay period. If the delay period is not set, the TimelockGuard is enabled with no delay.

### Transaction Execution Flow

1. Users sign the transaction normally, either via the UI or via `superchain-ops`.
2. Anyone can collect the signatures and send the transaction calldata along with the signatures
  to `TimelockGuard.schedule`, instead of executing via `Safe.execTransaction`. This function has
  the exact same inputs as `execTransaction` and is therefore compatible with both the standard
  Safe and the proposed nonceless execution module.
3. `TimelockGuard.schedule` verifies the signatures attached to the transaction and starts a timer
  for execution of the transaction.
4. After the specified time, anyone can then call `Safe.execTransaction` as normal and execution
  will complete as expected.

### Replayability Of Transactions
Each scheduled transaction would be identified by a hash derived from the transaction parameters,
which already include a nonce for replayability protection. Cancelled transactions will be marked
as cancelled so that they can't be scheduled again with the same signatures.

### Storage Of Scheduled Transactions
Scheduled transactions will be stored with as many of its parameters as possible, limited by the
gas required to execute the most complex expected transactions withi projected block gas limits.
Rich transaction data helps with identifying the effects of a transaction, in particular to
external operators such as bug bounty hunters that might lack the tools or the context from
internal operators.

### Transaction Cancellation Flow
To cancel a scheduled transaction, a `cancellation_threshold` number of owners would need to
signal that they agree with the cancellation. Cancelled transactions would need to be signed again
to be scheduled again.

In nested safe setups, the owners of a child multisig would need to signal the rejection of the
transaction in numbers no less than the `cancellation_threshold` of the child multisig, for the
child multisig to signal rejection of the transaction to the parent multisig.

#### Dynamic `cancellation_threshold`
A `cancellation_threshold` of 1 is convenient for usability as it requires the least
number of owners to signal their agreement to cancel a transaction in an emergency situation.

However, a `cancellation_threshold` that is set too low can also be abused by malicious owners to
permanently block the multisig from executing transactions, inducing a liveness failure.

A dynamic `cancellation_threshold` is chosen as a best compromise between security and usability.
The `cancellation_threshold` would start at 1 and increase by 1 with each successful consecutive
cancellation. The `cancellation_threshold` would reset to 1 with the successful execution of an
scheduled transaction.

The upper boundary to the `cancellation_threshold` would be the "blocking minority", defined as
the minimum number of `owners` that can stop the multisig from approving transactions, which is
`min(quorum, total_owners - quorum + 1)`.

### Interaction with the LivenessModule
The LivenessModule will execute an ownership change through the safe if a challenge is successful,
this ownership challenge should not be cancellable, so it shouldn't be subject to the scheduling
flow.

When using a LivenessModule, the liveness challenges are responded with a regular transaction which
is subject to the timelock delay, and as such the LivenessModule needs to have a challenge response
time that allows for both assembling a quorum of signers and executed a delayed transaction. An
integrated LivenessModule and TimelockGuard is a viable option.

## Alternatives Considered
<!-- Describe any alternatives that were considered during the development of this design. Explain
why the alternative designs were ultimately not chosen and where they failed to meet the product
requirements. -->

Other potential designs that were considered make trade-offs for security or usability that are
suboptimal. Examples of this are:

- Allowing any user to reveal a transaction for later execution, which creates more considerations
  for monitoring/offchain infrastructure and changes the transaction flow.
- Having multisigs queue transaction via the timelock directly and then allowing anyone to execute
  the transaction via a module, which changes the signing flow significantly and means transactions
  are challenging to simulate properly.
- Configurable `cancellation_thresholds` were considered, but the risk of misconfiguration over
  time was expected to be considerable.
- A fixed `cancellation_threshold` of 1 was considered, but the risk of operational disruption by
  malicious actors was considered unacceptable.

We could not find any alternative design with the simplicity and security of the one that was
ultimately presented in this document.

## Risks and Uncertainties

<!-- Explain any risks and uncertainties that this design includes. Highlight aspects of the design
that remain unclear and any potential issues we may face down the line. -->

### Timelock Overhead
Any type of mandatory timelock introduces overhead for multisigs that want to execute transactions
in a timely fashion. Multisig leads will have to plan ahead of time in a way that they were not
previously required to do.

### Time Critical Transactions
Some multisigs may face scenarios where they need to be able to execute transactions very quickly
in a worst-case scenario. We should do our best to avoid these situations as much as possible and
use other protocol features (like the Pause) to mitigate them entirely.

Instant execution is not required for finance multisigs. The delay will however be no longer than
the time required for a transaction review by trained finance staff.
