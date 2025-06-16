# Avail Core Protocol: High-Level Architectural Overview

This document provides a high-level architectural overview of the Avail core protocol, detailing its main components and their interactions. Avail is a blockchain designed specifically for data availability, leveraging the Substrate framework and custom logic to achieve its goals.

## Main Components and Responsibilities

The Avail node architecture is built upon the Substrate framework, inheriting many of its modular components while integrating specialized data availability (DA) features.

1.  **Substrate Framework:**
    *   **Description:** Provides the foundational layer for Avail, including the P2P networking (`libp2p`), consensus mechanisms, transaction management, and the FRAME runtime environment.
    *   **Responsibilities:** Basic blockchain functionalities like block production, finality, peer discovery, transaction propagation, and a modular system for building blockchain logic (pallets).

2.  **Avail Runtime (State Transition Function):**
    *   **Description:** The core logic of the Avail blockchain, compiled to Wasm. It includes standard Substrate pallets and custom Avail-specific pallets that define how state transitions occur.
    *   **Responsibilities:**
        *   Processing and validating incoming transactions (extrinsics), especially data submissions.
        *   Executing the DA logic: organizing submitted data, applying erasure coding (via Kate commitments), and generating the necessary commitments and proofs.
        *   Managing blockchain state (accounts, balances, DA metadata).
        *   Interfacing with the consensus layer to include processed data into blocks.

3.  **Custom Data Availability (DA) Logic:**
    *   **Description:** This is the specialized set of modules and algorithms that differentiate Avail. It's deeply integrated into the runtime and core block processing. Key files like `da_block.rs`, `kate_commitment.rs`, `data_proof.rs`, and the `kate` library are central to this.
    *   **Responsibilities:**
        *   **Data Submission Handling:** Processing `app_extrinsic.rs` which allow applications to submit arbitrary data.
        *   **Erasure Coding & Commitments:** Using Kate commitments (polynomial commitments) to encode block data into a 2D matrix, allowing for efficient data reconstruction and verification. This ensures that even if parts of the data are missing, it can be recovered.
        *   **Data Structuring:** Organizing data within blocks (`da_block.rs`) in a way that is amenable to sampling and proof generation. This includes creating the data matrix and corresponding commitments.
        *   **Proof Generation & Verification:** Creating compact proofs of data availability that light clients can use to verify that data was indeed included in a block without downloading the entire block.
        *   **Data Lookup & Sampling:** Providing mechanisms (`data_lookup/mod.rs`) for nodes and light clients to query specific parts of the data or request random samples for DA verification.

4.  **P2P Network (based on libp2p):**
    *   **Description:** The communication layer connecting Avail nodes.
    *   **Responsibilities:**
        *   Propagating transactions (including data submissions) and blocks.
        *   Broadcasting consensus messages (BABE block proposals, GRANDPA votes).
        *   Distributing DA-specific information: this includes sharing parts of the data matrix, responding to sampling requests from other nodes (especially light clients), and disseminating data proofs.

5.  **Consensus Mechanism (BABE & GRANDPA):**
    *   **Description:** Standard Substrate consensus protocols.
        *   **BABE (Blind Assignment for Blockchain Extension):** Primary block production mechanism.
        *   **GRANDPA (GHOST-based Recursive ANcestor Deriving Prefix Agreement):** Finality gadget.
    *   **Responsibilities:**
        *   **BABE:** Selects block producers, who then create and propose new blocks containing transactions and the crucial DA commitments (e.g., Kate commitment root in the block header).
        *   **GRANDPA:** Finalizes the chain of blocks, ensuring that blocks containing specific data commitments are irreversibly part of the Avail ledger.
    *   **Interaction with DA:** Block headers are extended (`header/extension/`) to include commitments (e.g., Kate commitment) over the block's data, which is essential for DA verification.

6.  **Transaction Pool:**
    *   **Description:** An off-chain component that queues and validates incoming extrinsics before they are included in a block.
    *   **Responsibilities:** Managing pending transactions, protecting against spam, and ordering transactions for block inclusion. Data submission extrinsics pass through this pool.

7.  **RPC (Remote Procedure Call) Layer:**
    *   **Description:** An interface for external clients (e.g., light clients, dApps, wallets, block explorers) to interact with an Avail node.
    *   **Responsibilities:**
        *   Allowing submission of extrinsics (e.g., `data_submit`).
        *   Providing methods to query block information, DA metadata, retrieve data proofs, and check the availability status of data.

## Interactions and Data Flow

1.  **Data Submission:**
    *   An application submits data via an `app_extrinsic` through the RPC layer.
    *   The extrinsic enters the Transaction Pool, is validated, and propagated over the P2P network.
2.  **Block Production (BABE):**
    *   A BABE-selected block producer picks up transactions from the pool.
    *   The Runtime executes these transactions. For data submissions, the Custom DA Logic is invoked:
        *   Data is arranged into the DA matrix.
        *   Kate commitments are computed.
    *   A block is formed, with its header including the Kate commitment root and other DA metadata.
    *   The block is broadcast over the P2P network.
3.  **Data Availability & Verification:**
    *   Full nodes receive the block, validate it (including DA commitments), and store the data.
    *   Light clients can:
        *   Fetch block headers.
        *   Perform Data Availability Sampling (DAS) by requesting random cells from the DA matrix via the P2P network.
        *   Request and verify data proofs for specific data they are interested in.
4.  **Block Finalization (GRANDPA):**
    *   GRANDPA validators vote on chains of blocks.
    *   Once a block (and thus its DA commitments) is finalized, it's considered irreversible.

## Mermaid Diagram: Avail High-Level Architecture

```mermaid
graph TD
    subgraph External World
        UserApp[User Application / dApp]
        LightClient[Avail Light Client]
    end

    subgraph Avail Node
        RPC[RPC Layer]
        TxPool[Transaction Pool]

        subgraph Consensus
            BABE[BABE Block Production]
            GRANDPA[GRANDPA Finality]
        end

        subgraph Runtime [Avail Runtime (Wasm)]
            direction LR
            RuntimeCore[Core State Transition]
            Pallets[FRAME Pallets]
            subgraph CustomDALogic [Custom DA Logic]
                direction TB
                DataSubmission[Data Submission Handling]
                KateCommit[Kate Commitments & Erasure Coding]
                ProofGeneration[Proof Generation/Verification]
                DataMatrix[Data Matrix Construction]
                DataLookup[Data Lookup & Sampling Interface]
            end
        end

        P2P[P2P Network (libp2p)]
        BlockStorage[Block & State Storage]
    end

    %% Interactions
    UserApp -- Extrinsics (Data Submission) --> RPC
    LightClient -- Queries/Sampling/Proof Requests --> RPC
    RPC -- Validated Extrinsics --> TxPool
    RPC -- Queries --> RuntimeCore
    RPC -- Queries --> BlockStorage

    TxPool -- Transactions --> BABE

    BABE -- Proposed Blocks --> P2P
    BABE -- Execute Block --> Runtime
    Runtime -- Processed Block Data --> BABE
    RuntimeCore -- Uses --> Pallets
    RuntimeCore -- Orchestrates --> CustomDALogic
    CustomDALogic -- Commitments/Proofs --> RuntimeCore

    P2P -- Blocks/Transactions/Consensus Msgs --> BABE
    P2P -- Blocks/Transactions/Consensus Msgs --> GRANDPA
    P2P -- Blocks/Transactions --> TxPool
    P2P -- DA Sampling/Proof Data --> LightClient
    P2P -- DA Sampling/Proof Data --> RuntimeCore %% For nodes serving data

    GRANDPA -- Finalized Blocks --> BlockStorage
    BABE -- Blocks to Finalize --> GRANDPA
    Runtime -- State Changes/New Blocks --> BlockStorage

    %% Data Flow within DA Logic
    DataSubmission --> DataMatrix
    DataMatrix --> KateCommit
    KateCommit --> ProofGeneration
    DataMatrix --> DataLookup
    ProofGeneration --> DataLookup %% Proofs are available for lookup

    %% Link to Substrate Base (conceptual)
    classDef substrateBase fill:#f9f,stroke:#333,stroke-width:2px;
    class BABE,GRANDPA,P2P,TxPool,Pallets,RuntimeCore substrateBase;
    classDef availCustom fill:#ccf,stroke:#333,stroke-width:2px;
    class CustomDALogic,DataSubmission,KateCommit,ProofGeneration,DataMatrix,DataLookup,RPC availCustom;

end
```
