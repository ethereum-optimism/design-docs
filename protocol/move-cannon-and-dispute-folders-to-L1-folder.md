# Proposal to move Cannon and Dispute folders to L1 folder

## Context

Currently the Cannon and Dispute contracts of the OP Stack are located in their own folders: `L2/cannon/` and `L2/dispute/` respectively. These contracts are critical components of the OP Stack's fault proof system:

- The Cannon contracts holds the fault proof VM implementation and related contracts and libraries that execute a single instruction after bisecting over the L2 execution trace.
- The Dispute contracts holds the fault dispute game implementation and related contracts and libraries that allows parties to dispute/defend invalid state transitions.

## Problem Statement

The Cannon and Dispute contracts are L1 contracts that operate on L1 to verify L2 execution. However, they are currently located in their own folders which does not accurately reflect their role in the system. This might lead to confusion about their purpose.

## Proposed Solution

There are 3 possible solutions to this problem:

1. Moving just the Cannon and Dispute contract `files` to the `L1/` folder:
   - Pros: The files are in an appropriate folder
   - Cons: Loses folder organization benefits. The L1 folder currently has 16 files, flattening cannon and dispute files into it will make these files harder to find.
2. Moving the entire Cannon and Dispute `folders` to the `L1/` folder:
   - Pros:
     - The files are in an appropriate folder
     - Folder structure is preserved
3. Leave everything as is and make no changes:
   - Pros: No migration effort needed
   - Cons: Continues architectural confusion and technical debt

The recommended approach is option 2 - moving the complete folders. This provides a clearer organization while preserving the internal structure.

## Risks, Uncertainties and Considerations

- All import paths referencing these folders will need to be updated
- Documentation and tooling may need path updates
- The migration should be done atomically to avoid broken references
