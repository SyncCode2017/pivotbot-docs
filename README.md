# PivotBot: Autonomous Yield Agent for the Agentic Internet

**Non-Custodial Leveraged Yield Agent for Base DeFi — with Autonomous Health Factor Guardian**

> *Users set intent once. The agent executes, monitors & protects.*

**Version 2.0 — April 2026**  
**Author:** Abolaji M. Adedeji · Syncedge Solutions  
**Live App:** [syncedgesolutions.xyz/pivot](https://syncedgesolutions.xyz/pivot)  
**Contact:** abolaji@syncedgesolutions.xyz

---

## Overview

PivotBot is a **Post Web-native, intent-driven autonomous yield agent** deployed on Base that:

- **Enables amplified leveraged lending** across Moonwell Base markets in a single atomic transaction using Balancer V2 zero-fee flashloans
- **Earns supply-side yield** on full leveraged collateral **without paying funding rates** (unlike perpetual futures)
- **Automates strategy selection** via AI reasoning powered by Coinbase CDP AgentKit based on your yield intent
- **Continuously monitors** position health 24/7 and auto-deleverages before liquidation strikes

**Live Performance:**  
✓ **+44.97% Net APY** on cbETH/wstETH delta-neutral pair (Health Factor: 1.33)  
✓ **5× Max Leverage** · **Single Atomic Transaction** · **Zero Funding Rates**  
✓ **Live on Base Mainnet** since Q1 2026

---

## The Problem

Traditional leveraged DeFi and margin trading have three critical flaws:

| Problem                 | Impact                                                                                                                             |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **8–20% Funding Rates** | Perpetual futures platforms charge continuous funding rates that destroy returns in sideways markets                               |
| **Zero Capital Yield**  | CEX margin accounts & perp positions earn nothing. Your collateral is idle.                                                        |
| **No Automation**       | 30% of liquidations occur when health factors are 1.0–1.2 — a 10-minute window where advance intervention could prevent total loss |

**Result:** Retail traders, yield farmers, and DAOs leave massive returns on the table or face sudden liquidation without warning.

---

## How PivotBot Solves It

### 1. **No Funding Rates** — Earn Yield Instead

PivotBot uses **lending protocol leverage** (not perpetual futures) so your collateral **generates yield while borrowed**. You pay zero funding rates and earn the spread between supply and borrow APYs — amplified by your chosen leverage.

**Yield Formula:**
```
Avg Net APY Calculation
= (Supply APY * Supply Value in WETH + WELL APY * WELL Assets Value in WETH − Borrow APY * Borrow Value in WETH) / Total Supply Value in WETH
```

### 2. **Atomic, Single-Transaction Leverage**

Unlike traditional "looping" strategies that execute over multiple transactions (exposing you to price swings between loops), PivotBot achieves full target leverage in **one atomic transaction**:

```
1. Request flashloan from Balancer (zero fee)
2. Supply userDeposit + flashloan to Moonwell
3. Borrow target asset against new collateral
4. Swap borrowed asset → flashloan denomination (Aerodrome)
5. Repay Balancer in same transaction
   ⚠ Any step fails → entire tx reverts. No partial state.
```

**Gas efficient.** **No liquidation window.** **Instant full leverage.**

### 3. **Intent-Driven Strategy Selection**

Instead of manually picking which pair to trade, you express **intent once** — choosing:
- **Sentiment:** Delta Neutral, Delta Long, or Delta Short
- **Target APY:** 5–50% range
- **Risk Tolerance:** Conservative, Moderate, Aggressive

The **agent reasoning engine** (5 steps) then:
1. **Parses** your intent
2. **Filters** 23 strategy pairs to sentiment-matching candidates
3. **Fetches live Moonwell rates** via on-chain Multicall3
4. **Checks Aerodrome liquidity** to estimate slippage
5. **Ranks & scores** by net APY and returns top 3 recommendations

You get **data-driven, optimal strategy selection** — not guesswork.

### 4. **24/7 Guardian Protection** (CDP AgentKit)

A **CDP AgentKit-powered agent** runs continuously, monitoring your position's health factor every block.

- **Default Monitoring:** Health factor < 1.25
- **User-Configurable:** Set your own liquidation threshold  
- **Operates through AgentVault:** The guardian's hot wallet is registered as the executor inside the user's AgentVault instance, enforcing balance caps, a function whitelist, cooldowns, and pause controls before any call reaches PivotBot.
- **Automatic Response:** When triggered, the guardian agent:
  - Calculates optimal partial or full deleverage
  - Executes `closePosition()` atomically in a single flashloan tx (via AgentVault)
  - Restores health factor above safe threshold
  - Notifies you with full audit trail on-chain

**Non-custodial guarantee maintained:** The guardian can only call `closePosition()` — it cannot open positions, transfer funds outside the close flow, or interact with any contract other than your bot. Position proceeds always return directly to the owner wallet.

---

## Architecture: Four-Contract System

### **PivotBot.sol** — Core Engine

The main execution contract handling all leverage and deleverage operations.

**Key Features:**
- Balancer V2 `receiveFlashLoan` callback integration
- Moonwell `CErc20` (mToken) supply, borrow, and redeem operations
- Aerodrome V2 router for DEX swaps
- Reentrancy guards on all external entry points
- Slippage tolerance enforcement
- Immutable (no upgradeability)

**Supported Tokens:** 11 Moonwell Base markets
- **LST (ETH):** cbETH, wstETH, rETH, WeETH
- **BTC:** cbBTC, LBTC
- **Stablecoins:** USDC, DAI
- **Base Native:** WETH, WELL, AERO

**Access Control:** 
- **Role-based access:** `DEFAULT_ADMIN_ROLE`, `MANAGER_ROLE` (granted to AgentVault), `PAUSER_ROLE`
- **`MANAGER_ROLE` scope:** In v2.1, this role is granted to the user's AgentVault instance, not directly to the CDP agent hot wallet. AgentVault enforces its own hard constraints before forwarding any call to PivotBot.
- **`DEFAULT_ADMIN_ROLE` scope:** Role reserved for the bot owner to execute transactions and reconfigure the bot including assigning roles, revoking roles and pausing/unpausing the bot in case of emergency.
- **`PAUSER_ROLE` scope:** Role can only pause and unpause the bot.

### **Factory.sol** — Deterministic Deployment

Deploys a **unique, per-user bot instance** using `CREATE2`:

```solidity
bytes32 salt = keccak256(abi.encodePacked(user));
bot = address(new PivotBot{salt: salt}(user, manager));
```

**Why per-user?** Each user's collateral and borrow positions are held by their own bot contract. **No commingling of funds.** Non-custodial guarantee: the user's wallet is always the ultimate owner.

### **Manager.sol** — Protocol Contract for Configuration & Fee Routing

Handles:
- **Role-based access:** `DEFAULT_ADMIN_ROLE`, `MANAGER_ROLE`, `PAUSER_ROLE`
- **Protocol fee collection:** 0.05% on flashloans + 0.05% on swaps
- **Token/market whitelist:** Only approved Moonwell tokens accepted

### **AgentVault.sol** — Agent Authority Boundary (New in v2.1)

A per-user intermediary contract that sits between the CDP agent hot wallet and the user's PivotBot instance. Instead of granting `MANAGER_ROLE` directly to the agent's hot wallet, the owner grants it to their AgentVault. The CDP agent operates through AgentVault but is constrained by four hard Solidity-enforced limits:

- **Balance Cap** — Owner sets a maximum WETH and USDC amount the agent can deploy. Bounds maximum agent exposure to a single position's margin.
- **Function Whitelist** — Only trade execution selectors are permitted. Role management, owner wallet changes, and other calls are rejected at the Solidity level.
- **Cooldown** — Owner-configurable minimum gap between consecutive executions. Limits damage if the agent hot wallet key is ever compromised.
- **Pause Flag** — Owner can block all agent execution instantly without revoking roles. A `drain()` function sweeps all WETH, USDC, and ETH dust directly to the owner wallet at any time with no timelock.

**Non-custodial guarantee preserved:** Position proceeds always return directly to the owner wallet. Agent hot wallet is never in the withdrawal path.

---

## Strategy Matrix: 23 Trading Pairs

### **Delta Neutral (8 pairs)** — Minimal directional risk, earn spread

Correlated assets that move together. Profit from APY spread between supply and borrow rates.

- cbETH ↔ wstETH ↔ rETH ↔ WeETH (LST cross-pairs)
- cbBTC ↔ LBTC (BTC cross-pairs)

**Best for:** Conservative yield seekers, APY spread arbitrage

### **Delta Long (8 pairs)** — Bullish directional exposure

Supply ETH/BTC assets, borrow stablecoins. Upside if ETH/BTC prices appreciate.

- cbETH/USDC · wstETH/USDC · rETH/USDC · WeETH/USDC
- cbETH/DAI · wstETH/DAI · cbBTC/USDC · cbBTC/DAI

**Best for:** Bullish traders, LST yield + price upside

### **Delta Short (7 pairs)** — Bearish directional exposure

Supply stablecoins, borrow appreciating assets. Profits if prices fall.

- USDC ↔ wstETH ↔ cbETH ↔ rETH ↔ cbBTC
- USDC ↔ WELL
- DAI ↔ wstETH ↔ cbETH

**Best for:** Bearish traders, hedge positions

---

## Fee Structure

| Fee Type             | Rate               | Applies To                                     |
| -------------------- | ------------------ | ---------------------------------------------- |
| **Protocol Fee**     | 0.05%              | Flash loan amount on every leverage/deleverage |
| **Swap Fee**         | 0.05%              | Aerodrome swap amount on every position event  |
| **PivotProPass NFT** | ETH (per tier)     | Access to agent execution                      |
| **B2B Licensing**    | Fixed (negotiated) | White-label factory deployments                |

**Free tier:** Non-subscribers receive **3 free strategy analyses per calendar month**. Only atomic execution requires a pass.

---

## PivotProPass: ERC-721 Subscription NFT

Time-limited access to agent execution via an on-chain gated NFT.

### **Subscription Tiers**

| Tier          | Duration | Notes           |
| ------------- | -------- | --------------- |
| One Month     | 30 days  | Starter access  |
| Three Months  | 90 days  | Discounted rate |
| Twelve Months | 365 days | Best value      |

### **Key Design Features**

- ✓ **One pass per wallet** — enforced at mint
- ✓ **Renewal stacks remaining time** — early renewals don't lose days
- ✓ **Transferable** — secondary market buyers inherit remaining time
- ✓ **On-chain expiry check** — cannot be bypassed via localStorage
- ✓ **Exact ETH payment** — no partial payments accepted

---

## Live Deployment

### **Base Mainnet Smart Contracts**

| Contract     | Address                                      | Role                                  |
| ------------ | -------------------------------------------- | ------------------------------------- |
| **PivotBot** | `0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3` | Core leverage/deleverage engine       |
| **Factory**  | `0xc4E537890e86fDD44aF936218f80d7326820d97d` | Deterministic per-user bot deployment |
| **Manager**  | `0x144F04807a6af905E3112Fe3Da9302D308c6DF26` | Access control, fee routing, upgrades |

**Recent Transaction:**  
[0xdc42587932cb065c44843ad8226fb65db4d7f6e0b6b5cd2f9f9709b67a67a476](https://basescan.org/tx/0xdc42587932cb065c44843ad8226fb65db4d7f6e0b6b5cd2f9f9709b67a67a476)

**Verification Links:**
- [PivotBot on Basescan](https://basescan.org/address/0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3)
- [Factory on Basescan](https://basescan.org/address/0xc4E537890e86fDD44aF936218f80d7326820d97d)
- [Manager on Basescan](https://basescan.org/address/0x144F04807a6af905E3112Fe3Da9302D308c6DF26)

**Live App:** https://syncedgesolutions.xyz/pivot

---

## Technology Stack

| Layer                | Technology                                 |
| -------------------- | ------------------------------------------ |
| **Smart Contracts**  | Solidity 0.8.33, Foundry                   |
| **Blockchain**       | Base (Ethereum L2 via OP Stack)            |
| **Flashloans**       | Balancer V2 (`IVault.flashLoan`)           |
| **Lending**          | Moonwell Base (Compound V2 fork)           |
| **DEX**              | Aerodrome V2 (Velodrome fork, Base)        |
| **Agent Framework**  | CDP AgentKit (Coinbase Developer Platform) |
| **Agent Wallets**    | CDP Server Wallets (non-custodial)         |
| **Agent Boundary**   | AgentVault.sol (per-user authority cap)    |
| **Frontend**         | React, Wagmi, Viem, Tailwind CSS           |
| **On-chain Queries** | Multicall3 (batched reads)                 |
| **NFT Standard**     | ERC-721 (OpenZeppelin 5.x)                 |

---

## Security

### **Measures in Place**

✓ Reentrancy guard on all external entry points  
✓ Balancer callback validation (only Balancer Vault can call `receiveFlashLoan`)  
✓ Token whitelist (only Moonwell Base market tokens accepted)  
✓ Slippage tolerance enforced atomically  
✓ No upgradeability on core engine (immutable after deployment)  
✓ No `delegatecall` used anywhere  
✓ Checks-Effects-Interactions pattern throughout  
✓ CDP AgentKit guardian scoped to deleverage only via `MANAGER_ROLE`

### **Testing**

- **Unit Tests:** Individual function behaviour for all PivotBot entry points
- **Integration Tests:** Full fork tests against Base mainnet state (Moonwell + Aerodrome + Balancer live contract state)
- **Fuzz Tests:** Randomised input coverage on leverage parameters, token amounts, and swap paths

### **Audit**

Security audit commissioned **Q2 2026**. Results will be published publicly.

---

## Competitive Advantage

| Feature                         | PivotBot | Gearbox   | Alpaca | Euler     | DeFi Perps |
| ------------------------------- | -------- | --------- | ------ | --------- | ---------- |
| **Non-custodial**               | ✓        | ⚠ Partial | ✗      | ✓         | ✗          |
| **Zero funding rate**           | ✓        | ✓         | ✓      | ✓         | ✗          |
| **Earn yield while leveraged**  | ✓        | ⚠ Partial | ✓      | ✓         | ✗          |
| **Autonomous agent guardian**   | ✓        | ✗         | ✗      | ✗         | ✗          |
| **Intent-based strategy**       | ✓        | ✗         | ✗      | ✗         | ✗          |
| **Atomic no-loop execution**    | ✓        | ✗         | ✗      | ⚠ Partial | ✗          |
| **Built on Base (Coinbase L2)** | ✓        | ✗         | ✗      | ✗         | ⚠ Partial  |
| **No token required to use**    | ✓        | ✗         | ✓      | ✗         | ✓          |

---

## Market Opportunity

PivotBot sits at the intersection of DeFi's two largest and fastest-growing segments:

### **DeFi Market Growth**

- **Total DeFi TVL:** $130–140B (Early 2026)
- **DeFi CAGR (2026–2030):** 43.3%
- **Projected by 2030:** $256B
- **Yield Farming Revenue:** 36.5% of all DeFi revenue

### **Base Chain Leadership**

- **46.6%** of all Layer 2 DeFi TVL
- **$5.6B** peak TVL in 2025; **$4B+** current
- **+180%** daily active addresses YoY (Q1 2026)
- **$580M/day** Aerodrome DEX volume
- **#3** Moonwell — lending protocol on Base

### **Target Segments**

| Segment        | TAM                                      |
| -------------- | ---------------------------------------- |
| Yield Farmers  | $89B in DeFi lending TVL                 |
| DeFi Traders   | 20M+ unique DeFi users (2025)            |
| DAO Treasuries | 11.5% of DeFi TVL (institutional)        |
| LST Holders    | cbETH, wstETH, rETH seeking yield        |
| Agent Builders | CDP AgentKit ecosystem (rapidly growing) |

---

## Roadmap

### **Q1 2026** — ✓ LIVE

- ✓ Contracts deployed on Base mainnet
- ✓ +44.97% APY demonstrated
- ✓ Balancer forum post published
- ✓ Live at syncedgesolutions.xyz/pivot

### **Q2 2026** — Building

- CDP AgentKit guardian integration
- Intent panel + agent reasoning UI
- PivotProPass NFT subscription launch
- Security audit commissioned

### **Q3 2026** — Scale

- Audit complete & published
- Full agent auto-deleverage live
- B2B white-label launched
- Unichain / Optimism deployment

### **Q4 2026+** — Institutional

- Institutional API access
- Multi-chain TVL $10M target
- Arbitrum One deployment
- Governance framework
- CEX/CeFi partnership integrations

---


## Documentation

- **[Technical Overview](./docs/technical-overview.md)** — Deep dive into contracts, reasoning engine, and guardian architecture
- **Smart Contract Code** — Available on [Basescan](https://basescan.org)
- **Live App** — [syncedgesolutions.xyz/pivot](https://syncedgesolutions.xyz/pivot)

---

## Key Data & References

### **On-Chain Verification**

All contracts are **verifiable and live on Basescan**:

- [PivotBot Contract](https://basescan.org/address/0x2d6781c28d77f8a446d9fa8d2ad421be9aa465e3)
- [Factory Contract](https://basescan.org/address/0xc4E537890e86fDD44aF936218f80d7326820d97d)
- [Manager Contract](https://basescan.org/address/0x144F04807a6af905E3112Fe3Da9302D308c6DF26)
- [Recent Tx](https://basescan.org/tx/0xdc42587932cb065c44843ad8226fb65db4d7f6e0b6b5cd2f9f9709b67a67a476)

### **Live Position Data (as of April 9, 2026)**

- **Strategy:** cbETH/wstETH (Delta Neutral)
- **Net APY:** +44.97%
- **Supply APY (cbETH):** 11.69%
- **Borrow APY (wstETH):** 1.02%
- **Leverage:** 4×
- **Health Factor:** 1.33 ✓

### **External Resources**

- Moonwell Base Markets — [app.moonwell.fi](https://app.moonwell.fi)
- Balancer V2 Vault — [docs.balancer.fi](https://docs.balancer.fi/reference/contracts/vault)
- Aerodrome V2 Router — [aerodrome.finance](https://aerodrome.finance/docs)
- CDP AgentKit — [developer.coinbase.com/agentkit](https://developer.coinbase.com/agentkit)
- Compound V2 Risk Model — [docs.compound.finance](https://docs.compound.finance)

---

## Legal & Disclaimers

**PivotBot is a non-custodial protocol.** Users retain custody of funds at all times via the per-user bot architecture. Funds are never held by any shared pool or company wallet.

**This document is for informational purposes only and does not constitute financial advice.** DeFi yield strategies carry risk including but not limited to: smart contract risk, liquidation risk, market risk, and slippage. Always conduct your own research before investing.

---

## Contact & Support

**Founder:** Abolaji M. Adedeji  
**Email:** abolaji@syncedgesolutions.xyz  
**Live App:** https://syncedgesolutions.xyz/pivot  

---

*© 2026 Syncedge Solutions · All rights reserved*
