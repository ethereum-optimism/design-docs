# Always using `_disableInitializers()` in constructors

## Context

Currently, most OP Stack proxied contracts prevent direct calls to their `initialize` functions on the implementation contract by calling it with default parameter values in the constructor. Additionally, these contracts do not expose a public `reinitialize` function to prevent reinitialization attempts.

## Problem Statement

So far this does the job but has a few drawbacks, namely:

- It's a manual process (per contract) given that `initialize` function's input arguments are different for each contract. Also, a change in the `initialize` function's arguments will require updating the call to `initialize` manually.
- If a public `reinitialize` function is added in the future, calling the `initialize` function in the constructor will not prevent reinitialization.
- It can get verbose for initializers with many arguments. Here's an [example](https://github.com/ethereum-optimism/optimism/blob/36b41baea45f07e8644ff778a1ffd81fb0ad9e77/packages/contracts-bedrock/src/L1/SystemConfig.sol#L152).

## Proposed Solution

A proposed solution is to always use `_disableInitializers()` in the constructor of the implementation contract instead. This will, like the current approach, prevent anyone from calling the `initialize` function directly on the implementation contract but also solves the drawbacks mentioned above.

- It has the benefit of being a one-time manual change for each contract regardless of how many arguments the `initialize` function expects or if it changes in the future.
- If a public `reinitialize` function is used in the future, `_disableInitializers()`, already called in the constructor, prevents reinitialization attempts on the implementation contract.
- It's not verbose regardless of how many arguments the `initialize` function expects.

Another advantage of this approach is that it's a simple change, and it's easy to use Semgrep to find and assert that proxied contracts follow this pattern.

Here are the contract files that would be affected by this change:

- DelayedWETH.sol
- DisputeGame.sol
- DataAvailabilityChallenge.sol
- L1CrossDomainMessenger.sol
- L1ERC721Bridge.sol
- L1StandardBridge.sol
- L2OutputOracle.sol
- OptimismPortal.sol
- OptimismPortal2.sol
- ProtocolVersions.sol
- SuperchainConfig.sol
- SystemConfig.sol
- L2CrossDomainMessenger.sol
- L2ERC721Bridge.sol
- L2StandardBridge.sol

## Alternatives Considered

No other alternatives considered.

## Risks, Uncertainties, and Considerations

### Consideration

- Another rule that might help this proposal is asserting via Semgrep that all `initialize()` functions have an `external` modifier and there's no occurence of `this.initialize()`. This would help in enforcing this rule. Is there any scenario where `initialize()` functions need to be called from within a contract or an inheriting contract?
