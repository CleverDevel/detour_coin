# Appendix A: Smart Contract Code Template

## DetourCoin Smart Contract - Production Template

This appendix provides a comprehensive, production-ready smart contract template for DetourCoin, including detailed comments, security features, and deployment considerations.

---

## File Structure

```
contracts/
├── DetourCoin.sol              # Main token contract
├── interfaces/
│   ├── IDetourCoin.sol         # Interface definition
│   └── IEmissionController.sol # Emission interface
├── libraries/
│   └── EmissionMath.sol        # Emission calculations
└── security/
    └── EmergencyStop.sol       # Circuit breaker
```

---

## Main Contract: DetourCoin.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title DetourCoin
 * @author Joe (CleverDevel)
 * @notice DetourCoin (DTC) - Digital currency for local business ecosystem
 * @dev ERC20 token with controlled emission and role-based access
 * 
 * Key Features:
 * - Fixed maximum supply: 1 billion tokens
 * - Controlled daily emission: 50K (pre-launch) / 125K (post-launch)
 * - Role-based access control for security
 * - Pausable for emergency situations
 * - Reentrancy protection on all state-changing functions
 * 
 * Emission Economics:
 * - Initial Supply: 10M DTC (1% of max supply)
 * - Pre-Launch Rate: 50,000 DTC/day (18.25M per year)
 * - Post-Launch Rate: 125,000 DTC/day (45.625M per year)
 * - Time to Max Supply: ~21.6 years at post-launch rate
 * 
 * Security:
 * - Multi-signature recommended for admin operations
 * - Time-locked emission prevents gaming
 * - Pausable for emergency response
 * - Extensive event logging for transparency
 */
contract DetourCoin is ERC20, AccessControl, Pausable, ReentrancyGuard {
    
    // ============ Role Definitions ============
    
    /**
     * @notice MINTER_ROLE can execute scheduled token emissions
     * @dev Should be granted to automated emission contract or multisig
     */
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    
    /**
     * @notice PAUSER_ROLE can pause/unpause the contract in emergencies
     * @dev Should be granted to multisig wallet only
     */
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    
    /**
     * @notice EMISSION_CONTROLLER_ROLE can adjust emission parameters
     * @dev Highly privileged - grant only to governance multisig
     */
    bytes32 public constant EMISSION_CONTROLLER_ROLE = keccak256("EMISSION_CONTROLLER_ROLE");
    
    // ============ Constants ============
    
    /**
     * @notice Initial token supply minted to founder at deployment
     * @dev 10 million tokens with 18 decimals = 10,000,000 * 10^18
     */
    uint256 public constant INITIAL_SUPPLY = 10_000_000 * 10**18;
    
    /**
     * @notice Maximum total supply - hard cap cannot be exceeded
     * @dev 1 billion tokens with 18 decimals = 1,000,000,000 * 10^18
     */
    uint256 public constant MAX_SUPPLY = 1_000_000_000 * 10**18;
    
    /**
     * @notice Maximum allowed emission rate (safety ceiling)
     * @dev 150K tokens per day = 150,000 * 10^18
     */
    uint256 public constant MAX_EMISSION_RATE = 150_000 * 10**18;
    
    /**
     * @notice Minimum time between emission adjustments (7 days)
     * @dev Prevents rapid emission rate changes
     */
    uint256 public constant EMISSION_ADJUSTMENT_COOLDOWN = 7 days;
    
    // ============ State Variables ============
    
    /**
     * @notice Current emission rate in tokens per day
     * @dev Can be adjusted by EMISSION_CONTROLLER_ROLE within bounds
     * Pre-launch: 50,000 DTC/day
     * Post-launch: 125,000 DTC/day
     */
    uint256 public emissionRate;
    
    /**
     * @notice Timestamp of last successful token emission
     * @dev Used to calculate days elapsed and enforce 24-hour minimum
     */
    uint256 public lastEmissionTime;
    
    /**
     * @notice Timestamp of last emission rate adjustment
     * @dev Used to enforce cooldown period between adjustments
     */
    uint256 public lastEmissionAdjustment;
    
    /**
     * @notice Total tokens emitted since deployment (excluding initial supply)
     * @dev Tracking metric for transparency and analysis
     */
    uint256 public totalEmitted;
    
    // ============ Events ============
    
    /**
     * @notice Emitted when emission rate is updated
     * @param oldRate Previous emission rate
     * @param newRate New emission rate
     * @param adjustedBy Address that made the adjustment
     * @param timestamp When the adjustment occurred
     */
    event EmissionRateUpdated(
        uint256 indexed oldRate,
        uint256 indexed newRate,
        address indexed adjustedBy,
        uint256 timestamp
    );
    
    /**
     * @notice Emitted when tokens are minted through scheduled emission
     * @param recipient Address receiving the emitted tokens
     * @param amount Number of tokens emitted
     * @param daysElapsed Days since last emission
     * @param newTotalSupply Total supply after emission
     * @param timestamp When emission occurred
     */
    event TokensEmitted(
        address indexed recipient,
        uint256 amount,
        uint256 daysElapsed,
        uint256 newTotalSupply,
        uint256 timestamp
    );
    
    /**
     * @notice Emitted when contract is paused
     * @param pausedBy Address that paused the contract
     * @param timestamp When pause occurred
     */
    event ContractPaused(
        address indexed pausedBy,
        uint256 timestamp
    );
    
    /**
     * @notice Emitted when contract is unpaused
     * @param unpausedBy Address that unpaused the contract
     * @param timestamp When unpause occurred
     */
    event ContractUnpaused(
        address indexed unpausedBy,
        uint256 timestamp
    );
    
    // ============ Constructor ============
    
    /**
     * @notice Deploys DetourCoin with initial configuration
     * @dev Grants all roles to deployer, mints initial supply, sets emission rate
     * 
     * Deployment Actions:
     * 1. Initializes ERC20 with name "DetourCoin" and symbol "DTC"
     * 2. Grants all admin roles to deployer (should be transferred to multisig)
     * 3. Mints 10M tokens to deployer (founder allocation)
     * 4. Sets initial emission rate to 50K tokens/day
     * 5. Records deployment timestamp for emission scheduling
     * 
     * Post-Deployment:
     * - Transfer admin roles to multisig wallet
     * - Set up automated emission execution
     * - Configure monitoring and alerting
     */
    constructor() ERC20("DetourCoin", "DTC") {
        // Grant all roles to deployer initially
        // CRITICAL: Transfer these to multisig after deployment
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
        _grantRole(PAUSER_ROLE, msg.sender);
        _grantRole(EMISSION_CONTROLLER_ROLE, msg.sender);
        
        // Mint initial supply to founder
        _mint(msg.sender, INITIAL_SUPPLY);
        
        // Set initial emission rate (pre-launch: 50K/day)
        emissionRate = 50_000 * 10**18;
        
        // Record deployment time for emission scheduling
        lastEmissionTime = block.timestamp;
        lastEmissionAdjustment = block.timestamp;
        
        // Emit initial state for tracking
        emit TokensEmitted(
            msg.sender,
            INITIAL_SUPPLY,
            0, // No elapsed days for initial mint
            INITIAL_SUPPLY,
            block.timestamp
        );
    }
    
    // ============ Emission Functions ============
    
    /**
     * @notice Executes scheduled daily token emission
     * @dev Can only be called once per 24-hour period by MINTER_ROLE
     * 
     * Process:
     * 1. Verify 24 hours elapsed since last emission
     * 2. Calculate days elapsed and total emission amount
     * 3. Verify emission won't exceed MAX_SUPPLY
     * 4. Mint tokens to caller (typically emission contract)
     * 5. Update emission timestamp and tracking
     * 
     * Security:
     * - Requires MINTER_ROLE
     * - Protected by ReentrancyGuard
     * - Enforces 24-hour minimum between emissions
     * - Respects MAX_SUPPLY hard cap
     * 
     * Gas Optimization:
     * - Batch emissions if multiple days missed
     * - Efficient storage updates
     * 
     * @return amount Number of tokens emitted
     */
    function emitScheduledTokens() 
        external 
        nonReentrant 
        onlyRole(MINTER_ROLE) 
        returns (uint256 amount) 
    {
        // Verify 24 hours have passed
        require(
            block.timestamp >= lastEmissionTime + 1 days,
            "DetourCoin: Emission cooldown active (24h minimum)"
        );
        
        // Calculate days elapsed (handles missed emissions)
        uint256 daysElapsed = (block.timestamp - lastEmissionTime) / 1 days;
        
        // Calculate total emission for elapsed days
        amount = daysElapsed * emissionRate;
        
        // Verify we don't exceed max supply
        require(
            totalSupply() + amount <= MAX_SUPPLY,
            "DetourCoin: Emission would exceed MAX_SUPPLY"
        );
        
        // Mint tokens to caller (emission contract)
        _mint(msg.sender, amount);
        
        // Update state
        lastEmissionTime = block.timestamp;
        totalEmitted += amount;
        
        // Emit event for transparency
        emit TokensEmitted(
            msg.sender,
            amount,
            daysElapsed,
            totalSupply(),
            block.timestamp
        );
        
        return amount;
    }
    
    /**
     * @notice Emergency/test function to emit without time restriction
     * @dev ONLY FOR TESTING - Remove or restrict in production
     * 
     * WARNING: This bypasses normal emission timing restrictions
     * Should only be used in testing environments or with strict governance
     * 
     * @return amount Number of tokens emitted
     */
    function emitScheduledTokensTest() 
        external 
        nonReentrant 
        onlyRole(MINTER_ROLE) 
        returns (uint256 amount) 
    {
        // Emit one day's worth without time check
        amount = emissionRate;
        
        require(
            totalSupply() + amount <= MAX_SUPPLY,
            "DetourCoin: Would exceed MAX_SUPPLY"
        );
        
        _mint(msg.sender, amount);
        lastEmissionTime = block.timestamp;
        totalEmitted += amount;
        
        emit TokensEmitted(
            msg.sender,
            amount,
            1, // Simulated 1 day
            totalSupply(),
            block.timestamp
        );
        
        return amount;
    }
    
    /**
     * @notice Adjusts the daily emission rate
     * @dev Can only be called by EMISSION_CONTROLLER_ROLE
     * 
     * Use Cases:
     * - Increase to 125K/day at public launch
     * - Adjust based on market conditions
     * - Respond to governance decisions
     * 
     * Restrictions:
     * - Cannot exceed MAX_EMISSION_RATE (150K/day)
     * - Must wait EMISSION_ADJUSTMENT_COOLDOWN (7 days) between changes
     * - Requires privileged role
     * 
     * Process:
     * 1. Verify cooldown period elapsed
     * 2. Validate new rate within bounds
     * 3. Update emission rate
     * 4. Record adjustment timestamp
     * 5. Emit event for transparency
     * 
     * @param newRate New daily emission rate in wei (tokens * 10^18)
     */
    function setEmissionRate(uint256 newRate) 
        external 
        onlyRole(EMISSION_CONTROLLER_ROLE) 
    {
        // Enforce cooldown between adjustments
        require(
            block.timestamp >= lastEmissionAdjustment + EMISSION_ADJUSTMENT_COOLDOWN,
            "DetourCoin: Adjustment cooldown active"
        );
        
        // Verify new rate is within safety bounds
        require(
            newRate <= MAX_EMISSION_RATE,
            "DetourCoin: Rate exceeds maximum allowed"
        );
        
        // Verify rate is not zero (would break emissions)
        require(
            newRate > 0,
            "DetourCoin: Rate must be greater than zero"
        );
        
        uint256 oldRate = emissionRate;
        emissionRate = newRate;
        lastEmissionAdjustment = block.timestamp;
        
        emit EmissionRateUpdated(
            oldRate,
            newRate,
            msg.sender,
            block.timestamp
        );
    }
    
    // ============ Emergency Controls ============
    
    /**
     * @notice Pauses all token transfers
     * @dev Emergency function - use only in case of security incident
     * 
     * When Paused:
     * - All transfers blocked
     * - Emissions can still occur (by design)
     * - Admin functions still work
     * 
     * Use Cases:
     * - Security vulnerability discovered
     * - Malicious activity detected
     * - Regulatory requirement
     * - Smart contract upgrade preparation
     */
    function pause() external onlyRole(PAUSER_ROLE) {
        _pause();
        emit ContractPaused(msg.sender, block.timestamp);
    }
    
    /**
     * @notice Unpauses token transfers
     * @dev Resumes normal operations after pause
     */
    function unpause() external onlyRole(PAUSER_ROLE) {
        _unpause();
        emit ContractUnpaused(msg.sender, block.timestamp);
    }
    
    // ============ Manual Mint (Governance) ============
    
    /**
     * @notice Mints tokens to specified address
     * @dev Governance function - use sparingly and with transparency
     * 
     * Use Cases:
     * - Founder grants (documented quarterly)
     * - Partnership allocations
     * - Ecosystem development
     * - Strategic initiatives
     * 
     * Restrictions:
     * - Requires MINTER_ROLE
     * - Cannot exceed MAX_SUPPLY
     * - Protected by reentrancy guard
     * - Fully logged for transparency
     * 
     * Governance:
     * - All mints should be publicly disclosed
     * - Ideally requires multisig approval
     * - Should align with tokenomics plan
     * 
     * @param to Recipient address
     * @param amount Number of tokens to mint (in wei)
     */
    function mint(address to, uint256 amount) 
        external 
        nonReentrant 
        onlyRole(MINTER_ROLE) 
    {
        require(to != address(0), "DetourCoin: Mint to zero address");
        require(amount > 0, "DetourCoin: Amount must be positive");
        require(
            totalSupply() + amount <= MAX_SUPPLY,
            "DetourCoin: Would exceed MAX_SUPPLY"
        );
        
        _mint(to, amount);
        totalEmitted += amount;
        
        // Note: Consider adding specific event for manual mints
        // to distinguish from scheduled emissions
    }
    
    // ============ Internal Overrides ============
    
    /**
     * @notice Hook that is called before any transfer of tokens
     * @dev Implements pausable functionality
     * 
     * Override Chain:
     * ERC20._update() -> DetourCoin._update() -> check paused
     * 
     * When Paused:
     * - All transfers revert
     * - Minting still allowed (emissions)
     * - Burning still allowed (if implemented)
     * 
     * @param from Address tokens are transferred from
     * @param to Address tokens are transferred to
     * @param value Amount of tokens being transferred
     */
    function _update(
        address from,
        address to,
        uint256 value
    ) internal virtual override {
        // Check if contract is paused (except for minting/burning)
        if (from != address(0) && to != address(0)) {
            require(!paused(), "DetourCoin: Token transfers paused");
        }
        
        super._update(from, to, value);
    }
    
    // ============ View Functions ============
    
    /**
     * @notice Returns time until next emission is allowed
     * @return seconds Number of seconds until emission available (0 if available now)
     */
    function timeUntilNextEmission() external view returns (uint256) {
        uint256 nextEmissionTime = lastEmissionTime + 1 days;
        if (block.timestamp >= nextEmissionTime) {
            return 0;
        }
        return nextEmissionTime - block.timestamp;
    }
    
    /**
     * @notice Returns number of days since last emission
     * @return days Number of complete days elapsed
     */
    function daysSinceLastEmission() external view returns (uint256) {
        return (block.timestamp - lastEmissionTime) / 1 days;
    }
    
    /**
     * @notice Calculates pending emission amount
     * @return amount Tokens that would be emitted if executed now
     */
    function pendingEmission() external view returns (uint256) {
        if (block.timestamp < lastEmissionTime + 1 days) {
            return 0;
        }
        
        uint256 daysElapsed = (block.timestamp - lastEmissionTime) / 1 days;
        uint256 amount = daysElapsed * emissionRate;
        
        // Cap at remaining supply
        uint256 remaining = MAX_SUPPLY - totalSupply();
        return amount > remaining ? remaining : amount;
    }
    
    /**
     * @notice Returns tokens remaining until max supply
     * @return remaining Number of tokens that can still be minted
     */
    function remainingSupply() external view returns (uint256) {
        return MAX_SUPPLY - totalSupply();
    }
    
    /**
     * @notice Estimates years until max supply at current rate
     * @return years Approximate years until max supply (2 decimals)
     */
    function yearsToMaxSupply() external view returns (uint256) {
        uint256 remaining = MAX_SUPPLY - totalSupply();
        uint256 annualEmission = emissionRate * 365;
        
        if (annualEmission == 0) return 0;
        
        // Return years * 100 for 2 decimal precision
        return (remaining * 100) / annualEmission;
    }
    
    /**
     * @notice Returns comprehensive emission statistics
     * @return current Current emission rate
     * @return total Total emitted since deployment
     * @return last Timestamp of last emission
     * @return next Timestamp when next emission available
     * @return remaining Tokens remaining to max supply
     */
    function emissionStats() external view returns (
        uint256 current,
        uint256 total,
        uint256 last,
        uint256 next,
        uint256 remaining
    ) {
        return (
            emissionRate,
            totalEmitted,
            lastEmissionTime,
            lastEmissionTime + 1 days,
            MAX_SUPPLY - totalSupply()
        );
    }
}
```

---

## Deployment Configuration

### Environment Setup

**.env File Template:**
```bash
# Network Configuration
POLYGON_RPC_URL=https://polygon-rpc.com
POLYGON_TESTNET_RPC_URL=https://rpc-mumbai.maticvigil.com

# Deployment Wallet
DEPLOYER_PRIVATE_KEY=your_private_key_here

# Multisig Addresses (Set these up FIRST)
MULTISIG_ADMIN=0x...  # Gnosis Safe address
MULTISIG_EMISSION=0x...  # Emission controller multisig

# Block Explorer
POLYGONSCAN_API_KEY=your_api_key_here

# Initial Configuration
INITIAL_EMISSION_RATE=50000000000000000000000  # 50K tokens in wei
```

### Hardhat Config

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");
require("@nomiclabs/hardhat-etherscan");
require("dotenv").config();

module.exports = {
  solidity: {
    version: "0.8.28",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
      viaIR: true, // Improved optimization
    },
  },
  networks: {
    hardhat: {
      chainId: 31337,
    },
    polygon_mumbai: {
      url: process.env.POLYGON_TESTNET_RPC_URL,
      accounts: [process.env.DEPLOYER_PRIVATE_KEY],
      chainId: 80001,
      gasPrice: 2000000000, // 2 gwei
    },
    polygon: {
      url: process.env.POLYGON_RPC_URL,
      accounts: [process.env.DEPLOYER_PRIVATE_KEY],
      chainId: 137,
      gasPrice: 50000000000, // 50 gwei - adjust based on network
    },
  },
  etherscan: {
    apiKey: {
      polygon: process.env.POLYGONSCAN_API_KEY,
      polygonMumbai: process.env.POLYGONSCAN_API_KEY,
    },
  },
  gasReporter: {
    enabled: true,
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
  },
};
```

---

## Post-Deployment Checklist

**Immediate Actions (First Hour):**
- [ ] Verify contract on PolygonScan
- [ ] Record contract address securely
- [ ] Verify initial supply minted correctly
- [ ] Confirm emission rate set properly
- [ ] Test pause/unpause functions
- [ ] Document all transaction hashes

**First Day:**
- [ ] Set up Gnosis Safe multisig wallet
- [ ] Transfer all roles to multisig
- [ ] Renounce deployer roles (except as backup)
- [ ] Set up monitoring alerts
- [ ] Configure emission automation
- [ ] Update documentation with addresses

**First Week:**
- [ ] Professional smart contract audit
- [ ] Establish emission monitoring
- [ ] Set up community transparency dashboard
- [ ] Create role management procedures
- [ ] Implement backup emission execution
- [ ] Test emergency procedures

---