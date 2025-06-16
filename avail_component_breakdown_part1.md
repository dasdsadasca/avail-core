# Avail Core Protocol: Component Breakdown Part 1

This document details the structure of blocks and application-specific transactions within the Avail core protocol.

## 1. Block Structure

The block structure in Avail is based on Substrate's generic block format but extended to include data availability specific information crucial for Avail's functionality.

### `DaBlock`

*   **Purpose:** `DaBlock<Header, Extrinsic>` is the fundamental container for a block in Avail. It's a generic structure that holds the block's header and the list of extrinsics (transactions) included in that block.
*   **Key Fields:**
    *   `header: Header`: Contains all the metadata for the block, including standard Substrate header fields and Avail's custom data availability extensions.
    *   `extrinsics: Vec<Extrinsic>`: A vector of extrinsics. These are the transactions processed and included in this block, which notably include `AppExtrinsic` for data submissions.

### `Header`

*   **Purpose:** `Header<N, H>` (where `N` is the block number type and `H` is the hash type) stores metadata about the block. It follows the standard Substrate header format but is critically augmented with an `extension` field for DA-specific data.
*   **Key Fields (Standard Substrate):**
    *   `parent_hash: H::Output`: The hash of the parent block.
    *   `number: N`: The height of this block in the chain.
    *   `state_root: H::Output`: The Merkle root of the blockchain's state trie after applying changes from this block.
    *   `extrinsics_root: H::Output`: The Merkle root of the extrinsics included in this block.
    *   `digest: Digest`: A list of cryptographic digests, often including consensus engine specific data (e.g., BABE pre-runtime digest, seal).
*   **Key Field (Avail Specific):**
    *   `extension: HeaderExtension`: This field contains Avail's data availability specific information.

### `HeaderExtension`

*   **Purpose:** `HeaderExtension` is an enum designed to encapsulate DA-related metadata within the block header. It is versioned to allow for future upgrades to the DA layer's header information. Currently, it primarily uses `V3`.
*   **Variants:**
    *   `V3(v3::HeaderExtension)`: Represents version 3 of the header extension. The actual DA metadata is contained within this variant.

### `v3::HeaderExtension`

*   **Purpose:** This struct holds the actual DA metadata for version 3 of the header extension.
*   **Key Fields:**
    *   `app_lookup: DataLookup`: A structure that provides a lookup mechanism for application-specific data within the block. It likely maps `AppId`s to information about the location or size of data submitted by each application, enabling efficient retrieval or verification of that data.
    *   `commitment: v3::KateCommitment`: Contains the Kate commitment for the block's data, along with related parameters.

### `v3::KateCommitment`

*   **Purpose:** This struct encapsulates all information related to the Kate polynomial commitment scheme used by Avail for data availability.
*   **Key Fields:**
    *   `rows: u16`: The number of rows in the 2D matrix representation of the block's data used for erasure coding and Kate commitments.
    *   `cols: u16`: The number of columns in the 2D data matrix. The product `rows * cols` indicates the total number of data cells.
    *   `commitment: Vec<u8>`: The serialized Kate commitment itself. These are cryptographic proofs (collections of elliptic curve points) that allow verifiers to check data availability without downloading the entire block.
    *   `data_root: H256`: A Merkle root of all submitted data that has been erasure-coded and committed to. This ensures that the Kate commitment corresponds to the actual data submitted to the block.

### Block Structure Mermaid Diagram

```mermaid
graph TD
    DaBlock["DaBlock<Header, Extrinsic>"] -- contains --> BlockHeader["Header<N, H>"]
    DaBlock -- contains --> Extrinsics["Vec<Extrinsic> (e.g., AppExtrinsic)"]

    BlockHeader -- contains --> ParentHash["parent_hash: H::Output"]
    BlockHeader -- contains --> Number["number: N"]
    BlockHeader -- contains --> StateRoot["state_root: H::Output"]
    BlockHeader -- contains --> ExtrinsicsRoot["extrinsics_root: H::Output"]
    BlockHeader -- contains --> Digest["digest: Digest"]
    BlockHeader -- contains --> Extension["extension: HeaderExtension"]

    Extension -- is an enum --> V3HeaderExt["V3(v3::HeaderExtension)"]

    V3HeaderExt -- contains --> AppLookup["app_lookup: DataLookup"]
    V3HeaderExt -- contains --> KateCommitmentStruct["commitment: v3::KateCommitment"]

    KateCommitmentStruct -- contains --> Rows["rows: u16"]
    KateCommitmentStruct -- contains --> Cols["cols: u16"]
    KateCommitmentStruct -- contains --> Commitment["commitment: Vec<u8> (Serialized Points)"]
    KateCommitmentStruct -- contains --> DataRoot["data_root: H256"]

    classDef struct fill:#lightgrey,stroke:#333,stroke-width:2px;
    class DaBlock,BlockHeader,Extension,V3HeaderExt,KateCommitmentStruct struct;
end
```

## 2. Transaction Handling (`AppExtrinsic`)

Avail uses a specialized extrinsic format, `AppExtrinsic`, to handle data submissions from different applications.

### `AppExtrinsic`

*   **Purpose:** `AppExtrinsic` (`core/src/app_extrinsic.rs`) is a wrapper around the raw data of an extrinsic, crucially associating it with an `AppId`. This allows Avail to identify which application submitted a particular piece of data and is fundamental for tracking and looking up application-specific data within blocks.
*   **Key Fields:**
    *   `app_id: AppId`: An identifier (likely a unique number) for the application that is submitting the data. This allows the system to differentiate data from various sources. `AppId(0)` might be reserved for general chain data or default submissions.
    *   `data: Vec<u8>`: The actual payload of the extrinsic, i.e., the data submitted by the application. This can be arbitrary binary data that the application wants to make available.
*   **Relation to Data Submission:**
    *   When an application wishes to submit data to be made available by Avail, it (or a client acting on its behalf) constructs an extrinsic. This extrinsic is then wrapped into or represented as an `AppExtrinsic`, including the application's `AppId` and the data payload.
    *   The `app_id` is used to populate the `app_lookup` table in the `HeaderExtension`, enabling light clients and other users to efficiently query for data belonging to specific applications.
    *   The `data` part is included in the block's body, erasure-coded, and covered by the Kate commitment in the block header.

### `AppExtrinsic` Mermaid Diagram

```mermaid
graph TD
    AppExtrinsic["AppExtrinsic"] -- contains --> AppIdField["app_id: AppId"]
    AppExtrinsic -- contains --> DataField["data: Vec<u8>"]

    classDef struct fill:#lightgrey,stroke:#333,stroke-width:2px;
    class AppExtrinsic struct;
end
```
