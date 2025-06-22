# 6 Tokenomics & Royalties

## 6.1 Royalty Calculation Model

Royalties for **derivative works** (remixes/inspired creations) are **programmatically enforced** via **onchain logic** embedded in minting contracts.

### **Royalty Formula:**

For any derivative content **Cₙ** inspired by parent **Cₙ₋₁**:

$$
\text{Royalty}(Cₙ) = P \cdot S(Cₙ₋₁) \cdot R(Cₙ₋₁)
$$

Where:

| Symbol      | Meaning                                                     |
| ----------- | ----------------------------------------------------------- |
| **P**       | Sale price of derivative content                            |
| **S(Cₙ₋₁)** | Royalty share in **Basis Points (BP)**, set by parent node  |
| **R(Cₙ₋₁)** | Reputation Multiplier (1.0 → 2.0) for incentivized creators |

---

### Example:

| Parameter             | Value       |
| --------------------- | ----------- |
| **Sale Price (P)**    | 1.0 ETH     |
| **Royalty Share (S)** | 300 BP → 3% |
| **Reputation Mult.**  | 1.2         |
| → **Royalty Paid**    | 0.036 ETH   |

---

## 6.2 Multi-Hop Royalty Splitting (Recursive)

For deep remix chains, **recursive royalty calculation** applies proportionally:

$$
\text{Royalty}_{\text{total}} = \sum_{i=1}^d (P \cdot S(C_i) \cdot R(C_i))
$$

* **Depth (d)** → Number of ancestor nodes in DAG.
* **Recursive Cap** → Set via governance; e.g., **max 3 hops** to avoid infinite recursion or micro-distributions.

### Onchain Example Flow:

```plaintext
Derivative NFT Sale → Contract → Traverses DAG → Splits payout to:
  - Immediate Parent(s)
  - Ancestor Nodes (if depth > 1)
  - Platform Fee Recipient
```

---

## 6.3 Role Responsibilities

| Role                 | Responsibility                                      | Reward Structure                            |
| -------------------- | --------------------------------------------------- | ------------------------------------------- |
| **Original Creator** | Right to set `royaltyShareBP` in minted metadata    | Receives upstream royalties from remixes    |
| **Inspired Creator** | Must declare `declaredInspiration=true` if remixing | Royalty from their own derivative sales     |
| **Fraud Detectors**  | Can submit fraud claims with proof                  | Receives **bounty** upon confirmation (DAO) |

---

## 6.4 Incentive Mechanisms

### 6.4.1 Platform Fees (Protocol Sustainability)

* Flat platform fee (e.g., **2.5%** of sale) directed to protocol treasury DAO.
* Can be redirected to staking pools for creators → **yield on creative work**.

### 6.4.2 Referrer Bonuses

* Referrals tracked via ephemeral **referral tokens** signed with **EIP-712 signatures**.
* Referrer receives **1–3%** of transaction volume depending on activity tier.

```plaintext
[User Clicks Referral Link] → Generates ephemeral EIP-712 Signature → Used at Checkout → Referrer Paid
```

---

## 6.5 Implementation: Solidity Architecture

```solidity
struct RoyaltySplit {
    address payable recipient;
    uint16 shareBP;
    uint8 depth;
    bytes32 inspirationId;
}
```

### Payout Function Flow:

```solidity
function distributeRoyalty(uint256 salePrice, RoyaltySplit[] calldata splits) external payable {
    for (uint i = 0; i < splits.length; i++) {
        uint256 amount = (salePrice * splits[i].shareBP) / 10000;
        (bool success, ) = splits[i].recipient.call{value: amount}("");
        require(success, "Royalty transfer failed");
    }
}
```

* **EIP-2981 Compatible** → Royalty standard compliant for cross-platform support.
* **Upgradeable Architecture** → Supports governance-controlled changes to recursion limits or BP ranges.

---


