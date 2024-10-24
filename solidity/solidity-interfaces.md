# Updated Solidity Interface Policy

## Context

Smart contract interfaces are useful tools that allow external tools, contracts, and tests to
interact with smart contracts using their ABIs instead of interacting with the underlying
implementation contract. Interfaces are generally more flexible than implementations because we
pin implementations to specific compiler versions whereas interfaces can have carat versions that
cover an entire Solidity major version.

Interfaces are also valuable because they create a separation between source contracts and other
contracts that need to interact with those source contracts. This separation significantly
decreases compilation time when changes need to be made to source contracts. Any changes to a
source contract would otherwise trigger a re-compilation of many other contracts that rely on that
source contract, resulting in untolerably long waits.

## Problem Statement

We recently undertook an update to the Optimism Monorepo that added interfaces for all smart
contracts. This resulted in a significant improvement to compilation time but simultaneously added
complexity for any developer that needs to introduce a new smart contract. Most of this complexity
arises because existing interface policy is non-obvious and the process for generating interfaces
is entirely manual. We would like to make certain changes to this policy to support a better
development process.

We wanted to entirely automate the interface generation process so that users do not need to
generate interfaces themselves. In exploratory work on this interface generation process, we've
found that there are a number of important edge cases that came up.

Specifically:

- Source contracts using interfaces creates a basic dependency problem. For instance, suppose that
  you'd like to add a new contract `ContractA`. You'd also like to create a new contract
  `ContractB` that depends on `ContractA`. Our current policy states that `ContractB` should use
  the interface for `ContractA`, `IContractA`. You would therefore need to generate the interface
  for `ContractA` *before* starting to work on `ContractB`, which is annoying.
- Circular dependencies in source contracts are basically impossible to resolve without manually
  crafting contract interfaces. For instance, if `ContractA` depends on `IContractB` and
  `ContractB` depends on `IContractA` but neither contract interface exists yet (new contracts)
  then the contracts cannot be compiled (because neither interface exists) and the interfaces can
  not be generated. Catch 22.
- Using interfaces within the source contracts leads to other awkward situations with function
  types that are challenging to resolve in an automated generation script.
- Contract interfaces don't have source code, so navigating the functions in the source files
  becomes awkward as each external call goes through an interface.

## Proposed Solution

All of the above seem to point to a change to our policy where contract interfaces are not used
*anywhere* in the source contracts and are only used in tests or other external/peripheral
locations that interact with the source contracts. Removing interfaces from source contracts
entirely would also be a much clearer mental model ("just don't use interfaces in src/").

In addition, interfaces would live in a new `interfaces` folder at the root of the contracts
package that has a structure that identically mirrors the `src` folder. As with the other changes,
this is meant to support automation and clearly separate interfaces from source contracts.

## Considerations

### Compilation Time

Proposed solution would increase compilation time for source contracts, but the impact would not be
nearly as significant as the original compilation time when no interfaces existed at all. We're
talking about something on the order of 5-20s instead of the current 2-10s. Not ideal but fine.

### Interface Usage

Developers must first generate contract interfaces before they can be used by things like tests or
scripts. Not a change from the status quo since this is already the case today.

### Continued Automation Challenges

Simply removing interface usage will solve the problems that come with using interfaces within the
source contracts but will not solve all of the challenges to automating interface generation. We do
have some concrete improvements like limiting complexity that comes with things like typecasting to
testing/peripheral code only and not source code.

### Future Proofing

More recent versions of Solidity are beginning to optimize for usecases like ours where the same
contracts are being used in many different places. It's possible that we will eventually not need
interfaces anywhere in the internal codebase to still maintain a solid compilation time. We don't
expect this to be true for at least another 6-12 months but it's something to consider. Making a
change like this now at least limits the impact of a more sweeping interface-removal change later.
