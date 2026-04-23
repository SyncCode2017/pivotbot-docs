# PivotBot: AI-Powered Leveraged Yield for Base

**Non-custodial leveraged yield automation with an autonomous health-factor guardian**

Set a yield target, choose your market view, and let PivotBot handle leverage, strategy selection, and risk monitoring without giving up custody of your assets.

**Version 2.1 - April 2026**  
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

## The Problem: Why Current Yield Strategies Fail

If you already farm yield or trade onchain, you usually face one of three bad trade-offs:

### 1. Funding rates drain returns

Perpetual futures and margin products often charge continuous funding. In a flat market, you can be directionally right and still give away a large part of your return.

### 2. Collateral often sits idle

On many centralized or manual DeFi workflows, the collateral backing your leverage does not work very hard. You borrow against it, but the setup itself is inefficient and slow.

### 3. Liquidations happen when no one is watching

Leveraged positions can move from healthy to dangerous quickly. By the time a user notices, the liquidation penalty may already be unavoidable.

For many users, the result is predictable: lower yield than expected, too much manual work, or unacceptable liquidation risk.

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

### 4. It monitors health factor with an AI guardian

PivotBot integrates a CDP AgentKit-powered guardian that monitors health factor and can act before liquidation. The default threshold is **1.25**, and the threshold is user-configurable.

The guardian does not receive broad control over the bot. It operates through a per-user AgentVault that enforces strict onchain constraints before any action reaches PivotBot.

---

## Why The Architecture Matters

PivotBot is designed so users keep custody while the automation layer stays tightly bounded.

### PivotBot.sol

This is the execution engine. It handles leverage and deleverage lifecycle logic, Balancer flashloan callbacks, Moonwell supply and borrow operations, and Aerodrome swaps.

### Factory.sol

This deploys a dedicated PivotBot instance for each user with CREATE2. Positions are isolated per user rather than pooled together, which prevents fund commingling.

### Manager.sol

This contract manages protocol configuration, access roles, fee routing, and the approved token and market set.

### AgentVault.sol

This is the authority boundary between the CDP agent hot wallet and the user bot. The agent executes through AgentVault, not directly against PivotBot.

AgentVault enforces four key constraints:

- **Pass validity first:** execution fails immediately if the wallet no longer has an active PivotProPass
- **Selector whitelist:** only approved execution selectors can be forwarded
- **Per-transaction spending cap:** owner-defined token spending limits bound each execution
- **Cooldown and pause controls:** owners can slow or halt execution at any time

### PivotProPass.sol

This is the time-gated subscription NFT used for agent execution access.

### AgentVaultFactory onboarding flow

Vault deployment and pass minting are coordinated atomically so users can set up the guarded automation layer in one flow.

---

## Strategy Universe

PivotBot currently works across **12 Moonwell Base markets**:

- **LSTs:** cbETH, wstETH, rETH, WeETH
- **BTC assets:** cbBTC, LBTC
- **Stablecoins:** USDC, DAI
- **Base-native or protocol assets:** WETH, WELL, AERO, VIRTUAL

The strategy engine evaluates **23 core pairs** across three styles:

| Strategy Type     | Count | What It Means                                                                        | Typical User               |
| ----------------- | ----- | ------------------------------------------------------------------------------------ | -------------------------- |
| **Delta Neutral** | 8     | Supply and borrow correlated assets to target spread with lower directional exposure | Conservative yield seekers |
| **Delta Long**    | 8     | Supply ETH or BTC-linked assets and borrow stablecoins                               | Bullish users              |
| **Delta Short**   | 7     | Supply stablecoins and borrow appreciating assets                                    | Bearish or hedging users   |

Examples include LST-vs-LST spreads such as cbETH/wstETH, BTC cross-pairs such as cbBTC/LBTC, long setups using ETH or BTC collateral against USDC or DAI debt, and short setups using stablecoins against ETH, BTC, or WELL exposure.

---

## PivotProPass: Access Without Custody Risk

PivotProPass is a **soulbound ERC-721 pass** that gates agent execution.

What that means for users:

- **One pass per wallet**
- **Non-transferable by design** to avoid secondary-market gaming
- **Renewals stack remaining time** so early renewal does not waste unused days
- **Execution access is checked onchain** inside AgentVault before any other execution rule

### Subscription durations

| Tier          | Duration |
| ------------- | -------- |
| One Month     | 30 days  |
| Three Months  | 90 days  |
| Six Months    | 180 days |
| Twelve Months | 365 days |

Pricing is USD-denominated and converted to ETH onchain using the Chainlink ETH/USD feed on Base. The pass contract validates stale price data before minting or renewal.

### Free usage before subscribing

Users can access **3 free strategy analyses per calendar month**. Analysis is free, but **agent execution requires an active PivotProPass**.

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

## Live Deployment On Base

| Contract     | Address                                      | Role                                    |
| ------------ | -------------------------------------------- | --------------------------------------- |
| **PivotBot** | `0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3` | Core leverage and deleverage engine     |
| **Factory**  | `0xc4E537890e86fDD44aF936218f80d7326820d97d` | Per-user bot deployment                 |
| **Manager**  | `0x144F04807a6af905E3112Fe3Da9302D308c6DF26` | Fee routing, config, and access control |

**Verification and activity:**

- [PivotBot on Basescan](https://basescan.org/address/0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3)
- [Factory on Basescan](https://basescan.org/address/0xc4E537890e86fDD44aF936218f80d7326820d97d)
- [Manager on Basescan](https://basescan.org/address/0x144F04807a6af905E3112Fe3Da9302D308c6DF26)
- [Recent Base transaction](https://basescan.org/tx/0xdc42587932cb065c44843ad8226fb65db4d7f6e0b6b5cd2f9f9709b67a67a476)

---

## Technology Stack

| Layer              | Technology                       |
| ------------------ | -------------------------------- |
| Smart contracts    | Solidity 0.8.33, Foundry         |
| Chain              | Base                             |
| Flashloans         | Balancer V2                      |
| Lending            | Moonwell Base                    |
| DEX routing        | Aerodrome V2                     |
| Agent framework    | CDP AgentKit                     |
| Agent wallets      | CDP Server Wallets               |
| Authority boundary | AgentVault                       |
| Frontend           | React, Wagmi, Viem, Tailwind CSS |
| Onchain reads      | Multicall3                       |
| NFT standard       | ERC-721 via OpenZeppelin 5.x     |

---

## Security And Risk Controls

### Built-in protections

- Reentrancy guards on mutating entry points
- Balancer callback validation for flashloan execution
- Approved token and market whitelist
- Atomic slippage enforcement
- No delegatecall-based execution path
- Checks-Effects-Interactions pattern throughout
- Agent execution restricted by AgentVault rather than direct manager access
- `drain()` support for owner recovery of WETH, USDC, and ETH dust from AgentVault

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
- Extend to Unichain and OP Mainnet
- Bring full autonomous deleveraging into wider production use

### Q4 2026 and beyond

- Arbitrum deployment
- Institutional API access
- Governance and multi-chain expansion

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

PivotBot is a **non-custodial protocol**, but leveraged DeFi is still risky. Smart contract risk, liquidation risk, market volatility, oracle issues, and slippage can all affect outcomes.

This README is for informational purposes only and should not be treated as financial advice.

---

**Founder:** Abolaji M. Adedeji  
**Email:** abolaji@syncedgesolutions.xyz  
**Company:** Syncedge Solutions
