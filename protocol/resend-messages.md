# Resend Messages

|                    |                             |
| ------------------ | --------------------------- |
| Author             | 0xDiscotech                 |
| Created at         | _2025-04-08_                |
| Initial Reviewers  | agusduha, skeletor-spaceman |
| Need Approval From |                             |
| Status             | In Review                   |

## Purpose

To provide a mechanism for users to re-emit a previously sent message event on origin, ensuring that stale messages can be picked up and relayed on destination.

## Summary

This feature introduces a new function that allows a re-emission of the SentMessage event for messages that have been sent but not yet relayed. The re-emission helps offchain infrastructure detect and relay messages that might otherwise be ignored due to their age. It does not affect messages that have already been relayed on the destination chain.

## Problem Statement + Context

This feature introduces a new function that allows a re-emission of the `SentMessage` event for messages that have been sent but not yet relayed. The re-emission helps offchain infrastructure detect and relay messages that might otherwise be ignored due to their age. It does not affect messages that have already been relayed on the destination chain.

## Proposed Solution

We propose adding a new `resendMessage` function on the `L2ToL2CrossDomainMessenger` contract. This function will accept message hash inputs as parameters (with `source` being the exception since it will be hardcoded as the current chain id), it will calculate the message hash, check that it was sent and it will re-emit the corresponding `SentMessage` event.

### Resource Usage

- It adds an additional `sstore` on `L2ToL2CrossDomainMessenger#sendMessage` function to store the `messageHash` with `true` as the value on the `sentMessages` mapping.
- It adds a new (small) function: `resendMessage`, which leads to a slightly higher deployment cost than the previous contract’s version.

## Failure Mode Analysis

- **Re-emitting a log for a valid message sent hash with different message params:**
  A hash collision could occur where message parameters that don't correspond to any sent message, result in the same message hash of a valid message sent. This would involve a bug in the `Hashing` library logic or a hash collision, for which its likelihood is very low.

## Risks

- **Already Relayed Messages:**
  Re-emitting a message already relayed MUST HAVE no effect because the system MUST process each message only once. The logic on `relayMessage` should always handle this case in this way, as it currently does.
