# PivotBot: Technical Overview

### Non-Custodial Leveraged Yield Agent for Base DeFi — with Autonomous Health Factor Guardian

**Version 2.0 — April 2026**  
**Author:** Abolaji M. Adedeji · Syncedge Solutions  
**Live App:** syncedgesolutions.xyz/pivot  
**Contact:** abolaji@syncedgesolutions.xyz

---

## Overview

PivotBot is a non-custodial, onchain yield automation protocol deployed on Base. It enables users to open amplified leveraged lending positions across Moonwell Base markets in a single atomic transaction using Balancer V2 zero-fee flashloans, then earn supply-side yield on the full leveraged collateral — without paying funding rates.

Version 2.0 extends the core protocol with an **intent-driven AI agent layer** powered by Coinbase CDP AgentKit, and introduces **AgentVault** — a per-user, owner-controlled intermediary contract that bounds agent authority to a hard-capped working capital allocation. Users express a yield intent once — choosing a sentiment bias (delta neutral, delta long, or delta short), a target APY, and a risk tolerance — and the agent scans live on-chain data, selects the optimal strategy pair, executes atomically, and then monitors position health continuously, auto-deleveraging before liquidation if the health factor crosses a user-defined threshold. At no point does the agent hold unbounded authority over user funds.

---

## Deployed Contracts (Base Mainnet)

| Contract     | Address                                      | Role                                  |
| ------------ | -------------------------------------------- | ------------------------------------- |
| **PivotBot** | `0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3` | Core leverage/deleverage engine       |
| **Factory**  | `0xc4E537890e86fDD44aF936218f80d7326820d97d` | Deterministic per-user bot deployment |
| **Manager**  | `0x144F04807a6af905E3112Fe3Da9302D308c6DF26` | Access control, fee routing, upgrades |

**Recent transaction:** `0xdc42587932cb065c44843ad8226fb65db4d7f6e0b6b5cd2f9f9709b67a67a476`

All contracts are live on Base mainnet and have been operational since Q1 2026. The protocol has demonstrated a verified net APY of **+44.97%** on the cbETH/wstETH delta-neutral pair (Health Factor: 1.33, supply APY 11.69%, borrow APY 1.02%, 4× leverage as of April 9, 2026).

---

## Architecture: Five-Contract System

PivotBot v2.0 uses five contracts working in concert:

1. **PivotBotFactory** — Deploys a unique PivotBot instance per user (deterministic via CREATE2)
2. **PivotBotFactoryManager** — Protocol-wide configuration, fee routing, and access control
3. **PivotBot** — Core leverage/deleverage engine; interacts with Moonwell, Aerodrome, Balancer
4. **AgentVaultFactory** — Deploys AgentVault instances; coordinates atomic vault + pass deployment
5. **AgentVault** — Per-user authority boundary; gates agent execution with hard constraints
6. **PivotProPass** — ERC-721 soulbound subscription NFT; time-gates agent access

```
User Wallet
    │
    ├── owns ──► PivotBot (deployed via PivotBotFactory)
    │                │
    │                └── MANAGER_ROLE granted to ──► AgentVault (deployed via AgentVaultFactory)
    │                                                     │
    │                                                     ├── executor ──► CDP Agent Hot Wallet
    │                                                     │
    │                                                     └── pass validation ──► PivotProPass
    │
    └── position proceeds always return directly to Owner Wallet
```

---

### 1. PivotBot.sol — Core Engine

The core contract handles the complete leverage and deleverage lifecycle. It integrates directly with Balancer V2's `IVault` interface to receive flashloans and with Moonwell's `mToken` (CErc20 / CERC20) interface for supply and borrow operations.

**Leverage flow (atomic, single transaction):**

```
User → PivotBot.openPosition(supplyAsset, borrowAsset, leverage, swapPath)
  └─ 1. Request flashloan from Balancer Vault (zero fee)
       Amount = (userDeposit × leverage) − userDeposit
  └─ 2. Supply: userDeposit + flashloanAmount → Moonwell mToken (collateral)
       Yield begins accruing immediately on full leveraged position
  └─ 3. Borrow: borrowAsset from Moonwell against new collateral
  └─ 4. Swap: Aerodrome V2 router swaps borrowedAsset → flashloanDenomination
  └─ 5. Repay: flashloan repaid to Balancer in same tx
       ⚠ If any step fails → entire tx reverts. No partial state.
```

**Avg Net APY Formula:**
```
Avg Net APY =
  (Supply APY × Supply Value in WETH
   + WELL APY × WELL Assets Value in WETH
   − Borrow APY × Borrow Value in WETH)
  / Total Supply Value in WETH
```

**Deleverage flow (atomic close):**
```
User → PivotBot.closePosition(borrowAsset, collateralAsset)
  └─ 1. Request flashloan from Balancer (amount = outstanding debt)
  └─ 2. Repay Moonwell borrow in full → collateral unlocked
  └─ 3. Redeem full collateral from Moonwell mToken
  └─ 4. Aerodrome swap: redeemed collateral → flashloan denomination
  └─ 5. Repay Balancer. Excess swept to user wallet.
       ⚠ Reverts atomically on failure. No liquidation window.
```

**Why no-loop matters:** Traditional DeFi leverage uses iterative "looping" — repeatedly supply, borrow, swap, re-supply across multiple transactions. Each loop iteration costs gas, creates partial state between transactions, and exposes the user to adverse price movement during execution. PivotBot's flashloan approach achieves full target leverage in one transaction, making it more gas-efficient and eliminating the liquidation window that exists between loop iterations.

**Key interfaces used:**
- `IVault` (Balancer V2) — flashloan provider; `receiveFlashLoan` callback
- `CErc20Interface` (Moonwell) — `mint()`, `borrow()`, `repayBorrow()`, `redeem()`
- `IRouter` (Aerodrome V2) — `swapExactTokensForTokens()` with dynamic routing

**Security measures in PivotBot.sol:**
- Reentrancy guard on all external entry points
- Balancer callback validation: only Balancer Vault can call `receiveFlashLoan`
- Token whitelist: only Moonwell Base market tokens accepted (CBETH, WSTETH, RETH, WeETH, CBBTC, LBTC, USDC, DAI, WETH, WELL, AERO)
- Slippage tolerance enforced at swap step; tx reverts if exceeded
- No upgradeability on the core engine (immutable after deployment)

**Access Control:**
- **Role-based access:** `DEFAULT_ADMIN_ROLE`, `MANAGER_ROLE` (granted to AgentVault), `PAUSER_ROLE`
- **`MANAGER_ROLE` scope:** In v2.0, this role is granted to the user's AgentVault instance, not directly to the CDP agent hot wallet. AgentVault then enforces its own hard constraints before forwarding any call to PivotBot.
- **`DEFAULT_ADMIN_ROLE` scope:** Reserved for the bot owner — executes transactions, assigns/revokes roles, pauses/unpauses in emergencies.
- **`PAUSER_ROLE` scope:** Can only pause and unpause the bot.

---

### 2. PivotBotFactory.sol — Deterministic Deployment

The Factory uses `CREATE2` to deploy a unique PivotBot instance for each user wallet. Each deployed bot is owned exclusively by the user's address.

**Why per-user bots:** Collateral and borrow positions on Moonwell are held by the user's own bot contract, not a shared pool. There is no commingling of user funds. The user's wallet is always the ultimate owner. This is the non-custodial guarantee.

```solidity
// Deterministic address per user
function deploy(address user) external returns (address bot) {
    bytes32 salt = keccak256(abi.encodePacked(user));
    bot = address(new PivotBot{salt: salt}(user, manager));
    userBots[user] = bot;
    emit BotDeployed(user, bot);
}
```

The `CREATE2` salt is the user's address, so the bot address is predictable before deployment. The frontend shows users their future bot address immediately, before any gas is spent.

---

### 3. PivotBotFactoryManager.sol — Protocol Configuration & Fee Routing

Protocol-internal contract for parameter management.

Handles:
- **Role-based access:** `DEFAULT_ADMIN_ROLE`, `MANAGER_ROLE`, `PAUSER_ROLE`
- **Protocol fee collection:** 0.05% on flashloans + 0.05% on swaps
- **Token/market whitelist:** Only approved Moonwell tokens accepted

---

### 4. AgentVaultFactory.sol — Vault & Pass Co-deployment (New in v2.0)

AgentVaultFactory is a factory contract that coordinates atomic deployment of AgentVault instances and minting of PivotProPass NFTs. It verifies that the user has already deployed a PivotBot before allowing vault creation.

**Deployment flow:**
```
User → AgentVaultFactory.deployMyAgentVault{value: X}(executor, tier)
  ├─ Verify: User has deployed PivotBot via PivotBotFactory
  ├─ Verify: User has not already deployed AgentVault
  ├─ Mint: Call PivotProPass.mintFor{value: X}(user, tier)
  │   └─ Consumes exact requiredEth; forwards to treasury; no refund
  ├─ Deploy: new AgentVault(pivotBot, executor, PIVOT_PRO_PASS) with deterministic salt
  ├─ Record: vault→owner and owner→vault mappings
  └─ Emit: VaultDeployed event
```

**Key detail:** AgentVaultFactory uses a simple counter for salt: `salt = bytes32(++vaultsDeployed)`. This makes each vault deployment deterministic but not indexed by user address (unlike PivotBotFactory).

---

### 5. AgentVault.sol — Agent Authority Boundary (New in v2.0)

AgentVault is a small Solidity contract that sits between the CDP agent hot wallet and the user's PivotBot instance. Instead of granting `MANAGER_ROLE` directly to the agent's hot wallet — which would give the agent unbounded authority over the bot — the owner grants `MANAGER_ROLE` to their AgentVault. The CDP agent hot wallet is registered inside AgentVault as the executor: it can trigger calls through AgentVault, but it holds no working capital of its own, and every call it makes is validated against four hard constraints before being forwarded.

**Setup flow:**
```
1. Owner deploys AgentVault(pivotBotAddress, ownerWallet) via factory
2. Owner calls PivotBot.grantRole(MANAGER_ROLE, agentVaultAddress)
3. Owner funds AgentVault with working capital (WETH/USDC) up to their chosen cap
4. AgentVault registers the CDP agent hot wallet as executor
5. Agent operates through AgentVault from this point forward
```

**Four hard constraints enforced in Solidity:**

**① Per-Transaction Spending Cap**  
The owner sets a maximum amount for each token (WETH, USDC, or any other) that can be spent in a single agent execution. AgentVault tracks the balance before and after each execution and reverts if the amount spent exceeds the configured cap for that token. This bounds the maximum impact of any single agent action, preventing runaway liquidations or unintended large positions.

```solidity
mapping(address => uint256) internal spendingCapPerTx;   // token → max spend per tx

// In execute():
if (spendingToken != address(0)) {
    balanceBefore = IERC20(spendingToken).balanceOf(address(this));
    IERC20(spendingToken).approve(PIVOT_BOT, type(uint256).max);
}
(bool success, bytes memory returnData) = PIVOT_BOT.call(data);
if (!success) revert ExecutionFailed(returnData);
if (spendingToken != address(0)) {
    IERC20(spendingToken).approve(PIVOT_BOT, 0);
    balanceAfter = IERC20(spendingToken).balanceOf(address(this));
    if (balanceBefore - balanceAfter > cap) {
        revert CapExceeded(spendingToken, balanceBefore - balanceAfter, cap);
    }
}
```

**② Function Selector Whitelist**  
AgentVault inspects the 4-byte calldata selector before forwarding any call to PivotBot. Only 9 selectors are whitelisted: `supplyAsset`, `borrowAsset`, `repayBorrow`, `repayBorrowBehalf`, `redeemAssetFromMw`, `repayOrSupplyAssetWithFlashloan`, `swapOnAerodromeV2`, `claimRewards`, and `accrueInterest`. Any other function call — including role management, fee changes, or token approvals — is rejected at the Solidity level, not the application layer.

```solidity
mapping(bytes4 => bool) internal whitelistedSelectors;

// In execute():
if (data.length < 4) revert CalldataTooShort();
bytes4 selector = bytes4(data);
if (!whitelistedSelectors[selector]) revert SelectorNotWhitelisted(selector);
```

The selectors are initialized once via `initDefaultSelectors()`, which can only be called by the owner:

```solidity
function initDefaultSelectors() external onlyRole(DEFAULT_ADMIN_ROLE) {
    if (_selectorsInitialized) revert SelectorsAlreadyInitialized();
    _selectorsInitialized = true;
    // 9 whitelisted MANAGER_ROLE functions
    whitelistedSelectors[bytes4(keccak256('supplyAsset(address,uint256)'))] = true;
    whitelistedSelectors[bytes4(keccak256('borrowAsset(address,uint256)'))] = true;
    // ... and 7 more
}
```

**③ Cooldown Between Executions**  
The owner sets a minimum gap in seconds between consecutive executions. Even if the agent hot wallet key is compromised and an attacker attempts to loop positions at speed, the cooldown limits execution frequency to one call per interval. The cooldown applies per AgentVault instance and is adjustable by the owner at any time.

```solidity
uint256 public cooldown;
uint256 public lastExecutionTime;

// In execute():
if (block.timestamp < lastExecutionTime + cooldown) {
    revert CooldownActive(lastExecutionTime + cooldown);
}
lastExecutionTime = block.timestamp;
```

**④ Pause Flag & Pass Validity Check**  
The owner can pause AgentVault at any time with a single transaction. While paused, all agent execution is blocked. Additionally, every call to `execute()` performs a **pass validity check first** — before any other constraint:

```solidity
function execute(bytes calldata data, address spendingToken) 
    external whenNotPaused nonReentrant onlyRole(EXECUTOR_ROLE) returns (bytes memory) 
{
    // ⭐ First check: Pass validity (before selector, cooldown, cap checks)
    if (!IPivotProPass(PIVOT_PRO_PASS).isActive(AGENT_VAULT_FACTORY.getVaultOwner(address(this)))) revert PassRequired();
    
    // Then proceed with remaining constraints...
}
```

If the owner's PivotProPass has expired, `execute()` reverts immediately with `PassRequired()`. The owner must renew their subscription before agent execution can proceed.

Critically, pausing AgentVault does not affect PivotBot's `MANAGER_ROLE` assignment — the owner does not need to revoke and re-grant roles to pause and resume agent operation. Unpausing re-enables execution immediately (assuming the pass is still valid).

**Owner control functions:**

| Function                                       | Description                                                                       |
| ---------------------------------------------- | --------------------------------------------------------------------------------- |
| `updateOwnerWallet(address _newWallet)`        | Rotates the owner/subscription holder wallet (principal identity for pass checks) |
| `setSpendingCapPerTx(address token, uint256)`  | Sets the per-transaction spend limit for a specific token (WETH, USDC, etc.)      |
| `setCooldown(uint256 seconds)`                 | Updates the minimum gap (in seconds) between consecutive executions               |
| `initDefaultSelectors()`                       | One-time initialization of the 9 whitelisted function selectors                   |
| `setWhitelistedSelector(bytes4, bool)`         | Adds or removes specific function selectors from the whitelist                    |
| `pause()` / `unpause()`                        | Blocks or re-enables all agent execution (pass remains valid; just blocks calls)  |
| `withdrawToken(address token, uint256 amount)` | Withdraws a specific ERC-20 token balance to owner wallet                         |
| `withdrawEther(uint256 amount)`                | Withdraws ETH balance to owner wallet                                             |
| `drain()`                                      | Sweeps all WETH, USDC, and ETH dust to owner wallet in a single atomic call       |

**Drain guarantee:** The `drain()` function transfers 100% of WETH, USDC, and any ETH held in AgentVault directly to the registered owner wallet in a single atomic call. There is no timelock, no agent approval required, and no dependency on PivotBot state:

```solidity
function drain() external nonReentrant onlyRole(DEFAULT_ADMIN_ROLE) {
    uint256 wethBalance = IERC20(WETH).balanceOf(address(this));
    uint256 usdcBalance = IERC20(USDC).balanceOf(address(this));
    uint256 ethBalance = address(this).balance;

    if (wethBalance > 0) {
        IERC20(WETH).transfer(_ownerWallet, wethBalance);
    }
    if (usdcBalance > 0) {
        IERC20(USDC).transfer(_ownerWallet, usdcBalance);
    }
    if (ethBalance > 0) {
        (bool success, ) = payable(_ownerWallet).call{value: ethBalance}('');
        if (!success) revert WithdrawalFailed();
    }

    emit Drained(_ownerWallet, wethBalance, usdcBalance, ethBalance);
}
```

The owner can drain at any time regardless of whether any positions are open or if the pass is expired.

**Non-custodial guarantee preserved:** When PivotBot closes a position or redeems collateral, proceeds are sent directly to the registered owner wallet — exactly as in v2.0. AgentVault is never in the withdrawal path. The only funds inside AgentVault at any time are the working capital the owner has explicitly deposited as agent margin.

**Maximum loss bound:** The maximum loss from a compromised agent hot wallet key is bounded by a small amount of ETH held in the agent's hot wallet for transaction fees. The agent can only *execute* positions through AgentVault—it cannot withdraw tokens from AgentVault (no `withdraw()` function accessible to executor role). Even if a position moves into loss:
- The CDP Guardian monitors health factor continuously
- If health factor drops below threshold (default: 1.25), the guardian auto-deleverages via AgentVault.execute()
- AgentVault enforces cooldown, selector whitelist, and spending cap on every call
- Position collateral held in Moonwell via PivotBot is never accessible to AgentVault or the agent hot wallet

---

## Token Universe (Moonwell Base Markets Only)

PivotBot is strictly scoped to the 12 tokens listed on Moonwell's Base deployment:

| Token   | Type       | Role in Strategies             |
| ------- | ---------- | ------------------------------ |
| CBETH   | LST (ETH)  | Supply (Neutral/Long)          |
| WSTETH  | LST (ETH)  | Supply/Borrow (Neutral)        |
| RETH    | LST (ETH)  | Supply/Borrow (Neutral)        |
| WeETH   | LST (ETH)  | Supply (Neutral)               |
| CBBTC   | BTC        | Supply/Borrow (Neutral/Long)   |
| LBTC    | BTC        | Supply/Borrow (Neutral)        |
| USDC    | Stablecoin | Supply (Short) / Borrow (Long) |
| DAI     | Stablecoin | Supply (Short) / Borrow (Long) |
| WETH    | ETH        | Supply/Borrow                  |
| WELL    | Protocol   | Borrow (Short)                 |
| AERO    | Protocol   | Supply/Borrow                  |
| VIRTUAL | Protocol   | Supply/Borrow                  |

---

## V2.0: The Intent Engine & AI Agent Layer

### Intent Panel Architecture

Users interact through an `IntentPanel` component with three inputs:

1. **Sentiment selector** — three modes:
   - **Delta Neutral:** Supply and borrow correlated assets (LST↔LST, BTC↔BTC). Minimal directional price risk; profit from the APY spread between supply and borrow rates.
   - **Delta Long:** Supply ETH/BTC assets, borrow stablecoins. Upside exposure if ETH/BTC appreciate.
   - **Delta Short:** Supply stablecoins, borrow appreciating assets. Profits if prices fall.

2. **Target APY slider** — 5% to 100% range

3. **Risk tolerance** — Conservative / Moderate / Aggressive (maps to leverage multiplier ceiling and drawdown budget)

### Agent Reasoning Chain (5 Steps)

After the user presses "Analyze," the agent executes five reasoning steps sequentially before presenting a recommendation:

**Step 1 — Parse Intent**  
Validates the user's chosen sentiment, target APY, and risk tolerance. Confirms which of the 23 strategy pairs are eligible for evaluation.

**Step 2 — Filter Candidate Pairs**  
Filters the full strategy matrix to only the pairs matching the selected sentiment:
- Delta Neutral: 8 pairs (LST/LST and BTC/BTC cross-pairs)
- Delta Long: 8 pairs (ETH/BTC supply → stablecoin borrow)
- Delta Short: 7 pairs (stablecoin supply → ETH/BTC/protocol borrow)

**Step 3 — Fetch Live On-Chain Rates**  
A single Multicall3 batch fetches `supplyRatePerTimestamp` and `borrowRatePerTimestamp` from every unique Moonwell mToken in the filtered candidate set, then converts to APY:

```javascript
// supplyRatePerTimestamp is in wei per second, scaled 1e18
const SECONDS_PER_YEAR = 31536000;
const supplyAPY = (supplyRatePerTimestamp / 1e18) * SECONDS_PER_YEAR * 100;
```

**Step 4 — Aerodrome Liquidity Check**  
For each candidate pair, reads pool reserves from the Aerodrome V2 PoolFactory (`getPool(tokenA, tokenB, false)`) to estimate execution slippage for the target position size:

```javascript
// Constant product slippage estimate
const priceImpact = tradeSize / (reserveIn + tradeSize);
```

Pairs with shallow liquidity (price impact above the threshold for the chosen risk level) are penalised or excluded.

**Step 5 — Rank & Score**  
Computes approximate net APY for every remaining candidate and sorts descending:

```javascript
const netAPY =
  (supplyAPY * leverage)
  - (borrowAPY * (leverage - 1))
  - protocolFee
  - estimatedSlippage;
```

Applies drawdown proxy by sentiment:
- Delta Neutral → very tight cap (correlated assets move together)
- Delta Long/Short → cap determined by user's risk tolerance setting

Returns top 3 ranked strategies with full APY breakdown.

### Full Strategy Matrix (25 Pairs)

**Delta Neutral (8 pairs):**  
cbETH/wstETH · cbETH/rETH · cbETH/WeETH · wstETH/rETH · wstETH/WeETH · rETH/WeETH · cbBTC/LBTC · CBBTC/LBTC

**Delta Long (8 pairs):**  
cbETH/USDC · wstETH/USDC · rETH/USDC · WeETH/USDC · cbETH/DAI · wstETH/DAI · CBBTC/USDC · CBBTC/DAI · VIRTUAL/USDC · All Delta Neutral Pairs

**Delta Short (7 pairs):**  
USDC/wstETH · USDC/cbETH · USDC/rETH · USDC/CBBTC · USDC/WELL · DAI/wstETH · DAI/cbETH

---

## V2.0: CDP AgentKit Guardian

### The Liquidation Problem

When a leveraged lending position's health factor drops below 1.0 on Moonwell, the position is subject to liquidation — a third party repays the debt at a penalty and claims the collateral at a discount. The user loses the liquidation penalty (typically 5–10%) on top of any market losses.

Approximately 30% of DeFi liquidations occur when the health factor is in the 1.0–1.2 range — a window where an automated 10-minute intervention would prevent the liquidation entirely.

PivotBot v2.0 addresses this with an always-on CDP AgentKit guardian operating through AgentVault.

### Guardian Architecture

The guardian is a CDP AgentKit-powered agent running continuously. Its hot wallet is registered as the executor inside the user's AgentVault instance. It never holds working capital directly.

**Monitoring loop:**
```
Every N blocks:
  1. Read healthFactor from Moonwell for user's bot address and/or calculate from open positions
     (comptroller.getAccountLiquidity(botAddress))
  2. If healthFactor < userThreshold (default: 1.25):
     → Trigger deleverage decision pipeline
  3. Deleverage pipeline:
     a. Calculate optimal partial vs full close
     b. Construct repayBorrow(), supplyAsset() or repayOrSupplyAssetWithFlashloan() calldata
     c. Call AgentVault.execute(calldata) via CDP AgentKit wallet
        → AgentVault validates selector, checks cooldown, checks pause
        → AgentVault forwards to PivotBot if all constraints pass
     d. Monitor tx confirmation
     e. Re-check healthFactor post-execution
     f. Notify user (webhook / frontend update)
```

**Non-custodial guarantee maintained:**  
The CDP AgentKit hot wallet can only call `AgentVault.execute()`. Position proceeds from closed positions always go directly to the owner wallet — not to Agent hot wallet. The agent never touches position collateral.

**Health factor formula (Moonwell / Compound V2 model):**
```
healthFactor = Σ(collateral_i × collateralFactor_i) / totalBorrowedWETH
```
Where `collateralFactor_i` is the per-asset liquidation threshold set by Moonwell governance (e.g. cbETH: 0.78, USDC: 0.82).

---

## V2.0: PivotProPass — ERC-721 Subscription NFT

Access to agent execution is gated behind a soulbound (non-transferable) time-limited ERC-721 NFT called `PivotProPass`. The pass is minted atomically alongside AgentVault deployment, funded with ETH sent by the user. Pricing is USD-denominated; real-time ETH conversion is calculated via Chainlink oracle (ETH/USD on Base). Revenue from all subscriptions and renewals flows immediately to the protocol treasury (`protocolFeeRecipient` on PivotBotFactoryManager) — PivotProPass never custodies ETH.

### Design Rationale: Why Co-deployed with AgentVault?

- **Atomic onboarding:** User deploys vault + subscribes to agent access in a single transaction
- **Simplified UX:** No separate "buy a pass then deploy vault" flow
- **Aligned incentives:** Vault owner has an active subscription at deployment time
- **Revenue efficiency:** Treasury receives payment immediately; no custody risk

### Deployment Architecture

```
User Wallet
    │
    └─ calls AgentVaultFactory.deployMyAgentVault{value: X}(executor, tier)
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
           1. Query Chainlink  2. Deploy AgentVault  3. Mint Pass
                    │               │               │
           requiredEth =        new AgentVault    PivotProPass.mintFor
           (tierPriceUsd        (pivotBot,         {value: X}
            * 1e18) /            owner,             ├─ Validate owner has no prior pass
            ethUsdPrice          executor,          ├─ requiredEth = getRequiredEth(tier)
                                 PASS_ADDR)        ├─ Update state: tokenOf[owner] = tokenId
           if X != required                         ├─ Update state: expiryOf[tokenId] = now + duration
           → revert                                ├─ _mint(owner, tokenId)
           InsufficientEth                         ├─ Forward requiredEth →
                                                   │   FACTORY_MANAGER.protocolFeeRecipient()
                                                  
```

All three operations are atomic within the AgentVaultFactory — if any step fails, the entire transaction reverts with no partial state.

### Chainlink Integration

PivotProPass maintains a live link to Chainlink's ETH/USD price feed on Base (`0x71041dddad3595F9CEd3DcCFBe3D1F4b0a16Bb70`, 8 decimals). The contract is initialized with four parameters:

```solidity
constructor(address _factoryManager, address _vaultFactory, address _priceFeed, uint256 _staleThreshold)
```

On every `mintFor()` and `renew()` call, the contract:

1. **Fetches latest price:** `(, int256 answer, , uint256 updatedAt, ) = PRICE_FEED.latestRoundData()`
2. **Validates staleness:** Revert if `block.timestamp - updatedAt > staleThreshold` (default: 60 seconds)
3. **Validates answer:** Revert if `answer <= 0`
4. **Calculates ETH equivalent:** 
   ```
   requiredEth = (tierPriceUsd * 1e18) / uint256(answer)
   ```
   Example: $30 USD price, ETH/USD at 2,000 → 30_00000000 * 1e18 / 2000_00000000 = 0.015 ETH

The `staleThreshold` is configurable by protocol owner via `setStaleThreshold(uint256)`, allowing adjustment as feed update intervals change. Default is 60 seconds.

### Revenue Routing — No Custodial Risk

**Critical distinction:** PivotProPass never custodies ETH. Payments are validated, state is updated, then ETH is immediately forwarded using the CEI pattern (Checks-Effects-Interactions):

```solidity
function mintFor(address user, Tier tier) external payable noZeroAddress(user) {
    // Checks: Verify authorization and preconditions
    if (msg.sender != AGENT_VAULT_FACTORY_ADDRESS) revert Unauthorized();
    if (tokenOf[user] != 0) revert OnePassPerWallet();

    uint256 requiredEth = getRequiredEth(tier);
    if (msg.value != requiredEth) revert InsufficientEth(requiredEth, msg.value);

    // Effects: Update state first
    uint256 tokenId = ++_nextTokenId;
    uint256 expiry = block.timestamp + tierDuration[tier];
    expiryOf[tokenId] = expiry;
    tokenOf[user] = tokenId;

    // Interactions: Forward ETH to protocol treasury (read live)
    address treasury = FACTORY_MANAGER.protocolFeeRecipient();
    (bool ok, ) = payable(treasury).call{value: requiredEth}('');
    if (!ok) revert TransferFailed();

    emit PassMinted(user, tokenId, tier, expiry, requiredEth);
    _safeMint(user, tokenId);
}
```

**Key properties:**
- Only AgentVaultFactory can call `mintFor()` (authorization check)
- User must send exactly `requiredEth` (no overpayment, no refunds)
- Minting fails atomically if any step fails
- Zero ETH custodied in PivotProPass at any time
- Revenue routing automatically follows `protocolFeeRecipient` changes on PivotBotFactoryManager (read live on every mint)
- No separate withdrawal function or governance delay needed

### Subscription Tiers & Pricing

| Tier          | Duration | USD Price (placeholder) | Use Case                            |
| ------------- | -------- | ----------------------- | ----------------------------------- |
| ONE_MONTH     | 30 days  | $10.00                  | Trial; short-term strategies        |
| THREE_MONTHS  | 90 days  | $25.00                  | Standard operational pass           |
| SIX_MONTHS    | 180 days | $45.00                  | Medium-term commitments             |
| TWELVE_MONTHS | 365 days | $80.00                  | Best value; year-round agent access |

Prices are stored on-chain as 8-decimal USD amounts (matching Chainlink decimals). Protocol owner calls `setTierPriceUsd(Tier, uint256)` to adjust pricing post-launch. ETH equivalents are always computed live from Chainlink to reflect market conditions.

### Key Design Decisions

**1. Soulbound (Non-Transferable)**
- Prevents secondary market gaming (mint, use, resell within 1 hour)
- Eliminates complexity of maintaining `tokenOf` mappings during transfers
- Aligns subscription with wallet identity — each wallet holds at most one pass
- If user switches wallets, they deploy new PivotBot, deploy new AgentVault and  mint a fresh pass from the new wallet; old pass stays dormant

**2. Time Stacking on Renewal**
- Caller holds a tokenId and calls `renew(tier)` with payment
- New expiry = `max(expiryOf[tokenId], block.timestamp) + tierDuration[tier]`
- Users who renew early do not lose unused days
- Stale renewals (after expiry) stack from block.timestamp, preventing unbounded time accumulation

**3. Pass Check in AgentVault.execute()**
- **First check (before selector whitelist, cooldown, cap):** `if (!IPivotProPass(PIVOT_PRO_PASS).isActive(AGENT_VAULT_FACTORY.getVaultOwner(address(this)))) revert PassRequired();`
- Retrieves vault owner dynamically from AgentVaultFactory (single source of truth)
- Not stored as a local variable; read on each execution
- Fail-fast semantics: expired passes block execution immediately with explicit `PassRequired()` error

**4. One Pass Per Wallet Enforcement**
- `tokenOf[address]` mapping prevents multiple mints to same wallet
- Checked at mint time; soulbound property preserves invariant through lifecycle

### Payment Flow Diagram

```
SCENARIO A: AgentVault Deployment (Initial Mint)
─────────────────────────────────────────────────
User Wallet ──{ETH: X}──> AgentVaultFactory.deployMyAgentVault(executor, tier)
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
              1. Check ETH    2. Deploy AgentVault   3. Mint Pass
                    │                     │                     │
              required = 0.015        new AgentVault    PivotProPass.mintFor
            if X != required         (pivotBot, user,   {value: 0.015}
              → revert                 executor,         ├─ _mint(user, tokenId)
                                       PASS_ADDR)        ├─ Forward 0.015 → treasury
                                                         
                    
```

```
SCENARIO B: Renewal (Any Time, User Initiated)
──────────────────────────────────────────────
User Wallet ──{ETH: Y}──> PivotProPass.renew(tier)
                               │
                   ┌───────────┼───────────┐
                   │           │           │
              1. Validate  2. Stack Time  3. Forward ETH
                   │           │           │
           tokenOf[user] != 0   │     address treasury =
           Y >= required   expiryOf =   FACTORY_MANAGER
                           max(old, now) .protocolFeeRecipient()
                           + duration    Forward Y → treasury
                                         Refund excess to user
```

### Pass Status & Execution Gate

When a CDP agent executor calls `AgentVault.execute()`, the function enforces all four constraints in strict sequence:

```solidity
function execute(bytes calldata data, address spendingToken) 
    external whenNotPaused nonReentrant onlyRole(EXECUTOR_ROLE) returns (bytes memory) 
{
    // ① Pass validity check (FIRST, before any other execution logic)
    if (!IPivotProPass(PIVOT_PRO_PASS).isActive(AGENT_VAULT_FACTORY.getVaultOwner(address(this)))) 
        revert PassRequired();

    // ② Calldata and selector validation
    if (data.length < 4) revert CalldataTooShort();
    bytes4 selector = bytes4(data);
    if (!whitelistedSelectors[selector]) revert SelectorNotWhitelisted(selector);

    // ③ Cooldown enforcement
    if (block.timestamp < lastExecutionTime + cooldown) {
        revert CooldownActive(lastExecutionTime + cooldown);
    }

    // Update last execution time
    lastExecutionTime = block.timestamp;
    
    // ④ Pre-execution: Track spending token balance
    uint256 balanceBefore;
    uint256 balanceAfter;
    if (spendingToken != address(0)) {
        balanceBefore = IERC20(spendingToken).balanceOf(address(this));
        IERC20(spendingToken).approve(PIVOT_BOT, type(uint256).max);
    }
    
    // Execute the call to PivotBot
    (bool success, bytes memory returnData) = PIVOT_BOT.call(data);
    if (!success) revert ExecutionFailed(returnData);
    
    // ④ Post-execution: Verify spending cap not exceeded
    if (spendingToken != address(0)) {
        IERC20(spendingToken).approve(PIVOT_BOT, 0);
        balanceAfter = IERC20(spendingToken).balanceOf(address(this));
        uint256 cap = spendingCapPerTx[spendingToken];
        if (balanceBefore - balanceAfter > cap) {
            revert CapExceeded(spendingToken, balanceBefore - balanceAfter, cap);
        }
    }
    
    emit Executed(msg.sender, selector, returnData);
    return returnData;
}
```

**Constraint enforcement sequence:**
1. **Pass validity (first):** Query AgentVaultFactory for vault owner, check if their PivotProPass is active. Reverts with `PassRequired()` if expired or inactive.
2. **Selector validation:** The 4-byte function selector must be in the whitelist. Reverts with `SelectorNotWhitelisted()` if not.
3. **Cooldown:** Minimum gap since last execution must have elapsed. Reverts with `CooldownActive()` if not.
4. **Spending cap (post-execution):** If `spendingToken` is not `address(0)`, measure token balance before and after the PivotBot call. Reverts with `CapExceeded()` if the amount spent exceeds `spendingCapPerTx[spendingToken]`.

**Fail-fast guarantee:** If any constraint fails, the entire transaction reverts atomically. No partial state or side effects. If all constraints pass, the call proceeds and the result (success or failure from PivotBot) is returned to the executor.

### Free Tier (Frontend Only)

Non-pass-holders can access **3 free strategy analyses per calendar month**. This is a frontend-only feature (localStorage-tracked) and does **not** bypass the on-chain execution gate:
- **Analysis** (intent → reasoning → recommendation): Always free
- **Atomic execution** (openPosition / closePosition on PivotBot): Requires active PivotProPass

Free analyses cannot be used to execute positions; they are purely informational.

### Implementation Scope

| Component                    | Location                                          | Status     |
| ---------------------------- | ------------------------------------------------- | ---------- |
| PivotProPass contract        | `src/PivotProPass.sol`                            | ✅ Complete |
| IPivotProPass interface      | `src/interfaces/IPivotProPass.sol`                | ✅ Complete |
| Chainlink interface (copied) | `src/interfaces/AggregatorV3Interface.sol`        | ✅ Complete |
| AgentVaultFactory updates    | `src/AgentVaultFactory.sol`                       | ✅ Complete |
| AgentVault updates           | `src/AgentVault.sol`                              | ✅ Complete |
| Deployment script            | `script/DeployPivotProPass.s.sol`                 | ✅ Complete |
| Mock for testing             | `test/mocks/MockPivotProPass.sol`                 | ✅ Complete |
| Unit tests                   | `test/PivotProPass.t.sol`                         | ✅ Complete |
| Integration tests            | `test/AgentVaultFactoryPivotProIntegration.t.sol` | ✅ Complete |

---

## Frontend: Agent Setup UI (New in v2.0)

The `IntentPanel`'s Agent Mode section gains an **Agent Setup card** backed by the `useAgentVault` hook.

### Agent Setup Card

The card surfaces all AgentVault state and controls in one place:

| Display                     | Description                                                         |
| --------------------------- | ------------------------------------------------------------------- |
| AgentVault address          | The deployed contract address for this vault                        |
| Vault owner (from factory)  | The principal wallet; used to check PivotProPass validity on calls  |
| WETH spending cap per tx    | Current max spend per transaction for WETH                          |
| USDC spending cap per tx    | Current max spend per transaction for USDC                          |
| Cooldown setting (seconds)  | Minimum gap (in seconds) between consecutive agent executions       |
| Last execution timestamp    | Unix timestamp of most recent agent execution                       |
| Next execution available at | Countdown (seconds remaining) until next execution is permitted     |
| Pause status                | Whether execution is currently paused (pause() called)              |
| PivotProPass expiry         | Owner's pass expiry timestamp; execution blocks if expired          |
| Pass active status          | Real-time boolean: is owner's pass currently valid and non-expired? |
| Drain All button            | Sweeps WETH, USDC, and ETH dust to owner wallet atomically          |
| Update spending cap form    | Adjusts per-tx cap for WETH or USDC independently                   |
| Update cooldown form        | Adjusts cooldown interval (in seconds)                              |

### `useAgentVault` Hook

Reads all AgentVault state in a single Multicall3 batch:

```javascript
const calls = [
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'getCurrentOwnerWallet') },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'getSpendingCapPerTx', [WETH]) },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'getSpendingCapPerTx', [USDC]) },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'cooldown') },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'lastExecutionTime') },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'paused') },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'getNextExecutionTime') },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'isInitialized') },
  { target: WETH, callData: encodeFunctionData(ERC20_ABI, 'balanceOf', [agentVaultAddress]) },
  { target: USDC, callData: encodeFunctionData(ERC20_ABI, 'balanceOf', [agentVaultAddress]) },
  { target: PIVOT_PRO_PASS, callData: encodeFunctionData(PASS_ABI, 'isActive', [vaultOwner]) },
  { target: PIVOT_PRO_PASS, callData: encodeFunctionData(PASS_ABI, 'expiryOf', [PIVOT_PRO_PASS.tokenOf(vaultOwner)]) },
];
```

Exposes write functions (for vault owner only):
- `setSpendingCapPerTx(token, amount)` — Update per-tx cap for a token
- `setCooldown(seconds)` — Update cooldown interval
- `pause()` / `unpause()` — Block/unblock agent execution
- `drain()` — Sweep all WETH, USDC, ETH to owner wallet
- `withdrawToken(token, amount)` — Withdraw specific token balance
- `withdrawEther(amount)` — Withdraw ETH balance
- `setWhitelistedSelector(bytes4, bool)` — Add/remove function selectors
- `initDefaultSelectors()` — One-time initialization of 9 default selectors

The `AGENT_VAULT_ABI`, `PASS_ABI`, and `AGENT_VAULT_FACTORY_ABI` are added to the constants file alongside the existing `NFT_ABI` and `AERODROME_ABI`.

---

## Protocol Fee Structure

| Fee Type         | Rate               | Applies To                                     |
| ---------------- | ------------------ | ---------------------------------------------- |
| Protocol Fee     | 0.05%              | Flash loan amount on every leverage/deleverage |
| Swap Fee         | 0.05%              | Aerodrome swap amount on every position event  |
| NFT Subscription | ETH (per tier)     | Access to agent execution                      |
| B2B Licensing    | Fixed (negotiated) | White-label factory deployments                |

Fees flow to the protocol treasury via Manager. No token required to use the protocol — fee revenue is ETH and ERC-20 denominated.

---

## Testing & Security

**Test suite (Foundry):**
- Unit tests: individual function behaviour for all PivotBot and AgentVault entry points
- Integration tests: full fork tests against Base mainnet state (Moonwell + Aerodrome + Balancer live contract state)
- Fuzz tests: randomised input coverage on leverage parameters, token amounts, swap paths, cap values, and cooldown intervals
- AgentVault-specific tests: selector rejection, cap enforcement, cooldown enforcement, drain atomicity, executor rotation

**Security measures:**
- Reentrancy guard on all public/external mutating functions
- Checks-Effects-Interactions pattern throughout
- Balancer flashloan callback authenticated — only `IVault(balancerVault)` can trigger `receiveFlashLoan`
- No `delegatecall` used anywhere
- Token input validation against approved registry (Manager whitelist)
- Slippage tolerance enforced atomically — position reverts rather than executing at unfavourable rates
- CDP AgentKit guardian scoped to only owner-whitelisted selectors via AgentVault (execute role only)
- AgentVault `drain()` has no timelock and requires no agent involvement
- PivotProPass pass check runs **first** in AgentVault.execute() before any other constraint (fail-fast)
- AgentVault authorization: executor role cannot call `drain()`, `setSpendingCapPerTx()`, `setCooldown()`, or any config functions — only owner (DEFAULT_ADMIN_ROLE)

**Security audit:** Commissioned Q2 2026. Results will be published publicly. AgentVault and PivotProPass are in scope.

---

## Technology Stack

| Layer            | Technology                                 |
| ---------------- | ------------------------------------------ |
| Smart Contracts  | Solidity 0.8.33, Foundry                   |
| Blockchain       | Base (Ethereum L2 via OP Stack)            |
| Flashloans       | Balancer V2 (`IVault.flashLoan`)           |
| Lending          | Moonwell Base (Compound V2 fork)           |
| DEX              | Aerodrome V2 (Velodrome fork, Base)        |
| Agent Framework  | CDP AgentKit (Coinbase Developer Platform) |
| Agent Wallets    | CDP Server Wallets (non-custodial)         |
| Agent Boundary   | AgentVault.sol (per-user authority cap)    |
| Frontend         | React, Wagmi, Viem, Tailwind CSS           |
| On-chain Queries | Multicall3 (batched reads)                 |
| NFT Standard     | ERC-721 (OpenZeppelin 5.x)                 |

---

## Supported Chains & Roadmap

**Currently live:** Base Mainnet

**Planned:**
- Q3 2026: Unichain, OP Mainnet (Optimism Superchain family)
- Q4 2026: Arbitrum One
- 2027: Additional EVM chains with Moonwell/Compound V2 deployments

Cross-chain deployments use the same Factory/Manager/PivotBot/AgentVault architecture — only the external protocol addresses (Moonwell, Aerodrome, Balancer) are chain-specific. The core logic is chain-agnostic.

---

## On-Chain Verification

All contracts can be verified on Basescan:

- **PivotBot:** https://basescan.org/address/0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3
- **Factory:** https://basescan.org/address/0xc4E537890e86fDD44aF936218f80d7326820d97d
- **Manager:** https://basescan.org/address/0x144F04807a6af905E3112Fe3Da9302D308c6DF26
- **Recent tx:** https://basescan.org/tx/0xdc42587932cb065c44843ad8226fb65db4d7f6e0b6b5cd2f9f9709b67a67a476

**Live app:** https://syncedgesolutions.xyz/pivot

---

## References

1. Moonwell Base Markets — app.moonwell.fi (live rate data, April 2026)
2. Balancer V2 Vault Interface — docs.balancer.fi/reference/contracts/vault
3. Aerodrome V2 Router — aerodrome.finance/docs
4. CDP AgentKit — developer.coinbase.com/agentkit
5. Compound V2 Risk Model (health factor) — docs.compound.finance
6. Messari DeFi Risk Report 2024 (liquidation statistics)
7. DeFi Llama — defillama.com (TVL data)

---

*PivotBot is a non-custodial protocol. Users retain custody of funds at all times via the per-user bot and AgentVault architecture. This document is for informational purposes only and does not constitute financial advice.*

*© 2026 Syncedge Solutions · abolaji@syncedgesolutions.xyz*