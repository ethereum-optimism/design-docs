# Proposal to reorganize OP Stack contracts folder structure

## Context

Currently, the OP Stack contracts are organized across multiple top level folders:

- `cannon/`: Fault proof VM implementation and single instruction execution contracts
- `dispute/`: Fault dispute game contracts for challenging invalid blocks and output root
- `governance/`: Protocol governance contracts
- `L1/`: Contracts deployed on L1
- `L2/`: Contracts deployed on L2
- `legacy/`: Deprecated contracts
- `libraries/`: Shared library contracts
- `periphery/`: Non core auxiliary contracts
- `safe/`: Safe related contracts and modules
- `universal/`: Contracts used on both L1 and L2
- `vendors/`: Third-party developed contracts

## Problem Statement

The current organization has several drawbacks:

1. Too many top-level folders make navigation difficult
2. Folder names don't always reflect contract purposes:
   - Cannon and Dispute contracts run on L1 but live in separate folders
   - Governance contracts run on L2 but are in a governance folder
   - Safe contracts are used on both L1/L2 but have their own folder
   - Legacy and periphery contracts have their own folders and files are flattened rather than organized by layer

## Proposed Solution

We propose reorganizing contracts based on their deployment layer:

1. Move layer specific contracts to their respective folders:

   - `cannon/` → `L1/cannon/`
   - `dispute/` → `L1/dispute/`
   - `governance/` → `L2/governance/`

2. Organize shared contracts:
   - `safe/` → `universal/safe/`
   - Split `legacy/` into `L1/legacy/`, `L2/legacy/`, `universal/legacy/`
   - Split `periphery/` into `L1/periphery/`, `L2/periphery/`, `universal/periphery/`

Afterwards, the `src/` folder will be organized as such:

- `L1/`
- `L2/`
- `universal/`
- `vendors/`

This restructuring will:

- Make contract purposes more immediately apparent
- Simplify navigation

## Risks, Uncertainties and Considerations

- All import paths referencing these folders will need to be updated
- Documentation and tooling may need path updates
- The migration should be done atomically to avoid broken references
