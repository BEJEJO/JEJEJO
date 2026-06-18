# JEJEJO Wager Protocol — Design Document

## Overview

JEJEJO is structured as three Soroban smart contracts, a React/TypeScript frontend, and a lightweight off-chain event indexer. The original Checkmate-Escrow two-contract architecture is extended with an OracleRegistry and the frontend/indexer layers that were planned but never built in the original roadmap.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Frontend (React/TS)                        │
│  Dashboard │ Create Wager │ Profile │ Analytics │ Settings        │
└───────────────────────────┬──────────────────────────────────────┘
                            │ Stellar SDK + Freighter
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
  ┌───────────────┐ ┌──────────────┐ ┌────────────────────┐
  │ WagerContract │ │OracleRegistry│ │ OracleStoreContract │
  │ (wager CRUD,  │ │ (per-category│ │ (audit log of       │
  │  escrow,      │ │  oracle auth)│ │  verified results)  │
  │  lifecycle)   │ └──────────────┘ └────────────────────┘
  └───────────────┘
          ▲
          │ submit_result (authorized oracle only)
  ┌───────────────┐
  │ Oracle Service│  (off-chain, per category)
  │ (Sports/       │
  │  Finance/etc.) │
  └───────────────┘
          │
          │ polls external APIs
  ┌────────────────────────┐
  │ External Data Sources  │
  │ (Sports APIs, price    │
  │  feeds, esports APIs)  │
  └────────────────────────┘

  ┌────────────────────────┐
  │  Off-chain Indexer     │  reads contract events → serves analytics API
  └────────────────────────┘
```

---

## Contract Architecture

### 1. WagerContract (`contracts/wager`)

The core contract. Derived from Checkmate-Escrow's `EscrowContract` with the following changes:
- Renamed types: `Match` → `Wager`, `Platform` → `EventCategory`, `game_id` → `event_id`, `event_metadata` added.
- Oracle authorization delegated to `OracleRegistry` instead of a single stored address.
- `Disputed` state added between `Active` and `Completed`.
- Bugs fixed: duplicate `DataKey` variant, storage tier mismatch, `initialize` error return.

#### Data Types

```rust
pub enum WagerState {
    Pending,    // created, awaiting deposits
    Active,     // both deposited, awaiting oracle result
    Disputed,   // result submitted, in dispute window
    Completed,  // finalized, payout executed
    Cancelled,  // cancelled or expired before activation
}

pub enum EventCategory {
    Sports,
    Finance,
    Esports,
    Custom,
}

pub enum Outcome {
    Participant1,
    Participant2,
    Draw,
}

pub struct Wager {
    pub id: u64,
    pub participant1: Address,
    pub participant2: Address,
    pub stake_amount: i128,
    pub token: Address,
    pub event_id: String,             // max 64 bytes, unique
    pub event_category: EventCategory,
    pub event_metadata: String,       // max 256 bytes, human-readable description
    pub state: WagerState,
    pub participant1_deposited: bool,
    pub participant2_deposited: bool,
    pub pending_outcome: Option<Outcome>,  // set when oracle submits, before finalization
    pub created_ledger: u32,
    pub completed_ledger: Option<u32>,
    pub dispute_deadline_ledger: Option<u32>,  // set when oracle submits result
    pub disputed: bool,
}

pub enum DataKey {
    Wager(u64),
    WagerCount,
    OracleRegistry,   // Address of OracleRegistry contract
    Admin,
    PendingAdmin,
    Paused,
    EventId(String),
    ActiveWagers,
    ParticipantWagers(Address),
    WagerTimeout,
    DisputeWindow,
    AllowedToken(Address),
    AllowedTokenCount,
    AllowlistEnforced,
    AllowedTokens,
    OracleRecord(u64),
}
```

#### Contract Functions

| Function | Authorized By | Description |
|---|---|---|
| `initialize(oracle_registry, admin)` | Deployer (once) | Sets OracleRegistry address and admin. Returns `Err(AlreadyInitialized)` on repeat. |
| `create_wager(p1, p2, stake, token, event_id, category, metadata)` | participant1 | Creates wager, validates inputs, emits `wager/created`. |
| `deposit(wager_id, participant)` | participant | Transfers stake to escrow; transitions to Active when both deposit. |
| `submit_result(wager_id, outcome, caller)` | Authorized oracle | Checks OracleRegistry auth; sets `pending_outcome`; transitions to `Disputed`; sets `dispute_deadline_ledger`. |
| `raise_dispute(wager_id, caller)` | participant1 or participant2 | Within dispute window only; sets `disputed = true`; emits `wager/disputed`. |
| `finalize_wager(wager_id)` | Anyone | After dispute window with no dispute; executes payout; transitions to `Completed`. |
| `resolve_dispute(wager_id, final_outcome)` | Admin | Overrides pending outcome; executes payout; transitions to `Completed`. |
| `cancel_wager(wager_id, caller)` | participant1 or participant2 | Pending state only; refunds deposits; transitions to `Cancelled`. |
| `expire_wager(wager_id)` | Anyone | After timeout in Pending state; refunds deposits; transitions to `Cancelled`. |
| `pause / unpause` | Admin | Blocks/unblocks create, deposit, submit_result. |
| `propose_admin / accept_admin` | Admin / PendingAdmin | Two-step admin transfer. |
| `set_wager_timeout(ledgers)` | Admin | Sets pending wager expiration window. |
| `set_dispute_window(ledgers)` | Admin | Sets dispute window duration. |
| `add_allowed_token / remove_allowed_token` | Admin | Manages token allowlist in instance storage. |
| `get_wager(id)` | Anyone | Read wager by ID. |
| `get_participant_wagers(address)` | Anyone | Returns all wager IDs for a participant. |
| `get_pending_wagers() / get_active_wagers()` | Anyone | Filtered state queries. |
| `get_escrow_balance(wager_id)` | Anyone | Returns current escrowed balance (0 if terminal). |
| `is_funded(wager_id)` | Anyone | True if both participants deposited. |

#### Wager Lifecycle State Machine

```
create_wager
    │
    ▼
 Pending ──── deposit(p1) ──── Pending
    │                              │
    │ deposit(p2)                  │ cancel_wager / expire_wager
    ▼                              ▼
 Active                        Cancelled
    │
    │ submit_result (oracle)
    ▼
 Disputed ──── raise_dispute ──── Disputed (disputed=true)
    │                                   │
    │ finalize_wager                     │ resolve_dispute (admin)
    │ (no dispute, window elapsed)       │
    ▼                                   ▼
 Completed                          Completed
```

---

### 2. OracleRegistry (`contracts/oracle-registry`)

New contract. Maintains a per-category mapping of authorized oracle addresses.

#### Data Types

```rust
pub struct OracleList {
    pub oracles: Vec<Address>,
}

pub enum DataKey {
    Admin,
    Paused,
    CategoryOracles(EventCategory),  // Vec<Address> per category
}
```

#### Contract Functions

| Function | Authorized By | Description |
|---|---|---|
| `initialize(admin)` | Deployer (once) | Sets admin. |
| `register_oracle(category, oracle_address)` | Admin | Adds oracle to category list (idempotent). |
| `remove_oracle(category, oracle_address)` | Admin | Removes oracle from category list; `OracleNotFound` if not registered. |
| `is_oracle_authorized(category, oracle_address)` | Anyone | Returns bool. |
| `get_oracles_for_category(category)` | Anyone | Returns Vec<Address>. |
| `update_admin / pause / unpause` | Admin | Standard admin controls. |

---

### 3. OracleStoreContract (`contracts/oracle-store`)

Renamed and lightly refactored from Checkmate-Escrow's `OracleContract`. Stores verified event results as an audit log. Not the payout authority — that is the WagerContract via OracleRegistry.

Changes from original:
- Renamed `OracleContract` → `OracleStoreContract`.
- `ResultEntry` uses `EventCategory` instead of `Platform`.
- `submit_result` signature updated to use `event_id` and `EventCategory`.
- Bugs fixed: consistent with renamed types.

---

## Frontend Architecture

### Technology Stack

| Layer | Choice | Reason |
|---|---|---|
| Framework | React 18 + TypeScript | Matches original roadmap spec |
| Build tool | Vite | Fast dev server, native ESM |
| Styling | TailwindCSS v3 | Utility-first, easy theming |
| Wallet | `@stellar/freighter-api` | Native Freighter integration |
| Stellar SDK | `@stellar/stellar-sdk` | Transaction building and submission |
| Charts | Recharts | React-native, accessible charts |
| State | React Context + useReducer | No external state library needed at MVP |
| Routing | React Router v6 | Standard SPA routing |
| Testing | Vitest + React Testing Library | Fast unit and component tests |

### Design System

**Color Palette (Dark/Light)**

| Token | Dark Mode | Light Mode |
|---|---|---|
| `bg-base` | `#0F1117` | `#F8F9FA` |
| `bg-surface` | `#1A1D27` | `#FFFFFF` |
| `bg-elevated` | `#252836` | `#F0F2F5` |
| `accent-primary` | `#6C63FF` | `#5B52E8` |
| `accent-secondary` | `#00D4AA` | `#00B894` |
| `text-primary` | `#F0F2F5` | `#1A1D27` |
| `text-secondary` | `#9BA3B2` | `#6B7280` |
| `state-pending` | `#F59E0B` | `#D97706` |
| `state-active` | `#3B82F6` | `#2563EB` |
| `state-disputed` | `#EF4444` | `#DC2626` |
| `state-completed` | `#10B981` | `#059669` |
| `state-cancelled` | `#6B7280` | `#9CA3AF` |

**Typography**

- Headings: `Inter` (700, 600)
- Body: `Inter` (400, 500)
- Monospace (addresses, IDs): `JetBrains Mono`
- Base size: 16px, scale: 1.25 (Major Third)

### Folder Structure

```
frontend/
├── public/
├── src/
│   ├── components/
│   │   ├── common/          # Button, Badge, Card, Modal, Input, Spinner
│   │   ├── layout/          # Navbar, Sidebar, Footer, PageWrapper
│   │   ├── wager/           # WagerCard, WagerDetail, WagerForm, WagerTable
│   │   ├── profile/         # ProfileStats, WagerHistory, ExportButtons
│   │   ├── analytics/       # StatsBar, CategoryChart, VolumeChart
│   │   └── notifications/   # NotificationBell, NotificationList
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   ├── CreateWager.tsx
│   │   ├── WagerDetail.tsx
│   │   ├── Profile.tsx
│   │   ├── Analytics.tsx
│   │   └── Settings.tsx
│   ├── hooks/
│   │   ├── useWallet.ts       # Freighter connection state
│   │   ├── useWagers.ts       # Contract read hooks
│   │   ├── useNotifications.ts
│   │   └── useTheme.ts
│   ├── context/
│   │   ├── WalletContext.tsx
│   │   └── ThemeContext.tsx
│   ├── lib/
│   │   ├── stellar.ts         # SDK helpers, transaction builders
│   │   ├── contracts.ts       # Contract client wrappers
│   │   ├── indexer.ts         # Off-chain indexer API client
│   │   └── export.ts          # CSV/JSON export utilities
│   ├── types/
│   │   └── index.ts           # Shared TypeScript types mirroring contract types
│   └── styles/
│       └── globals.css        # Tailwind base + custom tokens
├── tailwind.config.ts
├── vite.config.ts
└── package.json
```

### Page Layouts

**Dashboard** — two-column grid on desktop, single column on mobile. Left: filter sidebar. Right: wager card grid with pagination. Top: stats bar (active wagers, total volume, your open wagers).

**Wager Detail** — full-width card with event info, participant addresses, escrow balance, state timeline, and action buttons (deposit, cancel, raise dispute, finalize). Action buttons are shown/hidden based on connected wallet and wager state.

**Profile** — stats row at top (wins/losses/volume), tabbed wager history table below, export buttons in top-right.

**Analytics** — stats bar, category distribution bar chart, 30-day volume line chart, oracle performance table.

**Settings** — grouped form sections (Network, Display, Notifications) with save-on-change behavior.

### Responsive Breakpoints

| Breakpoint | Width | Layout changes |
|---|---|---|
| `sm` | 320px | Single column, hamburger nav, card stacks |
| `md` | 768px | Two-column grids, full tables |
| `lg` | 1024px | Sidebar appears, wider charts |
| `xl` | 1440px | Max-width content container (1280px) |

### Wave Sections and Animations

- Hero wave: SVG wave divider between header and dashboard content (CSS animated, subtle).
- Card hover: `transform: translateY(-2px)` with `box-shadow` transition (150ms ease).
- State badge: pulse animation on `Active` and `Disputed` states.
- Page transitions: fade-in on route change (150ms opacity).
- Skeleton loaders: animated shimmer while contract data loads.
- Notification badge: scale-in animation on new notification.

---

## Off-chain Indexer

A lightweight Node.js service that:
1. Subscribes to WagerContract events via Stellar RPC (`getEvents` streaming).
2. Persists events to a local SQLite database (or PostgreSQL for production).
3. Exposes a REST API used by the Analytics page.

### API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/stats` | Aggregate stats (total wagers, volume, resolution rate) |
| `GET` | `/api/wagers` | Paginated wager list with filters |
| `GET` | `/api/wagers/:id` | Single wager detail |
| `GET` | `/api/profile/:address` | Participant stats and history |
| `GET` | `/api/analytics/daily` | Daily creation counts (last 30 days) |
| `GET` | `/api/analytics/categories` | Wager counts by EventCategory |

---

## Codebase Refactor Plan

### Renames

| Old | New |
|---|---|
| `Checkmate-Escrow/` root | `JEJEJO/` |
| `contracts/escrow` | `contracts/wager` |
| `contracts/oracle` | `contracts/oracle-store` |
| `EscrowContract` | `WagerContract` |
| `OracleContract` | `OracleStoreContract` |
| `Match` struct | `Wager` struct |
| `MatchState` enum | `WagerState` enum |
| `Platform` enum | `EventCategory` enum |
| `Winner` enum | `Outcome` enum |
| `game_id` field | `event_id` field |
| `create_match` | `create_wager` |
| `cancel_match` | `cancel_wager` |
| `expire_match` | `expire_wager` |
| `get_player_matches` | `get_participant_wagers` |
| `get_pending_matches` | `get_pending_wagers` |
| `get_active_matches` | `get_active_wagers` |
| Event namespace `match` | `wager` |

### Bug Fixes Applied During Refactor

1. Remove duplicate `DataKey::AllowedTokenCount` from `types.rs`.
2. `WagerContract::initialize` returns `Result<(), Error>` and returns `Err(AlreadyInitialized)`.
3. `is_token_allowed` reads from `instance` storage (not `persistent`).
4. `submit_result_with_oracle_record` passes `caller` to the internal `submit_result` call.
5. `submit_result` now checks OracleRegistry instead of comparing against a single stored oracle address.
6. `remove_live_match` call in `submit_result` is removed (it was calling a non-existent function — was dead code).

---

## New Contract: OracleRegistry

This contract has no equivalent in Checkmate-Escrow and is written from scratch.

### Key Design Decisions

- **Per-category oracle lists**: Each `EventCategory` maps to a `Vec<Address>`. A single oracle may be registered for multiple categories.
- **No minimum oracle count**: The registry allows zero oracles for a category; `submit_result` will fail with `Unauthorized` if no oracle is registered.
- **Idempotent registration**: Registering the same oracle twice for the same category is a no-op.
- **Separate admin from WagerContract admin**: OracleRegistry has its own admin to enable independent governance of oracle authorization.

---

## Documentation Structure

```
JEJEJO/
├── README.md                     # Product overview, quick start, env vars
├── NOTICE                        # Attribution to Checkmate-Escrow
├── LICENSE                       # MIT (updated with JEJEJO contributors)
├── CHANGELOG.md                  # Version history
├── CONTRIBUTING.md               # Branching, commits, PR process
├── docs/
│   ├── architecture.md           # This document (updated)
│   ├── setup.md                  # Local dev setup (contracts + frontend)
│   ├── oracle.md                 # Oracle integration guide (updated)
│   ├── security.md               # Threat model (updated)
│   ├── deployment.md             # Testnet/mainnet deployment
│   └── roadmap.md                # Updated roadmap
├── contracts/
│   ├── wager/
│   ├── oracle-registry/
│   └── oracle-store/
├── frontend/
└── indexer/
```

---

## Git Commit Strategy

Commits are grouped by feature/concern so the development history is reviewable:

1. `chore: rename workspace to JEJEJO, update Cargo.toml`
2. `fix: resolve known bugs in escrow contract (duplicate DataKey, storage tier mismatch, initialize return type)`
3. `refactor: rename contracts/escrow → contracts/wager, update all types and event namespaces`
4. `feat: add Disputed state and dispute window to WagerContract`
5. `feat: add OracleRegistry contract`
6. `refactor: rename contracts/oracle → contracts/oracle-store, update types`
7. `feat: wire WagerContract to OracleRegistry for multi-oracle authorization`
8. `test: update all contract tests for renamed types and new lifecycle`
9. `docs: add NOTICE, update LICENSE, update README and architecture docs`
10. `docs: add CHANGELOG, CONTRIBUTING, setup guide`
11. `feat(frontend): scaffold React/TS frontend with Vite, Tailwind, routing`
12. `feat(frontend): implement wallet authentication with Freighter`
13. `feat(frontend): implement wager dashboard with filters and search`
14. `feat(frontend): implement create wager form and flow`
15. `feat(frontend): implement user profile page and stats`
16. `feat(frontend): implement analytics dashboard with charts`
17. `feat(frontend): implement notifications system`
18. `feat(frontend): implement dark/light mode theming`
19. `feat(frontend): implement data export (CSV/JSON)`
20. `feat(frontend): implement settings page`
21. `feat(frontend): accessibility pass (ARIA, keyboard nav, contrast)`
22. `feat(frontend): responsive design pass (mobile/tablet breakpoints)`
23. `feat(indexer): scaffold off-chain event indexer`
