# PivotBot: Non-Custodial Leveraged Yield Agent for Base DeFi

**AI-powered leveraged yield automation with an autonomous health-factor guardian**

Set a yield target, choose your market view, and let PivotBot handle leverage, strategy selection, and risk monitoring — all without giving up custody of your assets. The CDP Guardian continuously monitors your position and autonomously intervenes before liquidation, using only your pre-approved working capital.

**Version 2.1 — August 2026**  
**Author:** Abolaji M. Adedeji · Syncedge Solutions  
**Live App:** [syncedgesolutions.xyz/pivot](https://syncedgesolutions.xyz/pivot)  
**Technical Overview:** [docs/technical-overview.md](./docs/technical-overview.md)  
**LinkedIn Article:** [PivotBotV1.0: A Technical Look at Non-Custodial Leveraged Yield](https://linkedin.com/pulse/pivotbot-technical-look-non-custodial-leveraged-yield-adedeji-e2soe)  
**Demo Video:** [2-minute walkthrough of v1.0](https://youtu.be/ZfPAJjwvWgY?si=oICW7q6mE3kLogGq)  
**Contact:** abolaji@syncedgesolutions.xyz

---

## What PivotBot Does

PivotBot is a live yield automation protocol on Base. It helps DeFi users open and manage leveraged lending positions in one atomic flow instead of manually looping supply and borrow transactions.

In practical terms, PivotBot is built for three groups:

- **Yield farmers** who want higher yield on LSTs, BTC, and stablecoins without babysitting positions all day
- **Crypto traders** who want delta-neutral, long, or short exposure without paying perpetual funding rates
- **Treasuries and power users** who want non-custodial automation with hard onchain controls around agent behavior

**Live reference point (April 9, 2026):**

- **+44.97% net APY** on cbETH/wstETH
- **4x leverage**
- **Health factor: 1.33**
- **Live on Base mainnet since Q1 2026**

---

## The Problem with the Current Swing Trading / Leveraged Yield Farming Platforms

Today, most leveraged trading setups force users into at least one expensive compromise:

### 1. Funding-rate drag erodes returns

Perpetual futures platforms commonly impose recurring funding payments (often in the 8-20% annualized range, depending on market conditions). In sideways markets, these costs can quietly consume a meaningful share of profits.

### 2. Collateral stays underutilized

On many CEX margin and DeFi perp workflows, collateral is mostly just parked as backing capital. Instead of earning robust supply-side yield, users are often paying to keep exposure open.

### 3. Custody and counterparty risk remain

Centralized venues require users to hand over custody. That introduces non-market risk: exchange freezes, operational failure, or hacks can affect access to funds even when the trade thesis is correct.

**Bottom line:** traders, yield farmers, and DAOs either accept lower net returns, take on avoidable custody risk, or spend substantial time manually managing leverage.

---

## How PivotBot Solves It

### 1. It uses lending-market leverage instead of perpetual funding

PivotBot uses Moonwell lending markets and Balancer flashloans rather than perpetual contracts. That means the strategy is designed around earning supply-side yield on leveraged collateral, not paying recurring funding to stay in position.

### 2. It opens leverage atomically in one transaction

Instead of repeating multiple supply-borrow-swap loops, PivotBot completes the leverage flow atomically:

1. Borrow temporary liquidity from Balancer V2
2. Supply collateral to Moonwell
3. Borrow the paired asset
4. Swap through Aerodrome V2
5. Repay the flashloan before the transaction ends

If any step fails, the entire transaction reverts. There is no half-open position left behind.

### 3. It lets users express intent instead of building the trade manually

Users choose:

- **Sentiment:** Delta Neutral, Delta Long, or Delta Short
- **Target APY:** 5% to 100%
- **Risk tolerance:** Conservative, Moderate, or Aggressive

The strategy engine then filters the available pair universe, fetches live Moonwell rates through Multicall3, checks Aerodrome liquidity, and ranks the best candidates.

### 4. It monitors health factor with an AI guardian that acts autonomously

PivotBot integrates a **CDP AgentKit-powered guardian** that continuously monitors `% Credit Remaining` (Moonwell's formula: `collateralValueInEth × collateralFactor / borrowBalanceInEth × 100`) and can autonomously intervene before liquidation. The default threshold is **85% Credit Remaining** (equivalent to Health Factor ~1.18 when CF=0.825), and the threshold is user-configurable.

When % Credit Remaining drops below the user's defined threshold, the guardian performs an **iterative, 20% partial repayment** of the highest-value borrowed asset using only the working capital already sitting in the user's AgentVault:

1. **Read** Moonwell position data (collateral, borrow balances, prices)
2. **Check** cooldown status by reading `getNextExecutionTime()` from AgentVault
3. **Verify** user's PivotProPass is still active
4. **Determine** the highest-value borrowed asset and compute 20% of it
5. **Convert** AgentVault funds: USDC → WETH (via Aerodrome) first, then WETH → borrow token
6. **Dispatch** a `repayBorrow()` call through `AgentVault.execute()` with the swapped borrow token

**Key invariants the guardian respects at all times:**

| Constraint | Enforcement |
|---|---|
| **No collateral redemption** | `redeemAssetFromMw` is never called |
| **No flashloans** | `repayOrSupplyAssetWithFlashloan` is never called |
| **Vault funds only** | All repayments use working capital already in the user's AgentVault |
| **Cooldown respected** | Each action is dispatched through `AgentVault.execute()`, which enforces cooldown on-chain |
| **Pass-gated** | Users with expired PivotProPass are skipped |
| **Funds preferred in WETH** | USDC is converted to WETH before any borrow repayment |

Because the AgentVault cooldown permits only one `execute()` call per interval, a full recovery (swap + repay) requires at least **two cron cycles** — the guardian is patient and makes incremental progress each cycle, capped at 10 iterations per cron invocation.

**Non-custodial guarantee:** The CDP AgentKit hot wallet can only call `AgentVault.execute()` with whitelisted selectors. It can never withdraw tokens from AgentVault, redeem collateral, or access position proceeds. Position collateral held in Moonwell via PivotBot is never accessible to the agent hot wallet.

---

## Why The Architecture Matters (Six-Contract System)

PivotBot is designed so users keep custody while the automation layer stays tightly bounded.

### PivotBot.sol

The core execution engine. Handles leverage and deleverage lifecycle logic, Balancer V2 flashloan callbacks, Moonwell supply and borrow operations, and Aerodrome V2 swaps. Role-based access control with `DEFAULT_ADMIN_ROLE`, `MANAGER_ROLE` (granted to AgentVault), and `PAUSER_ROLE`. Immutable after deployment (no upgradeability).

### PivotBotFactory.sol

Deploys a unique, per-user PivotBot instance using CREATE2 deterministic addressing. Each user's collateral and borrow positions are held in their own bot contract — no commingling of funds. This is the non-custodial guarantee.

### PivotBotFactoryManager.sol

Manages protocol-wide configuration, role-based access control, fee collection (0.05% on flashloans + 0.05% on swaps), and the Moonwell token/market whitelist.

### AgentVault.sol

The authority boundary between the CDP agent hot wallet and the user's PivotBot. The agent does not hold direct access to PivotBot; it only executes through AgentVault.

AgentVault enforces four hard Solidity-level constraints:

1. **Pass validity check (first):** execution fails immediately if the wallet no longer has an active PivotProPass
2. **Function selector whitelist:** only these 9 approved execution selectors can be forwarded:
   - `supplyAsset` — supply collateral to Moonwell
   - `borrowAsset` — borrow from Moonwell
   - `repayBorrow` — repay borrowed assets
   - `repayBorrowBehalf` — repay on behalf of PivotBot
   - `redeemAssetFromMw` — redeem collateral (not used by guardian)
   - `repayOrSupplyAssetWithFlashloan` — flashloan-assisted operations
   - `swapOnAerodromeV2` — token swaps via Aerodrome
   - `claimRewards` — claim Moonwell WELL rewards
   - `accrueInterest` — trigger interest accrual on Moonwell
3. **Per-transaction spending cap:** owner-defined per-token spending limits bound each execution
4. **Cooldown enforcement:** configurable minimum gap (in seconds) between consecutive executions; owners can also pause execution instantly

Owners have full control: they can adjust caps, cooldowns, pause/unpause, and withdraw working capital atomically via `drain()`. Config functions (drain, setSpendingCap, setCooldown, pause, unpause) are restricted to the **owner only** — the executor role cannot call them.

### AgentVaultFactory.sol

Coordinates atomic co-deployment of AgentVault instances and PivotProPass NFT minting in a single transaction.

### PivotProPass.sol

Soulbound (non-transferable) ERC-721 NFT that time-gates agent execution access. Pricing is USD-denominated and converted to ETH on-chain via Chainlink oracle (ETH/USD on Base). Revenue flows immediately to the protocol treasury with zero custodial risk.

---

## Strategy Universe

PivotBot is strictly scoped to **12 Moonwell Base markets**:

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

The strategy engine evaluates **23 trading pairs** across three sentiment styles:

| Strategy Type     | Count | What It Means                                                                     | Typical User               |
| ----------------- | ----- | --------------------------------------------------------------------------------- | -------------------------- |
| **Delta Neutral** | 8     | Supply and borrow correlated assets (LST ↔ LST, BTC ↔ BTC) to capture APY spreads | Conservative yield seekers |
| **Delta Long**    | 8     | Supply ETH or BTC-linked assets, borrow stablecoins; profit if prices rise        | Bullish traders            |
| **Delta Short**   | 7     | Supply stablecoins, borrow appreciating assets; profit if prices fall             | Bearish or hedging users   |

**Examples:**
- **Neutral:** cbETH/wstETH, wstETH/rETH, cbBTC/LBTC (LST or BTC spread positions earning APY differentials)
- **Long:** cbETH/USDC, wstETH/DAI, cbBTC/USDC (leveraged long ETH/BTC with stablecoin liability)
- **Short:** USDC/cbETH, DAI/cbBTC, USDC/WELL (leveraged short exposure with stablecoin collateral)

---

## PivotProPass: Access Without Custody Risk

PivotProPass is a **soulbound ERC-721 pass** that gates agent execution on-chain.

What that means for users:

- **One pass per wallet** — enforced by `tokenOf[address]` mapping at mint time
- **Non-transferable by design** to avoid secondary-market gaming; soulbound property preserves this invariant through the pass lifecycle
- **Renewals stack remaining time** — early renewal does not waste unused days: `expiry = max(oldExpiry, block.timestamp) + duration`
- **Execution access is checked onchain** inside AgentVault before any other execution rule — this runs **first** (fail-fast: expired passes block execution immediately with `PassRequired()` error)

### Subscription durations

| Tier          | Duration |
| ------------- | -------- |
| One Month     | 30 days  |
| Three Months  | 90 days  |
| Six Months    | 180 days |
| Twelve Months | 365 days |

Pricing is USD-denominated and converted to ETH onchain using the Chainlink ETH/USD oracle on Base. The pass contract validates stale price data before minting or renewal, and enforces exact ETH amounts (no tolerance for overpayment). Revenue flows immediately to the protocol treasury with zero custodial risk.

### Free usage before subscribing

Users can access **3 free strategy analyses per calendar month** (frontend-only feature, tracked via localStorage). The distinction is important:

- **Analysis** (intent → reasoning → recommendation): Always free, up to 3/month
- **Atomic execution** (open/close position on PivotBot): Requires active PivotProPass

Free analyses cannot be used to execute positions — they are purely informational.

---

## Agent Setup UI (v2.1)

The frontend Agent tab provides a complete interface for AgentVault management and guardian configuration:

### Agent Setup Card

The card surfaces all AgentVault state and controls in one place:

- **AgentVault address** — the deployed contract address for this vault
- **Vault owner** — the principal wallet used for PivotProPass validity checks
- **WETH/USDC spending caps per tx** — current max spend per transaction for each token
- **Cooldown setting** — minimum gap (in seconds) between consecutive agent executions
- **Last/Next execution timestamps** — including a live countdown to next available execution
- **Pause status** — whether execution is currently paused
- **PivotProPass expiry & active status** — real-time boolean check
- **Drain All button** — sweeps WETH, USDC, and ETH dust to owner wallet atomically
- **Update spending cap/cooldown forms** — adjust constraints independently per token

### Additional Components

| Component | Location | Purpose |
|---|---|---|
| `useAgentVault` hook | `src/hooks/use-agent-vault.ts` | Batched Multicall3 reads for all vault state + write functions |
| `usePivotBot` hook | `src/hooks/use-pivot-bot.ts` | Bot position and strategy state |
| `HealthGauge` | `src/components/pivot/HealthGauge.tsx` | Visual % Credit Remaining gauge |
| `AgentMonitoringConfig` | `src/components/pivot/AgentMonitoringConfig.tsx` | Configure guardian thresholds |
| `LowVaultFundsWarning` | `src/components/pivot/PivotDashboard.tsx` | Warns when AgentVault working capital is low |
| `PassRenewalModal` | `src/components/pivot/PassRenewalModal.tsx` | Tier-based pass renewal flow |

The `useAgentVault` hook batches all reads (vault owner, spending caps, cooldown, execution times, pause status, balances, pass status) into a single Multicall3 call for gas-efficient UI rendering.

---

## Fee Model

| Fee Type             | Rate       | Applies To                                   |
| -------------------- | ---------- | -------------------------------------------- |
| **Protocol fee**     | 0.05%      | Flashloan amount on leverage and deleverage  |
| **Swap fee**         | 0.05%      | Aerodrome swap amount during position events |
| **Subscription fee** | Tier-based | PivotProPass minting and renewal             |
| **B2B licensing**    | Negotiated | White-label deployments                      |

Fees route through the protocol treasury via Manager. There is no separate utility token required to use the protocol.

---

## On-Chain Verification

All contracts are live on Base mainnet and verifiable on Basescan:

| Contract     | Address                                      | Role                                    |
| ------------ | -------------------------------------------- | --------------------------------------- |
| **PivotBot** | `0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3` | Core leverage and deleverage engine     |
| **Factory**  | `0xc4E537890e86fDD44aF936218f80d7326820d97d` | Deterministic per-user bot deployment   |
| **Manager**  | `0x144F04807a6af905E3112Fe3Da9302D308c6DF26` | Fee routing, config, and access control |

- [PivotBot on Basescan](https://basescan.org/address/0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3)
- [Factory on Basescan](https://basescan.org/address/0xc4E537890e86fDD44aF936218f80d7326820d97d)
- [Manager on Basescan](https://basescan.org/address/0x144F04807a6af905E3112Fe3Da9302D308c6DF26)
- [Recent Base transaction](https://basescan.org/tx/0xdc42587932cb065c44843ad8226fb65db4d7f6e0b6b5cd2f9f9709b67a67a476)

---

## Technology Stack

| Layer              | Technology                                  |
| ------------------ | ------------------------------------------- |
| Smart contracts    | Solidity 0.8.33, Foundry                    |
| Chain              | Base (Ethereum L2 via OP Stack)             |
| Flashloans         | Balancer V2 (`IVault.flashLoan`)            |
| Lending            | Moonwell Base (Compound V2 fork)            |
| DEX routing        | Aerodrome V2 (Velodrome fork, Base)         |
| Agent framework    | CDP AgentKit (Coinbase Developer Platform)  |
| Agent wallets      | CDP Server Wallets (non-custodial)           |
| Authority boundary | AgentVault.sol (per-user authority cap)     |
| Frontend           | React, Wagmi, Viem, Tailwind CSS            |
| Onchain reads      | Multicall3 (batched reads)                  |
| NFT standard       | ERC-721 via OpenZeppelin 5.x                |

---

## Security And Risk Controls

### Built-in protections

- Reentrancy guards on mutating entry points
- Balancer callback validation — only `IVault(balancerVault)` can trigger `receiveFlashLoan`
- Approved token and market whitelist (Manager-controlled)
- Atomic slippage enforcement — position reverts rather than executing at unfavourable rates
- No `delegatecall`-based execution path
- Checks-Effects-Interactions pattern throughout
- Agent execution restricted by AgentVault rather than direct manager access
- CDP AgentKit guardian scoped to only owner-whitelisted selectors via AgentVault (executor role only)
- AgentVault `drain()` has no timelock and requires no agent involvement
- PivotProPass pass check runs **first** in `AgentVault.execute()` before any other constraint (fail-fast)
- AgentVault authorization: executor role cannot call `drain()`, `setSpendingCapPerTx()`, `setCooldown()`, `pause()`, `unpause()`, or any config functions — only owner (`DEFAULT_ADMIN_ROLE`)
- Pass renewal enforces exact ETH amount (no tolerance for overpayment)
- Token input validation against approved registry

### Testing status

- Unit tests for PivotBot, AgentVault, and PivotProPass behaviors
- Integration tests against Base mainnet state for Moonwell, Aerodrome, and Balancer
- Fuzz tests for leverage inputs, swap paths, spending caps, and cooldowns

### Audit status

Security audit is commissioned for **Q2 2026**. AgentVault and PivotProPass are in scope alongside the core protocol.

---

## Roadmap

### Q1 2026
- Core contracts deployed on Base
- Live yield demonstration at +44.97% net APY on cbETH/wstETH
- Public app live at syncedgesolutions.xyz/pivot

### Q2 2026
- Expand guardian and intent UI experience
- Launch and refine PivotProPass workflows
- Complete and publish security audit work

### Q3 2026
- Scale B2B white-label deployments
- Extend to Unichain and OP Mainnet (Optimism Superchain family)
- Bring full autonomous deleveraging into wider production use

### Q4 2026
- Arbitrum One deployment
- Institutional API access
- Governance and multi-chain expansion

### 2027 and beyond
- Additional EVM chains with Moonwell/Compound V2 deployments
- Cross-chain deployments use the same Factory/Manager/PivotBot/AgentVault architecture — only the external protocol addresses (Moonwell, Aerodrome, Balancer) are chain-specific. The core logic is chain-agnostic.

---

## Resources

- [Technical Overview](./docs/technical-overview.md)
- [Live App](https://syncedgesolutions.xyz/pivot)
- [LinkedIn Article V1.0](https://linkedin.com/pulse/pivotbot-technical-look-non-custodial-leveraged-yield-adedeji-e2soe)
- [2-minute Demo Video V1.0](https://youtu.be/ZfPAJjwvWgY?si=oICW7q6mE3kLogGq)
- [Moonwell Base Markets](https://app.moonwell.fi)
- [Balancer V2 Vault Docs](https://docs.balancer.fi/reference/contracts/vault)
- [Aerodrome Docs](https://aerodrome.finance/docs)
- [CDP AgentKit Docs](https://developer.coinbase.com/agentkit)

---

## Important Disclaimer

PivotBot is a **non-custodial protocol** — users retain custody of funds at all times via the per-user bot and AgentVault architecture. However, leveraged DeFi is still risky. Smart contract risk, liquidation risk, market volatility, oracle issues, and slippage can all affect outcomes.

This README is for informational purposes only and should not be treated as financial advice.

---

**Founder:** Abolaji M. Adedeji  
**Email:** abolaji@syncedgesolutions.xyz  
**Company:** Syncedge Solutions
