# Purpose

To define a set of key principals to guide the evolution of the OP Stack release process.

# Summary

*Most (if not all) documents should have a summary. While the length will likely be proportional to the length of the
full document, the summary should be as succinct as possible.*

# Problem Statement + Context

The current OP Stack release process is ill-defined, paradoxical, error-prone, highly manual and ad-hoc.

ill-defined - ironically there are so many documents describing the release process (at least 6 different ones just with
a quick search, excluding those marked as outdated), that its difficult to know what the right release process to follow
is. Additionally, we have separate release processes for Go components like op-node and L1 contracts even though network
upgrades require coordinated releases of both.

error-prone -

superchain registry updates always get missed. superchain-registry tests are incompatible with the release process (
testnets are upgraded prior to governance but they are in the registry as standard chains which must use governance
approved contracts).

Contract releases only include some contracts, not all of the source code at that version.

Things that can easily go wrong in our release process (mostly based on them going wrong and thankfully being picked up
before it was too late):

* Forgetting to update absolute prestate for fault dispute games
* Schedule a different time in superchain-registry to what's in the governance post
* Not leaving sufficient time between veto period ending and upgrade activating to get security council signatures for
  contracts update
* Forgetting to add the -beta suffix to contract versions during development
* Not changing contract version suffix to -rc when cutting an rc (this also conflicts with the idea of tagging an
  existing commit as an RC)
* There's no actual release of the set of governance approved changes - has to be pieced together by finding the last
  release each contract was actually a part of
* superchain-ops script to execute the upgrade needs to be carefully manually reviewed. Easy to use the incorrect
  address for a deployed contract or role etc.
* Easy to miss describing some changes in the governance post
* Forgetting to release and deploy an updated op-challenger to work with latest contract changes

And our release process causes problems in our development flow:

* Have to sequence changes to contracts carefully as some changes block other features from being released if they
  change the same contracts

# Proposed Solution

These are principles to work towards, not things we will achieve in the short term. It's quite possible we won't ever
fully achieve them, but they are still useful to provide direction and guidance when making decisions about the release
process as we improve it step by step.

## One Release Process To Rule Them All

We need to have a single release process that includes both op-node and op-contracts. Some steps may be optional if not
everything is being released at once, but it should handle the case where everything is released at once (e.g. for a
network upgrade).

It is important for the process to scale up and down depending on what actually needs to be in the release. For example,
our most common release is just for op-node, op-batcher and op-proposer so that should be at least as efficient as it is
today. But a release like Granite which involved a network upgrade plus L1 contract changes (and multiple OP Stack
releases to support that) needs to be just as routine and reliable even if it is slower due to governance and extra
coordination/comms.

## Runs At Least Every Day

Any process that isn't run regularly is likely to be broken or forgotten by the time we next come to use it. To avoid
this we should ensure our release process is tested just like the rest of our code and actually run regularly as part of
CI.

Achievable: Upgrade scripts for contracts are built as changes are made and used as part of devnet tests in CI

Ambitious: Fully automated nightly releases that performs the entire release process and deploys to a devnet and creates
a new devnet from scratch

## Develop is Always Releasable

We should be able to create a new release whenever it's required without having carefully coordinate when different
changes are merged. We should use the same release process for urgent security critical releases as for planned
releases. If urgent releases use an ad-hoc process, we increase the risk of mistakes at the exact time that we're most
under pressure and can least afford mistakes.

Note that this doesn't mean that all the code merged to develop gets included and activated in a release. We will often
merge new work that isn't yet complete or requires audits or governance approval before being ready to release. The
release process needs to accommodate that and have a way to only include/activate the ready-to-release code.

## Manual Steps are a Smell

The release process should be as automated as possible. There will be manual reviews and approvals required in the
pipeline (e.g. governance) but those should be a point where the automation pauses and waits for approval before
continuing with automation.

### Do-nothing Scripts

While we don't have full automation for the release process, we should use
the [do-nothing script pattern](https://blog.danslimmon.com/2019/07/15/do-nothing-scripting-the-key-to-gradual-automation/).
Rather than yet another document describing our release process, create a system that steps through the release process.
When a step can be automated, it runs automatically, otherwise the instructions for how to manually perform the step is
provided.

## Clear Visibility

Our release process is unavoidably slow because of the need for governance and coordination with other chain operators.
We need everyone to be able to easily see where we're up to in the release process, if it's still on track, what
needs to be done next and when that needs to be done by.

## Deployment is Part of the Release

Software is not released until it is actually running in production. For the OP Stack, that means upgrades on multiple
chains and being able to create new chains with the latest release. This implies that every release must be suitable for
use on all supported chains, including creating new chains. We will need to be clear about what we directly support.

This also implies that any tools required to perform deployment need to be included in the release or have specifically
supported versions that are tested with this release.
