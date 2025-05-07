# Purpose

The purpose of this document is to standardize all error messages/handling for the Optimism Smart Contracts, which currently use a mix of error strings, and mix both `require` and `revert`.

# Summary

Going forward, all error messages should use the prefix of the `contract_` name. Additionally, all errors should be returned via `revert` rather than `require`. This is both for visual clarity and due to the efficiency concerns around `require` in the case of errors with arguments.

# Problem Statement + Context

The current problem is the Optimism smart contracts mix both `require` and `revert` statements inconsistently with error strings, and Solidity custom errors. For the sake of a consistent abi, the Optimism smart contracts should converge on a single method for reverting.

# Alternatives Considered

## Require and Error Strings

All errors could be handled using `require` statements with error strings. However, the main issue with this approach is that error strings take up significantly more bytecode size. Given the small margins of the contract size for staying within the Spurious Dragon bytecode size limit, it would be very useful to reduce contract size in favor of more optimizer runs.

## Require and Custom Errors

As of Solidity 0.8.24, `require` supports using custom errors, however, due to the limitation that `require` does not short-circuit (ie if there is a custom error which has a value passed in, and you pass in the result of some function call to that argument, the function call will be executed regardless of whether or not the error is returned) meaning that in all cases it is inefficient, and in some cases even dangerous.

# Proposed Solution - Revert and Custom Errors

All errors returned from Optimism Smart Contracts should use `revert` with custom errors for both efficiency and safety. In the case of scripts and tests which currently use `require` frequently, they should be instead replaced with `assertEq` or similar `assert*` where appropriate, including a string as a revert message. This will allow for semgrep rules to more easily enforce the no `require` constraint. Custom errors should also include the contract name as a prefix, followed by an underscore.

For example, under this new policy, the following statement inside of MIPS2.sol

```solidity
            require(_state.rightThreadStack != EMPTY_THREAD_ROOT, "empty right thread stack");
```

would now become

```solidity
    if (_state.rightThreadStack == EMPTY_THREAD_ROOT) {
        revert MIPS2_EmptyRightThreadStack();
    }
```

Additionally, all errors should live inside of the files they are used. This would mean the removal of `*Errors.sol` files, removing the need for files like `L1BlockErrors.sol`.
