# Custom Gas Token: Failure Modes and Recovery Path Analysis

| | |
|--------|--------------|
| Author | Mark Tyneway |
| Created at | 2024-05-13 |
| Needs Approval From | *Security Reviewer Name* |
| Other Reviewers | Dragan Zurzin, Kevin Ho |
| Status | In Review |

## Introduction

This document specifically covers the usage of the custom gas token as a feature for new chains. This does not cover the usage of the custom gas token enabled contracts for existing chains that use `ether` to pay for gas.

The custom gas token feature enables a L2 chain to collateralize its native asset using an L1 based ERC20 token. Users deposit their ERC20 token into the bridge and native asset is minted on L2.

Below are references for this project:

- [Custom Gas Token - Product Hub](https://www.notion.so/Custom-Gas-Token-Product-Hub-4eeb15de1c4545b5bef68fc24b573a71?pvs=21)
- [[EXTERNAL] How to run a Custom Gas Token chain](https://www.notion.so/EXTERNAL-How-to-run-a-Custom-Gas-Token-chain-d88c68306f934790be051f87213a0b1d?pvs=21)
- https://github.com/ethereum-optimism/specs/blob/08d3c7189309e9e739d11312ed2ac7c1a43cb582/specs/protocol/granite/custom-gas-token.md
- [https://github.com/ethereum-optimism/specs/discussions/140](https://github.com/ethereum-optimism/specs/discussions/140)
- https://github.com/ethereum-optimism/optimism/pull/10143

## **Failure Modes and Recovery Paths**

### Bridge Undercollateralization

- **Description**
    - It is possible that the bridge becomes undercollateralized. This means that the invariant is broken that the bridge owns greater than or equal to custom gas token on L1 compared to the amount of native asset minted on L2. If the bridge becomes undercollateralized, then users that deposited into L2 will not be able to withdraw. If the chain can interoperate with other chains, then this can impact the collateralization of the entire superchain.
- **Risk Assessment:**
    - High severity, low likelihood
    - A hack that steals money from the bridge can end the chain. This is not battle tested code but follows best practices like checks, effects, interactions, has thorough unit and end to end tests as well as follows a similar pattern to Fraxtal’s implementation of custom gas token. Fraxtal has not yet been hacked.
    - A hack that mints money on L2 without it being collateralized by the rollup should not be possible given a standard ERC20 token is used. This is certainly possible to do with a malicious ERC20 token and this is not possible to detect on chain. This means that social consensus must be used to determine the safety of a rollup using a custom gas token.
    - Another form of this attack involves using `ether` to mint native asset on L2 when the custom gas token feature is enabled. This would undercollaterize the bridge in terms on the custom gas token and turn the native asset into a proportionate representation of `ether` and the custom gas token. This would break withdrawals.
- **Mitigations:**
    - This can be mitigated by using offchain checks on the standardness of the ERC20 token before doing any sort of brand association with the custom gas token chain. These checks cannot be automated, manual inspection must be done to ensure that the token is [specs compliant](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/granite/custom-gas-token.md#properties-of-a-gas-paying-token).
- **Detection:**
    - An offchain monitoring service could exist that is able to utilize the APIs on the smart contracts to determine that the bridge is not drained on L1 in a malicious way
- **Recovery Path(s)**:
    - It is possible to recollateralize the bridge by directly transferring the ERC20 tokens to the bridge. This would enable users to be able to withdraw again.

### Non Standard ERC20 Usage

- **Description:**
    - This is related to [Bridge Undercollateralization](https://www.notion.so/Bridge-Undercollateralization-552a985fcbed4d95b468f4b4c5f22ec9?pvs=21) as it is one form that the bridge can become undercollaterized. Really anything is possible with a non standard ERC20
- **Risk Assessment:**
    - High severity, low likelihood
    - Given that we communicate with rollup as a service platforms, this is low probability. Chains that don’t listen to us can do whatever they want in their world but they will never be able to be considered standard.
- **Mitigations:**
    - We should not co-market with any custom gas token chain unless we have manually checked that their ERC20 token complies with the specs.
- **Detection:** *How do we detect if this occurs?*
    - Automation cannot fully detect that the ERC20 token meets the standardness checks.
- **Recovery Path(s)**:
    - We will need to remove the chain from the superchain if they are able to launch without us realizing that they are not meeting the ERC20 standardness requirements. It is risky to have the team migrate to a standard ERC20 because it would require a full solvency check first.

### Withdrawal Bricked Bug

- **Description:** *Details of the failure mode go here. What the causes and effects of this failure?*
    - It is possible that a bug exists that would prevent users to withdraw that is unrelated to bridge undercollaterization. This sort of bug would just prevent users from withdrawing.
- **Risk Assessment:** *Simple low/medium/high rating of impact (severity) + likelihood.*
    - High severity, low likelihood
    - There is test coverage for this, both unit testing and end to end testing. This has also been tested on live networks.
- **Mitigations:** *What mitigations are in place, or what should we add, to reduce the chance of this occurring?*
    - The live testing has shown that networks can operate with custom gas token enabled, see [https://github.com/ethereum-optimism/specs/discussions/140#discussioncomment-9426636](https://github.com/ethereum-optimism/specs/discussions/140#discussioncomment-9426636)
- **Detection:**
    - This isn’t immediately obvious if there is an automated way to detect but users will certainly let us know that there is a bug.
- **Recovery Path(s)**:
    - This may require an irregular state transition to fix

## Action Items

Below is what needs to be done before launch to reduce the chances of the above failure modes occurring, and to ensure they can be detected and recovered from:

- [ ]  Figure out existing chain monitoring solutions and if rollup as a service providers operate them
    - [ ]  If yes, then design the necessary changes to the monitoring service so that they can eventually be implemented
- [ ]  *Action item 3 (Assignee: tag assignee)*

## Appendix

### Appendix A: Relation to Existing Chains

The custom gas token feature can be enabled or disabled. When it is disabled, it is assumed that ether is used as the L2 native asset. This feature will eventually be rolled out to OP Mainnet with the custom gas token feature disabled. This will warrant a future failure mode analysis.
