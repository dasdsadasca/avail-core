# Conceptual PoC: F1 - `ShaTwo256` Bug Leading to Potential Fund Freezing/Loss

## 1. Recap of Finding F1: Incorrect Hash Function in `ShaTwo256`

Finding F1 identified a critical bug in `avail-core/core/src/sha2.rs`. The `ShaTwo256` struct, which is intended to provide SHA2-256 hashing functionality by implementing the `sp_core::Hasher` trait, incorrectly uses the `keccak_256` algorithm internally. This means that any code calling `ShaTwo256::hash()` will receive a Keccak-256 hash instead of the expected SHA2-256 hash.

## 2. Hypothetical Scenario Introduction

This Proof-of-Concept (PoC) is conceptual and describes a hypothetical scenario where the `ShaTwo256` bug in `avail-core` could contribute to the freezing or effective loss of funds. This scenario relies on a hypothetical runtime pallet (`pallet-outbound-messages`) that incorrectly assumes `avail_core::sha2::ShaTwo256` provides a standard SHA2-256 hash and uses it in a critical part of a cross-chain interaction, such as a bridge.

The exploit does not occur *solely* within `core/src` but materializes when the faulty `ShaTwo256` component is used by a runtime pallet in a way that interacts with external systems expecting standard cryptographic behavior.

## 3. Detailed PoC Steps

Let's assume the Avail runtime includes a `pallet-outbound-messages` designed to facilitate sending messages or assets to another blockchain (e.g., "Chain B") via a bridge mechanism.

**Hypothetical `pallet-outbound-messages` Logic:**

1.  **Message Creation:** A user on Avail initiates a transaction to send assets (e.g., wrapped tokens) to an address on Chain B. This transaction includes the destination address, amount, and other relevant details.
2.  **Message Hashing for Commitment/ID:** The `pallet-outbound-messages` serializes the core details of this outbound message (e.g., destination, amount, nonce). To create a unique identifier or a commitment for this message that will be used by the bridge relayer and Chain B, the pallet uses `avail_core::sha2::ShaTwo256::hash()` on this serialized message data. The pallet *intends* to generate a SHA2-256 hash.
    ```rust
    // Hypothetical pallet logic snippet (conceptual)
    // use avail_core::sha2::ShaTwo256;
    // use sp_core::Hasher;
    //
    // struct OutboundMessage { recipient: AccountIdOnChainB, amount: u128, nonce: u32 }
    // impl OutboundMessage {
    //     fn to_bytes(&self) -> Vec<u8> { /* ... serialization ... */ }
    // }
    //
    // fn process_outbound_message(message: OutboundMessage) -> H256 {
    //     let serialized_message = message.to_bytes();
    //     // Pallet developer INTENDS to use SHA2-256
    //     let message_hash = ShaTwo256::hash(&serialized_message);
    //     // This message_hash is stored on-chain or emitted in an event
    //     // for relayers to pick up.
    //     message_hash
    // }
    ```
    Due to the bug in `avail_core::sha2::ShaTwo256`, `message_hash` will actually be `keccak_256(serialized_message)`.

3.  **Relaying to Chain B:** A relayer service monitors Avail for these outbound message events. It picks up the `message_hash` (which is actually a Keccak-256 hash) and the message details. The relayer then attempts to submit this information to a bridge contract on Chain B.

4.  **Verification on Chain B:** The bridge contract on Chain B is designed to verify such messages. It might, for example, expect to recompute the hash of the message details using a standard SHA2-256 function and compare it with the `message_hash` provided by the relayer. Alternatively, this hash might be part of a signed attestation where the signature verification scheme on Chain B implicitly uses SHA2-256 for the message digest.

5.  **Hash Mismatch and Failure:**
    *   When the bridge contract on Chain B receives the message details and the `message_hash` (which is `keccak_256(data)`), it attempts to verify it by calculating `sha256(data)`.
    *   Since `keccak_256(data) != sha256(data)` (for virtually all inputs), the hash verification on Chain B will fail.
    *   The bridge contract on Chain B will reject the message and refuse to release the corresponding assets to the user on Chain B.

6.  **Fund Freezing/Loss:**
    *   The assets on the Avail side might have already been locked or burned by `pallet-outbound-messages` in anticipation of the transfer.
    *   Because the message cannot be verified and processed on Chain B due to the hash mismatch, the user's assets are effectively stuck or lost in the context of this cross-chain transfer.
    *   Rectifying this might require complex manual intervention, a bridge governance action, or even a hard fork of the Avail runtime if the `pallet-outbound-messages` has no mechanism to cancel or correct such failed outbound transfers once the incorrect hash is committed.

## 4. Mapping to Critical Impact

This scenario could potentially fall under:

*   **Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork):**
    *   If the `pallet-outbound-messages` (which is part of the WASM runtime) uses the faulty `ShaTwo256` for generating commitments that are then used in a bridge mechanism, and if rectifying the resulting stuck funds (due to hash mismatches on the receiving chain) requires a runtime upgrade (hard fork) to change how these commitments are generated or to introduce a recovery mechanism, then this category applies. The "exploit" here is the pallet's flawed logic stemming from the incorrect assumption about the hash function provided by `core/src`.
*   **Critical: WASM runtime exploits leading to direct loss of funds:**
    *   If the bridge mechanism involves burning tokens on Avail based on the (incorrectly calculated) hash and those tokens cannot be recovered or minted on the destination chain due to the verification failure, it could be considered a direct loss from the user's perspective.

## 5. Clarification of `core/src` vs. Runtime Responsibility

*   **`core/src` Responsibility:** `avail-core/core/src` is responsible for providing the `ShaTwo256` component that, due to a bug, does not function as its name and trait implementation imply (it produces Keccak-256 hashes instead of SHA2-256). This is a defect within `core/src`.
*   **Runtime Pallet Responsibility:** The actual fund freezing or loss vulnerability materializes because the hypothetical `pallet-outbound-messages` (part of the Avail runtime) *incorrectly assumes* that `avail_core::sha2::ShaTwo256` behaves as a standard SHA2-256 hasher and uses it in a cryptographically sensitive cross-chain communication protocol. The pallet's logic fails to account for the actual behavior of the component it consumes from `core/src`.

This PoC demonstrates a *consequence* of the `core/src` bug when integrated into a runtime pallet that makes standard assumptions about cryptographic primitives. It is not a self-contained exploit within `core/src` that directly manipulates funds; rather, `core/src` provides a faulty tool that, when misused (even if the misuse is based on a reasonable but incorrect assumption about the tool's behavior), can lead to critical failures in the runtime.

## 6. Reiteration of Direct Impact

While the above scenario illustrates a potential path to fund-related issues, the most direct and certain impact of the F1 bug, should the Avail runtime use `avail_core::sha2::ShaTwo256` for consensus-critical hashing (e.g., state roots, block content hashing), would be an **Unintended chain split**. This would occur if the bug is fixed in a future version, causing nodes with the fix to compute different hashes than nodes without the fix, leading to a fork in the blockchain.
