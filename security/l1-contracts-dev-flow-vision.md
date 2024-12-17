# Purpose

The goal of this design doc is to outline the vision we are working towards for the L1 contracts
development flow, so we can collect feedback from stakeholders on what we are working towards to
ensure we are all aligned on the vision.

# Summary

<!-- Most (if not all) documents should have a summary.
While the length will likely be proportional to the length of the full document,
the summary should be as succinct as possible. -->

The EVM Safety team is working on two projects—OPCM (OP Contracts Managr) Upgrades and superchain-ops
improvements—that aim to improve the tools and standardize the processes around the full L1 smart
contract development cycle. This includes development, testing, releasing, deployment, and upgrading.

# Problem Statement + Context

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

As the number of chains we need to manage increases, our existing L1 contract development processes
have proven to be insufficient, and will not scale well. We need to tighten up this end to end flow
with robust, easy to use tools and well-defined processes. Some existing issues we've had are:

- Deployment and upgrade processes are not standardized.
- Confusion around smart contract releases.
- Calldata and state changes are not part of governance proposals and not verifiable by governance.
- Task development and validation is a manual process that is time-consuming and error-prone.
- We do not have a good way to track the full changelog for a given contracts release.

# Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

The proposed UX upon for L1 smart contract development is described below.
This UX will be achieved once the ongoing OPCM Upgrades and superchain-ops projects are completed.
We start from assuming that `op-contracts/v2.0.0` with OPCM Upgrades is shipped as part of Isthmus
in Cycle 34. We intentionally exclude the upgrade to `op-contracts/v2.0.0` in this section, as it
may have some unique aspects because it puts this system in place, although we expect it to be
largely similar to the flow described here.

## Development

A team starts working on a feature. If, and only if, they are 100% sure this feature is going
into the next release, feature work can happen on contracts directly. Otherwise the feature must be
developed using the inheritance pattern described in [`smart-contract-feature-development.md`](../smart-contract-feature-development.md).

For example, if you are adding something to the SystemConfig that will ship in the next release, make
those changes directly to `SystemConfig.sol`. Tests would be added directly to `SystemConfig.t.sol` like normal.

If your feature may not go into the next release, create a child contract such as `contract SystemConfigInterop is SystemConfig`.
In this contract you can add your feature, and once we know it's ready for release it will be merged
into the parent contract it inherits from. Tests for `SystemConfigInterop` would live in `SystemConfigInterop.t.sol`
(TODO flesh out more details on how this should work with OPCM, e.g. do we need an `OPCM*` for each feature, like
`OPCMInterop`, etc? A simpler approach may be to always use OPCM as the base standard deploy, and write a `interopUpgrade()`
method that contains the code that will eventually go in `OPCM.upgrade`).

An alternate approach to inheritance is to put code directly into `SystemConfig`, but behind a
feature flag. This is the approach client code uses so they can ship releases without also shipping
features that are not production ready. However, in safety-critical systems dead code is typically
not allowed, so we likely will not allowed this approach for contracts. However, we mention it here
anyway for completeness.

## Testing

In the above section we described how standard unit tests would be written during development. Another
aspect is how deploy and upgrade scripts are written and tested during development, as this is a key
change.

Originally, `Deploy.s.sol` was the script that run as part of test `setUp`, so we can run tests
against the state resulting from the deploy script. In practice, that `Deploy.s.sol` script was
never used for real deploys. So `Deploy.s.sol` has been refactored to instead use OPCM to deploy
a chain's L1 contracts and use that fresh deploy as the base for tests. This means that all tests
will be known to pass against the state resulting after a new chain deploy. (In the future, we may
delete the full `Deploy.s.sol` file and replace it with OPCM code directly, as embedding OPCM into
this legacy deploy script was only done as it's a simpler transition).

OPCM contains a `deploy()` method that takes some arguments to deploy a chain. Some changes to
source code will require no changes to this method—for example, if you only change things like
code comments or function internals. Other changes will require changes to this method, like
adding new contracts or modifying the `initialize` signature. For the latter case, because OPCM
is what's used to scaffold chains for testing, you will be forced to modify the `deploy` method to
account for your changes in order to test them. This is great, as it's now trivial to keep deploy
scripts up to date, and we essentially get this for free.

OPCM will also contain an `upgrade` method as described in the [OPCM Upgrades design doc](l1-upgrades.md).
Some basic unit tests will test this method. Additionally, a flag can be set to have tests
run against the state resulting from running `OPCM.upgrade` against an existing chain (instead of the
default where tests are run against the state resulting from a fresh deploy via `OPCM.deploy()`). CI will be
responsible for running the tests against the `upgrade` method for all mainnet and sepolia chains
that the Security Council + Optimism Foundation (the 2/2) hold keys for. This enforces that
developers are also writing the upgrade logic at the same time, to provide guarantees that the full
test suite passes for new deploys and upgrades for all existing chains.

## Releases and Contract Deployments

After development, we are now ready to cut a release candidate and deploy the new implementation
contracts to a chain, such as devnet, sepolia, or mainnet. All contract deployments must be done
against a tag of the form `op-contracts/vX.Y.Z[-rc.n]`. To deploy the contract, create a `bootstrap`
command in op-deployer, and run that new command to deploy the new contracts.

An alternate approach that we might support that avoids the need for bootstrap commands is using
CREATE2 in the `DeploySuperchain.s.sol` and `DeployImplementations.s.sol` scripts. This way,
deploying contracts is as easy as re-running those scripts—if bytecode for a contract has not changed,
the resulting create2 address is the same, so we can skip deploying that unchanged contract.

The current [release and versioning process](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/meta/VERSIONING.md)
is unintuitive and requires a lot of manual work. It is not possible to have the source code for the
entirety of a release at a single commit, and therefore very difficult to test contract releases.
Additionally, deploy and upgrade scripts are not part of a release.

Now, OPCM serves as the source of truth for what a release is. A release is defined as the OPCM contract
itself, along with all contracts that are used in the `deploy()` or `upgrade()` methods of OPCM.

## Execution of Onchain Transactions

With new implementation contracts deployed, we can prepare playbooks, or tasks, in superchain-ops.
There will be two ways to create tasks: bespoke tasks, and template tasks.

Bespoke tasks are for one-off tasks that are not expected to be repeated. These will be created
by running a command such as `just new-task {l1ChainId} {taskName}`. This will scaffold a new
directory. In this directory would be a solidity file that inherits from some base contract. In here
you write two things: the calls to execute, and validations. The validations are similar to what we
currently do, and the calls to execute are different—instead of specifying the calldata in an `input.json`
file, calls are specified in solidity as if you are writing a regular forge script. The tooling
will extract the sequence of calls from the script and generate the calldata bundle.

Template tasks are for tasks that are expected to be repeated. These will be created by creating
a solidity template of the same format as a bespoke task, but with some parts replaced with variables.
These variables are read from a TOML file, and the path to that TOML file is the only input to the
template script. To create a new task from a template you would run a command such as
`just new-task-from-template {l1ChainId} {taskName} {templateName}`. This will scaffold a new directory,
but there will be no solidity file. Instead, a TOML template will be copied into the directory and
populated as required.

There will also be some other UX and safety improvements for writing these solidity files.

- Addresses for a chain can be fetched in Solidity via `addresses.getAddress("ContractName", l2ChainId)`.
- CI will automatically run for tasks that have not yet been executed.
- Commands to help autogenerate the `VALIDATIONS.md` file as much as possible.
- Additional safety checks, such as storage layout compatibility and data hash verification.

The data hash verification in particular is an important one. Currently, upgrade scripts are not part
of governance proposals—only contract bytecode is. The OPCM changes described above make it trivial to
add deploy and upgrade scripts to the governance proposals. As a result, because the OPCM address and
the inputs to the `upgrade()` call will be known at the time of the governance post, we can precompute
the data hash and include it in the governance proposal.

This means Security Council and Optimism Foundation signers only need to verify that the hash shown on
their ledger matches the hash in the governance proposal. If it does, we know all state changes are
what's expected and what governance has approved. This is much simpler and less error prone than the
current approach, which requires signers to validate every individual state change.
