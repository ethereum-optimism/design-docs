# Purpose

This document proposes a method to support multiple Cannon behaviors (state transition functions) and Virtual Machines (VMs).

# Goals

- Support multiple Cannon state transition functions without complicating the offchain VM.
- Maintain the readability of the offchain VM Go code.
- Allow easy addition of new state transition functions to the offchain VM.

# Problem Statement and Context

When the Cannon Fault Proof Virtual Machine (FPVM) is updated, challengers need to determine which version of the offchain VM to run to maintain consistency with the on-chain VM. This issue is complicated by the reuse of the existing Cannon VM type across upgrades.

For example, the Cannon FPVM requires additional support for `F_GETFD` in `fnctl(2)` to support programs compiled by Go 1.22. These changes alter the FPVM's behavior, effectively creating new Cannon VM implementations. Applying newer behaviors to prestates intended for older Cannon versions would be inappropriate.

This problem extends to handling single-threaded and multi-threaded Cannon VMs, where prestates of both VMs are incompatible. Cannon needs to switch between multi-threaded and single-threaded functionality. We addressed this by introducing a `StateVersion` to the serialized Cannon state, allowing Cannon to apply either single-threaded or multi-threaded state transitions based on the `StateVersion`.

As both the Go compiler used to build the fault proof program and the program itself are upgraded, the Cannon FPVM may need extensions to support new use cases. Note that these VM extensions are not limited to syscalls; even additional instructions not yet supported by Cannon may be used by later versions of Go.

# Alternatives Considered

## Cannon-as-a-Service (CaaS)

CaaS is an offchain VM that fulfills the existing role of the Go Cannon implementation while supporting various Go Cannon behaviors. It's an HTTP server that serves execution traces, accepting requests for an execution trace given a prestate and the inputs to a fault proof program.

CaaS has access to multiple Cannon binaries and routes the FPP inputs based on the prestate input. It uses the `StateVersion` field in the serialized prestate to determine the appropriate Cannon binary/implementation for generating a trace.

This approach requires changes to op-challenger to support a new type of execution server and additional infrastructure to run CaaS. However, it offers the advantage of decoupling op-challenger deployments from Cannon. CaaS can be extended to support new VMs without redeploying op-challenger.

# Proposed Solution

The proposed solution aims to avoid introducing non-uniformity from other VMs, including Asterisc, which don't (yet) have the multiple STF problem.
This solution requires no changes to op-challenger and is implemented using a new Cannon CLI capable of using multiple versions of the `cannon` VM.

The core of this solution lies in packaging each version of `cannon` into the new CLI using [`go:embed`](https://pkg.go.dev/embed).
This approach results in multiple Cannon binaries, one for each FPVM implementation, being embedded into the Cannon CLI binary.
By embedding these binaries directly into the CLI, we ensure that all necessary versions are readily available without the need for external dependencies or separate installations.
The new Cannon CLI maintains compatibility with the existing `cannon` CLI by supporting the same set of flags and subcommands.
This ensures a seamless transition for users familiar with the current system and minimizes disruption to existing workflows. Despite its enhanced capabilities, the interface remains consistent and intuitive.

One of the key features of this new CLI is its ability to detect which VM implementation to use based on the prestate input.
Similar to the CaaS approach, it reads the `StateVersion` from the prestate input to determine the appropriate version of Cannon to execute.
This automatic detection eliminates the need for manual version selection and reduces the risk of errors that could arise from using an incompatible VM version.

Once the CLI has identified the appropriate Cannon VM binary for the given prestate, it loads this binary via go:embed.
The selected Cannon program is then executed using [`execve(2)`](https://man7.org/linux/man-pages/man2/execve.2.html).
This execution method allows for efficient and isolated running of the chosen Cannon version, ensuring that each prestate is processed by the correct VM implementation.

An additional benefit of this approach is it facilitates multi-architecture MIPS Cannon. Particularly for the usecase where multiple Cannon binaries with different [build tags are needed to compile Cannon for MIPS32 and MIPS64 VMs](https://github.com/ethereum-optimism/optimism/pull/12029).

## State Version

This approach implies that the `StateVersion` must be used to also encode the VM version. As such, every change to the STF, no matter how little will require a new version.
However, `StateVersion` is a uint8 field and it's not terribly unlikely that there are over 256 VM implementations. 
As such, we need a new wider field in the encoded state file to encapsulate the VM version.

## Supporting new cannon embeds 

While every version of cannon can be embedded into the cannon CLI, this can result in a really large binary.
Instead, the Cannon CLI only needs to maintain the VM implemnetations that it's likely to use.
In practice, at most two `cannon` VMs are needed by the op-challenger, to support the transition between FP upgrades.

To provide optionality of embedded cannon VMs, we can use the [go:embed filesystem](https://pkg.go.dev/embed#hdr-File_Systems) that serves as a lookup table (via a simple file naming conventio) of `cannon` VMs.
