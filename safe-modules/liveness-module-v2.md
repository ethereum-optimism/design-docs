
# LivenessModule V2

## Context and Problem Statement

<!-- The Context and Problem Statement section is one of the most critical parts of the design
document process. Use this section to clearly highlight the background context for the problem,
the specific issues being faced by customers, and any constraints on a solution as defined either
by customers or by technical limitations. Context and Problem Statement is an opportunity to tell
the story that helps to motivate the rest of the design document. -->

[Safe Accounts](https://safe.global) are configured as a set of owner addresses where some
threshold of addresses can sign and ultimately execute a transaction through the account. We must
solve for two safety cases, first that a threshold of signers do not maliciously execute a
transaction (safety failure) and second that a blocking minority of signers
(`total - threshold + 1`) don't become unable to sign (liveness failure). Either case can be
equally catastrophic but we tend to focus on the safety failure case. Here we attempt to produce a
solution for the liveness failure case that fits within certain constraints.

We define multisig liveness as the ability for the multisig to reach consensus and execute a
transaction within a bounded amount of time. Note that a multisig does not need to reach consensus
over every possible transaction (because the members have the ability to reject transactions as
needed). We instead say that a multisig is live as long as it is able to reach consensus over a
transaction that states "this multisig is live".

At a high level, a multisig fails to be live if at least a blocking minority of signers prevents
the multisig from being able to reach consensus. This can happen if signers are honest but unable
to sign (e.g., keys are lost) or if signers are malicious and refuse to sign. We must address both
problems if we want to avoid multisig liveness failures. The existing `LivenessGuard` is
insufficient in this regard because it does not handle malicious signers.

Any system to address the above must also scale sufficiently such that a signer is not
unnecessarily burdened by the liveness mechanism (this is a product constraint). The
`LivenessGuard` is also insufficient here because it does not scale across multiple chains and it
demands signatures on a regular basis (as opposed to only demanding signatures as needed).

## Customer Description

<!-- Provide a brief summary of the customers for this design document. -->

The customers for this design doc are any participants in the multisig accounts that would utilize
this module, as well as any 3rd parties who rely on the security properties of these accounts.

### Customer Reviewers

<!-- Identify at least one customer who should be involved in the review of this document. -->

- Coinbase (potentially impacted 3rd party)
- Uniswap (`LivenessModule` user)
- Security Council (`LivenessModule` user)
- Optimism Foundation

## Requirements and Constraints

<!-- Identify the solution requirements and any additional design constraints from the Context and
Problem Statement section in a bulleted list. -->

- Solve both the owner liveness and account liveness problems *securely*.
- Keep code as minimal as possible to avoid complex logic in Safe modules.
- Reduce mental overhead for Safe owners and multisig leads.

## Proposed Solution

<!-- Explain the solution that you believe best addresses the problem described above. -->

We propose the introduction of a new module, `LivenessModuleV2` (name is unimportant) to replace
the existing `LivenessGuard`/`LivenessModule`. `LivenessModuleV2` would employ a challenge/response
mechanism as an alternative to the existing poke mechanism inside of `LivenessGuard`.

### Singleton

`LivenessModuleV2` is a singleton that exists at a known address on all chains and has no
constructor parameters. Any Safe on any network can choose to configure the module and then enable
the module within the Safe.

If possible, this module should be integrated with all other safe modules and guards in a single
contract that is enabled and configured in all Optimism Safes.

### Configuration

A Safe configures the module by setting a challenge period and a fallback owner.

### Challenge Protocol

The fallback owner may trigger an Account Liveness Challenge by targeting a specific Safe address.
In response to this challenge, a threshold of owners (same threshold as the Safe itself) must 
execute a regular transaction that confirms the liveness of the safe. If the Safe fails to execute
this transaction, all owners are removed from the Safe and ownership is transferred to the fallback
account. If the Safe successfully executes the transaction, the challenge fails and nothing happens.

The Account Liveness Challenge addresses both of the ways in which a Safe can fail to be live. If
sufficiently many honest users cannot sign that the account cannot reach a threshold, the account
falls back to the fallback owner. If sufficiently many users are malicious that the account cannot
reach a threshold, the honest users can refuse to sign and force a fallback.

## Alternatives Considered

<!-- Describe any alternatives that were considered during the development of this design. Explain
why the alternative designs were ultimately not chosen and where they failed to meet the product
requirements. -->

### Owner Liveness Challenge

We previously considered a model that would allow users to challenge either the entire Safe (as
happens in the Account Liveness Challenge) or an individual owner (i.e., an Owner Liveness
Challenge). An Owner Liveness Challenge would be more similar to the behavior of the existing
`LivenessModule`, where a user could target a specific owner and require that owner respond or be
removed from the Safe.

The Owner Liveness Challenge faces a number of problems:

- It creates a second challenge path.
- It only solves the honest user case, whereas the Account Liveness Challenge solves both cases.
- It requires that the Safe configure various new parameters like minimum owner count and threshold
  percentage which creates more surface area for potential issues.

Additionally, in most cases, the Owner Liveness Challenge would be unused because the Safe would
prefer to simply remove an owner that is unable to sign.

## Risks and Uncertainties

<!-- Explain any risks and uncertainties that this design includes. Highlight aspects of the design
that remain unclear and any potential issues we may face down the line. -->
### Malicious Fallback
While the fallback owner is chosen as a trusted actor, it can always happen that they are
eventually compromised. In that case, they might issue a liveness challenge and if unnoticed
gain control of the multisig enabling the LivenessModule.

To mitigate this, we will require monitoring that will alert us of unexpected liveness challenges.

### Blocking Minority is Below Quorum
In order for the module to be effective for the stated scenario, a multisig must require at least a simple majority (defined as `threshold > (total + 1 )/ 2`)  as quorum, otherwise malicious actors will reach quorum before reaching a blocking minority.

Instead of implementing a module that would address this scenario, we accept that this module
is intended for multisigs where the quorum is greater than a simple majority.

### Fallback Ownership

Both the existing `LivenessModule` and this updated iteration rely on fallback ownership. In the
case that the challenge fails, ownership of the account must be transferred to some fallback
address. We believe this is acceptable and reasonable. Most multisig accounts can simply name the
Optimism Foundation multisig as a fallback, and the Optimism Foundation multisig can name the
Security Council as a fallback.

One open question is what the Security Council should set as a fallback. We'd likely want this to
be another multisig operated by an independent and well-trusted entity.

### Blunt Instrument

It should be noted that the updated `LivenessModule` is more of a blunt instrument than the
existing module. We no longer have the ability to remove specific owners, we instead trigger a
fallback immediately if a threshold cannot be reached. This is less surgical, but we believe it is
equally effective and ultimately safer/simpler than the existing module because users don't need to
account for as many parameters.
