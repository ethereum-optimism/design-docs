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

Another concern specific to the Security Council multisig is that a quorum ofsigners have to review
and sign each upgrade transaction, and it is difficult to incentivise them against their competing
interests. While the Security Council should be maintained as a highly trusted body that secures
critical transactions, there is opportunity to use their time and focus more effectively.

## Customer Description

<!-- Provide a brief summary of the customers for this design document. -->

The customers for this design doc are any participants in the multisig accounts that would utilize
this module, as well as any 3rd parties who rely on the security properties of these accounts.

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

A Safe configures the Guard by setting a delay period that applies to transactions for which a
quorum is required. Optionally, a longer delay can be set that enables transactions to execute with
a single approval if there is no opposition from other sigenrs.

### Transaction Flow

`TimelockGuard` enables two execution paths, using the same function signatures but diferentiated
by the amount of signatures supplied.

#### Quorum Approval, Default Delay

1. User signs the transaction normally, either via the UI or via `superchain-ops`.
2. Anyone can collect the signatures and send them along with the transaction along to
  `TimelockGuard.schedule`. This function has the exact same inputs as `execTransaction` and is
  therefore compatible with both the standard Safe and the proposed nonceless execution module.
3. `TimelockGuard.schedule` verifies the signatures attached to the transaction and emits an event
  reporting the proposed transaction.
4. After the shorter configured delay, anyone can then call
  `Safe.execTransaction` as normal and execution will complete as expected.

#### Single Approval, Long Delay
This execution path is only enabled if a delay for it is set, and the delay is longer than the
default delay.

1. One single user signs the transaction normally, either via the UI or via `superchain-ops`.
2. Anyone can collect the signature and send it along with the transaction to
  `TimelockGuard.schedule`. This function has the exact same inputs as `execTransaction` and is
  therefore compatible with both the standard Safe and the proposed nonceless execution module.
3. `TimelockGuard.schedule` verifies the signature attached to the transaction and emits an event
  reporting the proposed transaction.
4. If there are no other actions, after the longer configured delay, anyone can then call
  `Safe.execTransaction` as normal and execution will complete as expected.
5. After scheduling but before execution, any signer can call `Safe.cancelTransaction` to make the
  transaction permanently non-executable.

## Security Considerations

### Malicious Majority, Blocking Minority, Malicious Signer
This design only mitigates instances of malicious signers that fall short of a blocking minority.
If a signer maliciously cancels a transaction with the intent of blocking it, the multisig should
submit and execute a transaction under quorum to remove the malicious signer, and then resubmit
the cancelled transaction. The resubmission can be done under quorum to avoid a long delay.

For situations in which there exists a blocking minority of malicious signers we should implement
the liveness guard. An scenario with a malicious majority can’t be mitigated as it is
indistinguishable from a non-malicious majority.


### Safety of Single-Approval transactions
A quorum of signers currently have to review each transaction. For the Security Council multisig it
is difficult to incentivise them to do this job properly against their competing interests. An
automated flow with a long delay allows OP Labs to task and adequately incentivise multiple parties
to carefully analyse each proposal before it gets executed, while safely avoiding any
man-in-the-middle attacks that would replace a verified proposal by a malicious one.

Some examples of parties that could be incentivised to verify proposals in the timelock are EVM
Safety, bounty hunters, and contracted auditors such as Spearbit or Guardian Audits.

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

### Stateful Timelock
The timelock could be implemented by keeping count of which signer approved each proposal, but an
implementation that just evaluates the signatures submitted on execution offloads that complexity
off chain without compromising in security, and offers a smaller attack surface.

### Multiple Cancellation Threshold
It was considered to require a threshold of signers to cancel a proposal, to avoid the scenario in
which a single malicious signer can block proposals. This approach would require a considerable
amount of logic. The current design allows to still execute a proposal as long as the number of
malicious signers is less than a blocking minority.

### Provide Signatures
A single-flow implementation is possible in which the proposal is first scheduled by a single
signer, and then the signatures are provided to a replacement for `Safe.execTransaction`. While
this would optimize the flow in the case of a non-blocking minority of malicious signers, it
would also significantly change the UX in the more common execution path of transactions that
require a quorum, which is the default if single-approval transactions are not enabled.

## Risks and Uncertainties

<!-- Explain any risks and uncertainties that this design includes. Highlight aspects of the design
that remain unclear and any potential issues we may face down the line. -->

### Timelock Overhead

Any type of mandatory timelock introduces overhead for multisigs that want to execute transactions
in a timely fashion. Multisig leads will have to plan ahead of time in a way that they were not
previously required to do. We can mitigate some of this by allowing the full threshold to execute
faster, but this will be difficult to coordinate regularly and leads should not expect this to be
a norm.

### Critical Transactions

Some multisigs may face scenarios where they need to be able to execute transactions very quickly
in a worst-case scenario. We should do our best to avoid these situations as much as possible and
use other protocol features (like the Pause) to mitigate them entirely. We need to confirm with
users of finance multisigs that this is not expected to be common. We should avoid instant
execution at all costs, even if a signature from all owners would reduce the time to, say, 1 hour.

### Temporary Malicious Majority - To discuss
An scenario can be assumed where a malicious party has obtained a quorum of signatures for a single
malicious transaction, but where obtaining a quorum for a second transaction is considerably more
difficult.

In that scenario, the multisig might want to cancel the malicious transaction during the default
delay period. A way of doing that is to allow a `min(blocking_minority, quorum)` of the multisig to
cancel with zero delay transactions approved by quorum. An alternative would be to allow a single
signer to cancel transactions, with the required number of cancelling signers growing by one with
each consecutive cancellation until reaching `min(blocking_minority, quorum)`.