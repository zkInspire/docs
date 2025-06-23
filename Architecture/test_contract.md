# Test Contract - Table of Contents

## [Technical Architecture](#technical-architecture-1)
- Core Protocol Stack
- Smart Contract Components

## [Uniswap V4 Integration](#uniswap-v4-integration-1)
- Hook-Based Revenue Distribution
- Pool Architecture
- Position Management

## [Zora Protocol Integration](#zora-protocol-integration-1)
- Token Deployment Pipeline
- Metadata Standards

## [Revenue Distribution Engine](#revenue-distribution-engine-1)
- Multi-Layer Distribution
- Inspiration Graph Traversal

## [Content Creation Flow](#content-creation-flow-1)
- Primary Content Creation
- Derivative Content Creation

## [Zero-Knowledge Proof Integration](#zero-knowledge-proof-integration-1)
- Proof Types
- ZK Verification Implementation

## [Data Structures](#data-structures-1)
- Core Storage
- State Mappings

## [Reputation System](#reputation-system-1)
- Scoring Algorithm
- Reputation Metrics

## [Administrative Functions](#administrative-functions-1)
- Owner Controls
- Dispute Mechanism

## [Revenue Claiming](#revenue-claiming-1)
- Manual Revenue Withdrawal
- Batch Revenue Claims
## [Query Functions](#query-functions-1)
- Content Discovery

## [Testing Framework](#testing-framework-1)
- Mock Contract Implementation
- Test Coverage

# Proof of Inspiration Protocol

A decentralized protocol for content provenance tracking and automated revenue distribution built on Uniswap V4 hooks architecture and Zora Protocol infrastructure.

## Technical Architecture

### Core Protocol Stack

```
┌─────────────────────────────────────────────────────────────┐
│                 Proof of Inspiration                       │
├─────────────────────────────────────────────────────────────┤
│  Content Graph   │  Revenue Engine  │  Reputation System   │
├─────────────────────────────────────────────────────────────┤
│          Uniswap V4 Hooks        │    Zora Protocol        │
├─────────────────────────────────────────────────────────────┤
│     Position Manager    │ Pool Manager │  Factory Contract  │
├─────────────────────────────────────────────────────────────┤
│                        Ethereum L2                         │
└─────────────────────────────────────────────────────────────┘
```

### Smart Contract Components

#### ProofOfInspiration.sol
Primary protocol contract implementing:
- Content registration and tokenization
- Inspiration claim validation
- Revenue distribution automation
- Reputation scoring algorithms

#### Integration Contracts
- **IZoraFactory**: ERC-20 token deployment interface
- **IUniswapV4PoolManager**: Pool creation and management
- **IPositionManager**: LP position handling with subscription hooks
- **IZkVerifier**: Zero-knowledge proof verification for inspiration claims

## Uniswap V4 Integration

### Hook-Based Revenue Distribution

```solidity
contract ProofOfInspiration is IPositionSubscriber {
    function notifyModifyLiquidity(
        uint256 tokenId,
        int256 liquidityDelta,
        BalanceDelta feeDelta
    ) external override {
        bytes32 contentId = positionToContentId[tokenId];
        if (contentId != bytes32(0) && feeDelta.amount0() > 0) {
            _distributeFeesToInspirationNetwork(contentId, feeDelta);
        }
    }
}
```

**Technical Flow:**
1. LP positions subscribe to fee notifications via `IPositionSubscriber`
2. When fees are collected, `notifyModifyLiquidity` triggers automatically  
3. Fee distribution cascades through inspiration graph using BFS traversal
4. Revenue shares calculated based on configurable basis points (max 5000 BPS = 50%)

### Pool Architecture

Each content piece deploys:
```solidity
struct V4PoolConfig {
    address pairedToken;      // WETH, USDC, or custom base token
    uint24 fee;              // Pool fee tier (500, 3000, 10000 BPS)
    int24 tickSpacing;       // Tick spacing for concentrated liquidity
    uint160 initialSqrtPriceX96; // Initial price in Q64.96 format
}
```

**Pool Initialization:**
```solidity
PoolKey memory poolKey = PoolKey({
    currency0: Currency.wrap(token0),
    currency1: Currency.wrap(token1), 
    fee: poolConfig.fee,
    tickSpacing: poolConfig.tickSpacing,
    hooks: IHooks(hookAddress)
});

PoolId poolId = poolManager.initialize(
    poolKey,
    poolConfig.initialSqrtPriceX96,
    hookData
);
```

### Position Management

```solidity
function _createLiquidityPosition(
    address coinAddress,
    V4PoolConfig memory poolConfig,
    IPositionManager.MintParams memory mintParams
) internal returns (uint256 positionTokenId) {
    // Mint LP position
    (positionTokenId,,,) = positionManager.mint(mintParams);
    
    // Subscribe to fee notifications
    positionManager.subscribe(positionTokenId, address(this));
    
    return positionTokenId;
}
```

## Zora Protocol Integration

### Token Deployment Pipeline

```solidity
function _deployContentToken(
    string memory name,
    string memory symbol,
    string memory uri,
    address[] memory owners,
    bytes memory poolConfig,
    address platformReferrer,
    bytes32 salt
) internal returns (address coinAddress, uint256 tokenId) {
    (coinAddress, tokenId) = zoraFactory.deploy{value: msg.value}(
        msg.sender,          // payout recipient
        owners,              // token owners array
        uri,                 // metadata URI
        name,                // token name
        symbol,              // token symbol  
        poolConfig,          // encoded pool configuration
        platformReferrer,    // referrer for fees
        address(0),          // post-deploy hook
        "",                  // hook data
        salt                 // deterministic deployment salt
    );
}
```

### Metadata Standards

Content metadata follows Zora's URI specification:
```json
{
  "name": "Content Title",
  "description": "Content description", 
  "image": "ipfs://QmHash/image.jpg",
  "content_hash": "QmContentHash123",
  "creator": "0x...",
  "creation_timestamp": 1234567890,
  "inspiration_claims": [
    {
      "original_content_id": "0x...",
      "proof_type": "ZK_SIMILARITY_PROOF",
      "revenue_share_bps": 1000
    }
  ]
}
```

## Revenue Distribution Engine

### Multi-Layer Distribution

```solidity
function _distributeRevenue(bytes32 claimId, uint256 amount) internal {
    InspirationClaim memory claim = inspirationClaims[claimId];
    
    // Calculate platform fee
    uint256 platformFee = (amount * platformFeeBps) / 10000;
    uint256 remainingAmount = amount - platformFee;
    
    // Calculate inspiration share
    uint256 inspirationShare = (remainingAmount * claim.revenueShareBps) / 10000;
    uint256 derivativeShare = remainingAmount - inspirationShare;
    
    // Distribute to inspiration network
    _cascadeRevenue(claim.originalContentId, inspirationShare);
    
    // Track metrics
    totalRevenueGenerated[claim.originalContentId] += inspirationShare;
    
    emit RevenueDistributed(
        claim.original,
        getContentCreator(claim.originalContentId),
        inspirationShare,
        claim.originalContentId
    );
}
```

### Inspiration Graph Traversal

```solidity
function _cascadeRevenue(bytes32 contentId, uint256 amount) internal {
    ContentPiece memory content = contentPieces[contentId];
    address creator = content.creator;
    
    // Direct revenue to original creator
    pendingRevenue[content.coinAddress][creator] += amount;
    
    // Propagate to parent inspirations (recursive)
    bytes32[] memory parentInspirations = getParentInspirations(contentId);
    if (parentInspirations.length > 0) {
        uint256 parentShare = amount / 10; // 10% to parent level
        for (uint i = 0; i < parentInspirations.length; i++) {
            _cascadeRevenue(parentInspirations[i], parentShare / parentInspirations.length);
        }
    }
}
```

## Content Creation Flow

### Primary Content Creation

```solidity
function createContent(
    string memory name,
    string memory symbol,
    string memory uri,
    string memory contentHash,
    V4PoolConfig memory poolConfig,
    IPositionManager.MintParams memory mintParams,
    address platformReferrer,
    bytes32 salt
) external payable returns (bytes32 contentId, address coinAddress, uint256 positionTokenId) {
    // Generate unique content ID
    contentId = keccak256(abi.encodePacked(
        msg.sender,
        contentHash,
        block.timestamp,
        salt
    ));
    
    // Deploy ERC-20 token via Zora
    address[] memory owners = new address[](1);
    owners[0] = msg.sender;
    
    (coinAddress,) = _deployContentToken(
        name,
        symbol,
        uri,
        owners,
        abi.encode(poolConfig),
        platformReferrer,
        salt
    );
    
    // Create Uniswap V4 pool and liquidity position
    _initializeV4Pool(coinAddress, poolConfig);
    positionTokenId = _createLiquidityPosition(coinAddress, poolConfig, mintParams);
    
    // Store content metadata
    contentPieces[contentId] = ContentPiece({
        creator: msg.sender,
        coinAddress: coinAddress,
        contentHash: contentHash,
        timestamp: block.timestamp,
        exists: true,
        totalDerivatives: 0,
        reputationScore: 0,
        inspirationClaims: new bytes32[](0),
        positionTokenId: positionTokenId
    });
    
    // Map position to content
    positionToContentId[positionTokenId] = contentId;
    
    emit ContentCreated(contentId, msg.sender, coinAddress, contentHash, block.timestamp, positionTokenId);
}
```

### Derivative Content Creation

```solidity
function createDerivativeWithInspiration(
    bytes32 originalContentId,
    string memory name,
    string memory symbol,
    string memory uri,
    string memory contentHash,
    V4PoolConfig memory poolConfig,
    IPositionManager.MintParams memory mintParams,
    address platformReferrer,
    bytes32 salt,
    uint256 revenueShareBps,
    bytes32 zkProofHash,
    InspirationProofType proofType
) external payable returns (bytes32 derivativeContentId, address coinAddress, uint256 positionTokenId) {
    require(contentPieces[originalContentId].exists, "Original content does not exist");
    require(revenueShareBps <= MAX_REVENUE_SHARE_BPS, "Revenue share too high");
    
    // Create derivative content
    (derivativeContentId, coinAddress, positionTokenId) = createContent(
        name, symbol, uri, contentHash, poolConfig, mintParams, platformReferrer, salt
    );
    
    // Create inspiration claim
    bytes32 claimId = keccak256(abi.encodePacked(originalContentId, derivativeContentId));
    
    inspirationClaims[claimId] = InspirationClaim({
        derivative: coinAddress,
        original: contentPieces[originalContentId].coinAddress,
        revenueShareBps: revenueShareBps,
        zkProofHash: zkProofHash,
        zkVerified: _verifyZkProof(zkProofHash, proofType),
        disputed: false,
        timestamp: block.timestamp,
        proofType: proofType
    });
    
    // Update graph structures
    contentDerivatives[originalContentId].push(derivativeContentId);
    contentInspirations[derivativeContentId].push(originalContentId);
    contentPieces[originalContentId].totalDerivatives++;
    
    emit InspirationClaimed(claimId, originalContentId, derivativeContentId, msg.sender, revenueShareBps, proofType);
}
```

## Zero-Knowledge Proof Integration

### Proof Types

```solidity
enum InspirationProofType {
    DECLARED_ONLY,           // Creator declaration only
    COMMUNITY_VERIFIED,      // Community consensus
    ZK_SIMILARITY_PROOF,     // Cryptographic content similarity
    ZK_PROVENANCE_PROOF,     // Cryptographic provenance chain
    CROSS_PLATFORM_VERIFIED  // External platform verification
}
```

### ZK Verification Implementation

```solidity
function _verifyZkProof(
    bytes32 zkProofHash,
    InspirationProofType proofType
) internal view returns (bool) {
    if (proofType == InspirationProofType.ZK_SIMILARITY_PROOF ||
        proofType == InspirationProofType.ZK_PROVENANCE_PROOF) {
        return zkVerifier.verify(zkProofHash);
    }
    return true; // Non-ZK proof types automatically verified
}
```

## Data Structures

### Core Storage

```solidity
struct ContentPiece {
    address creator;
    address coinAddress;
    string contentHash;        // IPFS hash
    uint256 timestamp;
    bool exists;
    uint256 totalDerivatives;
    uint256 reputationScore;
    bytes32[] inspirationClaims;
    uint256 positionTokenId;   // Uniswap V4 position
}

struct InspirationClaim {
    address derivative;        // Derivative token address
    address original;         // Original token address  
    uint256 revenueShareBps;  // Revenue share (0-5000 BPS)
    bytes32 zkProofHash;      // ZK proof identifier
    bool zkVerified;          // Verification status
    bool disputed;            // Dispute flag
    uint256 timestamp;
    InspirationProofType proofType;
}
```

### State Mappings

```solidity
mapping(bytes32 => ContentPiece) public contentPieces;
mapping(bytes32 => InspirationClaim) public inspirationClaims;
mapping(bytes32 => bytes32[]) public contentDerivatives;
mapping(bytes32 => bytes32[]) public contentInspirations;
mapping(uint256 => bytes32) public positionToContentId;
mapping(address => mapping(address => uint256)) public pendingRevenue;
mapping(bytes32 => uint256) public totalRevenueGenerated;
```

## Reputation System

### Scoring Algorithm

```solidity
function calculateRankingScore(bytes32 contentId) public view returns (uint256) {
    ContentPiece memory content = contentPieces[contentId];
    
    uint256 baseScore = 1000;
    uint256 derivativeBonus = content.totalDerivatives * 100;
    uint256 revenueBonus = (totalRevenueGenerated[contentId] / 1e18) * 50;
    uint256 timeDecay = _calculateTimeDecay(content.timestamp);
    uint256 zkBonus = _calculateZkVerificationBonus(contentId);
    
    return (baseScore + derivativeBonus + revenueBonus + zkBonus) * timeDecay / 100;
}

function _calculateTimeDecay(uint256 timestamp) internal view returns (uint256) {
    uint256 age = block.timestamp - timestamp;
    uint256 daysSinceCreation = age / 86400; // seconds per day
    
    if (daysSinceCreation < 30) return 100;      // No decay first month
    if (daysSinceCreation < 90) return 95;       // 5% decay after 3 months
    if (daysSinceCreation < 180) return 90;      // 10% decay after 6 months
    return 85;                                   // 15% decay after 6 months
}
```

### Reputation Metrics

```solidity
struct ReputationMetrics {
    uint256 totalOriginalContent;
    uint256 totalDerivatives;
    uint256 totalInspirations;
    uint256 totalRevenue;
    uint256 collaboratorScore;
    uint256 fraudFlags;
    uint256 communityScore;
}

function getCreatorReputation(address creator) external view returns (ReputationMetrics memory) {
    return creatorReputation[creator];
}
```

## Administrative Functions

### Owner Controls

```solidity
function setZkVerifier(address _zkVerifier) external onlyOwner {
    zkVerifier = IZkVerifier(_zkVerifier);
}

function setPlatformFee(uint256 _platformFeeBps) external onlyOwner {
    require(_platformFeeBps <= 1000, "Platform fee too high"); // Max 10%
    platformFeeBps = _platformFeeBps;
}
```

### Dispute Mechanism

```solidity
function disputeInspiration(bytes32 claimId, string memory reason) external {
    InspirationClaim storage claim = inspirationClaims[claimId];
    require(claim.timestamp > 0, "Claim does not exist");
    
    // Mark as disputed
    claim.disputed = true;
    
    // Apply reputation penalty
    address derivativeCreator = getTokenCreator(claim.derivative);
    creatorReputation[derivativeCreator].fraudFlags++;
    
    emit InspirationDisputed(claimId, msg.sender, reason);
}
```

## Revenue Claiming

### Manual Revenue Withdrawal

```solidity
function claimRevenue(address tokenAddress) external {
    uint256 pending = pendingRevenue[tokenAddress][msg.sender];
    require(pending > 0, "No pending revenue");
    
    pendingRevenue[tokenAddress][msg.sender] = 0;
    
    IERC20(tokenAddress).transfer(msg.sender, pending);
    
    emit RevenueClaimed(msg.sender, tokenAddress, pending);
}
```

### Batch Revenue Claims

```solidity
function claimMultipleRevenues(address[] calldata tokenAddresses) external {
    for (uint256 i = 0; i < tokenAddresses.length; i++) {
        uint256 pending = pendingRevenue[tokenAddresses[i]][msg.sender];
        if (pending > 0) {
            pendingRevenue[tokenAddresses[i]][msg.sender] = 0;
            IERC20(tokenAddresses[i]).transfer(msg.sender, pending);
            emit RevenueClaimed(msg.sender, tokenAddresses[i], pending);
        }
    }
}
```

## Query Functions

### Content Discovery

```solidity
function getInspirationGraph(bytes32 contentId) external view returns (
    bytes32[] memory derivatives,
    uint256 depth,
    uint256 totalRevenue
) {
    derivatives = contentDerivatives[contentId];
    depth = _calculateContentDepth(contentId);
    totalRevenue = totalRevenueGenerated[contentId];
}

function getV4PoolInfo(bytes32 contentId) external view returns (
    PoolKey memory poolKey,
    uint256 positionTokenId,
    uint160 sqrtPriceX96,
    int24 tick
) {
    ContentPiece memory content = contentPieces[contentId];
    positionTokenId = content.positionTokenId;
    
    // Construct pool key
    poolKey = _getPoolKey(content.coinAddress);
    
    // Get current pool state
    PoolId poolId = PoolId.wrap(keccak256(abi.encode(poolKey)));
    (sqrtPriceX96, tick,,) = poolManager.getSlot0(poolId);
}
```

## Testing Framework

### Mock Contract Implementation

```solidity
contract MockZoraFactory {
    uint256 public constant DEPLOYMENT_COST = 0.01 ether;
    
    function deploy(
        address payoutRecipient,
        address[] memory owners,
        string memory uri,
        string memory name,
        string memory symbol,
        bytes memory poolConfig,
        address platformReferrer,
        address postDeployHook,
        bytes memory hookData,
        bytes32 salt
    ) external payable returns (address coinAddress, uint256 tokenId) {
        require(msg.value >= DEPLOYMENT_COST, "Insufficient deployment fee");
        
        MockERC20 token = new MockERC20(name, symbol);
        coinAddress = address(token);
        tokenId = 1;
        
        token.mint(owners[0], 1000000 * 10**18);
        return (coinAddress, tokenId);
    }
}
```

### Test Coverage

The test suite includes:
- Content creation and tokenization
- Inspiration claim validation
- Revenue distribution automation
- Dispute mechanism testing
- Reputation scoring verification
- Administrative function testing
- Edge case handling# Proof of Inspiration Protocol

A decentralized protocol for content provenance tracking and automated revenue distribution built on Uniswap V4 hooks architecture and Zora Protocol infrastructure.

# Technical Architecture

# Core Protocol Stack

```
┌─────────────────────────────────────────────────────────────┐
│                 Proof of Inspiration                       │
├─────────────────────────────────────────────────────────────┤
│  Content Graph   │  Revenue Engine  │  Reputation System   │
├─────────────────────────────────────────────────────────────┤
│          Uniswap V4 Hooks        │    Zora Protocol        │
├─────────────────────────────────────────────────────────────┤
│     Position Manager    │ Pool Manager │  Factory Contract  │
├─────────────────────────────────────────────────────────────┤
│                        Ethereum L2                         │
└─────────────────────────────────────────────────────────────┘
```

# Smart Contract Components

## ProofOfInspiration.sol
Primary protocol contract implementing:
- Content registration and tokenization
- Inspiration claim validation
- Revenue distribution automation
- Reputation scoring algorithms

## Integration Contracts
- **IZoraFactory**: ERC-20 token deployment interface
- **IUniswapV4PoolManager**: Pool creation and management
- **IPositionManager**: LP position handling with subscription hooks
- **IZkVerifier**: Zero-knowledge proof verification for inspiration claims

# Uniswap V4 Integration

## Hook-Based Revenue Distribution

```solidity
contract ProofOfInspiration is IPositionSubscriber {
    function notifyModifyLiquidity(
        uint256 tokenId,
        int256 liquidityDelta,
        BalanceDelta feeDelta
    ) external override {
        bytes32 contentId = positionToContentId[tokenId];
        if (contentId != bytes32(0) && feeDelta.amount0() > 0) {
            _distributeFeesToInspirationNetwork(contentId, feeDelta);
        }
    }
}
```

**Technical Flow:**
1. LP positions subscribe to fee notifications via `IPositionSubscriber`
2. When fees are collected, `notifyModifyLiquidity` triggers automatically  
3. Fee distribution cascades through inspiration graph using BFS traversal
4. Revenue shares calculated based on configurable basis points (max 5000 BPS = 50%)

## Pool Architecture

Each content piece deploys:
```solidity
struct V4PoolConfig {
    address pairedToken;      // WETH, USDC, or custom base token
    uint24 fee;              // Pool fee tier (500, 3000, 10000 BPS)
    int24 tickSpacing;       // Tick spacing for concentrated liquidity
    uint160 initialSqrtPriceX96; // Initial price in Q64.96 format
}
```

**Pool Initialization:**
```solidity
PoolKey memory poolKey = PoolKey({
    currency0: Currency.wrap(token0),
    currency1: Currency.wrap(token1), 
    fee: poolConfig.fee,
    tickSpacing: poolConfig.tickSpacing,
    hooks: IHooks(hookAddress)
});

PoolId poolId = poolManager.initialize(
    poolKey,
    poolConfig.initialSqrtPriceX96,
    hookData
);
```

## Position Management

```solidity
function _createLiquidityPosition(
    address coinAddress,
    V4PoolConfig memory poolConfig,
    IPositionManager.MintParams memory mintParams
) internal returns (uint256 positionTokenId) {
    // Mint LP position
    (positionTokenId,,,) = positionManager.mint(mintParams);
    
    // Subscribe to fee notifications
    positionManager.subscribe(positionTokenId, address(this));
    
    return positionTokenId;
}
```

# Zora Protocol Integration

## Token Deployment Pipeline

```solidity
function _deployContentToken(
    string memory name,
    string memory symbol,
    string memory uri,
    address[] memory owners,
    bytes memory poolConfig,
    address platformReferrer,
    bytes32 salt
) internal returns (address coinAddress, uint256 tokenId) {
    (coinAddress, tokenId) = zoraFactory.deploy{value: msg.value}(
        msg.sender,          // payout recipient
        owners,              // token owners array
        uri,                 // metadata URI
        name,                // token name
        symbol,              // token symbol  
        poolConfig,          // encoded pool configuration
        platformReferrer,    // referrer for fees
        address(0),          // post-deploy hook
        "",                  // hook data
        salt                 // deterministic deployment salt
    );
}
```

## Metadata Standards

Content metadata follows Zora's URI specification:
```json
{
  "name": "Content Title",
  "description": "Content description", 
  "image": "ipfs://QmHash/image.jpg",
  "content_hash": "QmContentHash123",
  "creator": "0x...",
  "creation_timestamp": 1234567890,
  "inspiration_claims": [
    {
      "original_content_id": "0x...",
      "proof_type": "ZK_SIMILARITY_PROOF",
      "revenue_share_bps": 1000
    }
  ]
}
```

# Revenue Distribution Engine

## Multi-Layer Distribution

```solidity
function _distributeRevenue(bytes32 claimId, uint256 amount) internal {
    InspirationClaim memory claim = inspirationClaims[claimId];
    
    // Calculate platform fee
    uint256 platformFee = (amount * platformFeeBps) / 10000;
    uint256 remainingAmount = amount - platformFee;
    
    // Calculate inspiration share
    uint256 inspirationShare = (remainingAmount * claim.revenueShareBps) / 10000;
    uint256 derivativeShare = remainingAmount - inspirationShare;
    
    // Distribute to inspiration network
    _cascadeRevenue(claim.originalContentId, inspirationShare);
    
    // Track metrics
    totalRevenueGenerated[claim.originalContentId] += inspirationShare;
    
    emit RevenueDistributed(
        claim.original,
        getContentCreator(claim.originalContentId),
        inspirationShare,
        claim.originalContentId
    );
}
```

## Inspiration Graph Traversal

```solidity
function _cascadeRevenue(bytes32 contentId, uint256 amount) internal {
    ContentPiece memory content = contentPieces[contentId];
    address creator = content.creator;
    
    // Direct revenue to original creator
    pendingRevenue[content.coinAddress][creator] += amount;
    
    // Propagate to parent inspirations (recursive)
    bytes32[] memory parentInspirations = getParentInspirations(contentId);
    if (parentInspirations.length > 0) {
        uint256 parentShare = amount / 10; // 10% to parent level
        for (uint i = 0; i < parentInspirations.length; i++) {
            _cascadeRevenue(parentInspirations[i], parentShare / parentInspirations.length);
        }
    }
}
```

# Content Creation Flow

## Primary Content Creation

```solidity
function createContent(
    string memory name,
    string memory symbol,
    string memory uri,
    string memory contentHash,
    V4PoolConfig memory poolConfig,
    IPositionManager.MintParams memory mintParams,
    address platformReferrer,
    bytes32 salt
) external payable returns (bytes32 contentId, address coinAddress, uint256 positionTokenId) {
    // Generate unique content ID
    contentId = keccak256(abi.encodePacked(
        msg.sender,
        contentHash,
        block.timestamp,
        salt
    ));
    
    // Deploy ERC-20 token via Zora
    address[] memory owners = new address[](1);
    owners[0] = msg.sender;
    
    (coinAddress,) = _deployContentToken(
        name,
        symbol,
        uri,
        owners,
        abi.encode(poolConfig),
        platformReferrer,
        salt
    );
    
    // Create Uniswap V4 pool and liquidity position
    _initializeV4Pool(coinAddress, poolConfig);
    positionTokenId = _createLiquidityPosition(coinAddress, poolConfig, mintParams);
    
    // Store content metadata
    contentPieces[contentId] = ContentPiece({
        creator: msg.sender,
        coinAddress: coinAddress,
        contentHash: contentHash,
        timestamp: block.timestamp,
        exists: true,
        totalDerivatives: 0,
        reputationScore: 0,
        inspirationClaims: new bytes32[](0),
        positionTokenId: positionTokenId
    });
    
    // Map position to content
    positionToContentId[positionTokenId] = contentId;
    
    emit ContentCreated(contentId, msg.sender, coinAddress, contentHash, block.timestamp, positionTokenId);
}
```

## Derivative Content Creation

```solidity
function createDerivativeWithInspiration(
    bytes32 originalContentId,
    string memory name,
    string memory symbol,
    string memory uri,
    string memory contentHash,
    V4PoolConfig memory poolConfig,
    IPositionManager.MintParams memory mintParams,
    address platformReferrer,
    bytes32 salt,
    uint256 revenueShareBps,
    bytes32 zkProofHash,
    InspirationProofType proofType
) external payable returns (bytes32 derivativeContentId, address coinAddress, uint256 positionTokenId) {
    require(contentPieces[originalContentId].exists, "Original content does not exist");
    require(revenueShareBps <= MAX_REVENUE_SHARE_BPS, "Revenue share too high");
    
    // Create derivative content
    (derivativeContentId, coinAddress, positionTokenId) = createContent(
        name, symbol, uri, contentHash, poolConfig, mintParams, platformReferrer, salt
    );
    
    // Create inspiration claim
    bytes32 claimId = keccak256(abi.encodePacked(originalContentId, derivativeContentId));
    
    inspirationClaims[claimId] = InspirationClaim({
        derivative: coinAddress,
        original: contentPieces[originalContentId].coinAddress,
        revenueShareBps: revenueShareBps,
        zkProofHash: zkProofHash,
        zkVerified: _verifyZkProof(zkProofHash, proofType),
        disputed: false,
        timestamp: block.timestamp,
        proofType: proofType
    });
    
    // Update graph structures
    contentDerivatives[originalContentId].push(derivativeContentId);
    contentInspirations[derivativeContentId].push(originalContentId);
    contentPieces[originalContentId].totalDerivatives++;
    
    emit InspirationClaimed(claimId, originalContentId, derivativeContentId, msg.sender, revenueShareBps, proofType);
}
```

# Zero-Knowledge Proof Integration

## Proof Types

```solidity
enum InspirationProofType {
    DECLARED_ONLY,           // Creator declaration only
    COMMUNITY_VERIFIED,      // Community consensus
    ZK_SIMILARITY_PROOF,     // Cryptographic content similarity
    ZK_PROVENANCE_PROOF,     // Cryptographic provenance chain
    CROSS_PLATFORM_VERIFIED  // External platform verification
}
```

## ZK Verification Implementation

```solidity
function _verifyZkProof(
    bytes32 zkProofHash,
    InspirationProofType proofType
) internal view returns (bool) {
    if (proofType == InspirationProofType.ZK_SIMILARITY_PROOF ||
        proofType == InspirationProofType.ZK_PROVENANCE_PROOF) {
        return zkVerifier.verify(zkProofHash);
    }
    return true; // Non-ZK proof types automatically verified
}
```

# Data Structures

## Core Storage

```solidity
struct ContentPiece {
    address creator;
    address coinAddress;
    string contentHash;        // IPFS hash
    uint256 timestamp;
    bool exists;
    uint256 totalDerivatives;
    uint256 reputationScore;
    bytes32[] inspirationClaims;
    uint256 positionTokenId;   // Uniswap V4 position
}

struct InspirationClaim {
    address derivative;        // Derivative token address
    address original;         // Original token address  
    uint256 revenueShareBps;  // Revenue share (0-5000 BPS)
    bytes32 zkProofHash;      // ZK proof identifier
    bool zkVerified;          // Verification status
    bool disputed;            // Dispute flag
    uint256 timestamp;
    InspirationProofType proofType;
}
```

## State Mappings

```solidity
mapping(bytes32 => ContentPiece) public contentPieces;
mapping(bytes32 => InspirationClaim) public inspirationClaims;
mapping(bytes32 => bytes32[]) public contentDerivatives;
mapping(bytes32 => bytes32[]) public contentInspirations;
mapping(uint256 => bytes32) public positionToContentId;
mapping(address => mapping(address => uint256)) public pendingRevenue;
mapping(bytes32 => uint256) public totalRevenueGenerated;
```

# Reputation System

## Scoring Algorithm

```solidity
function calculateRankingScore(bytes32 contentId) public view returns (uint256) {
    ContentPiece memory content = contentPieces[contentId];
    
    uint256 baseScore = 1000;
    uint256 derivativeBonus = content.totalDerivatives * 100;
    uint256 revenueBonus = (totalRevenueGenerated[contentId] / 1e18) * 50;
    uint256 timeDecay = _calculateTimeDecay(content.timestamp);
    uint256 zkBonus = _calculateZkVerificationBonus(contentId);
    
    return (baseScore + derivativeBonus + revenueBonus + zkBonus) * timeDecay / 100;
}

function _calculateTimeDecay(uint256 timestamp) internal view returns (uint256) {
    uint256 age = block.timestamp - timestamp;
    uint256 daysSinceCreation = age / 86400; // seconds per day
    
    if (daysSinceCreation < 30) return 100;      // No decay first month
    if (daysSinceCreation < 90) return 95;       // 5% decay after 3 months
    if (daysSinceCreation < 180) return 90;      // 10% decay after 6 months
    return 85;                                   // 15% decay after 6 months
}
```

## Reputation Metrics

```solidity
struct ReputationMetrics {
    uint256 totalOriginalContent;
    uint256 totalDerivatives;
    uint256 totalInspirations;
    uint256 totalRevenue;
    uint256 collaboratorScore;
    uint256 fraudFlags;
    uint256 communityScore;
}

function getCreatorReputation(address creator) external view returns (ReputationMetrics memory) {
    return creatorReputation[creator];
}
```

# Administrative Functions

## Owner Controls

```solidity
function setZkVerifier(address _zkVerifier) external onlyOwner {
    zkVerifier = IZkVerifier(_zkVerifier);
}

function setPlatformFee(uint256 _platformFeeBps) external onlyOwner {
    require(_platformFeeBps <= 1000, "Platform fee too high"); // Max 10%
    platformFeeBps = _platformFeeBps;
}
```

## Dispute Mechanism

```solidity
function disputeInspiration(bytes32 claimId, string memory reason) external {
    InspirationClaim storage claim = inspirationClaims[claimId];
    require(claim.timestamp > 0, "Claim does not exist");
    
    // Mark as disputed
    claim.disputed = true;
    
    // Apply reputation penalty
    address derivativeCreator = getTokenCreator(claim.derivative);
    creatorReputation[derivativeCreator].fraudFlags++;
    
    emit InspirationDisputed(claimId, msg.sender, reason);
}
```

# Revenue Claiming

## Manual Revenue Withdrawal

```solidity
function claimRevenue(address tokenAddress) external {
    uint256 pending = pendingRevenue[tokenAddress][msg.sender];
    require(pending > 0, "No pending revenue");
    
    pendingRevenue[tokenAddress][msg.sender] = 0;
    
    IERC20(tokenAddress).transfer(msg.sender, pending);
    
    emit RevenueClaimed(msg.sender, tokenAddress, pending);
}
```

## Batch Revenue Claims

```solidity
function claimMultipleRevenues(address[] calldata tokenAddresses) external {
    for (uint256 i = 0; i < tokenAddresses.length; i++) {
        uint256 pending = pendingRevenue[tokenAddresses[i]][msg.sender];
        if (pending > 0) {
            pendingRevenue[tokenAddresses[i]][msg.sender] = 0;
            IERC20(tokenAddresses[i]).transfer(msg.sender, pending);
            emit RevenueClaimed(msg.sender, tokenAddresses[i], pending);
        }
    }
}
```

# Query Functions

## Content Discovery

```solidity
function getInspirationGraph(bytes32 contentId) external view returns (
    bytes32[] memory derivatives,
    uint256 depth,
    uint256 totalRevenue
) {
    derivatives = contentDerivatives[contentId];
    depth = _calculateContentDepth(contentId);
    totalRevenue = totalRevenueGenerated[contentId];
}

function getV4PoolInfo(bytes32 contentId) external view returns (
    PoolKey memory poolKey,
    uint256 positionTokenId,
    uint160 sqrtPriceX96,
    int24 tick
) {
    ContentPiece memory content = contentPieces[contentId];
    positionTokenId = content.positionTokenId;
    
    // Construct pool key
    poolKey = _getPoolKey(content.coinAddress);
    
    // Get current pool state
    PoolId poolId = PoolId.wrap(keccak256(abi.encode(poolKey)));
    (sqrtPriceX96, tick,,) = poolManager.getSlot0(poolId);
}
```

# Testing Framework

## Mock Contract Implementation

```solidity
contract MockZoraFactory {
    uint256 public constant DEPLOYMENT_COST = 0.01 ether;
    
    function deploy(
        address payoutRecipient,
        address[] memory owners,
        string memory uri,
        string memory name,
        string memory symbol,
        bytes memory poolConfig,
        address platformReferrer,
        address postDeployHook,
        bytes memory hookData,
        bytes32 salt
    ) external payable returns (address coinAddress, uint256 tokenId) {
        require(msg.value >= DEPLOYMENT_COST, "Insufficient deployment fee");
        
        MockERC20 token = new MockERC20(name, symbol);
        coinAddress = address(token);
        tokenId = 1;
        
        token.mint(owners[0], 1000000 * 10**18);
        return (coinAddress, tokenId);
    }
}
```

# Test Coverage

The test suite includes:
- Content creation and tokenization
- Inspiration claim validation
- Revenue distribution automation
- Dispute mechanism testing
- Reputation scoring verification
- Administrative function testing
- Edge case handling

---
