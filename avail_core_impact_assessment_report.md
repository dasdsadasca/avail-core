# Avail Core Protocol: Consolidated Impact Assessment Report for `avail-core/core/src`

## 1. Introduction

**Purpose:**
This document assesses the potential impact of the previously identified findings (F1-F6) from the security analysis of `avail-core/core/src`. The impact is evaluated against a predefined set of Critical and High impact categories, focusing on how vulnerabilities or weaknesses within `core/src` could contribute to these severe outcomes, either directly or through integration flaws in the consuming Avail runtime.

**Scope:**
The assessment focuses on impacts stemming from issues within `avail-core/core/src`. It differentiates between impacts that could arise directly from `core/src`'s logic or API contracts and those that are contingent upon how the consuming Avail runtime integrates and utilizes the `avail-core` library.

## 2. Consolidated Impact Assessment

---

### Finding F1: Incorrect Hash Function in `ShaTwo256`

*   **Relevant Impact Categories:**
    *   `[ ]` Critical: WASM runtime exploits leading to direct loss of funds
    *   `[ ]` Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork)
    *   `[ ]` High: RPC API crash affecting projects with >= 25% market capitalization on top of the respective layer
    *   `[ ]` High: Causing network processing nodes to process transactions from the mempool beyond set parameters
    *   `[ ]` High: Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
    *   `[X]` High: Unintended chain split (network partition)
*   **Justification:**
    The `ShaTwo256::hash` function in `core/src/sha2.rs` incorrectly uses `keccak_256`. If the Avail runtime uses this `ShaTwo256` implementation for any consensus-critical hashing (e.g., calculating state roots, transaction Merkle roots that form part of the block hash, or the block hash itself) and this bug is later corrected, nodes running different versions of the code (pre-fix and post-fix) would compute different hashes for the same data. This divergence in hash computation for consensus-critical elements would inevitably lead to an unintended chain split, as nodes would disagree on the validity of blocks. This impact is a direct consequence of `core/src` providing a misbehaving component that a runtime might reasonably use for a standard cryptographic purpose. Other listed critical/high impacts are less direct or unlikely to be solely caused by this specific bug within `core/src`.

---

### Finding F2: `AppId` Handling and Defaulting

*   **Relevant Impact Categories (Contingent on Runtime):**
    *   `[X]` Critical: WASM runtime exploits leading to direct loss of funds
    *   `[X]` Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork)
    *   `[ ]` High: RPC API crash...
    *   `[X]` High: Causing network processing nodes to process transactions from the mempool beyond set parameters
    *   `[ ]` High: Temporary freezing of network transactions by delaying one block...
    *   `[ ]` High: Unintended chain split (network partition)
*   **Justification:**
    The `core/src` module allows `AppId` to default to `AppId(0)`. While `core/src` itself doesn't assign special privileges to `AppId(0)`, if the *consuming runtime* implements flawed logic where `AppId(0)` (or any specific `AppId` that can be achieved via default) bypasses permission checks, has incorrect fee logic, or interacts with privileged pallets (e.g., treasury, staking) in an unintended way, this could be exploited. For instance, if a pallet allows fund transfers or state changes only for specific `AppId`s and `AppId(0)` is mistakenly given administrative rights or bypasses checks, it could lead to fund loss/freezing. Similarly, if `AppId(0)` transactions are processed with significantly lower weight or bypass certain validation steps, it could lead to nodes processing transactions beyond fair parameters. The exploitability is entirely dependent on the runtime's specific handling of `AppId`s.

---

### Finding F3: Nested Extrinsic Processing (`AppExtrinsic.data`)

*   **Relevant Impact Categories (Contingent on Runtime):**
    *   `[X]` Critical: WASM runtime exploits leading to direct loss of funds
    *   `[X]` Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork)
    *   `[ ]` High: RPC API crash...
    *   `[X]` High: Causing network processing nodes to process transactions from the mempool beyond set parameters (resource exhaustion)
    *   `[X]` High: Temporary freezing of network transactions by delaying one block... (computational DoS)
    *   `[X]` High: Unintended chain split (network partition) (if non-deterministic handling of nested calls or errors)
*   **Justification:**
    `core/src` defines `AppExtrinsic.data` as `Vec<u8>`, allowing for nested extrinsics if the runtime chooses to decode and dispatch them. If the runtime does not implement robust controls (e.g., recursion depth limits, weight accounting for inner calls, proper error handling and atomicity), various attacks become possible:
    *   **WASM Exploits (Critical):** If the inner extrinsic calls a vulnerable WASM pallet, it could lead to fund loss or freezing.
    *   **DoS (High):**
        *   Unbounded recursion or computationally intensive inner calls could cause nodes to exceed processing parameters or significantly delay block production.
    *   **Chain Split (High):** If the runtime's handling of errors or non-determinism within nested calls leads to different nodes reaching different states.

---

### Finding F4: `HeaderExtension` Integrity and Construction

*   **Relevant Impact Categories (Contingent on Runtime):**
    *   `[ ]` Critical: WASM runtime exploits leading to direct loss of funds
    *   `[X]` Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork) (in extreme cases of DA layer failure or state corruption requiring a hard fork to resolve)
    *   `[ ]` High: RPC API crash...
    *   `[X]` High: Causing network processing nodes to process transactions from the mempool beyond set parameters (e.g., due to oversized DA commitments if not limited by runtime)
    *   `[X]` High: Temporary freezing of network transactions by delaying one block... (if DA computations become too slow due to unconstrained parameters)
    *   `[X]` High: Unintended chain split (network partition)
*   **Justification:**
    The integrity of `HeaderExtension` is vital. If the runtime:
    *   Fails to ensure `KateCommitment.data_root` matches the actual `blob_root`.
    *   Does not constrain DA parameters like `rows`, `cols`, or `app_lookup` size.
    *   Mishandles errors from `DataLookup` construction.
    This can lead to:
    *   **DA Layer Failure (Critical if severe):** Light clients cannot verify data, or invalid data is accepted. A fundamental failure here could necessitate a hard fork.
    *   **DoS (High):** Maliciously crafted transactions could lead to oversized headers or computationally intensive DA processing, delaying blocks or exceeding processing parameters.
    *   **Chain Split (High):** If nodes have different interpretations or validation rules for `HeaderExtension` components due to runtime bugs or inconsistencies.

---

### Finding F5: General Robustness - Runtime Error Handling & Resource Limits

*   **Relevant Impact Categories (Contingent on Runtime):**
    *   `[ ]` Critical: WASM runtime exploits leading to direct loss of funds
    *   `[X]` Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork) (if runtime panics on errors from `core/src` lead to persistent state corruption)
    *   `[X]` High: RPC API crash affecting projects with >= 25% market capitalization on top of the respective layer (if node crashes due to unhandled errors)
    *   `[X]` High: Causing network processing nodes to process transactions from the mempool beyond set parameters (if runtime fails to limit resources for DA processing)
    *   `[X]` High: Temporary freezing of network transactions by delaying one block... (due to node crashes or resource exhaustion)
    *   `[X]` High: Unintended chain split (network partition) (if error handling is non-deterministic across nodes or leads to state divergence)
*   **Justification:**
    While `core/src` generally uses `Result`/`Option`, the runtime's handling of these is crucial.
    *   **Node Crashes (High - RPC, Block Delay):** If the runtime panics on an error returned by a `core/src` API, nodes can crash, affecting RPC availability and block production.
    *   **Resource Exhaustion DoS (High):** If the runtime doesn't enforce resource limits on data passed to `core/src` (e.g., for DA matrix construction, `app_lookup` size), it can lead to excessive processing times or memory usage, potentially delaying blocks or allowing nodes to exceed normal processing parameters.
    *   **State Corruption/Chain Split (Critical/High):** Inconsistent error handling or panics leading to state corruption could, in worst-case scenarios, freeze funds or cause chain splits.

---

### Finding F6: `BenchRandomness` Misuse Potential

*   **Relevant Impact Categories (Contingent on Runtime):**
    *   `[X]` Critical: WASM runtime exploits leading to direct loss of funds
    *   `[X]` Critical: WASM runtime exploits leading to permanent freezing of funds (fix requires hardfork)
    *   `[ ]` High: RPC API crash...
    *   `[ ]` High: Causing network processing nodes to process transactions from the mempool beyond set parameters
    *   `[ ]` High: Temporary freezing of network transactions by delaying one block...
    *   `[X]` High: Unintended chain split (network partition) (less direct, but possible if used in non-deterministic ways in consensus-adjacent logic, or if predictability leads to exploitable block production patterns)
*   **Justification:**
    The `core/src/bench_randomness.rs` module itself is not flawed for its intended purpose. However, if the *runtime* were to misuse this deterministic randomness source for any security-critical operation (e.g., key generation, nonce selection, leader election, shuffling validators, or any application logic within a pallet that relies on randomness for fairness or security), the consequences could be severe. Predictable outcomes in these areas can lead to:
    *   **Fund Loss/Freezing (Critical):** For example, if used in a poorly designed lottery pallet or for generating predictable private keys.
    *   **Consensus Manipulation (High/Critical):** If used in leader election or other consensus mechanisms, allowing attackers to predict or influence outcomes.
    *   **Chain Splits (High):** If its use leads to non-deterministic behavior across different nodes under certain conditions, although this is less direct than other causes.

## 3. Summary Conclusion

The `ShaTwo256` bug (F1) within `avail-core/core/src` directly presents a **High** risk of causing an "Unintended chain split (network partition)" if the runtime utilizes this component for consensus-critical hashing and the bug is subsequently fixed, leading to version incompatibilities.

For the remaining findings (F2, F3, F4, F5, F6), the potential for them to escalate to the specified "Critical" or "High" impact categories is primarily **contingent on the implementation, integration choices, and security diligence of the consuming Avail runtime**. The `avail-core/core/src` library provides foundational elements and APIs; however, without robust validation, resource limiting, correct error handling, and secure design patterns within the runtime, these elements could be misused or become vectors for the listed severe outcomes. The most significant risks involve WASM exploits via nested extrinsics, DA layer failures due to improper header construction, and various DoS or state inconsistency issues stemming from poor resource management or error handling by the runtime.
