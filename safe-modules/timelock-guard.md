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

A Safe configures the Guard by setting two delay periods, one that applies when a threshold is
reached and a second that applies if all members of the multisig sign the transaction.

### Transaction Flow

1. Users sign the transaction normally, either via the UI or via `superchain-ops`.
2. Anyone can collect the signatures and send the transaction along with the signatures to
  `TimelockGuard.schedule`. This function has the exact same inputs as `execTransaction` and is
  therefore compatible with both the standard Safe and the proposed nonceless execution module.
3. `TimelockGuard.schedule` verifies the signatures attached to the transaction and starts a timer
  for execution of the transaction. If a threshold of signers have signed the timer is set to one
  value, if all signers have signed the timer is set to some second (presumably shorter) value.
4. After the specified time, anyone can then call `Safe.execTransaction` as normal and execution
  will complete as expected.

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

### Critical Transactions

Some multisigs may face scenarios where they need to be able to execute transactions very quickly
in a worst-case scenario. We should do our best to avoid these situations as much as possible and
use other protocol features (like the Pause) to mitigate them entirely. We need to confirm with
users of finance multisigs that this is not expected to be common. We should avoid instant
execution at all costs, even if a signature from all owners would reduce the time to, say, 1 hour.
