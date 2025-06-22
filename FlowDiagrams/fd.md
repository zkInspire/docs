# ** Detailed User Flow (With Uniswap, Zora, IPFS)**

```mermaid
sequenceDiagram
    autonumber
    participant User as User (Creator)
    participant Wallet as Wallet (EIP-712 Signer)
    participant IPFS as IPFS (Offchain Storage)
    participant zkCircuit as zkCircuit (zkSNARK Generator)
    participant Verifier as zkVerifier (Onchain)
    participant InspireGraph as InspireGraph (Zora/ERC-721 Ext.)
    participant Treasury as Treasury (Royalty Contract)
    participant Uniswap as Uniswap Router
    participant GraphQL as Indexer / UI (e.g., The Graph)

    %% 1. Upload Media
    User->>IPFS: Upload Creative Work (metadata.json + content)
    IPFS-->>User: contentURI (CID)

    %% 2. Declare Remix
    User->>Wallet: Sign Remix Declaration (parentId, contentURI)
    Wallet-->>User: Signature 

    %% 3. Generate Proof
    User->>zkCircuit: (parentHash, remixHash)
    zkCircuit-->>User: zkProof, publicSignals

    %% 4. Mint on Zora Protocol
    User->>Wallet: Sign tx (contentURI, proofHash, royaltyBP)
    Wallet->>InspireGraph: mintRemixNFT(contentURI, parentId, proof, royalties)

    %% 5. zkProof Validation
    InspireGraph->>Verifier: verifyProof(proof, publicSignals)
    Verifier-->>InspireGraph: Success 

    %% 6. Royalty Distribution
    InspireGraph->>Treasury: Split Royalties (creator, ancestors, platform)
    Treasury->>Uniswap: Auto-swap portion to $ZORA or $ETH
    Treasury->>Wallet: Royalty payout in $ZORA/$ETH

    %% 7. Index + Discovery
    InspireGraph-->>GraphQL: Emit InspirationAdded (Event → Index)
    GraphQL-->>User: Updated Inspiration Graph + Revenue Flows

```

---

##  **Component Breakdown**

| **Layer**        | **Technology**                         | **Role**                                                 |
| ---------------- | -------------------------------------- | -------------------------------------------------------- |
| **Storage**      | IPFS                                   | Media (images/audio/video) storage; returns CID.         |
| **Proof**        | zkSNARK                         | Cryptographic proof of derivation (optional zkML later). |
| **NFT Mint**     | Zora Protocol (ERC-721 & Extensions)   | Tokenizing creative works as NFTs with inspiration link. |
| **Royalties**    | Custom contract + Zora/Uniswap         | Splitting & swapping royalties → \$ZORA, \$ETH.          |
| **Verification** | Onchain zkVerifier (Solidity Verifier) | Validating zkProof commitments on mint.                  |
| **Discovery**    | The Graph (or custom GraphQL)          | Indexing inspiration graph for search/discovery.         |

---

##  **Royalty Flow with Uniswap Integration**

1. **X% Royalty** → Sent to parent creators (based on DAG depth & declared splits).
2. **Y% Platform Fee** → Converted to \$ZORA via Uniswap.
3. **Referrer Cut (Optional)** → Routed to referrer wallet.
4. **Remaining Royalties** → Direct payout to remix creator in \$ETH or \$ZORA.

**Royalty Splitting Example:**

```plaintext
Royalty: 10%
 → 5% → Immediate parent creator
 → 2% → Ancestors (depth-based)
 → 2% → Platform (swapped to $ZORA)
 → 1% → Referrer (if any)
```

---

