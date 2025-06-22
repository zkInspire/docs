# zkInspire — **Inspiration Graph Ranking & Data Architecture**

## 1. Formal Ranking Algorithm: **zkInspire Discovery Score (ZIDS)**

For every content node `C` in the zkInspire network, we define its **ZIDS** as:

$$
\text{ZIDS}(C) = (B + \alpha I + \beta Z + \gamma G) \cdot R - F
$$

Where:

| Symbol | Meaning                                                             | Example     |
| ------ | ------------------------------------------------------------------- | ----------- |
| **B**  | **Base Engagement Score** (likes, reposts, downstream usage)        | 1.0         |
| **I**  | **Declared Inspiration Bonus**                                      | +0.2        |
| **Z**  | **zkProof Multiplier** → If **zkProof-of-Inspiration (zkPoI)**      | +0.5        |
| **G**  | **Graph Depth/Breadth Bonus** → Larger, active derivation chains    | +0.3        |
| **R**  | **Reputation Multiplier** → Creator’s onchain & offchain trust      | ×1 → 2      |
| **F**  | **Fraud Risk Penalty** → AI similarity mismatches, flagged disputes | -0.5 → -0.9 |

>  **Weights (α, β, γ)** → Dynamically tuned via DAO governance or quadratic voting for optimal ranking calibration.

---

###  Example ZIDS Calculation:

| Scenario                                | Score Example                                   |
| --------------------------------------- | ----------------------------------------------- |
| High-quality zk-proven remix, reputable | \$(1.0 + 0.2 + 0.5 + 0.3) \* 1.7 - 0.0 = 3.23\$ |
| Remix without zkProof, medium trust     | \$(1.0 + 0.2 + 0 + 0.2) \* 1.1 - 0.1 = 1.42\$   |
| Suspicious, disputed remix              | \$(1.0 + 0.2 + 0 + 0.1) \* 0.9 - 0.8 = 0.41\$   |

---
