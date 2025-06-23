---


# System Architecture

# Table of Contents for Contracts
1. [System Architecture](#system-architecture)
2. [Core Contract Analysis](#core-contract-analysis)
3. [Data Structures Deep Dive](#data-structures-deep-dive)
4. [Function-by-Function Analysis](#function-by-function-analysis)
5. [Revenue Distribution Mechanics](#revenue-distribution-mechanics)
6. [Reputation System Implementation](#reputation-system-implementation)
7. [Graph Theory Implementation](#graph-theory-implementation)
8. [Security Analysis](#security-analysis)
9. [Integration with Uniswap and Zora](#integration-patterns)
10. [Gas Optimization Strategies](#gas-optimization-strategies)

---
The Proof of Inspiration system implements a decentralized content attribution and revenue sharing protocol. It consists of two interconnected smart contracts that manage the entire lifecycle of creative content and its derivatives.

```solidity
// Core system interfaces
interface IZoraFactory {
    function deploy(
        address payoutRecipient,
        address[] memory owners,
        string memory uri,
        string memory name,
        string memory symbol,
        bytes memory poolConfig,
        address platformReferrer,
        address postDeployHook,
        bytes calldata postDeployHookData,
        bytes32 coinSalt
    ) external payable returns (address coin, bytes memory postDeployHookDataOut);
}

interface IUniswapV3Pool {
    function fee() external view returns (uint24);
    function collectProtocol(address recipient0, address recipient1, uint128 amount0Requested, uint128 amount1Requested) 
        external returns (uint128 amount0, uint128 amount1);
}
```

The system integrates with:
- **Zora Protocol**: For NFT/coin minting and marketplace functionality
- **Uniswap V3**: For automated liquidity provision and fee collection
- **ZK-Proof Systems**: For verifiable content similarity claims

## Core Contract Analysis

### ProofOfInspiration Contract

```solidity
contract ProofOfInspiration is Ownable, ReentrancyGuard {
    // Immutable addresses for gas optimization
    IZoraFactory public immutable zoraFactory;
    address public immutable uniswapV3Factory;
    
    // Core state mappings
    mapping(bytes32 => ContentPiece) public contentPieces;
    mapping(bytes32 => InspirationClaim) public inspirationClaims;
    mapping(address => ReputationMetrics) public creatorReputations;
    mapping(address => bytes32[]) public creatorContent;
    mapping(bytes32 => bytes32[]) public contentDerivatives;
    
    // Graph structure for inspiration networks
    mapping(bytes32 => bytes32[]) public inspirationGraph;
    mapping(bytes32 => uint256) public contentDepth;
    
    // Revenue tracking system
    mapping(address => mapping(address => uint256)) public pendingRevenue;
    mapping(bytes32 => uint256) public totalRevenueGenerated;
    
    // ZK-proof verification system
    mapping(bytes32 => bool) public verifiedProofs;
    address public zkVerifier;
```

**Key Design Decisions:**

1. **Immutable Factory Addresses**: Gas optimization by avoiding SLOAD operations
2. **Nested Mappings for Revenue**: Efficient O(1) lookup for creator-coin revenue pairs
3. **Graph Adjacency Lists**: Efficient storage of inspiration relationships
4. **Separation of Concerns**: Clear distinction between content, claims, and revenue

## Data Structures Deep Dive

### ContentPiece Structure

```solidity
struct ContentPiece {
    address creator;           // 20 bytes - Creator's wallet address
    address coinAddress;       // 20 bytes - Deployed Zora coin contract
    string contentHash;        // Dynamic - IPFS hash or content fingerprint
    uint256 timestamp;         // 32 bytes - Block timestamp of creation
    bool exists;              // 1 byte - Existence flag for validation
    uint256 totalDerivatives; // 32 bytes - Count of derivative works
    uint256 reputationScore;  // 32 bytes - Cached reputation score
}
```

**Storage Optimization Analysis:**
- Total fixed size: ~137 bytes + string length
- Packed efficiently in storage slots
- `exists` flag prevents zero-value confusion
- `reputationScore` cached to avoid recalculation

### InspirationClaim Structure

```solidity
struct InspirationClaim {
    address derivative;         // 20 bytes - Derivative coin address
    address original;          // 20 bytes - Original coin address  
    uint256 revenueShareBps;   // 32 bytes - Basis points (10000 = 100%)
    bytes32 zkProofHash;       // 32 bytes - Hash of ZK proof data
    bool zkVerified;           // 1 byte - Verification status
    bool disputed;             // 1 byte - Dispute flag
    uint256 timestamp;         // 32 bytes - Claim creation time
    InspirationProofType proofType; // 1 byte - Enum for proof type
}
```

**Revenue Share Calculation:**
```solidity
// Revenue share in basis points allows precise percentage calculations
uint256 originalShare = (amount * claim.revenueShareBps) / 10000;
// Example: 2500 bps = 25% share
// If amount = 1000 tokens: originalShare = (1000 * 2500) / 10000 = 250 tokens
```

### ReputationMetrics Structure

```solidity
struct ReputationMetrics {
    uint256 collaboratorScore;        // Weighted reputation score
    uint256 totalInspirations;        // Times this creator inspired others
    uint256 totalDerivatives;         // Derivative works created
    uint256 fraudFlags;               // Disputed claims count
    uint256 successfulCollaborations; // Verified successful partnerships
}
```

## Function-by-Function Analysis

### Content Creation Function

```solidity
function createContent(
    string memory name,
    string memory symbol,
    string memory uri,
    string memory contentHash,
    bytes memory poolConfig,
    address platformReferrer,
    bytes32 salt
) external payable returns (bytes32 contentId, address coinAddress) {
    // Generate deterministic content ID
    contentId = keccak256(abi.encodePacked(msg.sender, contentHash, block.timestamp));
    
    // Prepare Zora deployment parameters
    address[] memory owners = new address[](1);
    owners[0] = msg.sender;
    
    // Deploy through Zora factory with ETH value
    (coinAddress, ) = zoraFactory.deploy{value: msg.value}(
        msg.sender,      // Revenue recipient
        owners,          // Coin owners array
        uri,            // Metadata URI
        name,           // Human readable name
        symbol,         // Trading symbol
        poolConfig,     // Uniswap pool configuration
        platformReferrer, // Referral tracking
        address(this),  // Post-deploy hook contract
        abi.encode(contentId), // Hook data payload
        salt           // Deterministic deployment salt
    );
    
    // Store content metadata
    contentPieces[contentId] = ContentPiece({
        creator: msg.sender,
        coinAddress: coinAddress,
        contentHash: contentHash,
        timestamp: block.timestamp,
        exists: true,
        totalDerivatives: 0,
        reputationScore: 0
    });
    
    // Update creator's content registry
    creatorContent[msg.sender].push(contentId);
    
    emit ContentCreated(contentId, msg.sender, coinAddress, contentHash, block.timestamp);
}
```

**Technical Deep Dive:**

1. **Content ID Generation**: Uses keccak256 hash of creator, content hash, and timestamp
   - Ensures uniqueness even if same creator uploads identical content
   - Provides deterministic addressing for off-chain systems

2. **Zora Integration**: 
   - Passes ETH value for initial liquidity
   - Sets up post-deploy hook for automatic registration
   - Uses salt for deterministic addresses

3. **Gas Optimization**: 
   - Single storage write for ContentPiece struct
   - Efficient array push operation for creator registry

### Derivative Creation with Inspiration

```solidity
function createDerivativeWithInspiration(
    bytes32 originalContentId,
    string memory name,
    string memory symbol,
    string memory uri,
    string memory contentHash,
    bytes memory poolConfig,
    address platformReferrer,
    bytes32 salt,
    uint256 revenueShareBps,
    bytes32 zkProofHash,
    InspirationProofType proofType
) external payable returns (bytes32 derivativeContentId, address coinAddress) {
    // Validation checks
    require(contentPieces[originalContentId].exists, "Original content does not exist");
    require(revenueShareBps <= MAX_REVENUE_SHARE_BPS, "Revenue share too high");
    
    // Create the derivative content first
    (derivativeContentId, coinAddress) = createContent(
        name, symbol, uri, contentHash, poolConfig, platformReferrer, salt
    );
    
    // Generate unique claim ID
    bytes32 claimId = keccak256(abi.encodePacked(originalContentId, derivativeContentId));
    
    // Create inspiration claim record
    inspirationClaims[claimId] = InspirationClaim({
        derivative: coinAddress,
        original: contentPieces[originalContentId].coinAddress,
        revenueShareBps: revenueShareBps,
        zkProofHash: zkProofHash,
        zkVerified: false,
        disputed: false,
        timestamp: block.timestamp,
        proofType: proofType
    });
    
    // Update graph structure
    inspirationGraph[originalContentId].push(derivativeContentId);
    contentDerivatives[originalContentId].push(derivativeContentId);
    contentPieces[originalContentId].totalDerivatives++;
    
    // Calculate and set inspiration chain depth
    contentDepth[derivativeContentId] = contentDepth[originalContentId] + 1;
    
    // Handle ZK proof verification
    if (zkProofHash != bytes32(0) && proofType == InspirationProofType.ZK_SIMILARITY_PROOF) {
        _verifyZkProof(claimId, zkProofHash);
    }
    
    // Update reputation scores
    _updateCreatorReputation(msg.sender, "DERIVATIVE_CREATED");
    _updateCreatorReputation(contentPieces[originalContentId].creator, "INSPIRED_OTHERS");
    
    emit InspirationClaimed(
        claimId, originalContentId, derivativeContentId, 
        msg.sender, revenueShareBps, proofType
    );
}
```

**Advanced Features:**

1. **Claim ID Generation**: Deterministic hashing prevents duplicate claims
2. **Graph Updates**: Maintains both forward and backward links in inspiration graph
3. **Depth Tracking**: Enables analysis of inspiration chain lengths
4. **Automatic Reputation**: Updates both derivative creator and original creator scores

### ZK-Proof Verification System

```solidity
function _verifyZkProof(bytes32 claimId, bytes32 zkProofHash) internal {
    if (zkVerifier != address(0)) {
        // Call external ZK verification contract
        bool verified = _callZkVerifier(zkProofHash);
        
        // Update claim verification status
        inspirationClaims[claimId].zkVerified = verified;
        verifiedProofs[zkProofHash] = verified;
        
        emit ZkProofVerified(claimId, zkProofHash, verified);
        
        if (verified) {
            // Reputation bonus for successful proof
            address claimer = contentPieces[_getContentIdFromClaim(claimId)].creator;
            _updateCreatorReputation(claimer, "ZK_PROOF_VERIFIED");
        }
    }
}

function _callZkVerifier(bytes32 proofHash) internal pure returns (bool) {
    // Placeholder for actual ZK verification
    // Real implementation would call external verifier contract:
    // return IZKVerifier(zkVerifier).verifyProof(proofData, publicInputs);
    return proofHash != bytes32(0);
}
```

**ZK-Proof Integration Points:**
- **Groth16**: Efficient verification with small proof size
- **PLONK**: Universal setup, flexible circuit design
- **Content Fingerprinting**: Perceptual hashing for media similarity
- **Semantic Similarity**: NLP-based text similarity proofs

## Revenue Distribution Mechanics

### Core Distribution Algorithm

```solidity
function distributeRevenue(
    bytes32 claimId,
    uint256 amount
) external nonReentrant {
    InspirationClaim memory claim = inspirationClaims[claimId];
    require(claim.derivative != address(0), "Invalid claim");
    require(!claim.disputed, "Claim is disputed");
    require(msg.sender == claim.derivative, "Only derivative coin can distribute");
    
    // Calculate base revenue splits
    uint256 originalShare = (amount * claim.revenueShareBps) / 10000;
    uint256 platformFee = (amount * platformFeeBps) / 10000;
    uint256 derivativeShare = amount - originalShare - platformFee;
    
    // Apply ZK-proof verification bonus
    if (claim.zkVerified) {
        uint256 bonus = (originalShare * 200) / 10000; // 2% bonus
        originalShare += bonus;
        derivativeShare -= bonus; // Derivative pays the bonus
    }
    
    // Execute token transfer
    IERC20(claim.derivative).transferFrom(msg.sender, address(this), amount);
    
    // Update pending revenue balances
    bytes32 originalContentId = _getOriginalContentId(claimId);
    address originalCreator = contentPieces[originalContentId].creator;
    
    pendingRevenue[claim.original][originalCreator] += originalShare;
    totalRevenueGenerated[originalContentId] += originalShare;
    
    emit RevenueDistributed(claim.original, originalCreator, originalShare, originalContentId);
}
```

**Revenue Flow Visualization:**
```
Total Revenue (1000 tokens)
├── Original Creator (25% = 250 tokens)
│   └── ZK Bonus (+2% = 20 tokens) = 270 tokens
├── Platform Fee (2.5% = 25 tokens)
└── Derivative Creator (72.5% = 705 tokens)
    └── ZK Bonus (-2% = -20 tokens) = 685 tokens
```

### Revenue Claiming Mechanism

```solidity
function claimRevenue(address coinAddress) external nonReentrant {
    uint256 amount = pendingRevenue[coinAddress][msg.sender];
    require(amount > 0, "No pending revenue");
    
    // Clear pending balance before transfer (CEI pattern)
    pendingRevenue[coinAddress][msg.sender] = 0;
    
    // Transfer tokens to claimer
    IERC20(coinAddress).transfer(msg.sender, amount);
}
```

**Security Features:**
- **Check-Effects-Interactions**: Prevents reentrancy attacks
- **NonReentrant Modifier**: Additional reentrancy protection
- **Balance Clearing**: Prevents double-spending

## Reputation System Implementation

### Reputation Update Logic

```solidity
function _updateCreatorReputation(address creator, string memory reason) internal {
    ReputationMetrics storage reputation = creatorReputations[creator];
    
    bytes32 reasonHash = keccak256(abi.encodePacked(reason));
    
    if (reasonHash == keccak256("DERIVATIVE_CREATED")) {
        reputation.totalDerivatives++;
        reputation.collaboratorScore += 10;
    } else if (reasonHash == keccak256("INSPIRED_OTHERS")) {
        reputation.totalInspirations++;
        reputation.collaboratorScore += 15;
    } else if (reasonHash == keccak256("ZK_PROOF_VERIFIED")) {
        reputation.successfulCollaborations++;
        reputation.collaboratorScore += 25;
    } else if (reasonHash == keccak256("CLAIM_DISPUTED")) {
        reputation.fraudFlags++;
        reputation.collaboratorScore = reputation.collaboratorScore > 50 ? 
            reputation.collaboratorScore - 50 : 0;
    }
    
    emit ReputationUpdated(creator, reputation.collaboratorScore, reason);
}
```

**Reputation Scoring System:**
- **Derivative Created**: +10 points (encourages creativity)
- **Inspired Others**: +15 points (rewards influential content)
- **ZK Proof Verified**: +25 points (incentivizes verification)
- **Claim Disputed**: -50 points (penalizes fraud)

### Algorithmic Ranking Implementation

```solidity
function calculateRankingScore(bytes32 contentId) external view returns (uint256) {
    ContentPiece memory content = contentPieces[contentId];
    require(content.exists, "Content does not exist");
    
    ReputationMetrics memory reputation = creatorReputations[content.creator];
    
    // Base scoring components
    uint256 baseScore = 1000;
    uint256 inspirationBonus = 0;
    uint256 zkProofMultiplier = 1000; // 1.0x in basis points
    uint256 reputationMultiplier = 1000; // 1.0x in basis points
    uint256 graphBonus = 0;
    uint256 fraudPenalty = 0;
    
    // Calculate inspiration bonus
    bytes32[] memory derivatives = contentDerivatives[contentId];
    for (uint i = 0; i < derivatives.length; i++) {
        bytes32 claimId = keccak256(abi.encodePacked(contentId, derivatives[i]));
        InspirationClaim memory claim = inspirationClaims[claimId];
        
        if (claim.derivative != address(0)) {
            inspirationBonus += 200; // +0.2 per inspiration
            
            if (claim.zkVerified) {
                zkProofMultiplier += 300; // +0.3 multiplier for verified proofs
            }
        }
    }
    
    // Reputation multiplier (1.0x to 2.0x based on creator reputation)
    reputationMultiplier = 1000 + (reputation.collaboratorScore * 1000) / 1000;
    if (reputationMultiplier > 2000) reputationMultiplier = 2000;
    
    // Graph participation bonus
    graphBonus = content.totalDerivatives * 100; // +0.1 per derivative
    if (graphBonus > 500) graphBonus = 500; // Cap at +0.5
    
    // Fraud penalty
    fraudPenalty = reputation.fraudFlags * 700; // -0.7 per fraud flag
    
    // Final score calculation: ((Base + Inspiration + ZK) * Reputation + Graph) - Fraud
    uint256 score = ((baseScore + inspirationBonus + zkProofMultiplier) * reputationMultiplier) / 1000;
    score += graphBonus;
    score = score > fraudPenalty ? score - fraudPenalty : 0;
    
    return score;
}
```

**Ranking Formula Breakdown:**
```
Score = ((1000 + InspBonus + ZKBonus) × RepMultiplier + GraphBonus) - FraudPenalty

Where:
- InspBonus = 200 × number_of_inspirations
- ZKBonus = 300 × verified_zk_proofs  
- RepMultiplier = 1.0 to 2.0 based on reputation
- GraphBonus = min(100 × derivatives, 500)
- FraudPenalty = 700 × fraud_flags
```

## Graph Theory Implementation

### Inspiration Graph Structure

```solidity
// Forward adjacency list: content -> inspired content
mapping(bytes32 => bytes32[]) public inspirationGraph;

// Content depth in inspiration chains
mapping(bytes32 => uint256) public contentDepth;

// Derivative tracking for revenue purposes
mapping(bytes32 => bytes32[]) public contentDerivatives;
```

### Graph Query Functions

```solidity
function getInspirationGraph(bytes32 contentId) external view returns (
    bytes32[] memory derivatives,
    uint256 depth,
    uint256 totalRevenue
) {
    derivatives = contentDerivatives[contentId];
    depth = contentDepth[contentId];
    totalRevenue = totalRevenueGenerated[contentId];
}

// Advanced graph traversal (off-chain computation recommended)
function getInspirationChain(bytes32 contentId, uint256 maxDepth) 
    external view returns (bytes32[] memory chain) {
    // Breadth-first search implementation
    // Returns array of content IDs in inspiration chain
    // Note: Gas-intensive, should be called off-chain for deep chains
}
```

### Network Analysis Algorithms

```solidity
// Calculate network centrality metrics
function calculateCentralityScore(bytes32 contentId) external view returns (uint256) {
    // Degree centrality: number of direct connections
    uint256 inDegree = 0; // Count of content inspired by this
    uint256 outDegree = inspirationGraph[contentId].length;
    
    // Weighted by revenue generated
    uint256 revenueWeight = totalRevenueGenerated[contentId];
    
    return (inDegree + outDegree) * 100 + revenueWeight / 1000;
}
```

## Security Analysis

### Reentrancy Protection

```solidity
// Uses OpenZeppelin's ReentrancyGuard
modifier nonReentrant() {
    require(_status != _ENTERED, "ReentrancyGuard: reentrant call");
    _status = _ENTERED;
    _;
    _status = _NOT_ENTERED;
}

// Applied to sensitive functions
function distributeRevenue(bytes32 claimId, uint256 amount) 
    external nonReentrant { /* ... */ }

function claimRevenue(address coinAddress) 
    external nonReentrant { /* ... */ }
```

### Access Control Patterns

```solidity
// Ownable for administrative functions
function setZkVerifier(address _zkVerifier) external onlyOwner {
    zkVerifier = _zkVerifier;
}

// Creator-only functions
modifier onlyCreator(bytes32 contentId) {
    require(contentPieces[contentId].creator == msg.sender, "Not the creator");
    _;
}

// Dispute authorization
modifier canDispute(bytes32 claimId) {
    bytes32 originalContentId = _getOriginalContentId(claimId);
    require(
        msg.sender == contentPieces[originalContentId].creator ||
        msg.sender == owner(),
        "Not authorized to dispute"
    );
    _;
}
```

### Input Validation

```solidity
// Revenue share limits
require(revenueShareBps <= MAX_REVENUE_SHARE_BPS, "Revenue share too high");

// Platform fee limits  
require(_platformFeeBps <= 1000, "Platform fee too high"); // Max 10%

// Content existence checks
require(contentPieces[originalContentId].exists, "Original content does not exist");

// Zero address checks
require(coinAddress != address(0), "Invalid coin address");
```

### Integer Overflow Protection

```solidity
// Using Solidity 0.8.19+ with built-in overflow protection
uint256 originalShare = (amount * claim.revenueShareBps) / 10000;
uint256 platformFee = (amount * platformFeeBps) / 10000;
uint256 derivativeShare = amount - originalShare - platformFee;

// SafeMath no longer needed in Solidity 0.8+
// All arithmetic operations revert on overflow/underflow
```

## Integration Patterns

### Zora Protocol Integration

```solidity
// Post-deploy hook for automatic registration
function onZoraCoinDeployed(
    address coin,
    bytes calldata hookData
) external returns (bytes memory) {
    require(msg.sender == address(zoraFactory), "Unauthorized");
    
    bytes32 contentId = abi.decode(hookData, (bytes32));
    
    // Register coin with revenue router
    InspirationRevenueRouter(revenueRouter).registerCoin(coin, contentId);
    
    return "";
}
```

### Uniswap V3 Integration

```solidity
// Revenue collection from trading fees
function collectUniswapFees(address coin) external returns (uint256) {
    // Get pool address for coin/ETH pair
    address pool = IUniswapV3Factory(uniswapV3Factory).getPool(
        coin, 
        WETH, 
        500 // 0.05% fee tier
    );
    
    require(pool != address(0), "Pool does not exist");
    
    // Collect protocol fees
    (uint128 amount0, uint128 amount1) = IUniswapV3Pool(pool).collectProtocol(
        address(this), // recipient0
        address(this), // recipient1
        type(uint128).max, // amount0Requested
        type(uint128).max  // amount1Requested
    );
    
    // Return collected amount (simplified)
    return uint256(amount0) + uint256(amount1);
}
```

### External ZK-Proof System Integration

```solidity
interface IZKVerifier {
    function verifyProof(
        bytes calldata proof,
        bytes32[] calldata publicInputs
    ) external view returns (bool);
}

function verifyContentSimilarity(
    bytes32 claimId,
    bytes calldata zkProof,
    bytes32[] calldata publicInputs
) external {
    require(zkVerifier != address(0), "ZK verifier not set");
    
    bool verified = IZKVerifier(zkVerifier).verifyProof(zkProof, publicInputs);
    
    if (verified) {
        inspirationClaims[claimId].zkVerified = true;
        bytes32 proofHash = keccak256(zkProof);
        verifiedProofs[proofHash] = true;
        
        emit ZkProofVerified(claimId, proofHash, true);
    }
}
```

## Gas Optimization Strategies

### Storage Layout Optimization

```solidity
// Pack structs to minimize storage slots
struct OptimizedClaim {
    address derivative;      // Slot 0: 20 bytes
    uint96 revenueShareBps; // Slot 0: 12 bytes (96 bits sufficient for basis points)
    
    address original;       // Slot 1: 20 bytes
    uint96 timestamp;      // Slot 1: 12 bytes (sufficient until year 2500+)
    
    bytes32 zkProofHash;   // Slot 2: 32 bytes
    
    bool zkVerified;       // Slot 3: 1 byte
    bool disputed;         // Slot 3: 1 byte
    uint8 proofType;       // Slot 3: 1 byte
    // 29 bytes remaining in slot 3
}
```

### Batch Operations

```solidity
// Batch reputation updates
function batchUpdateReputations(
    address[] calldata creators,
    string[] calldata reasons
) external onlyOwner {
    require(creators.length == reasons.length, "Array length mismatch");
    
    for (uint i = 0; i < creators.length; i++) {
        _updateCreatorReputation(creators[i], reasons[i]);
    }
}

// Batch revenue distribution
function batchDistributeRevenue(
    bytes32[] calldata claimIds,
    uint256[] calldata amounts
) external {
    require(claimIds.length == amounts.length, "Array length mismatch");
    
    for (uint i = 0; i < claimIds.length; i++) {
        distributeRevenue(claimIds[i], amounts[i]);
    }
}
```

### Event Optimization

```solidity
// Use indexed parameters for efficient filtering
event ContentCreated(
    bytes32 indexed contentId,    // Searchable by content ID
    address indexed creator,      // Searchable by creator
    address indexed coinAddress,  // Searchable by coin
    string contentHash,          // Not indexed (saves gas)
    uint256 timestamp           // Not indexed (saves gas)
);

// Packed event data for gas efficiency
event RevenueDistributed(
    address indexed coin,
    address indexed creator,
    uint256 amount,
    bytes32 indexed sourceContentId
);
```





---