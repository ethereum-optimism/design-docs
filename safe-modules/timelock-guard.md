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

- Optimism Foundation (`LivenessModule` user)
- Optimism Security Council (`LivenessModule` user)

## Requirements and Constraints

<!-- Identify the solution requirements and any additional design constraints from the Context and
Problem Statement section in a bulleted list. -->

- Enforce a mandatory delay on multisig transactions.
- Keep code as minimal as possible to avoid complex logic in Safe modules/guards.
- Limit mental overhead for Safe owners and multisig leads, don't break workflows if possible.
- Apply cleanly for all of the major multisigs.
- Do not use sequential nonces, which we are planning to deprecate for operational reasons.

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

### Configuration

A Safe configures the Guard by setting a delay period which can be zero.

### Transaction Execution Flow

1. Users sign the transaction normally, either via the UI or via `superchain-ops`.
2. Anyone can collect the signatures and send the transaction along with the signatures to
  `TimelockGuard.schedule`. This function has the exact same inputs as `execTransaction` and is
  therefore compatible with both the standard Safe and the proposed nonceless execution module.
3. `TimelockGuard.schedule` verifies the signatures attached to the transaction and starts a timer
  for execution of the transaction. If a threshold of signers have signed the timer is set to one
  value, if all signers have signed the timer is set to some second (presumably shorter) value.
4. After the specified time, anyone can then call `Safe.execTransaction` as normal and execution
  will complete as expected.

### Replayability Of Transactions
Each scheduled transaction would be identified by a hash derived from the transaction parameters
plus a user-defined `salt`. This would allow to uniquely identify scheduled transactions while
allowing to schedule identical transaction calls more than once, different only in the time that 
they will execute.

### Storage Of Scheduled Transactions
Scheduled transactions will be stored with as much data as possible, limited by projected gas block
limits and gas required to execute the most complex expected transactions. Rich transaction data
helps with identifying the effects of a transaction, in particular to external operators such as
bug bounty huntesr that might lack the tools or the context from internal operators.

### Transaction Cancellation Flow
To cancel a scheduled transaction, a `cancellation_threshold` number of owners would need to signal
that they agree with it. Cancelled transactions never will be executable, unless rescheduled with a
different `salt` and fresh signatures.

To describe the cancellation flow, we will consider first a single multisig, and then an
arbitrarily nested multisig.

#### Single Multisig Cancellation FLow
Members of the Safe can propose to cancel by calling `reject(txHash)`.
    1. EOAs can call this function directly.
    2. Smart contracts can call this function directly.

Anyone can actually cancel the transaction by calling `cancel(txHash, owners[])` where `owners[]`
is the list of owners that called `reject`. TimelockGuard will then verify that each owner did, in
fact, call `reject` and that the number of owners that called `reject` is greater then or equal to
the `cancellation_threshold`.

#### Nested Multisig Cancellation Flow
In a nested group of Safes, some Safes are owners of other Safes, building a tree with a single
Safe at the root. 

To allow for cancelling of scheduled transactions a nested Safe setup with an arbitrary depth,
owners of any Safe still call `reject(txHash)` for a transaction hash scheduled within the
TimelockGuard, which is common for all the Safes in the nested group.

A new `cancel(SafeA, txHash, owners[])` allows to specify the executing safe (`SafeA`) and the list
of owners that signalled rejection of the scheduled transaction. Upon receiving this call, `SafeA`
will compare the list of signalling owners with its own owner members, and will request from any
owner members that are also Safes to recursively confirm rejection from their owners using the
unmatched owners from the original provided array. The process will continue until the
`cancellation_threshold` is met at the executing Safe, or all the originally provided owners have
been matched up without reaching the `cancellation_threshold` at the executing Safe.

#### Choosing a `cancellation_threshold`
Choosing an appropriate `cancellation_threshold` is not obvious, and needs to be considered
according to several factors.
1. Stage 1 requires that the compromise of a 75% or less of the Security Council can't be enough
to cause a safety-level incident. If the Security Council would be turned into a reactive multisig
with cancellation duties instead of approval duties, a compromise of 
`total_owners - cancellation_threshold + 1` SC owners would be enough to remove the ability of the
SC to cancel malicious transactions. For this reason, for the Security Council multisig,
`cancellation_threshold =< 0.25 * total_owners`
2. We define a "blocking minority" as the minimum number of `owners` that can stop the multisig
from approving transactions, which is `total_owners - quorum + 1`. If the `cancellation_threshold`
would be less than a blocking minority then it becomes the new blocking minority as it can stop any
transactions (including modifying ownership, or responding to liveness challenges) from execution.
A `cancellation_threshold` that is set too low can be abused in this way by malicious owners to
permanently block the multisig from executing transactions if a LivenessModule hasn't been
installed.

Therefore:
It's recommended that the LivenessModule is installed in all multisigs, otherwise the
`cancellation_threshold` should not be lower than the "blocking minority".

The `cancellation_threshold` should be high enough to limit having to use the liveness challenge
too frequently and disrupt operations, if that is a concern.

For the Security Council, the `cancellation_threshold` should be lower than `0.25 * total_owners`,
or 3.25 with the current 10/13 membership. That leaves an optimal `cancellation_threshold` of 3.

## Alternatives Considered

<!-- Describe any alternatives that were considered during the development of this design. Explain
why the alternative designs were ultimately not chosen and where they failed to meet the product
requirements. -->

Other potential designs here make trade-offs for security or usability that are undesireable enough
to not really be worth considering. Examples of this are:

- Allowing any user to reveal a transaction for later execution, which creates more considerations
  for monitoring/offchain infrastructure and changes the transaction flow.
- Having multisigs queue transaction via the timelock directly and then allowing anyone to execute
  the transaction via a module, which changes the signing flow significantly and means transactions
  are challenging to simulate properly.

We could not find any alternative design with the simplicity and security of the one that was
ultimately presented in this document.

## Risks and Uncertainties

<!-- Explain any risks and uncertainties that this design includes. Highlight aspects of the design
that remain unclear and any potential issues we may face down the line. -->

### Timelock Overhead
Any type of mandatory timelock introduces overhead for multisigs that want to execute transactions
in a timely fashion. Multisig leads will have to plan ahead of time in a way that they were not
previously required to do. We can mitigate some of this by allowing the full threshold to execute
faster, but this will be difficult to coordinate regularly and leads should not expect this to be
a norm.

### Time Critical Transactions
Some multisigs may face scenarios where they need to be able to execute transactions very quickly
in a worst-case scenario. We should do our best to avoid these situations as much as possible and
use other protocol features (like the Pause) to mitigate them entirely. We need to confirm with
users of finance multisigs that this is not expected to be common. We should avoid instant
execution at all costs, even if a signature from all owners would reduce the time to, say, 1 hour.

### Interaction with the LivenessModule
The `cancellation_threshold` becomes the new `blocking_minority`, and as such it can force not
meeting a liveness challenge. Given that such challenges should be issued by the trusted fallback
owner, this is not a concern.

### Interaction with the UnorderedExecutionModule
To avoid issues with a future implementation of an UnorderedExecutionModule, the scheduling of
transactions should not include their nonce, while still maintaining the principle that cancelled
transactions can't be executed again with the same signatures, but can be executed again with fresh
signatures.
