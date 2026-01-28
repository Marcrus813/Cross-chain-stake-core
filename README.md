# Cross-Chain Staking Protocol

## v1.0

### 📖 Project Overview

This is a **cross-chain staking protocol** based on Ethereum Beacon Chain staking, allowing users to stake assets on Layer 2, aggregate funds to Ethereum mainnet (L1) through a cross-chain bridge for validator staking, and distribute staking rewards back to L2.

#### Core Features

- ✅ **Cross-Chain Staking**: L2 users can participate in L1 Ethereum validator staking
- ✅ **Liquidity Token**: Receive dETH tokens after staking ETH, maintaining asset liquidity
- ✅ **Triple Yield**:
  - L1 Yield: Ethereum validator rewards (ETH)
  - L2 Yield: Platform native tokens (DappLink Token)
  - dETH restaking protocol rewards
- ✅ **Delegated Staking**: Support delegating assets to professional operators
- ✅ **Flexible Unstaking**: Support queued withdrawal mechanism, withdraw assets after completion
- ✅ **Secure and Reliable**: Multiple security safeguards including oracle dual-layer verification, pause mechanism, reentrancy protection, etc.

#### Tech Stack

- **Smart Contracts**: Solidity 0.8.20 / 0.8.24
- **Development Framework**: Foundry
- **Upgrade Pattern**: OpenZeppelin Upgradeable Contracts
- **Cross-Chain Communication**: Custom message bridging protocol

---

### 🏗️ System Architecture

For detailed architecture diagrams, see [docs/flows/architecture.md](./docs/flows/architecture.md)

#### L1 Layer Contract Architecture

| Contract Name | Function Description | Key Functions |
|---------------|---------------------|---------------|
| **StakingManager** | Staking management, ETH aggregation and dETH minting | `stake()`, `unstakeRequest()`, `allocateETH()` |
| **DETH** | Staking certificate token (ERC20) | `mint()`, `burn()`, `transfer()` (with bridging) |
| **UnstakeRequestsManager** | Unstaking request management and claiming | `create()`, `claim()`, `allocateETH()` |
| **OracleManager** | Oracle records validator status | `receiveRecord()`, `validateUpdate()`, `sanityCheckUpdate()` |
| **ReturnsAggregator** | Returns aggregation and fee collection | `processReturns()` |
| **ReturnsReceiver** | Receives validator withdrawals | `transfer()` |
| **L1PoolManager** | L1 fund pool management | `DepositAndStakingETH()`, `BridgeFinalizeETHForStaking()` |
| **L1Pauser** | L1 pause control | `pauseAll()`, `unpauseAll()` |
| **L1Locator** | Service locator | `coreComponents()` |

#### L2 Layer Contract Architecture

| Contract Name | Function Description | Key Functions |
|---------------|---------------------|---------------|
| **StrategyManager** | Strategy management and share allocation | `depositETHIntoStrategy()`, `removeShares()`, `withdrawSharesAsWeth()` |
| **DelegationManager** | Delegation management and operator shares | `delegateTo()`, `undelegate()`, `queueWithdrawals()`, `completeQueuedWithdrawal()` |
| **Strategy** | Specific staking strategy implementation | `deposit()`, `withdraw()`, `shares()`, `totalShares()` |
| **L1RewardManager** | L1 reward management (deployed on L2) | `depositETHRewardTo()`, `claimL1Reward()` |
| **L2RewardManager** | L2 token reward management | `calculateFee()`, `stakerClaimReward()`, `operatorClaimReward()` |
| **L2PoolManager** | L2 fund pool management | `WithdrawETHtoL1()`, `WithdrawWETHToL1()` |
| **L2Pauser** | L2 pause control | `pauseAll()`, `unpauseAll()` |
| **L2Locator** | Service locator | `coreComponents()` |

#### Bridge Layer Architecture

| Contract Name | Function Description | Key Functions |
|---------------|---------------------|---------------|
| **TokenBridgeBase** | Cross-chain bridge base contract | `BridgeInitiateETH()`, `BridgeFinalizeETH()`, `BridgeInitiateStakingMessage()` |
| **MessageManager** | Cross-chain message management | `sendMessage()`, `claimMessage()` |

---

### 🔑 Core Concepts

#### 1. dETH (Derivative ETH)
- **Definition**: Certificate token received after users stake ETH
- **Features**: ERC20 standard, transferable, cross-chain capable
- **Exchange Rate**: Dynamic rate calculated based on protocol total controlled ETH and dETH total supply
  ```
  dETH Exchange Rate = Protocol Total Controlled ETH / dETH Total Supply
  ```
- **Use Cases**:
  - As staking certificate
  - Can be transferred to others (automatically triggers L2 share transfer)
  - Burned during unstaking to redeem ETH

#### 2. Shares
- **Definition**: User's share in a specific strategy on L2
- **Calculation**: Calculated by each Strategy contract based on deposit amount and current exchange rate
- **Management**:
  - `stakerStrategyShares[staker][strategy]`: Recorded in StrategyManager
  - `operatorShares[operator][strategy]`: Recorded in DelegationManager for operator shares

#### 3. Strategies
- **Definition**: Staking strategy contracts on L2, managing different types of assets
- **Types**:
  - ETH Strategy: Manages native ETH
  - WETH Strategy: Manages Wrapped ETH
  - ERC20 Strategy: Manages other ERC20 tokens

#### 4. Operators
- **Definition**: Professional validator operators who accept user delegation
- **Responsibilities**:
  - Run validator nodes
  - Maintain node stability and security
  - Receive 8% of L2 token rewards

#### 5. Oracle
- **Definition**: Off-chain service that monitors Ethereum validator status and submits records
- **Security Mechanisms**:
  - **Integrity Verification** (`validateUpdate`): Check data consistency
  - **Sanity Check** (`sanityCheckUpdate`): Check if balance changes are within reasonable range
  - **Pending Mechanism**: Records failing sanity checks require admin approval
  - **Auto Pause**: Global pause when anomalies detected

---

### 🔄 Four Core Business Flows

For detailed flow diagrams, see [docs/flows/](./docs/flows/) directory.

#### Flow 1: User Staking (L1 → L2)

**Overview**: Users deposit ETH on L1, receive L2 shares through bridging, and optionally delegate to operators.

**Main Steps**:
1. User deposits ETH in L1PoolManager
2. Relayer triggers staking, calls StakingManager.stake()
3. StakingManager mints dETH to user
4. User transfers dETH, triggering cross-chain message
5. Relayer relays message to L2
6. L2 StrategyManager updates user shares
7. User deposits into strategy on L2 and optionally delegates

**Detailed Flow Diagram**: [1-staking-flow.md](./docs/flows/1-staking-flow.md)

---

#### Flow 2: Staking Reward Distribution

**Overview**: Validators generate rewards, recorded through oracle, fees collected via ReturnsAggregator, then L1 and L2 rewards distributed separately.

**L1 Reward Distribution**:
1. Validator rewards auto-withdraw to ReturnsReceiver
2. Oracle Updater submits validator status records
3. OracleManager validates and triggers ReturnsAggregator.processReturns()
4. ReturnsAggregator collects protocol fees (default 10%)
5. EL rewards bridge to L2's L1RewardManager
6. CL net rewards transfer to StakingManager's unallocatedETH
7. Users claim on L2 via L1RewardManager.claimL1Reward()

**L2 Reward Distribution**:
1. Admin deposits DappLink Token to L2RewardManager
2. Off-chain service calculates rewards and calls calculateFee()
3. Stakers receive 92%, operators receive 8%
4. Users claim via stakerClaimReward()

**Detailed Flow Diagram**: [2-rewards-flow.md](./docs/flows/2-rewards-flow.md)

---

#### Flow 3: Queued Withdrawal (L2 → L1)

**Overview**: Users initiate unstaking, create withdrawal request, wait for completion conditions.

**Main Steps**:
1. **L2 Undelegation**: User calls `DelegationManager.undelegate()` or `queueWithdrawals()`
2. **Cross-Chain Notify L1**: Call `StakingManager.unstakeRequest()`
3. **Wait for Completion**: Block waiting period + sufficient funds
4. **Query Claimable**: `UnstakeRequestsManager.requestInfo()`

**Detailed Flow Diagram**: [3-unstaking-flow.md](./docs/flows/3-unstaking-flow.md)

---

#### Flow 4: Withdrawal Completion

**Overview**: After conditions are met, Relayer triggers claim, burns dETH, bridges ETH to L2, user completes withdrawal.

**Main Steps**:
1. **Relayer Triggers L1 Claim**: `StakingManager.claimUnstakeRequest()`
2. **ETH Bridges to L2**: `BridgeInitiateETH()` → Relayer relays → `BridgeFinalizeETH()`
3. **Sync L1 Return Shares**: `StrategyManager.migrateRelatedL1StakerShares()`
4. **L2 Complete Withdrawal**: `DelegationManager.completeQueuedWithdrawal()`

**Detailed Flow Diagram**: [4-withdrawal-flow.md](./docs/flows/4-withdrawal-flow.md)

---

### 👥 Roles and Permissions

| Role | Description | Main Responsibilities |
|------|-------------|----------------------|
| **User** | Regular staking user | Stake, unstake, claim rewards, delegate |
| **Relayer** | Cross-chain message relayer | Listen events, relay messages, trigger cross-chain operations |
| **Oracle Updater** | Oracle updater | Submit validator status records |
| **Admin** | System administrator | Set contract parameters, grant roles, emergency management |
| **Operator** | Validator operator | Run validator nodes, accept delegations |

#### Permission Matrix

| Function | User | Relayer | Oracle | Admin |
|----------|:----:|:-------:|:------:|:-----:|
| Stake ETH | ✅ | ❌ | ❌ | ❌ |
| Bridge Staking | ❌ | ✅ | ❌ | ❌ |
| Submit Oracle Record | ❌ | ❌ | ✅ | ❌ |
| Allocate ETH | ❌ | ❌ | ❌ | ✅ |
| Start Validator | ❌ | ❌ | ❌ | ✅ |
| Initiate Unstake | ✅ | ❌ | ❌ | ❌ |
| Claim Unstake | ❌ | ✅ | ❌ | ❌ |
| Complete Withdrawal | ✅ | ❌ | ❌ | ❌ |
| Pause Contract | ❌ | ❌ | ❌ | ✅ |

---

### ⚙️ Key Parameter Configuration

#### Amount Limit Parameters

| Parameter Name | Default | Description |
|----------------|---------|-------------|
| `minimumDepositAmount` | 32 ETH | StakingManager minimum staking amount |
| `minimumUnstakeBound` | 0.01 ETH | Minimum unstaking amount |
| `maximumDETHSupply` | 1024 ETH | dETH maximum supply |

#### Time Limit Parameters

| Parameter Name | Default | Description |
|----------------|---------|-------------|
| `numberOfBlocksToFinalize` | Configurable | Blocks required for unstake request completion |
| `finalizationBlockNumberDelta` | 64 blocks | Oracle record finalize waiting period |

#### Fee Rate Parameters

| Parameter Name | Default | Description |
|----------------|---------|-------------|
| `feesBasisPoints` | 1000 (10%) | Protocol fee percentage |
| `stakerPercent` | 92% | Staker reward percentage |
| `operatorPercent` | 8% | Operator reward percentage |

---

### 🛡️ Security Mechanisms

#### 1. Pause Mechanism
- L1: Can pause staking, unstaking, validator startup, oracle submission
- L2: Can pause strategy deposits, delegation, undelegation, withdrawals
- Trigger: Manual by admin or automatic when oracle detects anomalies

#### 2. Oracle Dual-Layer Verification
- **Integrity Verification**: Check data consistency, revert on failure
- **Sanity Check**: Check if balance changes are reasonable, mark pending on failure
- **Pending Mechanism**: Anomalous records require admin approval
- **Auto Pause**: Automatically trigger global pause when anomalies detected

#### 3. Finalize Check
Prevent withdrawal rollbacks, ensure oracle record blocks are finalized

#### 4. Reentrancy Protection
Uses OpenZeppelin `ReentrancyGuard`, critical functions marked `nonReentrant`

#### 5. Access Control
Role-based access control (RBAC) using OpenZeppelin `AccessControlEnumerable`

---

### 💻 Development Guide

#### Environment Requirements

- **Solidity**: ^0.8.20 / ^0.8.24
- **Foundry**: Latest version
  ```bash
  curl -L https://foundry.paradigm.xyz | bash
  foundryup
  ```

#### Install Dependencies

```bash
# Clone repository
git clone <repository-url>
cd Crosschain-contract

# Install Foundry dependencies
forge install
```

#### Compile Contracts

```bash
# Compile all contracts
forge build

# Show detailed information
forge build --force
```

#### Run Tests

```bash
# Run all tests
forge test

# Show verbose output
forge test -vvvv

# Generate test coverage report
forge coverage
```

#### Format Code

```bash
# Format all Solidity files
forge fmt

# Check format (no modifications)
forge fmt --check
```

---

### ❓ FAQ (Frequently Asked Questions)

#### Q1: How is the dETH exchange rate calculated?

**A**: The dETH exchange rate dynamically adjusts using the formula:
```
dETH Exchange Rate = Protocol Total Controlled ETH / dETH Total Supply
```
When validators generate rewards, protocol total controlled ETH increases, and dETH exchange rate rises.

#### Q2: Why is L1RewardManager deployed on L2?

**A**: Although `L1RewardManager` is deployed on L2, it manages ETH staking rewards bridged from L1, hence the name L1RewardManager.

#### Q3: How long does unstaking take?

**A**: Unstaking time depends on:
1. Block waiting period: `numberOfBlocksToFinalize` blocks
2. Fund availability: Whether UnstakeRequestsManager has sufficient ETH

#### Q4: Why is a Relayer needed?

**A**: The Relayer is responsible for:
1. Listening to L1 and L2 cross-chain events
2. Relaying messages to target chain
3. Triggering operations requiring permissions
4. Synchronizing L1 and L2 state

#### Q5: How to become an operator?

**A**: Call `DelegationManager.registerAsOperator(operatorDetails, metadataURI)` to register as an operator.

#### Q6: How are protocol fees collected?

**A**: Protocol collects fees from L1 validator rewards:
- Default fee rate: 10%
- Only collected from rewards, principal not charged

---
