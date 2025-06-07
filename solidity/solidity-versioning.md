# Solidity Versioning

## Context

Solidity undergoes regular updates. These updates often introduce useful new features,
optimizations, and bug fixes that can enhance the functionality, security, and efficiency of OP
Stack smart contracts. OP Labs has been planning to propose an update of all existing OP Stack
contracts to Solidity 0.8.25 to take advantage of some of these improvements.

## Problem Statement

As we approach this update and consider future upgrades, we need to establish a coherent strategy
for managing Solidity version updates in our repository. This strategy should address the following
key points:

1. Ensure a consistent approach to each upgrade across the OP Stack codebase.
2. Facilitate easy review and approval processes for proposed updates.
3. Minimize the risk associated with adopting new compiler versions.
4. Protect the OP Stack from compiler bugs that could impact contract functionality or security.

## Proposed Solution

To address the challenges outlined in the problem statement, we propose the following structured
approach for Solidity version updates:

### Establish a Minimum Delay Period

- Implement a mandatory waiting period after a new Solidity release before it can be considered
  for adoption (6 months). Duration is based on an [analysis](https://oplabs.notion.site/Solidity-major-bugs-patch-duration-113f153ee162805592ced28631e5bbd8?pvs=4)
  of time typically required to identify and fix critical bugs in previous Solidity releases.

### Formal Version Update Design Doc Proposal Process

- Require the submission of a detailed proposal for any Solidity version upgrade.
- Proposal should be submitted as a PR to the design-docs repository.
- Submission must be made BEFORE any Solidity version upgrades are made.
- Proposed Solidity version must be at least 6 months old.
- The proposal must include the following components:
  - The proposed Solidity version number.
  - The release date of the proposed version.
  - Specific feature(s) in the new version that would benefit the OP Stack codebase, with a
    clear explanation of their potential impact and usefulness.
  - A comprehensive summary of all significant features, changes, and bug fixes introduced
    between the version currently used and the proposed upgrade.
  - Highlights of any high-risk features or notable bug fixes that require special attention
    during the review process.
  - Links to the Solidity changelog for each version in between the current version used in the
    codebase and the proposed new version.

### Review and Approval Process

- Establish a dedicated review panel.
- Conduct a thorough assessment of the upgrade proposal based on the following criteria:
  - Is the Solidity version old enough?
  - Does the proposed upgrade provide clear value to the codebase?
  - Do any new features or bug fixes pose an unnecessary risk to the codebase?
- Require unanimous approval from the review panel before proceeding with the upgrade.
