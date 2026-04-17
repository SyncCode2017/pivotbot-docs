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

## Architecture: Four-Contract System

PivotBot v2.1 uses four contracts working in concert. Factory deploys a PivotBot instance per user. Manager holds protocol-wide configuration and fee routing. AgentVault sits between the CDP agent hot wallet and the user's PivotBot, enforcing hard constraints on what the agent can do and how much capital it can touch. PivotBot is the core execution engine that interacts with Moonwell, Aerodrome, and Balancer.

```
User Wallet
    │
    ├── owns ──► PivotBot (deployed via Factory)
    │                │
    │                └── MANAGER_ROLE granted to ──► AgentVault
    │                                                     │
    │                                                     └── executor ──► CDP Agent Hot Wallet
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
- **`MANAGER_ROLE` scope:** In v2.1, this role is granted to the user's AgentVault instance, not directly to the CDP agent hot wallet. AgentVault then enforces its own hard constraints before forwarding any call to PivotBot.
- **`DEFAULT_ADMIN_ROLE` scope:** Reserved for the bot owner — executes transactions, assigns/revokes roles, pauses/unpauses in emergencies.
- **`PAUSER_ROLE` scope:** Can only pause and unpause the bot.

---

### 2. Factory.sol — Deterministic Deployment

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

### 3. Manager.sol — Protocol Configuration & Fee Routing

Protocol-internal contract for parameter management.

Handles:
- **Role-based access:** `DEFAULT_ADMIN_ROLE`, `MANAGER_ROLE`, `PAUSER_ROLE`
- **Protocol fee collection:** 0.05% on flashloans + 0.05% on swaps
- **Token/market whitelist:** Only approved Moonwell tokens accepted

---

### 4. AgentVault.sol — Agent Authority Boundary (New in v2.1)

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

**① Balance Cap**  
The owner sets a maximum WETH amount and a maximum USDC amount that AgentVault is permitted to hold as deployable working capital. If the owner funds it beyond the cap, the excess sits in the contract but cannot be deployed by the agent until the owner explicitly raises the cap. This bounds maximum agent exposure to a single position's margin at target leverage — nothing more.

```solidity
mapping(address => uint256) public caps;   // token → max deployable amount

modifier withinCap(address token, uint256 amount) {
    require(amount <= caps[token], "Exceeds agent cap");
    _;
}
```

**② Function Whitelist**  
AgentVault inspects the calldata selector before forwarding any call to PivotBot. Only two selectors are whitelisted: `openPosition` (supply-with-leverage) and `closePosition` (repay-with-flashloan). Any other function call — including role management, fee changes, or token approvals — is rejected at the Solidity level, not the application layer.

```solidity
bytes4 private constant SET_SLIPPAGE_SELECTOR  = bytes4(keccak256("setSlippageToleranceNumX10k(uint256)"));
bytes4 private constant SET_DURATION_SELECTOR = bytes4(keccak256("setDuration(uint256)"));

function execute(bytes calldata data) external onlyExecutor notPaused {
    bytes4 selector = bytes4(data[:4]);
    require(
        selector != SET_SLIPPAGE_SELECTOR && selector != SET_DURATION_SELECTOR,
        "Selector not whitelisted"
    );
    (bool ok,) = pivotBot.call(data);
    require(ok, "PivotBot call failed");
    lastExecutionTime = block.timestamp;
}
```

**③ Cooldown**  
The owner sets a minimum gap in seconds between consecutive executions. Even if the agent hot wallet key is compromised and an attacker attempts to loop positions at speed, the cooldown limits execution frequency to one call per interval. The cooldown applies per AgentVault instance and is adjustable by the owner at any time.

```solidity
uint256 public cooldown;
uint256 public lastExecutionTime;

modifier respectsCooldown() {
    require(block.timestamp >= lastExecutionTime + cooldown, "Cooldown active");
    _;
}
```

**④ Pause Flag**  
The owner can pause AgentVault at any time with a single transaction. While paused, all agent execution is blocked. Critically, pausing AgentVault does not affect PivotBot's `MANAGER_ROLE` assignment — the owner does not need to revoke and re-grant roles to pause and resume agent operation. Unpausing re-enables execution immediately.

**Owner control functions:**

| Function                                | Description                                                                                    |
| --------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `setExecutor(address)`                  | Rotates the CDP agent hot wallet address without redeployment                                  |
| `setCap(address token, uint256 amount)` | Adjusts the deployable cap for WETH or USDC                                                    |
| `setCooldown(uint256 seconds)`          | Updates the minimum gap between executions                                                     |
| `pause()` / `unpause()`                 | Blocks or re-enables all agent execution                                                       |
| `drain()`                               | Sweeps all WETH, USDC, and ETH dust to owner wallet instantly — no delay, no agent involvement |

**Drain guarantee:** The `drain()` function transfers 100% of WETH, USDC, and any ETH dust held in AgentVault directly to the registered owner wallet in a single call. There is no timelock, no agent approval required, and no dependency on PivotBot state. The owner can drain at any time regardless of whether any positions are open.

**Non-custodial guarantee preserved:** When PivotBot closes a position or redeems collateral, proceeds are sent directly to the registered owner wallet — exactly as in v2.0. AgentVault is never in the withdrawal path. The only funds inside AgentVault at any time are the working capital the owner has explicitly deposited as agent margin.

**Maximum loss bound:** The maximum loss from a compromised agent hot wallet key is bounded by `caps[WETH]` and `caps[USDC]` — the amounts the owner configured. These should be sized to cover a single position's margin at maximum leverage, nothing more. Position collateral held in Moonwell via PivotBot is not accessible to AgentVault or the agent hot wallet.

---

## Token Universe (Moonwell Base Markets Only)

PivotBot is strictly scoped to the 11 tokens listed on Moonwell's Base deployment:

| Token  | Type       | Role in Strategies             |
| ------ | ---------- | ------------------------------ |
| CBETH  | LST (ETH)  | Supply (Neutral/Long)          |
| WSTETH | LST (ETH)  | Supply/Borrow (Neutral)        |
| RETH   | LST (ETH)  | Supply/Borrow (Neutral)        |
| WeETH  | LST (ETH)  | Supply (Neutral)               |
| CBBTC  | BTC        | Supply/Borrow (Neutral/Long)   |
| LBTC   | BTC        | Supply/Borrow (Neutral)        |
| USDC   | Stablecoin | Supply (Short) / Borrow (Long) |
| DAI    | Stablecoin | Supply (Short) / Borrow (Long) |
| WETH   | ETH        | Supply/Borrow                  |
| WELL   | Protocol   | Borrow (Short)                 |
| AERO   | Protocol   | Supply/Borrow                  |

**WRSETH and tBTC are explicitly excluded** — they are not present on Moonwell Base.

---

## V2.1: The Intent Engine & AI Agent Layer

### Intent Panel Architecture

Users interact through an `IntentPanel` component with three inputs:

1. **Sentiment selector** — three modes:
   - **Delta Neutral:** Supply and borrow correlated assets (LST↔LST, BTC↔BTC). Minimal directional price risk; profit from the APY spread between supply and borrow rates.
   - **Delta Long:** Supply ETH/BTC assets, borrow stablecoins. Upside exposure if ETH/BTC appreciate.
   - **Delta Short:** Supply stablecoins, borrow appreciating assets. Profits if prices fall.

2. **Target APY slider** — 5% to 50% range

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

### Full Strategy Matrix (23 Pairs)

**Delta Neutral (8 pairs):**  
cbETH/wstETH · cbETH/rETH · cbETH/WeETH · wstETH/rETH · wstETH/WeETH · rETH/WeETH · cbBTC/LBTC · CBBTC/LBTC

**Delta Long (8 pairs):**  
cbETH/USDC · wstETH/USDC · rETH/USDC · WeETH/USDC · cbETH/DAI · wstETH/DAI · CBBTC/USDC · CBBTC/DAI · All Delta Neutral Pairs

**Delta Short (7 pairs):**  
USDC/wstETH · USDC/cbETH · USDC/rETH · USDC/CBBTC · USDC/WELL · DAI/wstETH · DAI/cbETH

---

## V2.1: CDP AgentKit Guardian

### The Liquidation Problem

When a leveraged lending position's health factor drops below 1.0 on Moonwell, the position is subject to liquidation — a third party repays the debt at a penalty and claims the collateral at a discount. The user loses the liquidation penalty (typically 5–10%) on top of any market losses.

Approximately 30% of DeFi liquidations occur when the health factor is in the 1.0–1.2 range — a window where an automated 10-minute intervention would prevent the liquidation entirely.

PivotBot v2.1 addresses this with an always-on CDP AgentKit guardian operating through AgentVault.

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

## V2.1: PivotProPass — ERC-721 Subscription NFT

Access to agent execution is gated behind a time-limited ERC-721 NFT called `PivotProPass`.

### Smart Contract Design

```solidity
contract PivotProPass is ERC721 {
    enum Tier { ONE_MONTH, THREE_MONTHS, TWELVE_MONTHS }

    mapping(uint256 => uint256) public expiryOf;   // tokenId → expiry timestamp
    mapping(address => uint256) public tokenOf;    // address → tokenId (one per wallet)

    function mint(Tier tier) external payable {
        require(tokenOf[msg.sender] == 0, "One pass per wallet");
        require(msg.value == tierPrices[tier], "Exact ETH required");
        uint256 tokenId = ++_nextTokenId;
        expiryOf[tokenId] = block.timestamp + tierDurations[tier];
        tokenOf[msg.sender] = tokenId;
        _mint(msg.sender, tokenId);
        emit PassMinted(msg.sender, tokenId, tier, expiryOf[tokenId]);
    }

    function renew(Tier tier) external payable {
        uint256 tokenId = tokenOf[msg.sender];
        require(tokenId != 0, "No active pass");
        require(msg.value == tierPrices[tier], "Exact ETH required");
        // Stacks remaining time — users who renew early don't lose days
        uint256 base = max(expiryOf[tokenId], block.timestamp);
        expiryOf[tokenId] = base + tierDurations[tier];
        emit PassRenewed(msg.sender, tokenId, tier, expiryOf[tokenId]);
    }

    function isActive(address user) external view returns (bool) {
        uint256 tokenId = tokenOf[user];
        return tokenId != 0 && expiryOf[tokenId] > block.timestamp;
    }
}
```

**Key design decisions:**
- **One pass per wallet** enforced at mint
- **Renewal stacks remaining time** — prevents churn by removing the cost of early renewal
- **Transferable with time inheritance** — expiry follows the tokenId, not the wallet
- **On-chain expiry check** — `usePivotPro` hook calls `isActive()` on every execution attempt; cannot be bypassed via localStorage
- **Exact ETH payment** — reverts on incorrect value

**Subscription tiers:**

| Tier          | Duration | Notes           |
| ------------- | -------- | --------------- |
| One Month     | 30 days  | Starter access  |
| Three Months  | 90 days  | Discounted rate |
| Twelve Months | 365 days | Best value      |

**Free tier:** Non-subscribers receive 3 free strategy analyses per calendar month. Analysis (intent → reasoning → recommendation) is always free; atomic execution requires a pass. The monthly counter is tracked in localStorage — no security implication, as the execution gate is enforced on-chain.

---

## Frontend: Agent Setup UI (New in v2.1)

The `IntentPanel`'s Agent Mode section gains an **Agent Setup card** backed by the `useAgentVault` hook.

### Agent Setup Card

The card surfaces all AgentVault state and controls in one place:

| Display                   | Description                                                 |
| ------------------------- | ----------------------------------------------------------- |
| AgentVault address        | The deployed contract address for this user                 |
| Executor address          | The currently registered CDP agent hot wallet               |
| WETH balance / cap        | Current holdings vs. the owner-set deployable cap           |
| USDC balance / cap        | Current holdings vs. the owner-set deployable cap           |
| Cooldown setting          | Minimum seconds between executions                          |
| Time since last execution | Live countdown to next permitted execution                  |
| Pause / Resume toggle     | Blocks or re-enables all agent execution                    |
| Drain All button          | Sweeps WETH, USDC, and ETH dust to owner wallet immediately |
| Fund Agent form           | Transfers working capital from user's wallet to AgentVault  |

### `useAgentVault` Hook

Reads all AgentVault state in a single Multicall3 batch:

```javascript
const calls = [
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'executor') },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'caps', [WETH]) },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'caps', [USDC]) },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'cooldown') },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'lastExecutionTime') },
  { target: agentVaultAddress, callData: encodeFunctionData(ABI, 'paused') },
  { target: WETH, callData: encodeFunctionData(ERC20_ABI, 'balanceOf', [agentVaultAddress]) },
  { target: USDC, callData: encodeFunctionData(ERC20_ABI, 'balanceOf', [agentVaultAddress]) },
];
```

Exposes write functions: `setExecutor`, `setCap`, `setCooldown`, `pause`, `unpause`, `drain`, and `fund`.

The `AGENT_VAULT_ABI` is added to the constants file alongside the existing `NFT_ABI` and `AERODROME_ABI`.

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
- CDP AgentKit guardian scoped to two selectors only via AgentVault whitelist
- AgentVault `drain()` has no timelock and requires no agent involvement

**Security audit:** Commissioned Q2 2026. Results will be published publicly. AgentVault is in scope.

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
- **Deployment tx:** https://basescan.org/tx/0xdc42587932cb065c44843ad8226fb65db4d7f6e0b6b5cd2f9f9709b67a67a476

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