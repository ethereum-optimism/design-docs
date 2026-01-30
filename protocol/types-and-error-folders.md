# Proposal to move types and errors files to dedicated folders

## Context

The OP Stack contracts in some instances define errors and custom types in a separate file from the contract(s) that use them. This is common among contracts that are part of a common folder e.g L1/, L2/ etc which might have a common error or/and types file where they import the needed errors and types from. This has the advantage of re-usage of errors and types across contracts without duplicating code, ease of modifiability and extensibility and makes it easy to compare errors and types used by different contracts.

## Problem Statement

These error and types files are most times located in a libraries or lib folder e.g `libraries/` `L1/libraries/`, `L2/libraries/` but these are not actual libraries and so do not belong in the libraries folder. This can also make it hard to find at first, especially to a developer newly exploring the codebase.

## Proposed Solution

The proposed solution is to move these error and types files to dedicated folders e.g `L1/errors/`, `L1/types/`, `L2/errors/`, `L2/types/` etc.

## Alternatives Considered

An alternative is to make errors follow [#143](https://github.com/ethereum-optimism/design-docs/pull/143) which proposes an error naming convention of `ContractName_ErrorName(..)`. This would mean that errors would be unique for each contracts (hence defined within them) and so we would not need dedicated folders for them. However this does not address types and so we might still need dedicated folders for them.

## Risks, Uncertainties and Considerations

### Consideration

- Should the dedicated folders be located at the `src/` level or within the same folder as the contracts that use them e.g `L1/`?
