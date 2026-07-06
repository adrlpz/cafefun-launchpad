# RobinLaunch — PRD (Product Requirements Document)

**Version:** 1.0  
**Date:** 2026-07-06  
**Author:** Fizz / SUPERAGENT IRONCLAW  
**Status:** Draft  
**Target Chain:** Robinhood Chain (ETH) — chainId 4663  
**Inspired By:** RobinFun (robinfun.live) — fair-launch bonding curve token launchpad

---

## 1. Executive Summary

### Problem Statement
Token launches today are dominated by insiders — presales, team allocations, and private rounds concentrate supply and information with a few, leaving retail buyers at a disadvantage. Liquidity rugs and honeypots kill trust in new projects. Existing fair-launch solutions are trapped on congested layer-1s with high fees.

### Proposed Solution
**RobinLaunch** is a permissionless, fair-launch token launchpad native to Robinhood Chain. Anyone deploys a token in one transaction. Every token starts trading immediately on a transparent bonding curve — no presale, no team allocation, no insider rounds. When a token reaches its graduation threshold, it auto-migrates to a public Uniswap V2 pool with 100% of LP tokens burned permanently. Built-in livestream + chat per token for community engagement. CTO (Community Takeover) mechanism for abandoned projects.

### Success Criteria (KPIs)

| KPI | Target | Measurement |
|---|---|---|
| Tokens launched in first 30 days | ≥ 100 | On-chain factory counter |
| Total graduation rate | ≥ 15% | Graduated / Total launched |
| Cumulative trading volume (30d) | ≥ 50,000 ETH | Volume from curve trades + DEX trades |
| Protocol treasury collected (30d) | ≥ 500 ETH | Half of 1% fee volume |
| Frontend uptime | ≥ 99.5% | Uptime monitoring |
| Smart contract exploit value at risk | $0 | Audit + invariant fuzzing pass |

---

## 2. User Experience & Functionality

### 2.1 User Personas

| Persona | Description | Key Need |
|---|---|---|
| **Token Creator** | Anyone with an ETH wallet who wants to launch a token | One-tx deploy, no coding, instant trade |
| **Trader / Speculator** | Retail user looking for early entries on new tokens | Transparent pricing, instant liquidity, no rug |
| **Community Member** | Users who find tokens they believe in | Buy/sell on curve, watch livestreams, chat |
| **CTO Initiator** | Community member claiming an abandoned token | Submit CTO request, redirect creator fees |

### 2.2 User Stories

#### Launch a Token
> **As a** token creator, **I want to** deploy a token in a single transaction **so that** my token is immediately tradable on a fair bonding curve.

**Acceptance Criteria:**
- Creator connects wallet (MetaMask/Rainbow/WalletConnect)
- Clicks "Launch a Token" → enters token name, symbol, optional image/livestream
- Signs one transaction → token created with deterministic CREATE2 address ending in `...4663`
- Token immediately appears on explore page with live curve stats
- Zero coding or technical knowledge required

#### Trade on the Bonding Curve
> **As a** trader, **I want to** buy and sell tokens on a transparent bonding curve **so that** I get the same price as everyone else with no slippage manipulation.

**Acceptance Criteria:**
- Buy: user enters ETH amount → contract calculates tokens at current curve price + 1% fee
- Sell: user enters token amount → contract calculates ETH return at current curve price − 1% fee
- Price updates deterministically on each trade (constant product curve x·y = k)
- Starting virtual market cap: $4,000 USD (snapshotted to ETH at creation)
- 1% fee split: 0.5% creator, 0.5% protocol treasury
- Fee always collected in ETH

#### Token Graduation
> **As a** trader, **I want** tokens to auto-migrate to a DEX once the curve fills **so that** the token gets permanent, locked liquidity.

**Acceptance Criteria:**
- Graduation trigger: $9,300 USD raised on curve (≈ ~$44k MC graduation)
- Permissionless graduation call (anyone can trigger)
- 300M tokens (30%) + raised ETH → deposited into Uniswap V2 pool
- LP tokens burned to 0xdead (100% locked forever)
- Post-graduation: 1% fee continues via token contract, auto-swapped to ETH when ≥$5 accumulated
- Graduation hardened against front-running (direct-to-pair LP provisioning)

#### Community Takeover (CTO)
> **As a** community member, **I want** to claim the creator fee on an abandoned token **so that** the community benefits from trading volume instead of an inactive creator.

**Acceptance Criteria:**
- Any wallet submits CTO request from token page (off-chain signature, no gas)
- Provides: new fee destination wallet, Telegram / X contact
- RobinFun team reviews manually (RobinLaunch team reviews)
- On approval: owner-restricted function reassigns creator fee to new wallet
- Only applies to tokens launched on the current factory

#### Livestream per Token
> **As a** creator, **I want** to livestream from my token's page **so that** I can pitch my project live.

**Acceptance Criteria:**
- Creator can go live from browser (WebRTC)
- Viewers see video + live viewer count (no camera/mic for viewers)
- Live chat for wallet-connected users (off-chain sign, no gas)
- Chat history persisted on page
- Creator can delete messages and ban wallets

#### Token Exploration
> **As a** user, **I want** to browse all tokens **so that** I can find new opportunities.

**Acceptance Criteria:**
- Explore page: cards sorted by recency, volume, market cap
- Live trade feed: buy/sell events flash and push card to top
- Each token page: curve chart, stats (holders, MC, volume, price), trade interface, live stream, chat

### 2.3 Non-Goals (v1.0)

- **No leverage trading** — spot-only on bonding curve
- **No vesting/cliff schedules** — tokens are 100% liquid from launch
- **No whitelist/KYC** — fully permissionless
- **No custom token parameters** — all tokens share same fixed supply & curve (v1)
- **No mobile native app** — responsive web only
- **No governance token** — protocol is fee-driven, no $OXILAUNCH token (v1)
- **No cross-chain** — ETH-only for v1

---

## 3. Technical Specifications

### 3.1 Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│                     Frontend (React + Next.js)              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ Explore   │  │ Token    │  │ Create   │  │ Live      │ │
│  │ Page      │  │ Detail   │  │ Token    │  │ Stream    │ │
│  └──────────┘  └──────────┘  └──────────┘  └───────────┘ │
│         └──────────── wagmi/viem ────────────┘             │
└──────────────────────────┬─────────────────────────────────┘
                           │ RPC
┌──────────────────────────▼─────────────────────────────────┐
│                 Robinhood Chain (ETH, chainId 4663)             │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Smart Contracts (Solidity)               │  │
│  │  ┌──────────────┐  ┌──────────────┐                 │  │
│  │  │ RobinFactory    │  │ RobinToken     │ (CREATE2)       │  │
│  │  │ (proxy+impl)  │  │ - Bonding    │ per token       │  │
│  │  │               │  │   Curve      │                 │  │
│  │  │ - createToken │  │ - buy/sell   │                 │  │
│  │  │ - graduate    │  │ - graduate   │                 │  │
│  │  │ - setFeeDest  │  │ - fee accrue │                 │  │
│  │  └──────────────┘  └──────────────┘                 │  │
│  │                                                      │  │
│  │  ┌──────────────┐  ┌──────────────────────────────┐ │  │
│  │  │ Uniswap V2   │  │ RobinFeeCollector              │ │  │
│  │  │ Pool (after  │  │ - auto-swap ETH fees         │ │  │
│  │  │ graduation)  │  │ - distribute 50/50           │ │  │
│  │  └──────────────┘  └──────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────┐
│                  Backend Services (Node.js/TypeScript)       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Indexer      │  │ WebRTC       │  │ Chat Server  │     │
│  │ (event logs) │  │ (LiveKit/SFU) │  │ (WebSocket)  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│  ┌──────────────┐  ┌──────────────┐                       │
│  │ API (REST)   │  │ DB (Postgres)│                       │
│  │ - tokens     │  │ - tokens     │                       │
│  │ - stats      │  │ - trades    │                       │
│  │ - chat       │  │ - chat msgs │                       │
│  └──────────────┘  └──────────────┘                       │
└────────────────────────────────────────────────────────────┘
```

### 3.2 Smart Contract Design

#### RobinFactory (Upgradeable)

| Aspect | Detail |
|---|---|
| Pattern | Minimal proxy (EIP-1167) with CREATE2 for deterministic child addresses |
| Child address | Vanity ending in `...4663` (chainId) via CREATE2 salt |
| State | Maps token address → TokenInfo (creator, feeDest, graduated, etc.) |
| Key functions | `createToken(name, symbol, imageURI)` → deploys RobinToken |
| | `graduate(address token)` → permissionless graduation trigger |
| | `setCreatorFeeDest(address token, address newDest)` → owner-only CTO |
| | `upgradeTo(address newImpl)` → UUPS upgradeable |
| Security | ReentrancyGuard, ownable, pausable |

#### RobinToken (Individual per token, created via factory)

| Parameter | Value |
|---|---|
| Total supply | 1,000,000,000 (1B) |
| Curve allocation | 700,000,000 (70%) |
| Liquidity reserve | 300,000,000 (30%) |
| Starting virtual MC | $4,000 USD (snapshotted to ETH at creation) |
| Graduation raise | $9,300 USD |
| Graduation MC | ~$44,127 USD |
| Curve type | Constant product with virtual reserves (x·y = k) |
| Trade fee | 1% flat (0.5% creator, 0.5% protocol) |

**Curve math:**
```
Virtual reserve invariant: x · y = k
where x = virtual ETH reserve, y = token reserve (70% of supply)
Starting price = x / y (denominated in ETH per token)

On buy (input ΔETH):
  Δtokens = y - k / (x + ΔETH)  // tokens received

On sell (input Δtokens):
  ΔETH = x - k / (y + Δtokens)  // ETH received

Fee: 1% of ΔETH taken on entry (buy) or exit (sell)
```

**Graduation logic:**
```
1. Verify cumulative ETH raised ≥ graduationThreshold
2. Take reserved tokens (300M) + raised ETH (minus treasury fee)
3. Create Uniswap V2 pair if not exists
4. Add liquidity: raisedETH * 0.97 (300 ETH to treasury) + 300M tokens
5. Mint LP tokens → burn to 0xdead
6. Mark token as graduated
7. After graduation: 1% fee collected in contract, auto-swapped to ETH when ≥$5
```

#### RobinFeeCollector

| Aspect | Detail |
|---|---|
| Role | Receives protocol fee share from all tokens |
| Distribution | Accumulates ETH, can be withdrawn by protocol multisig |
| Post-graduation | Receives swapped ETH from graduated tokens' fee accumulation |

### 3.3 Integration Points

| Integration | Purpose | Tech |
|---|---|---|
| Wallet Connection | User auth for dapp | Wagmi + Viem + RainbowKit / WalletConnect |
| RPC | Read/write chain state | ETH RPC (https://rpc.robinhoodchain.com) |
| Blockscout | Transaction explorer links | https://robinhoodchain.blockscout.com |
| DEX | Post-graduation trading | Uniswap V2 fork on ETH |
| LiveKit / SFU | WebRTC livestream | LiveKit open-source or custom SFU |
| Postgres | Token metadata, trades, chat history | pg + drizzle orm |
| Redis | Caching, rate limiting, job queues | ioredis + bullmq |
| IPFS / Arweave | Token images/metadata | Pinata or web3.storage |

### 3.4 Security & Privacy

| Concern | Mitigation |
|---|---|
| Reentrancy | OpenZeppelin ReentrancyGuard on all buy/sell/graduate functions |
| Per-curve isolation | Each RobinToken is a separate contract; funds are **never** commingled |
| Graduation front-running | Direct-to-pair LP seeding — no intermediate state to exploit |
| Oracle manipulation | No external oracle needed — curve price is deterministic from reserves |
| Fee payout bricking | Best-effort fee distribution (failure never blocks trades) |
| Upgrade authority | Factory upgradeable via timelock (≥48h delay) |
| Creator misbehavior | CTO mechanism allows community to reclaim abandoned project fees |
| RPC spam | Rate limiting on API endpoints |
| Chat moderation | Creator can delete messages + ban wallets from their stream |

### 3.5 AI / Livestream Requirements

| Component | Requirements |
|---|---|
| WebRTC infrastructure | LiveKit server (self-hosted or cloud) for SFU + selective forwarding |
| Stream recording | Optional — store last N minutes for replay |
| Chat | WebSocket server (Socket.IO or LiveKit chat) with off-chain wallet sig for auth |
| Moderation | Creator-controlled mute/ban list; profanity filter optional |

---

## 4. Tokenomics

### 4.1 Per-Token Economics (Fixed, non-configurable in v1)

```
Total Supply:   1,000,000,000 (100%)
├── Curve Sale:  700,000,000 (70%)  — sold on bonding curve
└── LP Reserve:  300,000,000 (30%)  — paired with raised ETH at graduation

Starting Virtual MC:   $4,000 USD (≈ ~X ETH at creation rate)
Graduation Raise:      $9,300 USD
Graduation MC:         ≈ $44,127 USD
Trade Fee:             1% (0.5% creator | 0.5% protocol)
```

### 4.2 Protocol Treasury

- Collects 0.5% of every trade (50% of total 1% fee)
- Pre-graduation: collected in ETH directly from curve trades
- Post-graduation: collected in ETH via auto-swap mechanism in token contract
- Treasury funds: protocol ops, marketing, future development
- Managed by multisig (≥2/3 signers)

### 4.3 Creator Economics

- Earns 0.5% of every trade (50% of total 1% fee) for life of the token
- Pre-graduation: ETH sent directly to creator wallet on each curve trade
- Post-graduation: ETH auto-swapped and sent to creator wallet
- If creator goes inactive: CTO → fee goes to community wallet instead

---

## 5. Risks & Roadmap

### 5.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Smart contract bug | Low | Critical | Extensive fuzz testing, invariant testing, external audit |
| Graduation front-running | Medium | Medium | Direct-to-pair LP provisioning design |
| RPC congestion on Robinhood Chain | Low | Medium | Rate limiting, fallback RPCs |
| WebRTC scalability | Medium | Low | Use LiveKit cloud with auto-scaling |
| Sybil attack on chat | High | Low | Off-chain wallet sig per message, creator ban list |
| Robinhood Chain market volatility | Medium | Low | Graduation threshold snapshotted in ETH at token creation |

### 5.2 Phased Rollout

#### Phase 1 — MVP (Weeks 1-6)

| Milestone | Deliverables |
|---|---|
| Week 1-2 | Core contracts: RobinFactory + RobinToken (bonding curve, buy/sell, fee accrual) |
| Week 3 | Graduation + Uniswap V2 migration + LP burn |
| Week 4 | Frontend: create token page, explore page, token detail + trade UI |
| Week 5 | Integration: wallet connect, curve chart, live trade feed |
| Week 6 | Testnet deploy, internal QA, invariant fuzzing |

#### Phase 2 — Launch (Weeks 7-8)

| Milestone | Deliverables |
|---|---|
| Week 7 | External audit (Pessimistic / Code4rena / individual) |
| Week 8 | Mainnet deploy on ETH, public launch, marketing push |

#### Phase 3 — Community Features (Weeks 9-12)

| Milestone | Deliverables |
|---|---|
| Week 9-10 | Livestream integration (LiveKit) + per-token chat |
| Week 11-12 | CTO mechanism, token page improvements, analytics dashboard |

#### Phase 4 — v1.1 & Beyond

| Feature | Description |
|---|---|
| Multi-token curve types | Graduated price, Dutch auction, linear curve |
| Token creator dashboard | Analytics, fee history, stream management |
| Referral system | Share token pages → earn percentage of fees |
| Mobile-responsive redesign | Full mobile web experience |
| Explorer integration | Direct Blockscout links from token page |

### 5.3 Go-to-Market

| Channel | Activity |
|---|---|
| Robinhood Chain community | Partner with existing ETH ecosystem projects |
| Telegram / X | Launch announcement, creator testimonials, graduation alerts |
| Fee rebates | First 50 tokens: 100% creator fee share (instead of 50%) |
| Bug bounty | $5k ETH bounty for critical vulnerabilities post-audit |

---

## 6. Dependencies

| Dependency | Version | Justification |
|---|---|---|
| Solidity | ^0.8.24 | Modern compiler with built-in overflow checks |
| Foundry | Latest | Fast compilation, fuzz testing, forge script deploy |
| OpenZeppelin | v5.x | ReentrancyGuard, Ownable, UUPS upgradeable |
| Uniswap V2 Core | v1.0.1 | DEX pairing + liquidity pool |
| Uniswap V2 Periphery | v1.1.1 | Router for swap + addLiquidity |
| Wagmi | v2.x | React hooks for wallet + contract interaction |
| Viem | v2.x | TypeScript-first ethers alternative |
| RainbowKit | Latest | Wallet connection UI |
| Next.js | 14+ | Frontend framework |
| LiveKit | Latest | WebRTC SFU for livestream |
| PostgreSQL | 15+ | Persistent storage for indexer |
| Redis | 7+ | Caching + job queues |

---

## 7. Glossary

| Term | Definition |
|---|---|
| Bonding Curve | Deterministic pricing function where price increases with demand |
| Graduation | Migration from bonding curve to permanent Uniswap V2 pool |
| Virtual Reserves | Initial liquidity provided by contract math, not real tokens/ETH |
| CTO | Community Takeover — reassigns creator fee to community wallet |
| LP Token | Liquidity Provider token representing share of DEX pool |
| SFU | Selective Forwarding Unit — WebRTC server for multi-party video |

---

*End of PRD — v1.0*
