# Avail Core Protocol: Component Breakdown Part 2

This document details the Data Availability and Proof System components of the Avail core protocol, focusing on how data inclusion and availability are proven.

## Data Availability & Proof System

Avail employs a two-pronged approach for data integrity and availability:
1.  **Merkle Roots and Proofs (`TxDataRoots`, `DataProof`):** For proving the inclusion of specific data (like individual transactions or bridged messages) within a set of transactions.
2.  **Kate Commitments (`KateCommitment` in Header):** For proving the general availability of the bulk data in a block through sampling.

These systems are interconnected, notably through the `data_root` in `KateCommitment` which corresponds to the `blob_root` in `TxDataRoots`.

### `TxDataRoots`

*   **Purpose:** `TxDataRoots` (`core/src/data_proof.rs`) is a structure that aggregates Merkle roots for different categories of data processed within a block or a set of transactions. It serves as a consolidated cryptographic commitment to these data sets.
*   **Key Fields:**
    *   `data_root: H256`: A global Merkle root, typically computed as `keccak_256(blob_root, bridge_root)`. It provides a single top-level hash representing all data covered by the other two roots.
    *   `blob_root: H256`: The Merkle root of all general-purpose application data submitted to the chain (e.g., via `AppExtrinsic`). This is the data that is erasure-coded and covered by Kate commitments for data availability. This root is expected to match the `data_root` field within the `KateCommitment` found in the block header's extension.
    *   `bridge_root: H256`: The Merkle root specifically for data intended for or received from bridges to other blockchains (e.g., Ethereum). This allows for separate verification of bridged data.

### `DataProof`

*   **Purpose:** `DataProof` (`core/src/data_proof.rs`) provides evidence that a specific piece of data (a "leaf" in a Merkle tree) was included in the set of data committed to by either the `blob_root` or the `bridge_root` within `TxDataRoots`.
*   **Key Fields:**
    *   `roots: TxDataRoots`: Contains the set of roots (`data_root`, `blob_root`, `bridge_root`) that this proof is relevant to. The reconstructed root from the proof will be checked against one of these.
    *   `proof: Vec<H256>`: A list of sibling hashes from the Merkle tree. These are the necessary intermediate hashes that, along with the `leaf` hash, allow for the reconstruction of the path up to the specific Merkle root (`blob_root` or `bridge_root`).
    *   `number_of_leaves: u32`: The total number of leaves in the Merkle tree from which this proof was generated. This is essential for correctly validating the proof.
    *   `leaf_index: u32`: The 0-based index of the `leaf` in the original sequence of data items that formed the Merkle tree.
    *   `leaf: H256`: The hash of the specific data item (e.g., a transaction or a bridged message) whose inclusion is being proven.
*   **Mechanism for Proving Data Inclusion:**
    1.  A client obtains a `DataProof` for a piece of data it's interested in.
    2.  The client hashes its data to get the `leaf` value (or uses a provided `leaf` hash).
    3.  Using this `leaf`, `leaf_index`, the `proof` (sibling hashes), and `number_of_leaves`, the client algorithmically reconstructs a Merkle root.
    4.  This reconstructed root is then compared with the relevant root in the `DataProof.roots` structure (e.g., `roots.blob_root` for application data, or `roots.bridge_root` for bridged data).
    5.  If the roots match, the inclusion of the specific data item is cryptographically verified.

### Role of `kate_commitment.rs`

*   The file `core/src/kate_commitment.rs` defines the `v3::KateCommitment` struct. This struct is a critical part of Avail's data availability scheme and is embedded directly into the block header via `Header` -> `HeaderExtension` -> `v3::HeaderExtension`.
*   **`v3::KateCommitment` fields (`rows`, `cols`, `commitment`, `data_root`)** are used by light clients to perform Data Availability Sampling (DAS).
    *   `rows` and `cols` define the dimensions of the 2D matrix into which block data is arranged and erasure-coded.
    *   `commitment` is the actual Kate polynomial commitment (a set of cryptographic proofs, likely serialized elliptic curve points). Light clients sample random cells from this matrix and use the `commitment` to verify the correctness of the samples they receive.
    *   `data_root` (within `KateCommitment`) is a Merkle root of the original submitted data (before erasure coding and matrix arrangement). This root **must match** the `blob_root` from `TxDataRoots`. This linkage ensures that the data proven available by Kate commitments is the same data whose specific inclusion can be proven by a `DataProof`.

### Conceptual: `Cell` and `Proof` in Kate Commitments

While the `kate-recovery` crate details are not explored here, we can infer the roles of `Cell` and cell-specific `Proof` from the context of Kate commitments and the information in `DataProof` and `KateCommitment`:

*   **Data Matrix and `Cell`s:**
    *   The block's data (specifically, data corresponding to the `blob_root` / `KateCommitment.data_root`) is conceptually arranged into a 2D matrix of `rows` and `cols` (as specified in `KateCommitment`).
    *   A `Cell` represents a single unit of data at a specific coordinate `(row, col)` within this matrix. Each `Cell` contains a fragment of the total submitted data.
*   **Cell-Specific `Proof` (Kate Proof):**
    *   For a light client to verify a `Cell` during Data Availability Sampling (DAS), it would request the `Cell`'s content along with a corresponding `Proof`.
    *   This `Proof` is a cryptographic witness derived from the Kate commitment (the `commitment` field in `KateCommitment`). It allows the client to verify that the received `Cell` data is consistent with the overall Kate commitment for the entire data matrix published in the block header.
    *   By successfully verifying proofs for a sufficient number of randomly sampled cells, a light client gains high confidence that the entire dataset (represented by `KateCommitment.data_root`) is available.
*   **Distinction from `DataProof`:**
    *   **Kate Cell Proof:** Verifies a small piece of data (a `Cell`) against the *overall Kate commitment* in the header. Used for DAS to ascertain bulk data availability.
    *   **`DataProof` (Merkle Proof):** Verifies a *specific data item's hash* (e.g., a transaction) against the `blob_root` (or `bridge_root`). Used to prove a specific transaction is part of the dataset whose availability has been confirmed by DAS.

### Data Proof System Mermaid Diagram

```mermaid
graph TD
    subgraph BlockHeader ["Block Header"]
        HeaderExtension["HeaderExtension (V3)"] -- contains --> KC["v3::KateCommitment"]
        KC -- contains --> KCRows["rows: u16"]
        KC -- contains --> KCCols["cols: u16"]
        KC -- contains --> KCCommitment["commitment: Vec<u8> (Kate Points)"]
        KC -- contains --> KCDataRoot["data_root: H256 (Matches TxDataRoots.blob_root)"]
    end

    subgraph TransactionDataCommitments
        TDR["TxDataRoots"] -- contains --> TDRDataRoot["data_root: H256 (Global)"]
        TDR -- contains --> TDRBlobRoot["blob_root: H256 (For App Data)"]
        TDR -- contains --> TDRBridgeRoot["bridge_root: H256 (For Bridged Data)"]
    end

    subgraph ProofOfInclusion
        DP["DataProof"] -- contains --> DPRoots["roots: TxDataRoots"]
        DP -- contains --> DPProof["proof: Vec<H256> (Sibling Hashes)"]
        DP -- contains --> DPNumLeaves["number_of_leaves: u32"]
        DP -- contains --> DPLeafIndex["leaf_index: u32"]
        DP -- contains --> DPLeaf["leaf: H256 (Hash of specific data)"]
    end

    OriginalData["Original Submitted Data (e.g., AppExtrinsics)"]
    OriginalData -- Merkleized into --> TDRBlobRoot

    BridgedData["Original Bridged Data"]
    BridgedData -- Merkleized into --> TDRBridgeRoot

    TDRBlobRoot -- combined with --> TDRBridgeRoot
    TDRBridgeRoot -- to form --> TDRDataRoot

    DPLeaf -- part of --> OriginalData
    DPProof -- proves inclusion of --> DPLeaf
    DPProof -- against --> TDRBlobRoot_Ref["(TxDataRoots.blob_root in DataProof.roots)"]
    DPRoots -- references --> TDR

    KCDataRoot -- logically equals --> TDRBlobRoot

    subgraph ConceptualKateVerification ["Conceptual Kate Verification (DAS)"]
        direction LR
        DataMatrix["Data Matrix (rows x cols)"]
        Cell["Cell(row, col)"]
        CellProof["Kate Cell Proof"]
        DataMatrix -- contains --> Cell
        CellProof -- verifies --> Cell
        CellProof -- against --> KCCommitment
    end

    OriginalData -- arranged into --> DataMatrix


    classDef mainStruct fill:#lightgrey,stroke:#333,stroke-width:2px;
    class TDR,DP,KC mainStruct;

    classDef field fill:#white,stroke:#333;
    class KCRows,KCCols,KCCommitment,KCDataRoot,TDRDataRoot,TDRBlobRoot,TDRBridgeRoot,DPRoots,DPProof,DPNumLeaves,DPLeafIndex,DPLeaf field;

    TDRBlobRoot_Ref(TDRBlobRoot)
    style TDRBlobRoot_Ref fill:#lightblue,stroke:#333,stroke-width:1px,color:black
end

```
