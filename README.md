# RobinLaunch — Fair-Launch Bonding Curve Token Launchpad on Robinhood Chain

> Permissionless token launchpad with transparent bonding curves, auto-graduation to DEX, and built-in community engagement.

Inspired by [RobinFun](https://robinfun.live) — built for **Robinhood Chain**.

---

## ✨ Features

- **One-click token launch** — deploy a token in a single transaction
- **Bonding curve pricing** — transparent, deterministic, same price for everyone
- **Auto-graduation** — tokens migrate to Uniswap V2 at $44k MC with **100% LP burned**
- **1% trade fee** — 0.5% to creator, 0.5% to protocol
- **Community Takeover (CTO)** — claim abandoned project fees
- **Livestream per token** — WebRTC streaming + live chat
- **Verifiable on-chain** — everything is on ETH explorer

---

## Repo Structure

```
oxi-launchpad/
├── contracts/    — Solidity smart contracts (Foundry)
├── frontend/     — Next.js dapp (React + Wagmi + RainbowKit)
├── backend/      — API, indexer, chat, WebRTC services
├── docs/         — PRD, plan, whitepaper, deployment guide
└── scripts/      — Deploy & maintenance scripts
```

---

## Tech Stack

| Layer | Tech |
|---|---|
| Smart Contracts | Solidity ^0.8.24, Foundry |
| Chain | Robinhood Chain (ETH, chainId 4663) |
| Frontend | Next.js 14, Tailwind, Wagmi, RainbowKit |
| Backend | Node.js, Express, PostgreSQL, Redis |
| Streaming | LiveKit (WebRTC SFU) |
| Chat | Socket.IO |
| DEX | Uniswap V2 |

---

## Quick Start

```bash
# Contracts
cd contracts
forge build
forge test -vvv

# Frontend
cd frontend
npm install
npm run dev

# Backend
cd backend
npm install
npm run dev
```

---

## Whitepaper

See [docs/WHITEPAPER.md](docs/WHITEPAPER.md) for the full protocol specification.

---

## License

MIT
