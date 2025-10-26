# AuraFi - Creator Vaults

AuraFi is a dual-collateral protocol where creators and fans jointly back creator tokens on Celo. Creators stake CELO to unlock minting stages, while fans deposit CELO to mint tokens. A creator's Aura score (derived from Farcaster activity) dynamically adjusts token value (peg) and supply caps, with forced contraction and liquidation mechanisms ensuring aligned incentives.


# Deployed Contract Address : 0xCB1799Aa5A71ccB295dD9f1fD5Fc7D21A89D4E98

  -factory: "0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0"
  
  -oracle: "0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512"
  
  -treasury: "0x5FbDB2315678afecb367f032d93F642f64180aa3"


## Overview

AuraFi implements a creator economy protocol with the following key features:

- **Dual Collateral Model**: Both creator stakes and fan deposits back the token supply, ensuring skin in the game for all participants
- **Aura-Anchored Economics**: Token peg (CELO-per-token) and supply caps derive from verifiable Farcaster metrics, creating dynamic value tied to creator reputation
- **Staged Progression**: Creators unlock higher mint capacity by staking more collateral, enabling gradual growth
- **Position-Based Accounting**: Each mint creates a traceable position for fair FIFO redemptions and proportional forced burns
- **Forced Contraction**: When aura drops and supply exceeds cap, tokens are burned proportionally after a 24-hour grace period
- **Liquidation Mechanism**: When vault health falls below 120%, liquidators can inject CELO to restore health and earn 1% bounties

### Key Concepts

**Positions**: Each time a fan mints tokens, a Position struct is created recording the quantity, collateral deposited (minus fees), stage at mint time, and timestamp. Positions are stored in an array per fan address, enabling:
- Fair FIFO redemptions (oldest positions redeemed first)
- Proportional forced burns (each position loses tokens/collateral pro-rata)
- Accurate collateral attribution (no global pool confusion)

**Stages**: Discrete progression levels (0-4 in default config) that gate minting capacity. Creators must stake cumulative CELO amounts to unlock each stage, which increases the maximum token supply fans can mint. Stage 0 = vault created but not bootstrapped (no minting allowed).

**Peg Calculation**: The CELO-per-token exchange rate is calculated dynamically as `P(aura) = BASE_PRICE * (1 + K * (aura/A_REF - 1))`, bounded between 0.3 and 3.0 CELO. Higher aura = higher peg = more valuable tokens.

**Health Ratio**: Vault collateralization calculated as `Health = totalCollateral / (totalSupply * peg)`. Must be ≥150% for minting, liquidatable if <120%.

**Forced Burn**: When oracle updates cause supply to exceed the aura-based supply cap, anyone can call `checkAndTriggerForcedBurn()` to start a 24-hour grace period. After the deadline, anyone can call `executeForcedBurn(maxOwners)` to proportionally burn tokens and write down collateral across all positions. Batched processing prevents gas limit issues with many position owners.

**Liquidation**: When health <120%, liquidators pay CELO via `liquidate()` to buy down supply, earning a 1% bounty. The protocol calculates how many tokens to burn to restore health to 150%, burns them pro-rata across positions, pays the bounty, adds remaining CELO to vault collateral, and extracts a creator penalty.

## Architecture

### Core Contracts

```
┌─────────────────┐
│  VaultFactory   │ ──creates──> ┌──────────────┐
└─────────────────┘               │ CreatorVault │
                                  └──────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
              ┌─────▼──────┐      ┌─────▼──────┐      ┌─────▼──────┐
              │CreatorToken│      │ AuraOracle │      │  Treasury  │
              └────────────┘      └────────────┘      └────────────┘
```

**VaultFactory** (`VaultFactory.sol`)
- Deploys CreatorVault and CreatorToken pairs for each creator
- Configures stage parameters (stake requirements, mint caps) per vault
- Maintains registry mapping creators to their vaults
- Owner-controlled for governance and configuration

**CreatorVault** (`CreatorVault.sol`)
- Core vault logic managing dual collateral (creator + fan)
- Enforces stage-gated minting with collateralization checks
- Tracks per-position accounting for fair FIFO redemptions
- Fetches current aura from AuraOracle dynamically (no stored aura)
- Calculates peg and supply cap on-demand based on oracle aura
- Executes forced contraction when supply exceeds aura-based cap
- Handles liquidations when health falls below threshold
- Implements reentrancy guards and pausable functionality

**CreatorToken** (`CreatorToken.sol`)
- Standard ERC20 token representing fan ownership
- Restricted mint/burn operations (vault-only access control)
- Each creator gets their own unique token contract
- Fully transferable between fans

**AuraOracle** (`AuraOracle.sol`)
- Stores aura values per vault with IPFS evidence hashes
- Single source of truth for all vaults to read from via `getAura(vault)`
- Enforces 6-hour update cooldown per vault to prevent manipulation
- Access-controlled to registered oracle address(es)
- Emits AuraUpdated events with vault, aura, ipfsHash, and timestamp
- Vaults never store aura; they fetch it dynamically for all calculations
- Oracle script updates only AuraOracle contract, not individual vaults

**Treasury** (`Treasury.sol`)
- Collects protocol fees from minting operations (0.5% of collateral)
- Owner-controlled withdrawal function for fee distribution
- Simple accumulator with event logging

### Key Mechanisms

**Health Ratio**: `Health = totalCollateral / (totalSupply * peg)`
- Minimum 150% (MIN_CR) required for minting and redemptions
- Below 120% (LIQ_CR) triggers liquidation eligibility
- Calculated dynamically using current peg from oracle aura

**Dynamic Peg**: `P(aura) = BASE_PRICE * (1 + K * (aura/A_REF - 1))`
- Linear interpolation with aura normalization
- Bounded between 0.3 CELO (P_MIN) and 3.0 CELO (P_MAX) per token
- Fetched dynamically from AuraOracle on every mint/redeem/liquidation
- Higher aura → higher peg → more valuable tokens

**Supply Cap**: `SupplyCap(aura) = BaseCap * (1 + s * (aura - A_REF) / A_REF)`
- Grows/shrinks with aura to maintain backing ratio
- Sensitivity parameter s = 0.75
- Clamped between BaseCap * 0.25 and BaseCap * 4
- Forced contraction triggered when supply exceeds cap after aura drop

**Forced Contraction Process**:
1. Oracle updates aura to lower value in AuraOracle contract
2. Anyone calls `checkAndTriggerForcedBurn()` on CreatorVault to detect supply > cap
3. 24-hour grace period begins (FORCED_BURN_GRACE), SupplyCapShrink event emitted
4. Fans can redeem during grace period to avoid forced burn
5. After deadline, anyone calls `executeForcedBurn(maxOwners)` to burn tokens pro-rata
6. Both token quantities and collateral are reduced proportionally across all positions
7. Multiple calls may be needed for vaults with many position owners (batched processing)

**Liquidation Process**:
1. Vault health falls below 120% (due to aura drop, redemptions, or peg increase)
2. Liquidator calls `liquidate()` with CELO payment (payCELO)
3. Protocol calculates tokens to burn: `x = supply - floor((collateral + payCELO) / (peg * MIN_CR))`
4. Tokens burned pro-rata across all positions
5. Liquidator receives 1% bounty immediately
6. Remaining payCELO added to vault collateral
7. Creator penalty extracted from creatorCollateral
8. Vault health restored to ≥150%

## Project Structure

```
├── contracts/              # Solidity smart contracts
│   ├── CreatorVault.sol    # Core vault logic with dual collateral
│   ├── VaultFactory.sol    # Factory for deploying vaults
│   ├── CreatorToken.sol    # ERC20 token with restricted minting
│   ├── AuraOracle.sol      # Oracle for storing aura values
│   ├── Treasury.sol        # Fee collection contract
│   └── DependencyCheck.sol # OpenZeppelin dependency verification
├── script/                 # Deployment scripts
│   └── Deploy.s.sol        # Main deployment script
├── test/                   # Comprehensive contract tests
│   ├── CreatorStake.t.sol  # Creator staking and stage unlock tests
│   ├── FanMinting.t.sol    # Fan minting with position tracking tests
│   ├── TokenRedemption.t.sol # Token redemption tests
│   ├── ForcedContraction.t.sol # Forced burn mechanism tests
│   ├── Liquidation.t.sol   # Liquidation mechanism tests
│   ├── AuraOracle.t.sol    # Oracle tests
│   └── ...                 # Additional test files
└── oracle/                 # Off-chain oracle for aura computation
    ├── oracle.js           # Main oracle script
    ├── test-oracle.js      # Oracle tests
    └── README.md           # Oracle documentation
```

## Quick Start

### 1. Install Dependencies

**Smart Contracts:**

```shell
forge install
```

**Oracle:**

```shell
cd oracle
npm install
```

### 2. Build Contracts

```shell
forge build
```

### 3. Run Tests

**Contract Tests:**

```shell
forge test
```

**Verbose output:**

```shell
forge test -vvv
```

**Oracle Tests:**

```shell
cd oracle
npm test
```

### 4. Configure Oracle

Copy the example environment file:

```shell
cd oracle
cp .env.example .env.local
```

Edit `.env.local` with your configuration:

- `NEYNAR_API_KEY` - For Farcaster data (free tier works)
- `PINATA_API_KEY` / `PINATA_SECRET_KEY` - For IPFS pinning
- `ORACLE_PRIVATE_KEY` - Oracle wallet private key
- `RPC_URL` - Celo Alfajores RPC endpoint
- `AURA_ORACLE_ADDRESS` - Deployed AuraOracle contract address

### 5. Deploy Contracts

Deploy to Celo Alfajores testnet:

```shell
forge script script/Deploy.s.sol --rpc-url https://alfajores-forno.celo-testnet.org --private-key <your_private_key> --broadcast
```

This deploys:
- Treasury contract
- AuraOracle contract
- VaultFactory contract

Deployment addresses are saved to `deployments.json`.

### 6. Create a Creator Vault

Use the VaultFactory to create a vault (baseCap in wei, e.g., 100000e18 = 100,000 tokens):

```shell
cast send <FACTORY_ADDRESS> "createVault(string,string,address,uint256)" "CreatorName" "CNAME" <CREATOR_ADDRESS> 100000000000000000000000 --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>
```

The factory will emit a `VaultCreated` event with the new vault and token addresses.

### 7. Run Oracle

Test with mock data:

```shell
cd oracle
node oracle.js --vault <vault_address> --fid <farcaster_id> --mock --dry-run
```

Run with live data:

```shell
node oracle.js --vault <vault_address> --fid <farcaster_id>
```

## Key View Functions

### CreatorVault View Functions

```solidity
// Get current aura from oracle
function getCurrentAura() external view returns (uint256)

// Get current peg (CELO per token)
function getPeg() external view returns (uint256)

// Get current supply cap based on aura
function getCurrentSupplyCap() external view returns (uint256)

// Get comprehensive vault state
function getVaultState() external view returns (
    uint256 creatorCollateral,
    uint256 fanCollateral,
    uint256 totalCollateral,
    uint256 totalSupply,
    uint256 peg,
    uint8 stage,
    uint256 health
)

// Get specific position for a fan
function getPosition(address owner, uint256 index) external view returns (Position memory)

// Get number of positions for a fan
function getPositionCount(address owner) external view returns (uint256)
```

### AuraOracle View Functions

```solidity
// Get current aura for a vault
function getAura(address vault) external view returns (uint256)
```

## Usage Examples

### Creator Flow

1. **Create Vault**: Call `VaultFactory.createVault(name, symbol, creator, baseCap)` with token name, symbol, creator address, and base supply cap
2. **Bootstrap Stake**: Call `CreatorVault.bootstrapCreatorStake()` payable with sufficient CELO to unlock stage 1 (default: 100 CELO)
3. **Unlock Stages**: Call `CreatorVault.unlockStage()` payable with additional CELO to unlock higher stages and increase mint capacity
4. **Monitor Health**: Watch vault health via `getVaultState()` to avoid liquidation penalties

### Fan Flow

1. **Check Requirements**: Call `getPeg()` and calculate required collateral: `qty * peg * 1.5 * 1.005` (150% collateral + 0.5% fee)
2. **Mint Tokens**: Call `CreatorVault.mintTokens(qty)` payable with required CELO, creates a Position
3. **Hold Tokens**: Token value tracks creator's aura performance (higher aura = higher peg)
4. **Monitor Positions**: Call `getPosition(address, index)` and `getPositionCount(address)` to view your positions
5. **Redeem Tokens**: Call `CreatorVault.redeemTokens(qty)` to burn tokens and recover collateral (FIFO order)
6. **Watch for Forced Burns**: Monitor SupplyCapShrink events and redeem during grace period if needed

### Oracle Flow

1. **Fetch Metrics**: Oracle script fetches Farcaster metrics for creator (followers, engagement, verification)
2. **Compute Aura**: Weighted normalization with log-based scaling to 0-200 range
3. **Pin Evidence**: Store raw metrics and computation on IPFS via Pinata
4. **Update On-Chain**: Call `AuraOracle.pushAura()` with vault address, aura, and IPFS hash
5. **Vault Reads**: CreatorVault automatically fetches latest aura from AuraOracle on all operations

### Forced Contraction Flow

1. **Aura Drops**: Oracle updates aura to lower value in AuraOracle contract
2. **Trigger Check**: Anyone calls `CreatorVault.checkAndTriggerForcedBurn()` to detect supply > cap
3. **Grace Period**: 24-hour window begins, SupplyCapShrink event emitted with deadline
4. **Fan Exit Window**: Fans can call `redeemTokens()` during grace period to avoid forced burn
5. **Execute Burn**: After deadline, anyone calls `CreatorVault.executeForcedBurn(maxOwners)` to burn tokens pro-rata
6. **Batched Processing**: For vaults with many owners, multiple calls may be needed (e.g., maxOwners=20 per call)

### Liquidation Flow

1. **Health Degrades**: Vault health falls below 120% (e.g., due to aura drop, redemptions, or peg increase)
2. **Check Liquidatable**: Anyone can call `getVaultState()` to check if health < LIQ_CR (120%)
3. **Liquidator Acts**: Liquidator calls `CreatorVault.liquidate()` payable with CELO payment (must be >= minimum threshold)
4. **Calculate Burn**: Protocol calculates tokens to burn: `x = supply - floor((collateral + payCELO) / (peg * MIN_CR))`
5. **Tokens Burned**: Burn x tokens pro-rata across all positions
6. **Bounty Paid**: Liquidator receives 1% bounty immediately
7. **Health Restored**: Remaining payCELO added to vault collateral, creator penalty extracted, health restored to ≥150%

## Deployed Contracts (Celo Alfajores)

- **Treasury**: `0x1205E28b0e1A0E3Bf968908d9AD9Ac073A1F12eE`
- **AuraOracle**: `0xa585e63cfAeFc513198d70FbA741B22d8116C2d0`
- **VaultFactory**: `0x3A788A0d02BD1691E46aCcF296518574fcd919A6`

View on [Celo Explorer](https://alfajores.celoscan.io/)

## Key Parameters (MVP)

### Peg Function
- `BASE_PRICE`: 1 CELO
- `A_REF`: 100 (baseline aura)
- `K`: 0.5 (sensitivity)
- `P_MIN`: 0.3 CELO
- `P_MAX`: 3.0 CELO

### Collateralization
- `MIN_CR`: 150% (minimum collateralization ratio)
- `LIQ_CR`: 120% (liquidation threshold)
- `MINT_FEE`: 0.5%
- `LIQUIDATION_BOUNTY`: 1%

### Time Windows
- `FORCED_BURN_GRACE`: 24 hours
- `ORACLE_UPDATE_COOLDOWN`: 6 hours

### Default Stage Configurations
- Stage 0: 0 CELO stake, 0 tokens capacity
- Stage 1: 100 CELO stake, 500 tokens capacity
- Stage 2: 300 CELO stake, 2500 tokens capacity
- Stage 3: 800 CELO stake, 9500 tokens capacity
- Stage 4: 1800 CELO stake, 34500 tokens capacity

## Security Considerations

### Access Control
- CreatorToken mint/burn restricted to vault contract only
- AuraOracle updates restricted to registered oracle address
- VaultFactory admin functions restricted to owner
- All state-changing functions protected with reentrancy guards

### Economic Security
- Dual collateral ensures skin in the game for both creators and fans
- Forced contraction maintains peg integrity after aura drops
- Liquidation mechanism prevents undercollateralization
- Grace periods allow orderly exits before forced burns
- Minimum payment thresholds prevent griefing attacks

### Oracle Trust Model
- MVP uses single oracle address (dev/CI key)
- All updates include IPFS evidence hash for verification
- Cooldown prevents manipulation via rapid updates
- Future: Multi-oracle consensus (Chainlink/UMA)

### Audit Status
- **Status**: Unaudited MVP
- **Recommendation**: Do not use with significant funds
- **Testnet Only**: Currently deployed on Celo Alfajores testnet
- **Future**: Professional audit required before mainnet deployment

## MVP Limitations

### Current Scope
- Single oracle address (centralized trust)
- No governance mechanism for parameter changes
- No vault insurance fund
- No cross-vault liquidity
- Manual forced burn execution (not automated)
- Basic aura algorithm (can be gamed with Sybil accounts)

### Known Issues
- Gas costs for large position sets (100+ positions) require batched forced burn execution
- No TWAP for peg calculations (potential flash loan risk in extreme scenarios)
- Creator can abandon vault after fans mint (no ongoing creator obligations)
- No slashing mechanism for creator misbehavior beyond liquidation penalties
- Forced burn must be triggered manually (not automated on oracle updates)

### Not Implemented
- Quest/prediction markets
- Multi-oracle consensus
- Governance token
- Dynamic fee structures
- NFT-gated stages
- Layer 2 deployment

## Future Enhancements

### Phase 2 (Post-MVP)
- Multi-oracle consensus with Chainlink/UMA
- Governance token for parameter tuning
- Vault insurance fund from protocol fees
- Automated forced burn execution
- Enhanced aura algorithm with Sybil resistance

### Phase 3 (Advanced)
- Cross-vault liquidity pools
- Quest and prediction markets
- NFT-gated stages and perks
- Dynamic fee structures based on vault performance
- Layer 2 deployment for lower fees

### Optimizations
- Gas-optimized position storage (packed structs)
- Merkle trees for large position sets
- Batch operations for multiple vaults
- TWAP oracle integration

## Documentation

- [Oracle Documentation](oracle/README.md) - Detailed oracle setup and usage
- [Design Document](.kiro/specs/aurafi-creator-vaults/design.md) - Comprehensive design and architecture
- [Requirements Document](.kiro/specs/aurafi-creator-vaults/requirements.md) - Formal requirements specification
- [Foundry Book](https://book.getfoundry.sh/) - Smart contract development framework

## Foundry Commands

### Build

```shell
forge build
```

### Test

```shell
forge test
```

### Test with Verbosity

```shell
forge test -vvv
```

### Format

```shell
forge fmt
```

### Gas Snapshots

```shell
forge snapshot
```

### Local Node

```shell
anvil
```

### Deploy

```shell
forge script script/Deploy.s.sol --rpc-url <RPC_URL> --private-key <PRIVATE_KEY> --broadcast
```

### Cast (Interact with Contracts)

```shell
cast call <CONTRACT_ADDRESS> "functionName()" --rpc-url <RPC_URL>
cast send <CONTRACT_ADDRESS> "functionName(args)" --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>
```

### Help

```shell
forge --help
anvil --help
cast --help
```

## Troubleshooting

### Common Issues

**"StageNotUnlocked" error when minting**
- Creator must call `bootstrapCreatorStake()` first to unlock stage 1
- Check current stage with `getVaultState()`

**"InsufficientCollateral" error when minting**
- Calculate required amount: `qty * getPeg() * 1.5 * 1.005`
- Peg changes with aura, so check `getPeg()` before minting

**"ExceedsSupplyCap" error when minting**
- Current supply exceeds aura-based cap
- Wait for aura to increase or for forced burn to execute
- Check `getCurrentSupplyCap()` vs `totalSupply`

**"HealthTooLow" error when redeeming**
- Redemption would drop health below 150%
- Reduce redemption quantity or wait for more minting
- Check `getVaultState()` for current health

**"GracePeriodActive" error when executing forced burn**
- Must wait 24 hours after `checkAndTriggerForcedBurn()`
- Check `forcedBurnDeadline` in vault state

**"NotLiquidatable" error when liquidating**
- Vault health is still >= 120%
- Check `getVaultState()` for current health ratio

**Oracle update fails with "CooldownNotElapsed"**
- Must wait 6 hours between aura updates per vault
- Check `lastUpdateTimestamp` in AuraOracle

## Contributing

This is an MVP implementation for demonstration and testing purposes. Contributions are welcome, but please note:

- All code should include comprehensive tests
- Follow existing code style and patterns
- Document all public functions and complex logic
- Consider gas optimization for production use
- Security-critical changes require thorough review

## License


MIT


