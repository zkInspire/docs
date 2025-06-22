##  2. Inspiration Graph: Technical Data Model

### 2.1 Graph Representation

The **Inspiration Graph** is formalized as a **Directed Acyclic Graph (DAG)**:

$$
\mathcal{G} = (\mathcal{N}, \mathcal{E})
$$

* **𝓝 (Nodes)** → Canonical, unique representations of minted creative works.

  * **Canonicalization** → Performed via **Keccak-256** hashing of normalized metadata.
  * Identity = `id = Keccak256(IPFS_CID || creatorAddress || timestamp)`
* **𝓔 (Edges)** → Directed inspiration links:

$$
e_{j \to i} \in \mathcal{E} : I_j \to I_i
$$

* Each edge **must** be cryptographically provable via commitment to an onchain **zk-SNARK proof**.
* **Acyclic Constraint** → Cycles invalid → enforced at verification level to prevent infinite recursive payouts.

---

### 2.2 Node Metadata Schema (EVM-Indexed)

#### Solidity-Friendly Metadata Schema (Struct):

```solidity
struct InspirationNode {
    bytes32 id;                // keccak256(IPFS || creator || timestamp)
    address creator;           // EOA or contract
    bytes32 parent;            // Parent node hash (0x0 for originals)
    uint8 depth;               // Depth in DAG
    bytes32 proofHash;         // Commitment to zkProof onchain/offchain
    bool declaredInspiration;  // Explicit declaration
    uint16 royaltyShareBP;     // Basis points (e.g., 500 = 5%)
    uint256 timestamp;         // Unix epoch
}
```

* **Gas Optimizations**: `uint8`, `uint16`, and packing for efficient storage.
* **Canonicalization Contract** → Optional pre-deployment content integrity check.

---

### 2.3 Proof Integrity and Verification System

#### a) **zkProof System Design**

* **`proofHash`** → SHA256 commitment of zkSNARK output.
* zk Circuit → Takes as inputs:

→ Computes a **proof of derivation** where:

1. `parent_hash` → Keccak256 of parent content
2. `child_CID` → IPFS CID (content address)
3. `salt` → Random nonce for uniqueness in commitments

>  **Zero-Knowledge Privacy** → The **content of derivation is hidden**; only the proof verifies cryptographically that derivation took place from parent to child.

---

#### b) **Advanced zkML Proofs (Future Planned)**

* **zkML Integration (WIP)** → Use zkSNARKs to encode *style similarity* or *content feature extraction*.

  * Example circuits → **ResNet embeddings**, **audio fingerprints**, or **text semantic embeddings**.
  * Prove **“inspiration proximity”** in latent feature spaces **without revealing actual embeddings**.

```plaintext
Input → Embedding(child) - Embedding(parent) → zk circuit → ≤ threshold → Proof
```



---

#### c) **Onchain Verification Pipeline:**

```plaintext
1. submitProof(bytes calldata proof, bytes32 parent, bytes32 child)
2. zkVerifier.verify(proof, parent, child) → returns (bool)
3. emit InspirationProven(child, parent, msg.sender)
```

* **Verifier Contract** → Modular, supports circuit upgrades.
* **Proof Hash Storage** → Immutable commitment → protects against forgery or post-hoc manipulation.
---

