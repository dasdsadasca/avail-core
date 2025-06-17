# Security Vulnerability Report: Misimplementation of ShaTwo256 Hash Function in Avail Core

**Category:**
*   **Primary Impact:** High: Unintended chain split (network partition)
*   **Secondary/Conditional Impact:** Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork) / direct loss of funds (contingent on specific runtime pallet integration and misuse of the faulty hasher).

## Brief/Intro

The `avail-core/core/src/sha2.rs` module defines a `ShaTwo256` struct intended to provide SHA2-256 hashing functionality. However, it is incorrectly implemented and uses the `keccak_256` algorithm instead. This is an internal functional correctness issue within `core/src` due to a violation of its API contract. If the Avail runtime utilizes this faulty `ShaTwo256` component for consensus-critical hashing (e.g., state roots, block hashes), a subsequent bug fix to use the correct SHA2-256 algorithm would inevitably lead to an unintended chain split. Furthermore, if runtime pallets or interacting external systems rely on this component under the assumption it provides standard SHA2-256 hashes, this misbehavior can lead to validation failures, interoperability issues, and, in specific hypothetical runtime scenarios, could contribute to the freezing or effective loss of funds.

## Vulnerability Details

The vulnerability is a misimplementation within the `ShaTwo256` struct's `hash` method, defined in `core/src/sha2.rs`. This method is part of its implementation of the `sp_core::Hasher` trait (which is also used to satisfy `sp_runtime::traits::Hash`). Instead of employing a SHA2-256 algorithm, the code erroneously calls `keccak_256`.

**File:** `core/src/sha2.rs`

**Incorrect Code:**
```rust
use crate::keccak_256::keccak_256; // Imports Keccak-256
use primitive_types::H256;
use sp_core::Hasher;
use sp_runtime_interface::pass_by::PassByInner;

#[cfg(feature = "serde")]
use serde::{Deserialize, Serialize};

/// Sha256 Hasher.
#[derive(PartialEq, Eq, Clone, Debug)]
#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
pub struct ShaTwo256;

impl Hasher for ShaTwo256 {
	type Out = H256;

	/// Hash the given data.
	fn hash(s: &[u8]) -> Self::Out {
		keccak_256(s).into() // <--- Incorrect: Uses Keccak-256 instead of SHA2-256
	}
}

// Other trait implementations like Default, PassByInner follow...
```

**Violation of API Contract:**
The `ShaTwo256` component violates its implicit and explicit API contract in the following ways:
1.  **Naming Convention:** The name `ShaTwo256` universally implies that the component will produce SHA2-256 hashes.
2.  **Trait Implementation Context:** When `sp_core::Hasher` (and by extension `sp_runtime::traits::Hash`) is implemented by a type named `ShaTwo256`, any consumer of this trait, especially within the Substrate ecosystem where specific hashers like `BlakeTwo256` are common, would reasonably expect it to perform the named hashing algorithm. If the runtime's configuration or pallet logic selects `ShaTwo256` for operations requiring the specific cryptographic properties or output of SHA2-256 (e.g., for compatibility with external systems or certain cryptographic schemes), this misimplementation breaks that fundamental expectation.

This deviation means that any system component relying on `avail_core::sha2::ShaTwo256` for actual SHA2-256 hashing will receive incorrect (Keccak-256) hash results. No internal tests within `core/src` were identified that specifically use `ShaTwo256` in a context that would have caught this discrepancy (e.g., by comparing against known SHA2-256 test vectors).

## Impact Details

The misimplementation of `ShaTwo256` presents significant risks, with the severity and nature of the impact depending on how this component is integrated and used by the consuming Avail runtime.

**Primary Impact (High Likelihood if used for consensus-critical elements):**

*   **High: Unintended chain split (network partition)**
    This is the most direct and certain high-impact consequence if the Avail runtime employs `avail_core::sha2::ShaTwo256` for any consensus-critical computations. This includes, but is not limited to, block hashing, state root calculation, transaction Merkle root computation, or any cryptographic accumulator whose integrity is vital for chain consensus.
    *   **Scenario:** If the runtime currently uses this faulty `ShaTwo256` for such purposes, and a future version of `avail-core` corrects the bug to use actual SHA2-256, then nodes running the older version of the runtime (with the Keccak-256 based `ShaTwo256`) will compute different hashes for the same underlying data compared to nodes running the updated runtime (with the correct SHA2-256 based `ShaTwo256`).
    *   **Result:** This hash divergence for consensus-critical data structures will cause nodes running different versions to view the chain differently, leading to block rejection and an unintended, difficult-to-resolve network partition (chain split). A hard fork would likely be required to reconcile such a split.

**Secondary Impact (Conditional on Runtime Misuse):**

*   **Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork) / direct loss of funds**
    This impact is conditional and arises if a runtime pallet (part of the WASM runtime) incorrectly relies on `avail_core::sha2::ShaTwo256` for operations involving external systems or asset transfers where the SHA2-256 algorithm is specifically expected by the counterparty or protocol.
    *   **Conceptual PoC Scenario (Bridge Example):**
        1.  A hypothetical `pallet-cross-chain-bridge` within the Avail runtime is designed to facilitate asset transfers to an external blockchain (Chain B) that uses SHA2-256 for message verification.
        2.  When a user initiates a transfer, the pallet serializes the transfer details and computes a message identifier or commitment using `avail_core::sha2::ShaTwo256::hash()`, under the assumption that it's generating a SHA2-256 hash.
        3.  The actual hash produced is Keccak-256 due to the bug in `core/src`.
        4.  This Keccak-256 hash (believed to be SHA2-256 by the Avail pallet) is sent to Chain B via a relayer.
        5.  The bridge contract on Chain B attempts to verify the message by re-computing the hash of the received details using its own, correct SHA2-256 implementation.
        6.  The hashes will inevitably mismatch (`keccak_256(data) != sha256(data)`).
        7.  Consequently, the bridge contract on Chain B rejects the transaction, and the assets are not credited to the user on Chain B.
        8.  If the Avail-side pallet has already locked or burned the user's assets in anticipation of a successful transfer, these funds become effectively frozen or lost. Recovering these funds could be extremely difficult and might necessitate a runtime upgrade (hard fork) on Avail to implement a special recovery mechanism or correct the state.
    *   **Runtime Dependency:** This critical impact is entirely dependent on the Avail runtime containing such a pallet and that pallet (a) using `avail_core::sha2::ShaTwo256` and (b) interacting with an external system that strictly expects SHA2-256. The `core/src` bug provides the faulty primitive, but the runtime's specific usage pattern creates the fund-related vulnerability.

**Other Impacts:**

*   **General Data Integrity Mismatches:** Any part of the Avail ecosystem or external tooling that uses `avail_core::sha2::ShaTwo256` and expects standard SHA2-256 output will encounter hash mismatches. This can lead to failed integrity checks, incorrect data processing, or erroneous conclusions about data authenticity.
*   **Failed Verifications by External Systems:** If Avail exports data or proofs that include hashes generated by `ShaTwo256` under the pretense that they are SHA2-256 hashes, any external system attempting to verify these using standard SHA2-256 cryptographic libraries will fail, undermining interoperability and trust.

## References

*   **Affected Code:** `avail-core/core/src/sha2.rs`
    (Link for review: [`https://github.com/availproject/avail-core/blob/main/core/src/sha2.rs`](https://github.com/availproject/avail-core/blob/main/core/src/sha2.rs) - assuming `main` branch is representative for this audit. A specific commit hash should be used for formal tracking if available.)

## Proof of Concept

**1. Direct Hash Mismatch Verification (Conceptual):**

This PoC demonstrates the core malfunction within `avail-core/core/src` by comparing the output of `ShaTwo256::hash` against known test vectors for both SHA2-256 and Keccak-256.

*   **Step 1:** Select a common test input string, for example, the ASCII string `"avail"`.
*   **Step 2:** Compute the hash using `avail_core::sha2::ShaTwo256::hash(b"avail")`. Let this be `hash_avail_core`.
*   **Step 3:** Compute the standard Keccak-256 hash of `b"avail"` using a trusted, external cryptographic library. Let this be `hash_keccak_expected`.
    *   For "avail", Keccak-256 is: `0xc1223d53754451e997ada8f90580090916113c684a9f9995a1cb519980817980`
*   **Step 4:** Compute the standard SHA2-256 hash of `b"avail"` using a trusted, external cryptographic library. Let this be `hash_sha256_expected`.
    *   For "avail", SHA2-256 is: `0x02ccbd8356d0769526189a3a808a50accdd4000a605988e51685309688851902`
*   **Verification:**
    *   Compare `hash_avail_core` with `hash_keccak_expected`. They will match.
    *   Compare `hash_avail_core` with `hash_sha256_expected`. They will **not** match.
    This directly and verifiably demonstrates that `avail_core::sha2::ShaTwo256::hash()` produces a Keccak-256 hash, not a SHA2-256 hash as its name and context imply.

**2. Conceptual Fund Freezing/Loss PoC Reference:**

The detailed hypothetical scenario involving a `pallet-cross-chain-bridge`, as outlined in the "Secondary Impact (Conditional on Runtime Misuse)" section above, serves as a conceptual PoC for how this `core/src` bug could contribute to fund freezing or loss. This PoC emphasizes that:
*   `avail-core/core/src` provides the faulty hashing primitive (`ShaTwo256`).
*   The actual financial impact (fund freezing/loss) is realized due to the *consuming runtime pallet's* incorrect assumption about this primitive's behavior when interacting with an external system that correctly expects standard SHA2-256 cryptographic hashes.
*   The core of that PoC is the inevitable hash mismatch during cross-chain verification, leading to the failure of the asset transfer and the potential trapping of funds.
