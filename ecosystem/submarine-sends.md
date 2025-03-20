# Purpose

Superchain interop enables new low-latency DeFi actions across chains. The protocol ensures integrity of the messages passed between chains, but additional app-level measures must also be in place to ensure users get the same single-chain DeFi experience.

## Goals

- Cross chain DeFi actions do not expose the user to additional MEV risk
- 3rdparty can relay submarine transactions to remove friction for users

## Non Goals

-

# Problem Statement + Context

When using the `L2ToL2CrossDomainMessenger`, the target address and calldata is immediately made public when the message is sent. The outbox on the source chain becomes a visible mempool of transactions for a given destination.

For many use cases, this can be fine but in DeFi can undermine fairness, due to front running. [Submarine Sends](https://libsubmarine.org/) is a way to mitigate these kinds of problems, where the transaction is made private until executed in a commit-reveal scheme.

[Submarine Sends](https://libsubmarine.org/) also obfuscates the the flow of funds by generating a fresh address the funds are escrowed apart of the commitment phase. Making it impossible differentiate between a simple eth send and a user privately interacting with a dapp.

# Proposed Solution

This is not an important feature to replicate for Superchain interactions as the `L2ToL2CrossDomainMessenger` is the single entrypoint for the native flow of funds across all Superchain L2s. As long as the target contract and calldata is made hidden, a user observing outgoing messages wont be able to differentiate between the the different types of submarine sends.

- TODO: maybe note that people can detect which messages are submarine sends prior to the reveal

By replicating a commit-reveal format for interoperable transactions, user can privately send and relay messages between chains.

## Commitment

The commitment is simply the sensitive information in the outbound `SentMessage` which is the `target` and `message`. For submarine sends where the same target & message can be used more than once, salt can be used to ensure unique commitments.

```solidity
bytes memory payload = abi.encodePacked(salt, target, message)
bytes32 commitment = keccak256(payload)
```

By posting just the commitment, **only the the submitting user with knowledge of the preimage would be able to reveal the transaction** when relaying. If this user encounters issues between submission and relay, the preimage for that data is also lost.

To solve for this, the payload is encrypted and posted as calldata to the submitting transaction. By using ECDH, Elliptic Curve Diffie-Hellman, the data can be encrypted against any public key, allowing for the user to designate a 3rdparty to conduct the relay on their behalf.

1. Generate A Shared Secret Using ECDH. The user locally generates an empheral keypair to create a shared secret with the private key and the relaying public key
   ```ts
   const emphemeralPrivateKey = generatePrivateKey();
   const sharedSecret = secp.ecdh(relayerPublicKey, emphemeralPrivateKey);
   ```
2. AES encrypt the data using the shared secret and the first 12 bytes of the commitment as the iv tag
   ```ts
   const iv = commitment.slice(0, 12);
   const cipher = crypto.createCipheriv("aes-256-gcm", sharedSecret, iv);
   const encryptedData = cipher.update(payload);
   ```
3. Publish Commitment, Encrypted Data, Ephemeral Public Key, Cipher Auth Tag onchain. Only the commitment hash needs to stored onchain. All else can be referenced in the calldata by the relayer from the sending transaction to decrypt the payload.
   ```solidity
   function commit(
     bytes32 commitmentHash,
     address relayer, // included in emitted events for relayers to listen to
     bytes calldata data,
     bytes calldata ephemeralPubKey,
     bytes calldata cipherAuthTag
   ) external
   ```

## Reveal

A designated relayer can hook into events emitted by the submarine contract to detect a new commitment that they can decrypt. The reveal takes the following steps

1. Parse necessary data from calldata for decryption -- Data, Empheral Public Key, Cipher Auth Tag. The IV is just the first 12 bytes of the commitment hash.
2. Recompute the shared secret using the relayer's private key & decrypt the payload
   ```ts
   const sharedSecret = secp.ecdh(emphemeralPublicKey, relayerPrivateKey);
   const decipher = crypto.createDecipheriv("aes-256-gcm", sharedSecret, iv);
   decipher.setAuthTag(ciperAuthTag);
   const payload = decipher.update(encryptedData);
   ```
3. Reveal the payload with the submarine. The submarine contract ensures a pending commitment and matching preimage.
   ```solidity
   function reveal(bytes32 commitmentHash, uint256 salt, address target, bytes calldata message) external
   ```

# Risks & Uncertainties
