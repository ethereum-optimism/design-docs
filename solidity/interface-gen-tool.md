# Purpose

<!-- This section is also sometimes called "Motivations" or "Goals". -->

<!-- It is fine to remove this section from the final document,
but understanding the purpose of the doc when writing is very helpful. -->

Design document for an interface auto generator script. This removes the need to manually create and modify interfaces after the creation/modification of a contract.

<!-- Describe the specific problem that the document is seeking to address as well
as information needed to understand the problem and design space.
If more information is needed on the costs of the problem,
this is a good place to that information. -->

## Context

Interfaces have recently been introduced throughout the OP Stack codebase, resulting in significant improvements to the development workflow. This addition has dramatically reduced compile time after the modification of most contracts from several minutes to a few seconds. Furthermore, the introduction of interfaces also makes it easier for external developers to work with the OP Stack codebase.

## Problem Statement

However, this improvement comes with a trade-off, the creation and modification of interfaces is currently a manual process. When changes are made to a contract's ABI, its corresponding interface must be manually updated. This approach lacks scalability and efficiency, especially when making large changes and as the codebase grows and evolves.

## Solutions Considered

<!-- List out a short summary of each possible solution that was considered.
Comparing the effort of each solution -->

While using a script that leverages `forge inspect` and `cast interface` to automatically generate interfaces for all relevant contract files could be a potential solution, it has several limitations:

- Enum representation: Enums are treated as user-defined value types with an underlying uint8 type. This approach is problematic because it prevents accessing enum variants from the interface by name and makes working with the underlying type error-prone especially when the enum variants grow, are reordered or deleted.
- No pseudo-constructor support: The OP Stack codebase heavily relies on pseudo-constructors for type-safe constructor argument passing, which are not supported by this method.
- Unnecessary custom error replication: The generated interfaces include custom errors from the contract file, which is not relevant or necessary for our use case and cannot be optionally omitted.
- Interface Name: The name of the generated interface class is not customizable and is currently hardcoded to be the same as the contract name. This does not allow for a clear distinction between interfaces and contracts.
- No support for importing of Libraries, Types, Events, Errors etc, they are just re-generated.
- Contract types are turned into address types.
- No support for File-level defined types in generated interfaces.

These limitations make this approach suboptimal for our specific needs in the OP Stack codebase. Developers still have to manually review and update interfaces after they are generated.

## Goals

- Automate the process of creating and updating interfaces for contracts in the OP Stack codebase. This reduces the overhead of manually creating and updating interfaces.
- Ensure generated interfaces correctly: represent enum types and preserve their named variants, support pseudo-constructors, exclude custom errors from the generated interface files, allow for customizable interface names.
- Maintain or improve the current efficiency gains in compile times achieved through the use of interfaces. If compile times are increased, the benefits of automatically generated interfaces should still outweigh the additional time.
- Enhance the developer experience by reducing the overhead of interface management.

# Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

The proposed solution is an interface generation script that will function similarly to `forge inspect` and `cast interface`, but with enhanced capabilities to address our specific needs:

- Checks for loose imports in each contract file reports any found
- Support for pseudo-constructors
- Support for importation of Libraries, Types, Events, Errors etc, from other files where applicable rather than just re-generating them.
- Full support for enums, contract/interface types and not representing them as a user defined type with an underlying uint8 type and address respectively
- Interface names are prefixed with `I`
- Support for file level types in the generated interfaces

All these and more are not supported by the current forge inspect and cast interface methods but are supported by the interface generation script.

## Resource Usage

<!-- What is the resource usage of the proposed solution?
Does it consume a large amount of computational resources or time? -->

It should be a relatively fast script.

# Risks & Uncertainties

<!-- An overview of what could go wrong.
Also any open questions that need more work to resolve. -->

None so far.
