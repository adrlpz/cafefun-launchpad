# CafeFun — Implementation Plan

**Version:** 1.0  
**Date:** 2026-07-06  
**Stack:** Solidity (Foundry) + TypeScript (Backend/API) + React/Next.js (Frontend)  
**Target Chain:** Robinhood Chain (ETH) — chainId 4663  
**Timeline:** 12 weeks to Mainnet Launch  

---

## Table of Contents

1. [Phase 0: Foundation & Tooling](#phase-0-foundation--tooling)
2. [Phase 1: Core Smart Contracts](#phase-1-core-smart-contracts)
3. [Phase 2: Frontend Dapp](#phase-2-frontend-dapp)
4. [Phase 3: Backend Services](#phase-3-backend-services)
5. [Phase 4: Testing & Security](#phase-4-testing--security)
6. [Phase 5: Deployment & Launch](#phase-5-deployment--launch)
7. [Phase 6: Post-Launch Features](#phase-6-post-launch-features)
8. [Architecture Reference](#architecture-reference)
9. [File Tree](#file-tree)
10. [Key Decisions Log](#key-decisions-log)

---

## Phase 0: Foundation & Tooling (Week 1)

### 0.1 Repository Setup

```bash
# Create monorepo
mkdir oxi-launchpad && cd oxi-launchpad
git init
mkdir contracts frontend backend docs scripts

# Smart contracts
cd contracts
forge init --force
forge install OpenZeppelin/openzeppelin-contracts@v5.0.2
forge install transmissions11/solmate
forge install PaulRBerg/prb-math  # for fixed-point math
forge install foundry-rs/forge-std

# Frontend
cd ../frontend
npx create-next-app@latest . --typescript --tailwind --eslint --app
npm i viem wagmi @rainbow-me/rainbowkit
npm i recharts @tanstack/react-query lucide-react clsx
npm i @livekit/components-react livekit-client

# Backend
cd ../backend
npm init -y
npm i typescript tsx @types/node
npm i viem
npm i express cors helmet
npm i pg drizzle-orm drizzle-kit
npm i ioredis bullmq
npm i livekit-server-sdk
npm i socket.io @types/socket.io
```

### 0.2 CI/CD (GitHub Actions)

Create `.github/workflows/` with:

| Workflow | Trigger | Actions |
|---|---|---|
| `contracts.yml` | PR/merge to `contracts/` | `forge build`, `forge test`, `forge snapshot`, slither |
| `frontend.yml` | PR/merge to `frontend/` | `npm run lint`, `npm run build`, `npm run test` |
| `backend.yml` | PR/merge to `backend/` | `tsc --noEmit`, `npm run test` |
| `deploy.yml` | Tag `v*` | Forge deploy to Robinhood Chain mainnet + verify on Blockscout |
| `audit.yml` | Weekly | Slither + solhint + dependency audit |

### 0.3 Development Environment

```bash
# Local ETH fork via Anvil
anvil --fork-url https://rpc.robinhoodchain.com --chain-id 4663 --port 8545

# Deploy Uniswap V2 on local fork (if not on ETH yet)
forge script script/DeployUniswapV2.s.sol --rpc-url http://localhost:8545 --broadcast

# Set env
cp .env.example .env
# Fill in: RPC_URL, DEPLOYER_PK, ETHERSCAN_API_KEY (Blockscout), etc.
```

### 0.4 Project Structure Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Solidity version | ^0.8.24 | Latest stable, overflow checks built-in |
| Contract upgrade | UUPS (OpenZeppelin) | Less gas than transparent proxy; factory may need upgrades |
| Factory pattern | Minimal proxy + CREATE2 | Cheaper deploys, deterministic addresses |
| Frontend framework | Next.js 14 App Router | SSR for token pages, SEO, fast builds |
| Wallet connection | RainbowKit + Wagmi | Best UX, multi-wallet support |
| Indexer | Custom (not The Graph) | Full control, lighter infra for ETH; revisit v2 |
| DB | PostgreSQL + Drizzle | Type-safe queries, migrations |
| WebRTC | LiveKit self-hosted | Open-source SFU, low latency, simple API |
| Chat | Socket.IO + Postgres | Real-time, persistent history, simple |
| Deploy target | VPS (systemd) | Existing ETH infra; Docker optional |

---

## Phase 1: Core Smart Contracts (Weeks 2-4)

### 1.1 Contract Architecture

```
contracts/
├── src/
│   ├── CafeFunFactory.sol          # Factory: create tokens, manage protocol
│   ├── CafeFunToken.sol            # Individual token: bonding curve + fee logic
│   ├── CafeFunTreasury.sol     # Protocol treasury
│   ├── interfaces/
│   │   ├── ICafeFunFactory.sol
│   │   ├── ICafeFunToken.sol
│   │   └── ICafeFunTreasury.sol
│   ├── libraries/
│   │   ├── BondingCurveLib.sol  # Pure math for curve calculations
│   │   └── FeeLib.sol          # Fee accrual + distribution logic
│   └── utils/
│       ├── Create2Address.sol  # CREATE2 address derivation
│       └── Constants.sol       # Protocol constants
├── test/
│   ├── unit/
│   │   ├── CafeFunFactory.t.sol
│   │   ├── CafeFunToken.t.sol
│   │   ├── BondingCurveLib.t.sol
│   │   └── FeeLib.t.sol
│   ├── integration/
│   │   ├── CreateAndTrade.t.sol
│   │   ├── Graduation.t.sol
│   │   └── CTO.t.sol
│   ├── fuzz/
│   │   ├── CurveInvariants.t.sol
│   │   └── SolvencyInvariants.t.sol
│   └── adversarial/
│       └── AttackScenarios.t.sol
├── script/
│   ├── DeployCafeFunFactory.s.sol
│   └── UpgradeCafeFunFactory.s.sol
├── foundry.toml
└── remappings.txt
```

### 1.2 CafeFunFactory.sol — Detailed Spec

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {UUPSUpgradeable} from "@oz/contracts/proxy/utils/UUPSUpgradeable.sol";
import {OwnableUpgradeable} from "@oz/contracts/access/OwnableUpgradeable.sol";
import {ReentrancyGuardUpgradeable} from "@oz/contracts/utils/ReentrancyGuardUpgradeable.sol";
import {PausableUpgradeable} from "@oz/contracts/utils/PausableUpgradeable.sol";

contract CafeFunFactory is UUPSUpgradeable, OwnableUpgradeable, ReentrancyGuardUpgradeable, PausableUpgradeable {
    // ── Storage ───────────────────────────────────────
    struct TokenConfig {
        address creator;
        address feeDestination;  // creator fee recipient
        uint256 createdAt;
        bool graduated;
        address uniswapPair;
    }

    address public immutable uniswapV2Router;
    address public immutable weth;       // Wrapped ETH
    address public feeCollector;         // protocol treasury
    uint256 public graduationThreshold; // in ETH (snapshot of $9,300)
    uint256 public curveSupply;          // 70% of 1B
    uint256 public lpReserve;            // 30% of 1B
    uint256 public tradeFeeBps;          // 100 = 1%
    uint256 public protocolFeeShareBps;  // 50 = 50% of tradeFee = 0.5%

    mapping(address => TokenConfig) public tokens;
    address[] public allTokens;

    // ── Events ────────────────────────────────────────
    event TokenCreated(address indexed token, address indexed creator, string name, string symbol, uint256 deployBlock);
    event Graduated(address indexed token, address indexed pair, uint256 liquidity, uint256 ethAmount);
    event FeeDestinationChanged(address indexed token, address indexed oldDest, address indexed newDest);

    // ── Functions ─────────────────────────────────────
    constructor() { _disableInitializers(); }

    function initialize(address _router, address _weth, address _feeCollector) external initializer {
        __UUPSUpgradeable_init();
        __Ownable_init(msg.sender);
        __ReentrancyGuard_init();
        __Pausable_init();
        // set constants
    }

    /// @notice Deploy a new token via CREATE2 with vanity address ending in 4663
    function createToken(string calldata name, string calldata symbol, string calldata imageURI)
        external payable whenNotPaused returns (address token);

    /// @notice Permissionless graduation — anyone can trigger for any token
    function graduate(address token) external nonReentrant;

    /// @notice Owner-only: reassign creator fee destination (CTO)
    function setFeeDestination(address token, address newDest) external onlyOwner;

    /// @notice Calculate CREATE2 address deterministically
    function predictTokenAddress(bytes32 salt) external view returns (address);

    /// @notice Get all tokens
    function getAllTokens() external view returns (address[] memory);

    // UUPS: only owner can upgrade
    function _authorizeUpgrade(address newImpl) internal override onlyOwner;
}
```

### 1.3 Bonding Curve Math — BondingCurveLib.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {UD60x18} from "prb-math/UD60x18.sol";

library BondingCurveLib {
    struct CurveState {
        uint256 virtualOxcReserve;   // x in x·y = k
        uint256 tokenReserve;        // y in x·y = k
        uint256 k;                   // invariant
        uint256 tokensSold;          // cumulative tokens sold from curve
        uint256 ethRaised;           // cumulative ETH raised
    }

    /// @notice Calculate tokens received for a given ETH amount
    /// @dev Implements: Δtokens = y - k / (x + ΔETH)
    function getTokensForOxc(CurveState memory state, uint256 ethIn, uint256 feeBps)
        external pure returns (uint256 tokensOut, uint256 fee);

    /// @notice Calculate ETH received for a given token amount
    /// @dev Implements: ΔETH = x - k / (y + Δtokens)
    function getOxcForTokens(CurveState memory state, uint256 tokensIn, uint256 feeBps)
        external pure returns (uint256 ethOut, uint256 fee);

    /// @notice Get current spot price (ETH per token)
    function getPrice(CurveState memory state) external pure returns (uint256);

    /// @notice Get market cap in ETH
    function getMarketCap(CurveState memory state) external pure returns (uint256);

    /// @notice Initialize curve with virtual reserves
    function initialize(uint256 virtualOxc, uint256 tokenSupply)
        external pure returns (CurveState memory);
}
```

### 1.4 CafeFunToken.sol — Key Logic

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@oz/contracts/token/ERC20/ERC20.sol";
import {ReentrancyGuard} from "@oz/contracts/utils/ReentrancyGuard.sol";
import {BondingCurveLib} from "./libraries/BondingCurveLib.sol";

contract CafeFunToken is ERC20, ReentrancyGuard {
    ICafeFunFactory public factory;
    BondingCurveLib.CurveState public curve;

    // Fee accumulation post-graduation
    uint256 public accumulatedCreatorFee;
    uint256 public accumulatedProtocolFee;
    uint256 public constant FEE_SWAP_THRESHOLD = 0.005 ether; // $5 in ETH terms

    // ── Buy ────────────────────────────────────────────
    function buy() external payable nonReentrant {
        // 1. Take 1% fee from msg.value → split to creator + protocol treasury
        // 2. Calculate tokens from curve
        // 3. Update curve reserves
        // 4. Mint tokens to buyer
        // 5. Emit Buy event
    }

    // ── Sell ───────────────────────────────────────────
    function sell(uint256 tokenAmount) external nonReentrant {
        // 1. Burn tokens from seller
        // 2. Calculate ETH return from curve (minus 1% fee)
        // 3. Update curve reserves
        // 4. Send ETH to seller
        // 5. Emit Sell event
    }

    // ── Graduation ─────────────────────────────────────
    function graduate() external nonReentrant {
        // Only callable from factory
        // 1. Take liquidity reserve tokens + raised ETH
        // 2. Approve Uniswap V2 router
        // 3. Add liquidity → burn LP tokens
        // 4. After graduation: fee accrual mode changes
    }

    // ── Post-Graduation Fee Swap ───────────────────────
    function swapAccruedFees() external {
        // Auto-swap accumulated fees ≥ $5 to ETH
        // Send 50% to creator / feeDestination, 50% to protocol
    }
}
```

### 1.5 Week-by-Week Sprint Plan

#### Week 2: Core Math + Factory

| Day | Task | Deliverable |
|---|---|---|
| Mon | Fork ETH via Anvil, deploy Uniswap V2 locally | Local dev env ready |
| Tue | Write `BondingCurveLib.sol` — pure math functions | Library compiles, unit tests pass |
| Wed | Fuzz test curve math — invariants across 10k random trades | Invariants hold |
| Thu | Write `CafeFunFactory.sol` — creation logic + CREATE2 vanity | Factory deploys tokens |
| Fri | Unit tests: factory + token creation | ≥90% coverage on factory |

#### Week 3: CafeFunToken + Trading

| Day | Task | Deliverable |
|---|---|---|
| Mon | Write `CafeFunToken.sol` — buy/sell with curve + fee accrual | Token trades on curve |
| Tue | Unit tests: buy/sell edge cases (empty curve, max supply, rounding) | All edge cases handled |
| Wed | Integration test: create → buy → sell → verify fees | End-to-end flow tested |
| Thu | Fee distribution logic + `CafeFunTreasury.sol` | Fees split correctly |
| Fri | Integration test: multiple tokens, concurrent trades | Isolation proven |

#### Week 4: Graduation + CTO

| Day | Task | Deliverable |
|---|---|---|
| Mon | Graduation logic: flash-loan-resistant LP provisioning | Graduation works |
| Tue | Unit + fuzz: graduation edge cases (partial fills, race conditions) | Graduation hardened |
| Wed | CTO mechanism + fee destination management | CTO flow tested |
| Thu | Invariant fuzzing: solvency across 5k create/buy/sell/graduate sequences | All invariants hold |
| Fri | Slither + solhint pass, gas snapshot baseline | Contracts audit-ready |

**Invariant Checks (must hold after every sequence):**

| Invariant | Description |
|---|---|
| `curve.k` constant | x·y = k never changes outside of trades |
| Token supply invariant | Total supply = burned + circulating + LP reserve |
| Per-token ETH isolation | One token's curve balance never overlaps another |
| Fee correctness | Cumulative fees = 1% of volume (within rounding) |
| Graduation atomicity | Graduated tokens cannot re-enter curve |
| LP burn final | Graduated LP tokens at 0xdead permanently |

---

## Phase 2: Frontend Dapp (Weeks 4-6)

### 2.1 Component Architecture

```
frontend/
├── app/
│   ├── layout.tsx           # Providers (Wagmi, RainbowKit, QueryClient)
│   ├── page.tsx             # Landing / Hero
│   ├── explore/
│   │   └── page.tsx         # Token explorer (cards, sort, live feed)
│   ├── create/
│   │   └── page.tsx         # Token creation form
│   └── token/
│       └── [address]/
│           ├── page.tsx     # Token detail: chart, trade, stream, chat
│           └── cto.tsx      # CTO request page
├── components/
│   ├── layout/
│   │   ├── Navbar.tsx
│   │   ├── Footer.tsx
│   │   └── WalletConnect.tsx
│   ├── explore/
│   │   ├── TokenCard.tsx
│   │   ├── TokenGrid.tsx
│   │   ├── LiveFeed.tsx
│   │   └── SortFilter.tsx
│   ├── token/
│   │   ├── CurveChart.tsx     # Bonding curve chart (Recharts)
│   │   ├── TradePanel.tsx     # Buy/sell interface
│   │   ├── TokenStats.tsx     # MC, volume, holders, price
│   │   ├── HolderList.tsx     # Top holders (from indexer)
│   │   ├── TradeHistory.tsx   # Recent trades table
│   │   └── CTORequest.tsx     # CTO submission widget
│   ├── stream/
│   │   ├── LiveVideo.tsx      # WebRTC player
│   │   ├── StreamControls.tsx # Go live / stop
│   │   └── LiveChat.tsx       # Chat component
│   └── common/
│       ├── Spinner.tsx
│       ├── Toast.tsx
│       ├── Modal.tsx
│       ├── Pagination.tsx
│       └── CopyButton.tsx
├── hooks/
│   ├── useToken.ts           # Token data + contract calls
│   ├── useCurve.ts           # Curve state + price calculation
│   ├── useStream.ts          # LiveKit connection
│   ├── useChat.ts            # Socket.IO chat
│   └── useCTO.ts             # CTO request submission
├── lib/
│   ├── config.ts             # Wagmi config, chain config
│   ├── contracts.ts          # Contract ABIs + addresses
│   ├── curve.ts              # Client-side curve math
│   └── format.ts             # Number/currency formatting
└── public/
    └── (static assets)
```

### 2.2 Key UI Screens

#### Landing Page (`/`)
- Hero section: "Fair-launch bonding curves on Robinhood Chain"
- Stats bar: total tokens launched, total volume, total graduated
- Featured/latest tokens (top 3 by volume)
- CTA: "Launch a Token" + "Explore Tokens"

#### Explore Page (`/explore`)
- Grid of token cards with live price, MC, volume, 24h change
- Sort: newest, highest volume, highest MC, about to graduate
- Live feed sidebar: recent buy/sell events with flash animation
- Search bar (by name/symbol/address)
- Pagination (20 per page)

#### Create Token (`/create`)
- Form: name, symbol, image upload (IPFS)
- Preview: address, curve stats
- Gas estimate + network info (ETH)
- "Create" button → sends tx → redirects to token page

#### Token Detail (`/token/[address]`)
- Left: CurveChart (interactive, shows price curve + current position)
- Middle: TradePanel (buy amount → tokens, sell amount → ETH, 1% fee shown)
- Below: TradeHistory table, Top Holders
- Right sidebar: TokenStats + Creator info
- Bottom section: LiveVideo (if creator streaming) + LiveChat

#### CTO Page (`/token/[address]/cto`)
- Shows current creator fee destination
- Form: new wallet address, Telegram/X links
- "Submit CTO Request" (off-chain signature)

### 2.3 Smart Contract Interaction Layer

```typescript
// lib/config.ts
import { http, createConfig } from 'wagmi';
import { oxiChain } from './chain';  // ETH chain definition

export const wagmiConfig = createConfig({
  chains: [oxiChain],
  transports: {
    [oxiChain.id]: http('https://rpc.robinhoodchain.com'),
  },
});

// lib/curve.ts — client-side curve math
export function calculateTokensForOxc(
  virtualOxcReserve: bigint,
  tokenReserve: bigint,
  ethIn: bigint,
  feeBps: number
): { tokensOut: bigint; fee: bigint } {
  const k = virtualOxcReserve * tokenReserve;
  const fee = (ethIn * BigInt(feeBps)) / 10000n;
  const ethAfterFee = ethIn - fee;
  const newVirtualOxc = virtualOxcReserve + ethAfterFee;
  const newTokenReserve = k / newVirtualOxc;
  const tokensOut = tokenReserve - newTokenReserve;
  return { tokensOut, fee };
}

// hooks/useToken.ts
export function useToken(address: `0x${string}`) {
  const tokenContract = {
    address,
    abi: CafeFunTokenABI,
  };

  const { data: curveState } = useReadContract({
    ...tokenContract,
    functionName: 'curve',
  });

  const { data: isGraduated } = useReadContract({
    ...tokenContract,
    functionName: 'graduated',
  });

  const { writeContract: buy } = useWriteContract();

  // ...
}
```

---

## Phase 3: Backend Services (Weeks 5-7)

### 3.1 Architecture

```
backend/
├── src/
│   ├── index.ts               # Entry: Express + Socket.IO
│   ├── config.ts              # Env vars, constants
│   ├── db/
│   │   ├── schema.ts          # Drizzle schema
│   │   ├── migrations/        # Auto-generated migrations
│   │   └── index.ts           # DB client
│   ├── indexer/
│   │   ├── indexer.ts         # Event log parser
│   │   ├── handlers.ts        # Event → DB transformers
│   │   └── scheduler.ts       # Cron for missed blocks
│   ├── api/
│   │   ├── tokens.ts          # GET /api/tokens, GET /api/tokens/:address
│   │   ├── trades.ts          # GET /api/trades (by token)
│   │   ├── stats.ts           # GET /api/stats (global)
│   │   └── cto.ts             # POST /api/cto/request, GET /api/cto/requests
│   ├── chat/
│   │   ├── server.ts          # Socket.IO chat handler
│   │   ├── auth.ts            # Wallet signature verification
│   │   └── moderation.ts      # Ban/mute lists
│   ├── stream/
│   │   └── livekit.ts         # LiveKit token generation
│   └── queue/
│       ├── index.ts           # BullMQ setup
│       └── workers/
│           ├── feeSwap.ts      # Post-graduation fee swap jobs
│           └── indexer.ts      # Indexer retry queue
├── drizzle.config.ts
├── tsconfig.json
└── package.json
```

### 3.2 Database Schema (Postgres + Drizzle)

```typescript
// db/schema.ts

// ── Tokens (from indexer) ──
export const tokens = pgTable('tokens', {
  address:       text('address').primaryKey(),
  creator:       text('creator').notNull(),
  name:          text('name').notNull(),
  symbol:        text('symbol').notNull(),
  deployTx:      text('deploy_tx').notNull(),
  deployBlock:   integer('deploy_block').notNull(),
  createdAt:     timestamp('created_at').defaultNow().notNull(),
  imageURI:      text('image_uri'),
  graduated:     boolean('graduated').default(false),
  graduationTx:  text('graduation_tx'),
  marketCap:     text('market_cap'),       // latest (bigint as string)
  price:         text('price'),
  volume24h:     text('volume_24h'),
  holders:       integer('holders'),
  lastTradeAt:   timestamp('last_trade_at'),
});

// ── Trades (from indexer) ──
export const trades = pgTable('trades', {
  id:            serial('id').primaryKey(),
  tokenAddress:  text('token_address').references(() => tokens.address),
  type:          text('type', { enum: ['buy', 'sell'] }).notNull(),
  trader:        text('trader').notNull(),
  tokenAmount:   text('token_amount').notNull(),
  ethAmount:     text('eth_amount').notNull(),
  price:         text('price').notNull(),
  txHash:        text('tx_hash').notNull().unique(),
  blockNumber:   integer('block_number').notNull(),
  timestamp:     timestamp('timestamp').defaultNow().notNull(),
});

// ── Chat Messages ──
export const chatMessages = pgTable('chat_messages', {
  id:            serial('id').primaryKey(),
  tokenAddress:  text('token_address').notNull(),
  sender:        text('sender').notNull(),  // wallet address
  message:       text('message').notNull(),
  signature:     text('signature').notNull(), // EIP-191 off-chain sig
  createdAt:     timestamp('created_at').defaultNow().notNull(),
  deleted:       boolean('deleted').default(false),
});

// ── CTO Requests ──
export const ctoRequests = pgTable('cto_requests', {
  id:              serial('id').primaryKey(),
  tokenAddress:    text('token_address').notNull(),
  requester:       text('requester').notNull(),
  newFeeDest:      text('new_fee_dest').notNull(),
  telegramContact: text('telegram_contact'),
  xContact:        text('x_contact'),
  status:          text('status', { enum: ['pending', 'approved', 'rejected'] }).default('pending'),
  signature:       text('signature').notNull(),
  createdAt:       timestamp('created_at').defaultNow().notNull(),
});
```

### 3.3 Indexer Design

```
                     ┌──────────────┐
                     │  ETH RPC     │
                     └──────┬───────┘
                            │ polling every 2s (or newHeads subscription)
                            ▼
                    ┌───────────────┐
                    │  Scheduler    │
                    │  (BullMQ)     │
                    └───────┬───────┘
                            │ enqueue block range
                            ▼
                    ┌───────────────┐
                    │  Indexer      │
                    │  Worker       │
                    └───────┬───────┘
                            │ parse logs & events
                            ▼
                    ┌───────────────┐
                    │  Event        │
                    │  Handlers     │
                    └───────┬───────┘
                            │ upsert DB records
                            ▼
                    ┌───────────────┐
                    │  PostgreSQL   │
                    └───────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  REST API     │
                    │  (for FE)     │
                    └───────────────┘
```

**Events to index:**

| Event | Source Contract | Fields |
|---|---|---|
| `TokenCreated` | CafeFunFactory | token, creator, name, symbol |
| `Buy` | CafeFunToken | trader, tokenAmount, ethAmount, price |
| `Sell` | CafeFunToken | trader, tokenAmount, ethAmount, price |
| `Graduated` | CafeFunFactory | token, pair, liquidity, ethAmount |
| `FeeDestinationChanged` | CafeFunFactory | token, oldDest, newDest |
| `Transfer` | CafeFunToken | from, to, value → holder tracking |

### 3.4 Chat Server Design

```
WebSocket (Socket.IO) flow:
1. User connects → requests to join room `/token/:address`
2. User sends message → server verifies off-chain EIP-191 signature
   Signed message: "CafeFun Chat\nToken: {address}\nMessage: {text}\nNonce: {nonce}"
3. Server validates signature against sender wallet address
4. If valid: store in DB + broadcast to room
5. Creator can send `ban_wallet` event → server adds to ban list
6. Creator can send `delete_message` → server soft-deletes
```

---

## Phase 4: Testing & Security (Weeks 6-8)

### 4.1 Smart Contract Testing Strategy

#### Unit Tests

| File | Tests |
|---|---|
| `CafeFunFactory.t.sol` | Creating tokens, edge cases (duplicate salt, zero address), fee config |
| `CafeFunToken.t.sol` | Buy/sell at various curve positions, fee calculation rounding |
| `BondingCurveLib.t.sol` | Math correctness across price range, overflow safety |
| `FeeLib.t.sol` | Fee split accuracy, cumulative accounting |

#### Integration Tests

| Test | Scenario |
|---|---|
| `CreateAndTrade.t.sol` | Create → Buy multiple → Sell partial → Verify all balance changes + fees |
| `Graduation.t.sol` | Fill curve → Graduate → Verify Uniswap pool creation → Verify LP burned → Trade on DEX |
| `CTO.t.sol` | Create → Request CTO → Approve → Verify fee redirect |

#### Fuzz + Invariant Tests

| Test | Approach |
|---|---|
| `CurveInvariants.t.sol` | `x * y == k` invariant, ghost variable tracking, handler-based fuzzing |
| `SolvencyInvariants.t.sol` | `address(this).balance == sum(curveBalance_i + collectedFees_i)` across all tokens |
| `Adversarial.t.sol` | Reentrancy attempts, front-running graduation, griefing (dust trades) |

#### Gas Benchmarks

| Operation | Target Gas | Measurement Tool |
|---|---|---|
| Token create | ≤ 500k | `forge snapshot` |
| Buy (curve active) | ≤ 100k | `forge snapshot` |
| Sell (curve active) | ≤ 100k | `forge snapshot` |
| Graduate | ≤ 300k | `forge snapshot` |
| Post-grade swap fees | ≤ 200k | `forge snapshot` |

### 4.2 Security Checklist

| # | Item | Status |
|---|---|---|
| 1 | Reentrancy guard on all mutative functions | ☐ |
| 2 | CEI (Checks-Effects-Interactions) pattern throughout | ☐ |
| 3 | No delegatecall to user-controlled addresses | ☐ |
| 4 | Ownable/OwnableUpgradeable for admin functions | ☐ |
| 5 | Timelock (≥48h) on factory upgrade | ☐ |
| 6 | CREATE2 salt not user-controllable (prevents address squatting) | ☐ |
| 7 | Fee total never exceeds 100% (integer overflow checked) | ☐ |
| 8 | Graduation: cannot double-graduate | ☐ |
| 9 | Graduation: flash loan resistant LP provisioning | ☐ |
| 10 | Per-curve ETH isolation (one token ≠ another's balance) | ☐ |
| 11 | Best-effort fee payout (failure never blocks trades) | ☐ |
| 12 | All external calls guarded (try-catch or gas limit) | ☐ |
| 13 | Slither: no high-severity findings | ☐ |
| 14 | Solhint: no errors | ☐ |
| 15 | External audit (≥2 independent reviewers) | ☐ |
| 16 | Bug bounty program live at launch | ☐ |

### 4.3 Testing Commands

```bash
# Unit + integration
forge test --match-path "test/unit/*" -vvv
forge test --match-path "test/integration/*" -vvv

# Fuzz (high runs)
forge test --match-path "test/fuzz/*" -vv --fuzz-runs 10000

# Gas report
forge snapshot

# Slither (in CI)
slither src/ --solc-remaps @oz=...

# Coverage
forge coverage --report lcov
```

---

## Phase 5: Deployment & Launch (Weeks 7-8)

### 5.1 Contract Deployment (Forge Script)

```solidity
// script/DeployCafeFunFactory.s.sol
contract DeployCafeFunFactory is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PK");
        address deployer = vm.addr(deployerKey);
        address uniswapV2Router = vm.envAddress("UNISWAP_V2_ROUTER");
        address weth = vm.envAddress("WETH");
        address feeCollector = vm.envAddress("FEE_COLLECTOR");

        vm.startBroadcast(deployerKey);

        // 1. Deploy CafeFunTreasury
        CafeFunTreasury feeCollectorImpl = new CafeFunTreasury();

        // 2. Deploy CafeFunFactory implementation
        CafeFunFactory factoryImpl = new CafeFunFactory();

        // 3. Deploy ERC1967Proxy for factory
        ERC1967Proxy factoryProxy = new ERC1967Proxy(
            address(factoryImpl),
            abi.encodeWithSelector(
                CafeFunFactory.initialize.selector,
                uniswapV2Router, weth, address(feeCollectorImpl)
            )
        );

        // 4. Transfer ownership
        CafeFunFactory(address(factoryProxy)).transferOwnership(deployer);

        vm.stopBroadcast();

        // Output
        console.log("Factory proxy:", address(factoryProxy));
        console.log("Factory impl:", address(factoryImpl));
        console.log("FeeCollector:", address(feeCollectorImpl));
    }
}
```

### 5.2 Mainnet Launch Checklist

```bash
# 1. Deploy Uniswap V2 on ETH (if not already deployed)
forge script script/DeployUniswapV2.s.sol --rpc-url https://rpc.robinhoodchain.com --broadcast

# 2. Deploy core contracts
forge script script/DeployCafeFunFactory.s.sol --rpc-url https://rpc.robinhoodchain.com --broadcast --verify

# 3. Verify implementation contracts on Blockscout
forge verify-contract <address> src/CafeFunFactory.sol:CafeFunFactory --chain 4663

# 4. Deploy backend
cd backend && npm run build
# Copy to VPS via rsync
rsync -avz --exclude node_modules ./ vps-eth:~/oxi-backend/
# Start API + indexer + chat via systemd

# 5. Deploy frontend
cd frontend && npm run build
# Static export or Next.js standalone on VPS
rsync -avz --exclude node_modules .next/ vps-eth:~/oxi-frontend/
# Configure nginx reverse proxy

# 6. DNS + SSL
# Point oxilaunch.io → VPS-ETH IP
# certbot --nginx -d oxilaunch.io

# 7. Smoke test
# Create first token, buy, sell, graduate, verify
```

### 5.3 Deployment Architecture

```
                         ┌──────────────┐
                         │   User       │
                         │   Browser    │
                         └──────┬───────┘
                                │ HTTPS
                    ┌───────────▼───────────┐
                    │     Nginx (reverse     │
                    │      proxy)            │
                    │  oxilaunch.io          │
                    └────┬──────┬─────┬──────┘
                         │      │     │
               ┌─────────▼┐ ┌───▼──┐ ┌▼────────┐
               │ Frontend │ │ API  │ │ Socket  │
               │ (Next.js)│ │(Exp) │ │ .IO     │
               └──────────┘ └──┬───┘ └─────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Node.js Services   │
                    │  (systemd managed)   │
                    │  ├─ api.service       │
                    │  ├─ indexer.service    │
                    │  ├─ chat.service       │
                    │  ├─ stream.service     │
                    │  └─ worker.service     │
                    └──────────┬────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   PostgreSQL         │
                    │   + Redis            │
                    └─────────────────────┘
```

### 5.4 Systemd Service (Template)

```ini
# /etc/systemd/system/oxi-api.service
[Unit]
Description=CafeFun API Server
After=network.target postgresql.service redis.service

[Service]
Type=exec
User=ubuntu
WorkingDirectory=/home/ubuntu/oxi-backend
ExecStart=/usr/bin/node /home/ubuntu/oxi-backend/dist/index.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=/home/ubuntu/oxi-backend/.env

[Install]
WantedBy=multi-user.target
```

---

## Phase 6: Post-Launch Features (Weeks 9-12)

### 6.1 Livestream Integration (Week 9-10)

**Architecture:**

```
Creator Browser                     LiveKit SFU               Viewer Browser
┌─────────────────┐                ┌──────────────┐          ┌─────────────────┐
│  WebRTC Publish │ ──video/audio→│  Selective    │──broadcast→│  WebRTC Play    │
│  (MediaStream)  │               │  Forwarding  │            │  (VideoPlayer)  │
│  LiveKit SDK    │               │  Unit (SFU)  │            │  LiveKit SDK    │
└─────────────────┘               └──────────────┘            └─────────────────┘
        │                                │                            │
        │ REST: createRoom()             │ WebSocket:                 │
        │                                │ participant events         │
        │                                │                            │
        ▼                                ▼                            ▼
┌─────────────────┐               ┌──────────────┐
│  Oxi Backend    │               │  Redis (room  │
│  LiveKit Token  │               │   state)      │
│  Generation     │               └──────────────┘
└─────────────────┘
```

**Implementation steps:**
1. Deploy LiveKit server (self-hosted Docker on VPS or LiveKit Cloud)
2. Backend: `POST /api/stream/token` → verify creator → generate LiveKit token → return room name + token
3. Backend: `POST /api/stream/join/:tokenAddress` → generate viewer token
4. Frontend: `LiveVideo.tsx` with `@livekit/components-react` for playback
5. Frontend: `StreamControls.tsx` with publish button (camera + mic picker)
6. Creator dashboard: stream health, viewer count, end stream

### 6.2 Enhanced Chat (Week 10)

- Profanity filter (optional config)
- Chat commands (`/price`, `/holders`, `/mcap`)
- Emoji reactions on messages
- Archived chat replay on token page

### 6.3 Creator Dashboard (Week 11)

- **Token list**: all tokens launched by connected wallet
- **Fee history**: cumulative fees earned (chart), recent payouts
- **CTO status**: pending/approved/rejected CTO requests
- **Stream management**: start/stop stream, viewer analytics
- **Share links**: direct token page URL, tweet intent

### 6.4 Analytics Dashboard (Week 12)

- Global stats: total tokens, volume, graduated, fees collected
- Top tokens by volume/MC/holders
- Daily active traders chart
- Fee treasury balance
- Recent CTO activity

---

## Architecture Reference

### Data Flow Diagram

```
                 ┌──────────────┐
                 │  User Wallet │
                 └──────┬───────┘
                        │ TX
                        ▼
┌─────────────────────────────────┐
│        Robinhood Chain (ETH)         │
│  ┌───────────────────────────┐  │
│  │  CafeFunFactory               │  │
│  │  ├─ createToken() ────────┼──┼──→ deploys CafeFunToken (CREATE2)
│  │  ├─ graduate() ──────────┼──┼──→ sends LP to UniswapV2
│  │  └─ setFeeDestination()  │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │  CafeFunToken #1 (0x...cafe)│  │
│  │  ├─ buy() ───────────────┼──┼──→ mints tokens, updates curve
│  │  ├─ sell() ──────────────┼──┼──→ burns tokens, returns ETH
│  │  ├─ graduate() ──────────┼──┼──→ locks liquidity, flags graduated
│  │  └─ swapAccruedFees()    │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │  UniswapV2 Pair (if grad) │  │
│  └───────────────────────────┘  │
└──────────────┬──────────────────┘
               │ Events
               ▼
┌──────────────────────────────┐
│     Backend Indexer           │
│  (polling newHeads every 2s)  │
│  ┌──────────────────────────┐ │
│  │  Parse logs → handlers   │ │
│  └────────┬─────────────────┘ │
└───────────┼──────────────────┘
            │ Write
            ▼
┌──────────────────────────────┐
│     PostgreSQL                │
│  + Redis (cache/queue)        │
└───────────┬──────────────────┘
            │ Read
            ▼
┌──────────────────────────────┐
│     REST API                  │
│  (Express, served to FE)      │
└───────────┬──────────────────┘
            │ JSON
            ▼
┌──────────────────────────────┐
│     Next.js Frontend          │
│  (Wagmi for chain, API for    │
│   indexed data)               │
└──────────────────────────────┘
```

### Fee Flow Diagram

```
Pre-Graduation (Curve):
────────────────────────
Buyer sends X ETH
  ├─ 1% fee (0.01 * X / 2)  → Creator wallet (0.5%)
  ├─ 1% fee (0.01 * X / 2)  → Protocol treasury (0.5%)
  └─ 99%                      → Curve virtual reserve (updates k)

Seller receives Y ETH
  ├─ 1% fee (0.01 * Y / 2)  → Creator wallet (0.5%)
  ├─ 1% fee (0.01 * Y / 2)  → Protocol treasury (0.5%)
  └─ 99%                      → From curve virtual reserve

Post-Graduation (Uniswap V2):
──────────────────────────────
Trade on Uniswap V2 (via CafeFunTreasury hook?)
  └─ Better approach: 1% fee collected in token contract
     ├─ Accumulates in contract (both creator + protocol shares)
     ├─ When accumulated ≥ $5 → auto-swap to ETH
     │  ├─ 50% → Creator wallet
     │  └─ 50% → Protocol treasury
     └─ Swap via Uniswap V2 router (sell fee tokens for ETH)
```

---

## File Tree (Final)

```
oxi-launchpad/
├── .env.example
├── .github/
│   └── workflows/
│       ├── contracts.yml
│       ├── frontend.yml
│       ├── backend.yml
│       └── deploy.yml
├── contracts/
│   ├── foundry.toml
│   ├── remappings.txt
│   ├── src/
│   │   ├── CafeFunFactory.sol
│   │   ├── CafeFunToken.sol
│   │   ├── CafeFunTreasury.sol
│   │   ├── interfaces/
│   │   │   ├── ICafeFunFactory.sol
│   │   │   ├── ICafeFunToken.sol
│   │   │   └── ICafeFunTreasury.sol
│   │   ├── libraries/
│   │   │   ├── BondingCurveLib.sol
│   │   │   └── FeeLib.sol
│   │   └── utils/
│   │       ├── Create2Address.sol
│   │       └── Constants.sol
│   ├── test/
│   │   ├── unit/
│   │   │   ├── CafeFunFactory.t.sol
│   │   │   ├── CafeFunToken.t.sol
│   │   │   ├── BondingCurveLib.t.sol
│   │   │   └── FeeLib.t.sol
│   │   ├── integration/
│   │   │   ├── CreateAndTrade.t.sol
│   │   │   ├── Graduation.t.sol
│   │   │   └── CTO.t.sol
│   │   ├── fuzz/
│   │   │   ├── CurveInvariants.t.sol
│   │   │   └── SolvencyInvariants.t.sol
│   │   └── adversarial/
│   │       └── AttackScenarios.t.sol
│   └── script/
│       ├── DeployCafeFunFactory.s.sol
│       └── UpgradeCafeFunFactory.s.sol
├── frontend/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── explore/page.tsx
│   │   ├── create/page.tsx
│   │   └── token/[address]/
│   │       ├── page.tsx
│   │       └── cto.tsx
│   ├── components/
│   │   ├── layout/
│   │   ├── explore/
│   │   ├── token/
│   │   ├── stream/
│   │   └── common/
│   ├── hooks/
│   ├── lib/
│   ├── package.json
│   └── next.config.js
├── backend/
│   ├── src/
│   │   ├── index.ts
│   │   ├── config.ts
│   │   ├── db/
│   │   │   ├── schema.ts
│   │   │   └── index.ts
│   │   ├── indexer/
│   │   ├── api/
│   │   ├── chat/
│   │   ├── stream/
│   │   └── queue/
│   ├── package.json
│   └── tsconfig.json
├── docs/
│   ├── PRD.md
│   ├── IMPLEMENTATION_PLAN.md
│   ├── WHITEPAPER.md
│   └── DEPLOYMENT.md
└── README.md
```

---

## Key Decisions Log

| # | Decision | Option Chosen | Alternative | Rationale |
|---|---|---|---|---|
| 1 | Target chain | Robinhood Chain (ETH) | Base, Arbitrum | Native chain, lower fees, ecosystem building |
| 2 | Factory upgrade pattern | UUPS | Transparent proxy | Cheaper per-token creation, standard for Foundry |
| 3 | Token deploy pattern | Minimal proxy + CREATE2 | Cloning factory | Vanity address, cheaper deploys |
| 4 | Curve type | Constant product (x·y=k) | Linear, logarithmic | Battle-tested, same as top launchpads |
| 5 | Graduation DEX | Uniswap V2 | V3, Aerodrome | Simpler LP management, proven |
| 6 | Fee mechanism (curve) | Direct ETH split | ERC-20 fee token | Simple, no extra token |
| 7 | Fee mechanism (post-grade) | Token contract accrual + swap | Direct DEX fee hook | Works without modifying DEX |
| 8 | Indexer | Custom (polling) | The Graph | Full control, lighter for custom chain |
| 9 | Chat auth | Off-chain EIP-191 sig | On-chain tx | No gas cost, instant |
| 10 | WebRTC | LiveKit self-hosted | Daily, Zoom API | Open-source, self-sovereign |
| 11 | Token parameters | Fixed (70/30 split) | Configurable per creator | Simpler, fair for all, easier to audit |
| 12 | Graduation threshold | $9,300 USD (snapshotted to ETH) | Fixed ETH | USD peg = consistent for all creators |

---

## Open Questions (TBD)

| Question | Impact | Notes |
|---|---|---|
| Is Uniswap V2 already deployed on ETH? | Blocking | If not, need to deploy it first |
| What's the current ETH/USD rate? | Medium | Affects graduation threshold snapshot |
| VPS spec for backend services? | Medium | RAM/CPU determines indexing performance |
| Domain name for frontend? | Low | oxilaunch.io? oxi.fun? |
| Liquidity: initial protocol treasury? | Low | Need ETH for gas + initial operations |
| Audit budget? | Medium | Determines audit scope / firm choice |

---

*End of Implementation Plan — v1.0*
