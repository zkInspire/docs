# zkInspire Test Suite

A comprehensive test suite for the zkInspire smart contract system using Vitest and Viem.


# Table of Contents for tests

- [zkInspire Test Suite](#zkinspire-test-suite)
- [Overview](#overview)
- [Test Architecture](#test-architecture)
- [Contract Deployment](#contract-deployment)
- [Test Accounts](#test-accounts)
- [Core Features Tested](#core-features-tested)
- [Query Functions](#query-functions)
- [Error Handling](#error-handling)
- [Gas Optimization](#gas-optimization)
- [Test Constants](#test-constants)
- [Event Tracking Examples](#event-tracking-examples)



## Overview

zkInspire is a blockchain-based platform that enables content creators to track inspiration relationships, claim derivatives, and distribute revenue through a decentralized system. The platform integrates with Zora for NFT creation and includes zero-knowledge proof verification for inspiration claims.

## Test Architecture

### Dependencies

```javascript
import {
  createTestClient,
  createWalletClient,
  createPublicClient,
  http,
  parseEther,
  formatEther,
  keccak256,
  toHex,
  stringToBytes,
  bytesToHex,
  zeroHash,
  getContract
} from "viem";
import { foundry } from "viem/chains";
import { generatePrivateKey, privateKeyToAccount } from "viem/accounts";
```

### Test Clients Setup

```javascript
// Create test accounts
owner = privateKeyToAccount(generatePrivateKey());
creator1 = privateKeyToAccount(generatePrivateKey());
creator2 = privateKeyToAccount(generatePrivateKey());

// Initialize clients
testClient = createTestClient({
  chain: foundry,
  mode: "anvil",
  transport: http(),
});

publicClient = createPublicClient({
  chain: foundry,
  transport: http(),
});

walletClient = createWalletClient({
  chain: foundry,
  transport: http(),
  account: owner,
});
```

### Account Funding

```javascript
// Fund test accounts with ETH
await testClient.setBalance({
  address: owner.address,
  value: parseEther("100"),
});
await testClient.setBalance({
  address: creator1.address,
  value: parseEther("100"),
});
```

## Contract Deployment

```javascript
// Deploy ProofOfInspiration contract
const deployHash = await walletClient.deployContract({
  abi: proofOfInspirationAbi,
  bytecode: "0x...",
  args: [ZORA_FACTORY_ADDRESS, UNISWAP_V3_FACTORY_ADDRESS, zkVerifier.address],
});

const deployReceipt = await publicClient.waitForTransactionReceipt({
  hash: deployHash,
});

proofOfInspirationAddress = deployReceipt.contractAddress!;
```

## Test Accounts

```javascript
let owner: ReturnType<typeof privateKeyToAccount>;
let creator1: ReturnType<typeof privateKeyToAccount>;
let creator2: ReturnType<typeof privateKeyToAccount>;
let creator3: ReturnType<typeof privateKeyToAccount>;
let zkVerifier: ReturnType<typeof privateKeyToAccount>;
let platformReferrer: ReturnType<typeof privateKeyToAccount>;
```

## Core Features Tested

### 1. Contract Deployment Tests

```javascript
describe("Deployment", () => {
  it("Should set the correct owner", async () => {
    const contractOwner = await proofOfInspirationContract.read.owner();
    expect(contractOwner.toLowerCase()).toBe(owner.address.toLowerCase());
  });

  it("Should set the correct factory addresses", async () => {
    const zoraFactory = await proofOfInspirationContract.read.zoraFactory();
    const uniswapFactory = await proofOfInspirationContract.read.uniswapV3Factory();
    
    expect(zoraFactory.toLowerCase()).toBe(ZORA_FACTORY_ADDRESS.toLowerCase());
    expect(uniswapFactory.toLowerCase()).toBe(UNISWAP_V3_FACTORY_ADDRESS.toLowerCase());
  });
});
```

### 2. Content Creation

**Content Parameters Structure:**
```javascript
const contentParams = {
  name: "Test Content",
  symbol: "TEST",
  uri: "ipfs://test-uri",
  contentHash: CONTENT_HASH_1,
  poolConfig: "0x",
  salt: keccak256(stringToBytes("test-salt")),
};
```

**Basic Content Creation:**
```javascript
const hash = await creator1Contract.write.createContent([
  contentParams.name,
  contentParams.symbol,
  contentParams.uri,
  contentParams.contentHash,
  contentParams.poolConfig,
  platformReferrer.address,
  contentParams.salt,
], {
  value: parseEther("0.1"),
});
```

**Event Parsing:**
```javascript
const contentCreatedLog = logs.find(log => {
  try {
    const decoded = publicClient.decodeEventLog({
      abi: proofOfInspirationAbi,
      data: log.data,
      topics: log.topics,
    });
    return decoded.eventName === "ContentCreated";
  } catch {
    return false;
  }
});
```

**Content Verification:**
```javascript
// Verify content piece storage
const contentPiece = await proofOfInspirationContract.read.contentPieces([contentId]);
expect(contentPiece[0].toLowerCase()).toBe(creator1.address.toLowerCase()); // creator
expect(contentPiece[2]).toBe(contentParams.contentHash); // contentHash
expect(contentPiece[4]).toBe(true); // exists
expect(contentPiece[5]).toBe(0n); // totalDerivatives
```

### 3. Derivative Content Creation

**Creating Original Content First:**
```javascript
beforeEach(async () => {
  const hash = await creator1Contract.write.createContent([
    "Original Content",
    "ORIG",
    "ipfs://original-uri",
    CONTENT_HASH_1,
    "0x",
    platformReferrer.address,
    keccak256(stringToBytes("original-salt")),
  ], {
    value: parseEther("0.1"),
  });
  
  // Extract contentId from events...
});
```

**Creating Derivative with Inspiration:**
```javascript
const revenueShareBps = 1000n; // 10%
const zkProofHash = keccak256(stringToBytes("test-proof"));

const hash = await creator2Contract.write.createDerivativeWithInspiration([
  originalContentId,
  "Derivative Content",
  "DERIV",
  "ipfs://derivative-uri",
  CONTENT_HASH_2,
  "0x",
  platformReferrer.address,
  keccak256(stringToBytes("derivative-salt")),
  revenueShareBps,
  zkProofHash,
  1, // ZK_SIMILARITY_PROOF
], {
  value: parseEther("0.1"),
});
```

**Validation Tests:**
```javascript
it("Should fail if original content doesn't exist", async () => {
  const nonExistentContentId = keccak256(stringToBytes("non-existent"));

  await expect(async () => {
    await creator2Contract.write.createDerivativeWithInspiration([
      nonExistentContentId,
      // ... other params
    ], {
      value: parseEther("0.1"),
    });
  }).rejects.toThrow();
});

it("Should fail if revenue share is too high", async () => {
  await expect(async () => {
    await creator2Contract.write.createDerivativeWithInspiration([
      originalContentId,
      // ... other params
      6000n, // 60% - exceeds maximum
      // ... remaining params
    ], {
      value: parseEther("0.1"),
    });
  }).rejects.toThrow();
});
```

### 4. Zero-Knowledge Proof System

**ZK Proof Types:**
```javascript
// Proof types enum
const DECLARED_ONLY = 0;
const ZK_SIMILARITY_PROOF = 1;
```

**ZK Proof Verification:**
```javascript
it("Should verify ZK proof and emit event", async () => {
  const claim = await proofOfInspirationContract.read.inspirationClaims([claimId]);
  
  // In the current implementation, ZK proof is auto-verified if zkProofHash is not zero
  expect(claim[4]).toBe(true); // zkVerified
});

it("Should track verified proofs", async () => {
  const zkProofHash = keccak256(stringToBytes("test-proof"));
  const isVerified = await proofOfInspirationContract.read.verifiedProofs([zkProofHash]);
  expect(isVerified).toBe(true);
});
```

### 5. Reputation System

**Reputation Tracking:**
```javascript
it("Should track creator reputation correctly", async () => {
  // Create content to trigger reputation update
  await creator1Contract.write.createContent([
    "Test Content",
    "TEST",
    "ipfs://test-uri",
    CONTENT_HASH_1,
    "0x",
    platformReferrer.address,
    keccak256(stringToBytes("test-salt")),
  ], {
    value: parseEther("0.1"),
  });

  const reputation = await proofOfInspirationContract.read.getCreatorReputation([creator1.address]);
  expect(reputation[1]).toBe(0n); // totalInspirations
  expect(reputation[2]).toBe(0n); // totalDerivatives
  expect(reputation[3]).toBe(0n); // fraudFlags
  expect(reputation[4]).toBe(0n); // successfulCollaborations
});
```

### 6. Dispute Resolution

**Dispute Creation:**
```javascript
it("Should allow original creator to dispute claim", async () => {
  const reason = "This is not inspired by my work";
  
  const hash = await creator1Contract.write.disputeInspiration([claimId, reason]);
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  
  const disputeEvent = receipt.logs.find(log => {
    try {
      const decoded = publicClient.decodeEventLog({
        abi: proofOfInspirationAbi,
        data: log.data,
        topics: log.topics,
      });
      return decoded.eventName === "InspirationDisputed";
    } catch {
      return false;
    }
  });

  expect(disputeEvent).toBeTruthy();
});
```

**Access Control:**
```javascript
it("Should not allow unauthorized users to dispute", async () => {
  const reason = "I don't like this";
  
  await expect(async () => {
    await creator3Contract.write.disputeInspiration([claimId, reason]);
  }).rejects.toThrow();
});
```

### 7. Revenue Distribution

**Mock ERC20 Setup:**
```javascript
beforeEach(async () => {
  // Deploy mock ERC20 token
  const erc20DeployHash = await walletClient.deployContract({
    abi: mockErc20Abi,
    bytecode: "0x...",
    args: ["Test Token", "TEST", parseEther("1000000")],
  });

  mockErc20Contract = getContract({
    address: mockErc20Address,
    abi: mockErc20Abi,
    client: { public: publicClient, wallet: walletClient },
  });
});
```

**ERC20 Revenue Distribution:**
```javascript
it("Should distribute revenue correctly", async () => {
  const revenueAmount = parseEther("10");
  
  // Transfer tokens to the contract for distribution
  await mockErc20Contract.write.transfer([proofOfInspirationAddress, revenueAmount]);

  const creator1BalanceBefore = await mockErc20Contract.read.balanceOf([creator1.address]);
  const creator2BalanceBefore = await mockErc20Contract.read.balanceOf([creator2.address]);

  await proofOfInspirationContract.write.distributeRevenue([
    derivativeContentId,
    mockErc20Address,
    revenueAmount,
  ]);

  const creator1BalanceAfter = await mockErc20Contract.read.balanceOf([creator1.address]);
  const creator2BalanceAfter = await mockErc20Contract.read.balanceOf([creator2.address]);

  // Check that revenue was distributed according to the 20% share
  const expectedCreator1Share = (revenueAmount * 2000n) / 10000n; // 20%
  const expectedCreator2Share = revenueAmount - expectedCreator1Share; // 80%

  expect(creator1BalanceAfter - creator1BalanceBefore).toBe(expectedCreator1Share);
  expect(creator2BalanceAfter - creator2BalanceBefore).toBe(expectedCreator2Share);
});
```

**ETH Revenue Distribution:**
```javascript
it("Should handle ETH revenue distribution", async () => {
  const revenueAmount = parseEther("1");
  
  // Send ETH to contract
  await testClient.setBalance({
    address: proofOfInspirationAddress,
    value: revenueAmount,
  });

  await proofOfInspirationContract.write.distributeRevenue([
    derivativeContentId,
    "0x0000000000000000000000000000000000000000", // ETH address
    revenueAmount,
  ]);
});
```

### 8. Ranking Algorithm

```javascript
describe("Ranking Algorithm", () => {
  it("Should calculate base ranking score", async () => {
    const score = await proofOfInspirationContract.read.calculateRankingScore([contentId]);
    expect(score).toBeGreaterThan(0n);
  });

  it("Should fail for non-existent content", async () => {
    const nonExistentId = keccak256(stringToBytes("non-existent"));
    
    await expect(async () => {
      await proofOfInspirationContract.read.calculateRankingScore([nonExistentId]);
    }).rejects.toThrow();
  });
});
```

### 9. Administrative Functions

**Owner Controls:**
```javascript
it("Should allow owner to set ZK verifier", async () => {
  const newVerifier = creator1.address;
  
  await proofOfInspirationContract.write.setZkVerifier([newVerifier]);
  const verifier = await proofOfInspirationContract.read.zkVerifier();
  expect(verifier.toLowerCase()).toBe(newVerifier.toLowerCase());
});

it("Should allow owner to set platform fee", async () => {
  const newFeeBps = 300n; // 3%
  
  await proofOfInspirationContract.write.setPlatformFeeBps([newFeeBps]);
  const platformFee = await proofOfInspirationContract.read.platformFeeBps();
  expect(platformFee).toBe(newFeeBps);
});
```

**Access Control Validation:**
```javascript
it("Should not allow non-owner to set ZK verifier", async () => {
  const creator1Contract = getContract({
    address: proofOfInspirationAddress,
    abi: proofOfInspirationAbi,
    client: { public: publicClient, wallet: creator1WalletClient },
  });

  await expect(async () => {
    await creator1Contract.write.setZkVerifier([creator1.address]);
  }).rejects.toThrow();
});
```

**Parameter Validation:**
```javascript
it("Should not allow setting platform fee above 10%", async () => {
  const invalidFeeBps = 1100n; // 11%
  
  await expect(async () => {
    await proofOfInspirationContract.write.setPlatformFeeBps([invalidFeeBps]);
  }).rejects.toThrow();
});
```

## Query Functions

### Content Queries
```javascript
// Get content metadata
const contentPiece = await proofOfInspirationContract.read.contentPieces([contentId]);

// Get creator's content by index
const creatorContentId = await proofOfInspirationContract.read.creatorContent([
  creator1.address,
  0n,
]);

// Get inspiration graph
const graph = await proofOfInspirationContract.read.getInspirationGraph([contentId]);
```

### Creator Queries  
```javascript
// Get reputation metrics
const reputation = await proofOfInspirationContract.read.getCreatorReputation([creator1.address]);

// Get content ranking score
const score = await proofOfInspirationContract.read.calculateRankingScore([contentId]);
```

### Claim Queries
```javascript
// Get inspiration claim details
const claim = await proofOfInspirationContract.read.inspirationClaims([claimId]);

// Check proof verification status
const isVerified = await proofOfInspirationContract.read.verifiedProofs([zkProofHash]);
```

## Error Handling

```javascript
describe("Error Conditions", () => {
  it("Should fail to create content with empty content hash", async () => {
    await expect(async () => {
      await creator1Contract.write.createContent([
        "Test Content",
        "TEST",
        "ipfs://test-uri",
        "", // Empty content hash
        "0x",
        platformReferrer.address,
        keccak256(stringToBytes("test-salt")),
      ], {
        value: parseEther("0.1"),
      });
    }).rejects.toThrow();
  });

  it("Should fail to create content with insufficient payment", async () => {
    await expect(async () => {
      await creator1Contract.write.createContent([
        // ... params
      ], {
        value: parseEther("0.01"), // Insufficient payment
      });
    }).rejects.toThrow();
  });
});
```

## Gas Optimization

```javascript
describe("Gas Optimization", () => {
  it("Should have reasonable gas costs for content creation", async () => {
    const hash = await creator1Contract.write.createContent([
      // ... content params
    ], {
      value: parseEther("0.1"),
    });

    const receipt = await publicClient.waitForTransactionReceipt({ hash });
    
    // Check that gas used is within reasonable limits
    expect(receipt.gasUsed).toBeLessThan(500000n);
  });

  it("Should have reasonable gas costs for derivative creation", async () => {
    const hash2 = await creator2Contract.write.createDerivativeWithInspiration([
      // ... derivative params
    ], {
      value: parseEther("0.1"),
    });

    const receipt2 = await publicClient.waitForTransactionReceipt({ hash: hash2 });
    
    // Check gas limits for derivative creation
    expect(receipt2.gasUsed).toBeLessThan(700000n);
  });
});
```

## Test Constants

```javascript
// Mock addresses for integration testing
const ZORA_FACTORY_ADDRESS: Address = "0x777777C338d93e2C7adf08D102d45CA7CC4Ed021";
const UNISWAP_V3_FACTORY_ADDRESS: Address = "0x1F98431c8aD98523631AE4a59f267346ea31F984";

// Test content hashes
const CONTENT_HASH_1 = "QmTestHash1";
const CONTENT_HASH_2 = "QmTestHash2";
const CONTENT_HASH_3 = "QmTestHash3";

// Platform configuration
const PLATFORM_FEE_BPS = 250n; // 2.5%
const MAX_REVENUE_SHARE_BPS = 5000n; // 50%
```

## Event Tracking Examples

```javascript
// Parse ContentCreated event
const contentCreatedLog = receipt.logs.find(log => {
  try {
    const decoded = publicClient.decodeEventLog({
      abi: proofOfInspirationAbi,
      data: log.data,
      topics: log.topics,
    });
    return decoded.eventName === "ContentCreated";
  } catch {
    return false;
  }
});

if (contentCreatedLog) {
  const decoded = publicClient.decodeEventLog({
    abi: proofOfInspirationAbi,
    data: contentCreatedLog.data,
    topics: contentCreatedLog.topics,
  });
  
  const { contentId, creator, contentHash } = decoded.args as any;
}
```

---