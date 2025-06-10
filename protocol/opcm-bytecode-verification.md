# OPCM Bytecode Verification: Design Doc

|                    |            |
| ------------------ | ---------- |
| Author             | Maurelian  |
| Created at         | 2025-06-04 |
| Initial Reviewers  | TBD        |
| Need Approval From | TBD        |
| Status             | Draft      |

## Purpose

Ensure that the contracts referenced in `OPContractsManager` (OPCM) are verifiably built from
trusted source code by introducing a bytecode verification step into the release process.

## Summary

We propose integrating the `VerifyOPCM.s.sol` Foundry script into the release process via
`op-deployer` to validate that deployed contract bytecode matches locally built artifacts. This
prevents accidental or malicious mismatches and establishes trust in deployments originating from a
known commit. The step will be automated in CI, and become part of the documented SDLC.

## Problem Statement + Context

Currently, there is no enforced mechanism to ensure that the OPCM used in an upgrade is built from a
trusted commit, which should be one labelled as an `op-contracts/vX.Y.Z` tag, and approved by
governance.

This creates a risk of human error or maliciousness leading to an upgrade performed by an incorrect
version of the OPCM (or the implementation contracts it sets). It also presents a risk of a failed
upgrade resulting from a misconfigured OPCM (ie. if any [constructor
vars](https://github.com/ethereum-optimism/optimism/blob/a10fd5259a3af9a465955b035e16f516327d51d5/packages/contracts-bedrock/src/L1/OPContractsManager.sol#L266-L269)
are set incorrectly).

We want to eliminate this risk by automating the [Contract's Release
Verification](https://www.notion.so/oplabs/Contracts-Release-Checklist-1f8f153ee162805e8236f022ebb8c868?source=copy_linkhttps:/)
process, making it easy to demonstrate that an OPCM at a given address corresponds to a trusted
commit.

The solution should be:

- contained within the release commit in the monorepo
- easily runnable locally with a single command accepting only an RPC URL and the address of the
  OPCM
- runnable in CI in the `superchain-registry` repo
- incorporated into the upgrade process in a way that ensures it is run by multiple people

## Proposed Solution

### Bytecode verification against the local source

A new command should be added to `op-deployer`.

```
op-deployer verify-bytecode <opcm-address>

```

This command will invoke the `VerifyOPCM.s.sol` script's [default
entrypoint](https://github.com/ethereum-optimism/optimism/blob/158e990b76a85acbb018577bd4079190b2d97281/packages/contracts-bedrock/scripts/deploy/VerifyOPCM.s.sol#L126-L129)
to verify the OPCM.

```
op-deployer verify-bytecode --single-contract <contract-name> <contract-address>
```

This command will invoke the `VerifyOPCM.s.sol` script's [runSingle
entrypoint](https://github.com/ethereum-optimism/optimism/blob/158e990b76a85acbb018577bd4079190b2d97281/packages/contracts-bedrock/scripts/deploy/VerifyOPCM.s.sol#L135)
in order to verify any contracts involved in the upgrade which are not included in the OPCM (ie. the
new `DeputyPauseModule` introduced in upgrade 16).

### Artifacts source

By default, `op-deployer verify-bytecode` will use locally built forge-artifacts to check bytecode.
In order to facilitate quickly running in CI, without having to checkout and rebuild different
commits, the command will also accept a tag locator, with the following invocation:

```
op-deployer verify-bytecode --dangerously-use-remote-artifacts --artifacts-locator tag://op-contracts/vX.Y.Z

```

The flag `--dangerously-use-remote-artifacts` is intended to discourge the use of remote artifacts
when running locally, while still enabling a fast mechanism to run in CI.

### OPCM config verification

`op-deployer verify-bytecode` will require one of the following groups of arguments to confirm the
configuration of the OPCM.

Address flags:

```
--upgrade-controller 0x... \
  --superchain-config 0x... \
  --protocol-versions 0x... \
  --superchain-proxy-admin 0x...
```

Superchain-registry flag:

```
--superchain-registry <path/to/registry>
```

The argument groups should be mutually exclusive, either all of the address flags or the
superchain-registry flag should be provided, but not both.

### Integration points

It is important to ensure that the bytecode verification process is run by multiple people and on
multiple different machines.

The specific points that we should include the process are:

1. Automated in superchain-registry CI (as discussed above).
2. Included in the [contracts release checklist](https://www.notion.so/oplabs/Contracts-Release-Checklist-1f8f153ee162805e8236f022ebb8c868?source=copy_link#1f8f153ee16280998c6bfa1140a5854d) process.
3. Additionally for consideration: run by signers during the upgrade process.

### Resource Usage

N/A

## Single Point of Failure and Multi Client Considerations

N/A

## Impact on Developer Experience

- No impact on app developers.
- Protocol developers and release managers will have one additional verification step that is
  largely automated.

## Risks & Uncertainties

- Reliance on Etherscan APIs.
- We exclude from this discussion the possibility that the code in the repo is malicious, and are
  only concerned with verifying the bytecode against a monorepo commit and a
