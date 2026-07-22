# Purpose

Superchain interop enables new low-latency DeFi actions across chains. The protocol ensures integrity of the messages passed between chains, but additional app-level measures must also be in place to ensure users get the same single-chain DeFi experience.

## Goals

- Cross chain DeFi actions do not expose the user to additional 3rdparty risk

## Non Goals

- Masquerade submarine sends from regular transfer of funds.

# Problem Statement + Context

When using the `L2ToL2CrossDomainMessenger`, the target address and calldata is immediately made public when the message is sent. The outbox on the source chain becomes a visible mempool of transactions for a given destination.

For many use cases, this can be fine but in DeFi can undermine fairness, due to front running. [Submarine Sends](https://libsubmarine.org/) is a way to mitigate these kinds of problems, where the transaction is made private until executed in a commit / reveal structure.

# Proposed Solution

We can take inspiration by replicating a commit / reveal scheme for data sent between chains. Enabling cross chain transactions between contracts where the intent remains hidden until the reveal on the destination chain.

Unlike [Submarine Sends](https://libsubmarine.org/) which also provides the ability to make the commit transaction indistinguishable from a normal value transfer, the funds and commitment have to be accounted for and propogated through the `L2ToL2CrossDomainMessenger` which can allow for an external entinty to differentiate between a hashed cross domain submarine send and not. However this is not a problem since DeFi attacks rely on understanding the intent of the transaction which we show remains masked.

- Maybe there's some additional design where the destination escrow address is fresh on every call, achieving the same property but that is left out of scope for now.

## Commitment

The commitment for a submarine send is the sensitive information in the outbound `SentMessage` event, `target` and `message`. For submarine sends where the same target & message can be used more than once, a salt can be provided to ensure unique commitments per transaction.

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
   const data = cipher.update(payload);
   ```
3. Publish Commitment, Encrypted Data, Ephemeral Public Key, Cipher Auth Tag onchain. Only the commitment needs to be propogated. All else can be referenced in the calldata by the relaying entity from the sending transaction to decrypt the data.
   ```solidity
   function commit(bytes32 commitment, address relayer, bytes calldata data, bytes calldata ephemeralPubKey, bytes calldata cipherAuthTag) external
   ```

## Reveal

A designated relayer can hook into events emitted by the submarine contract to detect a new commitment that they can decrypt. The reveal takes the following steps

1. Parse necessary data from calldata for decryption -- commitment, data, empheral public key, cipher auth tag
2. Decrypt the payload with the recomputed shared secret using the relayer's private key and emphemeral public key.
   ```ts
   const iv = commitment.slice(0, 12);
   const sharedSecret = secp.ecdh(emphemeralPublicKey, relayerPrivateKey);
   const decipher = crypto
     .createDecipheriv("aes-256-gcm", sharedSecret, iv)
     .setAuthTag(cipherAuthTag);
   const payload = decipher.update(encryptedData);
   ```
3. Reveal the payload with the submarine contract. The commitment hash can be recomputed and checked.
   ```solidity
   function reveal(bytes32 commitment, uint256 salt, address target, bytes calldata message) external
   ```

## Cross Domain Submarine

By sending the commitment hash between L2s via the L2ToL2CrossDomainMessenger, we gain the ability to privately send transactions. However, submarine sends are typically accompanied with funds with the transaction. i.e Bridge and Swap.

The SuperchainTokenBridge & ETH bridge are designed as single cross chain messages. Thus the cross domain submarine contract needs to be able to receive and escrow sent funds prior to excuting the reveal transaction. Thus, a cross domain submarine transaction requires 2 relays on the destination

1. Send and escrowed funds in the submarine contract
2. Relay message the propogated commitment and details about funds owed to the target.

Sample snippet on how this can work for ETH. Generalizable to SuperchainERC20s

```solidity
IL2ToL2CrossDomainMessenger messenger;
ISuperchainETHBridge ethBridge;
mapping(bytes32 => uint256) ethEscrow;
mapping(bytes32 => bool) pendingCommitments;

/// @notice include the calldata parameters required for the encrypted payload
function submarineSendETH(uint256 _chainId, bytes32 commitment, ...) external payable {
   uint256 ethAmount = msg.value;
   bytes32 sentETHMsg = bridge.sendETH{value: ethAmount}(address(this), _chainId);
   bridge.sendMessage(_chainId, address(this), abi.encodeCall(this.handleSubmarineSendETH, sentETHMsg, commitment, ethAmount));
}

/// @notice keep accounting the commitment and funds received
function handleSubmarineSendETH(bytes32 sentETHMsg, bytes32 commitment, uint256 value) onlyMessenger external payable {
   // Ensure delivery before marking the commitment as pending
   require(messenger.successfulMessages(sentETHMsg));

   // If the same commitment has been used, ensure delivery of the first.
   // Ideally we can ensure every commitment is unique to avoid this edge case
   require(!pendingCommitments[commitment]);

   ethEscrow[commitment] = value;
   pendingCommitments[commitment] = true;
}
```

By asserting that the bridged funds are delivered first we can be sure that during the reveal, those funds can be delivered to the target.

```solidity
/// @notice reveal the pre-image to the commitment which then invokes the submarine call
function submarineReveal(bytes32 commitment, uint256 salt, address target, bytes calldata message) external {
   require(pendingCommitment[commitment]);
   require(validCommitmentReveal(...));

   uint256 value = ethEscrow[commitment];
   delete pendingCommitment[commitment];
   delete ethEscrow[commitment];

   (bool success, ) = target.call{ value: value }(message);
   require(success);
}
```

### Cross Domain Submarine Entrypoint

With the current structure of the `L2ToL2CrossDomainMessenger`, the reveal transaction, `submarineReveal()` is a followup transaction to the cross domain message that propogates the commitment, `handleSubmarineSendETH()`. However a relayer can batch all three calls to conduct a reveal in 1 transaction. `[relayETH(), handleSubmarineSendETH(), submarineReveal()]`.

With [Entrypoints](https://github.com/ethereum-optimism/design-docs/pull/163), it is possible to provide an entrypoint contract where all three calls must happen in 1 invocation. For demonstrative purposes, we'll keep `relayETH` seperate and show `handleSubmarineSendETH()` and `submarineReveal()` can be consolidated into a single entrypoint contract.

```solidity
contract CrossDomainSubmarineEntrypoint {
   function submarineReveal(Identifier _id, bytes calldata payload, uint256 salt, address target, bytes calldata message) external {
      // (1) Delivers the message which invokes handleSubmarineSendETH()
      bytes32 commitment = messenger.relayMessage(_id, payload);

      // (2) Reveal with the provided preimage
      submarineReveal(commitment, salt, target, message);
   }

   function handleSubmarineSendETH(...) onlyMessenger external returns (bytes32);
   function submarineReveal(...) internal;
}
```

This entrypoint ensures the commitment hash never has to be stored onchain since the reveal takes place in the same transaction, thus never "pending". The funds tied to the submarine send could similarly be attached to this entrypoint.

# Risks & Uncertainties

## Various `msg.sender ` assertions

It is typical for applications that integrate the `L2ToL2CrossDomainMessenger` to include an `onlyMessenger` modifier to ensure integrity of their cross domain calls. However with submarine sends, the developer must make sure to include a similar `onlySubmarineMessenger` modifier that asserts a different `msg.sender`, the submarine contract.

Making the right assertions is already an error-prone process and we're adding some complexity for developers. However this complexity is likely worth the cost since a developer must be intentionally aware how submarine functionality works when integrating their dapp & contracts.

## Commitment Uniqueness

The salt can ensure commitments are always unique. However it is the responsibility of the dapp to ensure this. It's already unlikely in DeFi that calldata might be the exact same as they depend on token inputs and minimum expected outputs. And even if they are the same, they must be pending during the same time window.

However an attacker that realizes that a dapp is using the same salt could try to fontrun some expected (target,message) combos with submarine sends that where the target always fails. All this would achieve in the current sample code snippet is halting delivery of submarine sends with the same commitment (already unlikely).

- In productionizing this implementation, this is an edge case we can explicitly handle.
