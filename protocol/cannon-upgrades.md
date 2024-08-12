# Purpose

To define the technical approach and roll out plan for upcoming upgrades to cannon, including multi-threaded cannon and
moving to 64bit.

# Summary

`cannon` is the first fault proof VM developed and currently the only one in production. There are two major upgrades
for it in the near future, adding threading support (including enabling GC) and moving from MIPS32 to MIPS64. Since
cannon is actively used in the current fault dispute system, the upgrade process needs to be defined to ensure we can
safely roll out the changes while preserving our ability to respond to games prior to the upgrade and potentially fix
urgent issues with the existing system while the upgrades are in development.

# Problem Statement + Context

With major upgrades to cannon in progress we need to identify a roll out plan. The plan needs to account for:

* Enabling urgent fixes to the existing system which may be required while the upgrades are in development/governance
* Ensuring op-challenger can respond to games both before and after the upgrade.
  * Existing games will continue to use the old version of cannon even after the upgrade so there is a period where both
    old and new versions of cannon will be required by op-challenger simultaneously
  * op-challenger needs to be able to determine which version of cannon to use for a specific game (currently it always
    uses the latest cannon with the prestate matching the game)
* Minimising impact on users (e.g. withdrawals being invalidated when switching game types)

Work on cannon MT is well underway with multiple threads supported but further investigation required to make the GC
work. PRs have been opened with support for 64bit (without backwards compatibility) which haven't been reviewed yet.
Both upgrades need some bake-in time being run in the vm runner to ensure they work reliably.

# Alternatives Considered

There's two axis to this problem:

1. Do we ship cannon MT and 64 bit support as two separate upgrades or combine them into a single upgrade?
2. Do we maintain backwards compatibility for 32-bit cannon in the monorepo develop branch until the 64-bit rollout is
   complete?

## Separate vs Combined Upgrades

### Separate Upgrades

Separating the two upgrades potentially allows cannon MT to ship earlier as it is not blocked by work on 64-bit support.
However, it means incurring the overhead of governance and coordinating upgrades twice.

If we need to use a new game type, it would invalid user's withdrawals twice.

### Combined Upgrades

Combining both changes into a single upgrade reduces the overhead of governance and upgrade coordination. It ensures
user's withdrawals are at most invalidated once. If we maintain backwards compatibility in the monorepo, it reduces the
number of versions being supported from 3 to 2.

## Backwards Compatibility

### New Game Type

The simplest way to maintain backwards compatibility is to deploy the updated cannon VM as a new game type. Changing the
respected game type in OptimismPortal will invalidate any already proven withdrawals, impacting users.

`op-challenger` will need to be updated to select the version of cannon to use based on the game type. This is
straight-forward but involves a bunch of mostly boilerplate code.

The legacy version of `cannon` could be made available to `op-challenger` in one of two ways:

1. Retain the ability to build it in the monorepo develop branch.

* `cannon` is automatically included in the `op-challenger` docker image.
* The `--type` argument to `cannon` can be used to tell it which VM version to run

2. Remove legacy cannon from the monorepo and copy it from an existing docker image when deploying `op-challenger`.

* We will need to make other VMs like `asterisc` available to `op-challenger` in a similar way to this
* It makes it harder to fix urgent issues in the legacy cannon if required as they would have to be done on a fork
  from before legacy cannon was removed
* Has the advantage of removing the legacy code sooner and not needing to make them work in parallel

#### Change Game Type Without Invalidating Withdrawals

The impact to users could be avoided if we upgraded `OptimismPortal` to allow changing the respected game type without
also invalidating all withdrawals. This is something we would like to do to improve incident response abilities anyway
but would mean changing the security critical air-gap code. This would need to be audited, delaying deployment.

### Specify Cannon Version in Prestate

Add a flag to the cannon prestate that indicates which version of the VM is required. No new game type is used
and `op-challenger` will simply provide the appropriate prestate to cannon which then operates in the appropriate mode.
This makes the upgrade to cannon MT or 64bit work just like any other prestate upgrade.

As the game type isn't changed, users would not need to reprove their withdrawals.

This is really only feasible if we maintain support for legacy cannon in the monorepo until after the upgrade (and any
existing games are completed). The legacy code could likely shortly after chains have upgraded to the new version. At
worst chains would have to upgrade the prestate to support the next hard fork.

#### State Format

Detecting the VM type is going to be very expensive with the current JSON based snapshot format as it takes a number of
seconds to parse it and it would likely need to be parsed twice - once to find the field that defines the version and
once to parse it as that version. This could be resolved by moving to a binary state format that encodes the specific
version at the start of the file for easy access. In doing this we could also design the format to be able to be
streamed to make it more efficient to parse and serialize than the JSON version. That may be required with the move to
64bit anyway due to the increase in potential client memory size creating much larger snapshots.

Support for the existing JSON format could be retained for backwards compatibility (switching version based on the
file extension similar to how gzip is enabled by file extension).

`op-challenger` would need some updates to handle the different file formats. This would also allow a nice cleanup of
challenger to stop it importing code from cannon to parse the state. It can be made almost entirely agnostic to the
state format by executing cannon to calculate the witness hash of a snapshot rather than parsing it itself.

# Proposed Solution

While it complicates development slightly, I recommend shipping cannon MT and 64-bit together under the existing game
type. The single-threaded cannon stays as 32-bit while multi-threaded moves to 64-bit. The VM type is encoded at the
start of a new binary state format with backwards compatibility for loading JSON states.

The move to multithreaded already meant introducing a number of abstractions to share code and shows that cannon can
support multiple VM implementations at once smoothly. By shipping both versions at once we reduce the number of versions
we need to support and simplify testing.

The extra development time required to preserve backwards compatibility is likely to be offset by not being blocked on
updating OptimismPortal to allow changing game types without invalidating withdrawals. We've also seen multiple times
through fault proofs that developing in a backwards compatible way pays big dividends for testing and flexibility in
deploying. With a major project underway to upgrade chains to fault proofs we want to make it as simple as possible to
run `op-challenger` correctly and keep its deployment simple.

# Risks & Uncertainties

## Cost of Backwards Compatibility

Preserving compatibility for 32-bit and 64-bit in the same codebase will be more challenging than the single vs
multithreaded changes. It may prove too difficult to be worth it and we will need to spike out a solution. It is likely
that compatibility will be preserved by duplicating a bunch of code with it varying only slightly (`uint32`
vs `uint64`).
