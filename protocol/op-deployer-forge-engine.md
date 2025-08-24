# Use Forge as the EVM for OP Deployer

## Summary

We propose switching out the Go-based EVM inside OP Deployer for a wrapper around Forge. This would reduce our
maintenance burden and allow us to take advantage of new Forge features without requiring us to reimplement them.

## Problem Statement + Context

OP Deployer was developed on top of a Golang-based EVM. This allows OP Deployer to execute Forge scripts and
simulate transactions prior to executing them like Forge does. Now that OP Deployer has been in production for over
a year, we're running into several issues with this approach:

1. **Difficulty debugging.** Rather than outputting a trace of the error, OP Deployer simply panics with
   "revision ID 1 cannot be reverted." Users must then parse through the logs to determine the cause of the error.
2. **Incomplete featureset compared to Forge.** OP Deployer is often compared to Forge, since both deploy contracts.
   Therefore, users perceive OP Deployer as incomplete from a feature perspective. For example, OP Deployer does not
   support verifying all the contracts it can deploy, and the contracts it _does_ support can only be verified on
   Etherscan.
3. **Maintenance burden.** The Golang EVM is a significant maintenance burden. Every new cheatcode in Forge must be
   reimplemented in Go. Additionally, we must maintain a "bridge" to allow Golang to call into Solidity scripts.
   This is a lot of work that provides no user benefit since the Golang EVM is simply a backend that allows contracts to
   be deployed. This is table stakes for a deployment tool, and is much better solved in Forge.

We originally adopted the Golang solution because we wanted to avoid adding a dependency on Forge. However, there
are many ways around this that do not incur the maintenance burdens described above.

## Proposed Solution

We propose switching all contract calls in OP Deployer to use Forge instead of the Golang EVM. OP Deployer would
essentially become a thin, stateful wrapper around Forge. Think of it like an installer for our contracts. We've
outlined some implementation details of this approach below.

### Getting Forge

OP Deployer will autodetect Forge based on the following heuristics:

1. If Forge is already on the user's machine, and it is a supported version, use it.
2. Otherwise, download a pinned version of Forge and use that.

### Calling Forge

For now, OP Deployer will simply call the Forge binary as a CLI tool. To communicate with Forge, OP Deployer will
parse Forge's standard output stream. This will require updating some of our smart contracts to output JSON. We may
adopt solutions like calling Forge as a library via FFI, but that is currently out of scope.

Forge's output will otherwise be streamed directly to the user's terminal so that they can see the progress of the
deployment.

## Impact on Developer Experience

This change should make maintaining OP Deployer much easier:

1. The Platforms team would no longer need to maintain the Golang EVM.
2. Changes to Forge can be immediately adopted.
3. Smart contract devs don't need to worry about how their contracts will be called from Golang anymore.
4. Error messages and contract deployment output will be much better.

Users running OP Deployer in a container will either need to use a container that contains Forge, or allow that 
container to download the Forge binary. We can provide a prebuilt container with Forge installed to avoid this issue.

## Alternatives Considered

### Continue Maintaining the Golang EVM

This is our default path. However, we're getting asked to build things like Blockscout contract verification, which 
would be entirely duplicative of what Forge already does really well.

## Risks & Uncertainties

None really. This is a pretty straightforward change that reduces risk overall.