# Problem Statement + Context

The fault proof system for interop, continues to use the split game where the bottom half of the game bisects the cannon
execution trace. However the top half of the game no longer progresses from one block to the next, but breaks down the
transition between one super root to the next into 1024 individual steps. As a result, the trace for the top half of the
game requires 2048 steps per 2 seconds compared to the pre-interop game when using 2 second block times, or 1024 times
more when using 1 second block times.

To operate correctly, the game's split depth must always be large enough for there to be enough leaf nodes in the top 
half of the game to progress from the anchor state to the current chain safe head. Otherwise the valid trace for the
top half of the game would exceed the maximum supported trace length.

The current configuration has a split depth of 30, which allows a maximum trace length of 2^31. With a 2 second block 
time that allows the anchor state to be approximately 136 years old. However, it would be infeasible for the fault proof
VM to walk back that many L1 blocks to find the batch data required for the earliest blocks within the 3.5 day chess
clock.

With the same split depth, the interop fault proof game would only support an anchor state up to 24 days old. That may
be insufficient if there was an issue in the fault proof system which required investigation and then governance
approval to fix.


# Proposed Solution

Reduce the number of steps required per super root to 128.

# Alternatives Considered

## Reduce the Number of Steps Per Super Root

The 1024 steps between each super root was chosen to align with a power of two and allow up to 1023 chains in the
dependency set before needing to change the game mechanics (the 1024th step is to check interop messages). There is
no chance of us reaching 1023 chains any time soon and a significantly reduced number could be used.

Such a high number was originally chosen to avoid the complexity of adjusting the number of steps at a hard fork, and
potentially needing to deal with games that spanned timestamps with different numbers of blocks between them. However,
the number of steps chosen doesn't actually affect the derivation process - only the dispute game. The number of steps
could simply be moved from a constant in the fault proof program and op-challenger to an input from the dispute game
contract, supplied to the challenger via a getter and via a local key to op-program. This would require some work
but is reasonably straight forward and doesn't require complex transitional handling.

If the number of steps per timestamp were reduced to 128, the current split depth would allow for a maximum anchor state
age of 194 days - just over 6 months. It is reasonable to assume that the security council could take action to
manually update the anchor state within 6 months if required. This would also support adding 127 chains to the
dependency set before the number of steps needs to be increased.

Given the consolidation step is currently done for all chains in a single fault proof VM execution, it is not feasible
to support 127 chains in the dependency without making changes to the game execution anyway.

## Increase Split Depth and Max Game Depth

The split depth could be increased to increase the maximum trace supported by the top game and increase the maximum
supported anchor state age. To keep the same maximum trace length for the bottom game, the max depth would need to be
increased by the same amount.

The [bond scaling](https://specs.optimism.io/fault-proof/stage-one/bond-incentives.html#bond-scaling) is designed such
that for any max depth, the bond required at the max depth covers the potential gas for posting a large preimage. Thus,
the bond amounts automatically scale as appropriate. However, the total bonds that would need to be paid by the honest
actor to reach the maximum depth significantly increases as the max depth increases because they need to perform more
moves and additional bonds. This decreases the capital efficiency of the fault proof system further when it is already
a weak point of the system.

## No Changes

Accept that the anchor state can not be more than 24 days old. It would be expected that the security council could fall
back to the permissioned game before this limit was actually reached but it seems unnecessarily short. In the case of
an incident it could easily restrict the available options and would be a constant consideration.
