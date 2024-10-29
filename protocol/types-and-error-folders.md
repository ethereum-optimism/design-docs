# Proposal to move types and errors files to dedicated folders

## Context

The OP Stack contracts in some instances define errors and custom types in a separate file from the contract(s) that use them. This is common among contracts that are part of a common folder e.g L1/, L2/ etc which might have a common error or/and types file where they import the needed errors and types from. This has the advantage of re-usage of errors and types across contracts without duplicating code, ease of modifiability and extensibility and makes it easy to compare errors and types used by different contracts.

## Problem Statement

These error and types files are most times located in a libraries or lib folder e.g `libraries/` `L1/libraries/`, `L2/libraries/` but these are not actual libraries and so do not belong in the libraries folder. This can also make it hard to find at first, especially to a developer newly exploring the codebase.

## Proposed Solution

The proposed solution is to move these error and types files to dedicated folders e.g `L1/errors/`, `L1/types/`, `L2/errors/`, `L2/types/` etc.

## Alternatives Considered

No alternatives considered.

## Risks, Uncertainties and Considerations

### Consideration

- Should the dedicated folders be located at the `src/` level or within the same folder as the contracts that use them e.g `L1/`?
