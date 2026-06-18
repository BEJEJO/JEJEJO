# JEJEJO Wager Protocol — Implementation Tasks

## Task Overview

Tasks are organized into phases matching the git commit strategy. Each task maps to one or more requirements and design components. Tasks within a phase can generally be done in parallel; phases must be completed in order.

---

## Phase 1: Codebase Foundation

- [ ] 1. Set up workspace and Cargo configuration
  - [ ] 1.1 Set the workspace name to `JEJEJO` in `Cargo.toml` workspace metadata
  - [ ] 1.2 Update `Cargo.toml` workspace members to `contracts/wager`, `contracts/oracle-registry`, `contracts/oracle-store`
  - [ ] 1.3 Update `environments.toml` to reference `JEJEJO`
  - [ ] 1.4 Update `.env.example` with new contract variable names (`CONTRACT_WAGER`, `CONTRACT_ORACLE_REGISTRY`, `CONTRACT_ORACLE_STORE`)
  - **Requirement**: REQ-2 (AC-2.1, AC-2.2)

- [ ] 2. Fix known bugs in the escrow contract before refactoring
  - [ ] 2.1 Remove the duplicate `DataKey::AllowedTokenCount` variant from `contracts/escrow/src/types.rs`
  - [ ] 2.2 Change `initialize()` in `contracts/escrow/src/lib.rs` to return `Result<(), Error>` and return `Err(Error::AlreadyInitialized)` instead of silently returning
  - [ ] 2.3 Fix `is_token_allowed` to read from `instance` storage instead of `persistent` storage
  - [ ] 2.4 Fix `submit_result_with_oracle_record` to pass the `caller` parameter to the internal `submit_result` call
  - [ ] 2.5 Remove the `remove_live_match` call inside `submit_result` (references a non-existent function)
  - [ ] 2.6 Verify all existing tests still pass: `cargo test -p escrow`
  - **Requirement**: REQ-2 (AC-2.5 through AC-2.10)

---

## Phase 2: Contract Refactor — Wager Contract

- [ ] 3. Rename `contracts/escrow` to `contracts/wager`
  - [ ] 3.1 Copy `contracts/escrow` to `contracts/wager` and update `contracts/wager/Cargo.toml` (package name `wager`, `[lib]` name `wager`)
  - [ ] 3.2 Rename `EscrowContract` → `WagerContract` in `lib.rs`
  - [ ] 3.3 Rename types in `types.rs`: `Match` → `Wager`, `MatchState` → `WagerState`, `Platform` → `EventCategory`, `Winner` → `Outcome`
  - [ ] 3.4 Rename all `DataKey` variants: `Match(u64)` → `Wager(u64)`, `MatchCount` → `WagerCount`, `GameId` → `EventId`, `ActiveMatches` → `ActiveWagers`, `PlayerMatches` → `ParticipantWagers`, `MatchTimeout` → `WagerTimeout`
  - [ ] 3.5 Add new fields to `Wager` struct: `event_category: EventCategory`, `event_metadata: String`, `pending_outcome: Option<Outcome>`, `dispute_deadline_ledger: Option<u32>`, `disputed: bool`
  - [ ] 3.6 Add `Disputed` variant to `WagerState` enum
  - [ ] 3.7 Rename `EventCategory` variants: `Lichess` → `Sports`, `ChessDotCom` → `Esports`, add `Finance` and `Custom`
  - [ ] 3.8 Rename all contract functions: `create_match` → `create_wager`, `cancel_match` → `cancel_wager`, `expire_match` → `expire_wager`, `get_player_matches` → `get_participant_wagers`, `get_pending_matches` → `get_pending_wagers`, `get_active_matches` → `get_active_wagers`
  - [ ] 3.9 Update all event namespace strings from `"match"` to `"wager"` in `lib.rs`
  - [ ] 3.10 Update `create_wager` to accept `event_category` and `event_metadata` parameters; validate metadata max 256 bytes
  - [ ] 3.11 Add `DataKey::OracleRegistry` to store the OracleRegistry contract address; remove single `DataKey::Oracle`
  - [ ] 3.12 Update `initialize` signature to `(oracle_registry: Address, admin: Address)`
  - **Requirement**: REQ-2, REQ-3, REQ-6

- [ ] 4. Implement Disputed state and dispute window in WagerContract
  - [ ] 4.1 Update `submit_result` to transition wager to `Disputed` state (not `Completed`), set `pending_outcome`, and set `dispute_deadline_ledger = current_ledger + DISPUTE_WINDOW_LEDGERS`
  - [ ] 4.2 Implement `raise_dispute(wager_id: u64, caller: Address)`: validate caller is participant, validate within dispute window, set `disputed = true`, emit `wager/disputed`
  - [ ] 4.3 Implement `finalize_wager(wager_id: u64)`: callable by anyone; requires `Disputed` state, no dispute raised, window elapsed; executes payout from `pending_outcome`; transitions to `Completed`
  - [ ] 4.4 Implement `resolve_dispute(wager_id: u64, final_outcome: Outcome)`: admin-only; overrides `pending_outcome`; executes payout; transitions to `Completed`
  - [ ] 4.5 Add `DataKey::DisputeWindow` and `set_dispute_window(ledgers)` admin function
  - [ ] 4.6 Add error variants to `errors.rs`: `DisputeWindowClosed`, `AlreadyDisputed`, `DisputeWindowOpen`, `OracleNotAuthorized`, `InvalidMetadata`
  - [ ] 4.7 Add `DISPUTE_WINDOW_LEDGERS` constant (default 720 ledgers ≈ 1 hour)
  - **Requirement**: REQ-5

- [ ] 5. Wire WagerContract to OracleRegistry for authorization
  - [ ] 5.1 Update `submit_result` to call `OracleRegistry::is_oracle_authorized(wager.event_category, caller)` via cross-contract call; return `Error::OracleNotAuthorized` if false
  - [ ] 5.2 Add `get_oracle_registry(env) -> Address` read function
  - **Requirement**: REQ-4 (AC-4.4, AC-4.5)

---

## Phase 3: New Contract — OracleRegistry

- [ ] 6. Scaffold OracleRegistry contract
  - [ ] 6.1 Create `contracts/oracle-registry/Cargo.toml` with package name `oracle-registry`
  - [ ] 6.2 Create `contracts/oracle-registry/src/lib.rs` with `OracleRegistry` contract struct
  - [ ] 6.3 Create `contracts/oracle-registry/src/types.rs` with `DataKey` enum (`Admin`, `Paused`, `CategoryOracles(EventCategory)`) and import `EventCategory` from `wager` crate
  - [ ] 6.4 Create `contracts/oracle-registry/src/errors.rs` with `Error` enum (`AlreadyInitialized`, `Unauthorized`, `ContractPaused`, `OracleNotFound`, `Overflow`)
  - **Requirement**: REQ-4

- [ ] 7. Implement OracleRegistry contract functions
  - [ ] 7.1 Implement `initialize(admin: Address)`: one-time init, sets admin, returns `Err(AlreadyInitialized)` on repeat
  - [ ] 7.2 Implement `register_oracle(category: EventCategory, oracle: Address)`: admin-only; reads current list, appends if not present (idempotent), saves back; emits `registry/oracle_added`
  - [ ] 7.3 Implement `remove_oracle(category: EventCategory, oracle: Address)`: admin-only; removes from list; returns `Err(OracleNotFound)` if not registered; emits `registry/oracle_removed`
  - [ ] 7.4 Implement `is_oracle_authorized(category: EventCategory, oracle: Address) -> bool`: reads list, returns true if present
  - [ ] 7.5 Implement `get_oracles_for_category(category: EventCategory) -> Vec<Address>`: returns current list for category
  - [ ] 7.6 Implement `pause`, `unpause`, `update_admin` with same patterns as WagerContract
  - [ ] 7.7 Apply TTL extension on all invocations
  - **Requirement**: REQ-4

- [ ] 8. Write OracleRegistry tests
  - [ ] 8.1 Test: register oracle for each category, verify `is_oracle_authorized` returns true
  - [ ] 8.2 Test: remove oracle, verify `is_oracle_authorized` returns false
  - [ ] 8.3 Test: duplicate registration is idempotent (no error, not duplicated in list)
  - [ ] 8.4 Test: remove non-existent oracle returns `OracleNotFound`
  - [ ] 8.5 Test: non-admin cannot register or remove oracles
  - [ ] 8.6 Test: `is_oracle_authorized` cross-contract call from WagerContract test environment
  - **Requirement**: REQ-4, P-10

---

## Phase 4: OracleStore Contract Refactor

- [ ] 9. Rename and refactor `contracts/oracle` to `contracts/oracle-store`
  - [ ] 9.1 Copy `contracts/oracle` to `contracts/oracle-store`, update `Cargo.toml` (package name `oracle-store`)
  - [ ] 9.2 Rename `OracleContract` → `OracleStoreContract` in `lib.rs`
  - [ ] 9.3 Update `types.rs`: replace `Platform` import from `escrow` with `EventCategory` import from `wager`; update `ResultEntry` to use `EventCategory`
  - [ ] 9.4 Rename `Winner` → `Outcome` in `types.rs` and all usages
  - [ ] 9.5 Ensure all existing oracle tests pass with updated types
  - **Requirement**: REQ-2 (AC-2.3)

---

## Phase 5: Update Tests

- [ ] 10. Update WagerContract test suite
  - [ ] 10.1 Update `tests/helpers.rs`: rename all helpers (`create_default_match` → `create_default_wager`, `fund_match` → `fund_wager`, `run_full_match` → `run_full_wager`, etc.)
  - [ ] 10.2 Update `tests/mod.rs` setup function to pass OracleRegistry address to `initialize`
  - [ ] 10.3 Update all test modules to use new type names (`WagerState`, `EventCategory`, `Outcome`)
  - [ ] 10.4 Add dispute window tests: submit_result → Disputed; raise_dispute within window; finalize after window; resolve_dispute by admin
  - [ ] 10.5 Add property-based tests for P-1 (fund conservation) across create/deposit/cancel/expire/finalize sequences
  - [ ] 10.6 Add property-based tests for P-2 (state machine monotonicity)
  - [ ] 10.7 Add property-based tests for P-3 (double deposit prevention)
  - [ ] 10.8 Add property-based tests for P-5 (dispute window enforcement)
  - [ ] 10.9 Add property-based tests for P-6 (payout completeness)
  - [ ] 10.10 Add property-based tests for P-7 (event ID uniqueness)
  - [ ] 10.11 Add property-based tests for P-8 (pause completeness)
  - [ ] 10.12 Verify `cargo test --workspace` passes with zero failures
  - **Requirement**: REQ-2 (AC-2.10), P-1 through P-10

---

## Phase 6: Documentation

- [ ] 11. Update root-level documentation
  - [ ] 11.1 Rewrite `README.md` for JEJEJO: product description, quick start commands, env vars, API reference, roadmap summary, acknowledgements section
  - [ ] 11.2 Update `LICENSE`: MIT license crediting JEJEJO contributors
  - [ ] 11.3 Create `CHANGELOG.md` starting with `v1.0.0` (JEJEJO initial release)
  - [ ] 11.4 Rewrite `CONTRIBUTING.md`: branching strategy (`main`, `develop`, `feature/*`, `fix/*`), conventional commit format, PR checklist, code style (rustfmt, clippy)
  - **Requirement**: REQ-1, REQ-19

- [ ] 12. Update technical documentation
  - [ ] 12.1 Rewrite `docs/architecture.md` to reflect three-contract architecture, frontend, and indexer
  - [ ] 12.2 Create `docs/setup.md`: step-by-step local dev for contracts (Rust, Soroban CLI) and frontend (Node, npm)
  - [ ] 12.3 Update `docs/oracle.md` to describe OracleRegistry, multi-category oracles, and updated result flow
  - [ ] 12.4 Update `docs/security.md` to reflect multi-oracle trust model and dispute window security properties
  - [ ] 12.5 Update `docs/roadmap.md` to JEJEJO roadmap (MVP → analytics → mobile)
  - **Requirement**: REQ-19

---

## Phase 7: Frontend Scaffold

- [ ] 13. Initialize frontend project
  - [ ] 13.1 Create `frontend/` directory with `npm create vite@latest -- --template react-ts`
  - [ ] 13.2 Install dependencies: `tailwindcss`, `@tailwindcss/forms`, `react-router-dom`, `@stellar/stellar-sdk`, `@stellar/freighter-api`, `recharts`, `clsx`
  - [ ] 13.3 Configure `tailwind.config.ts` with custom color tokens (dark/light palette from design doc), `Inter` and `JetBrains Mono` fonts, custom animation utilities
  - [ ] 13.4 Configure `vite.config.ts` with env variable prefix `VITE_`
  - [ ] 13.5 Create folder structure: `src/components/`, `src/pages/`, `src/hooks/`, `src/context/`, `src/lib/`, `src/types/`
  - [ ] 13.6 Create `src/types/index.ts` with TypeScript types mirroring contract types (`Wager`, `WagerState`, `EventCategory`, `Outcome`)
  - [ ] 13.7 Create `src/styles/globals.css` with Tailwind directives and CSS custom properties for theme tokens
  - [ ] 13.8 Set up React Router in `src/main.tsx` with routes: `/`, `/wager/:id`, `/create`, `/profile`, `/profile/:address`, `/analytics`, `/settings`
  - **Requirement**: REQ-9 through REQ-18

- [ ] 14. Implement shared layout and navigation components
  - [ ] 14.1 Create `src/components/layout/Navbar.tsx`: logo, nav links, wallet connect button, theme toggle, notification bell; collapses to hamburger on mobile
  - [ ] 14.2 Create `src/components/layout/PageWrapper.tsx`: max-width container, padding, fade-in animation
  - [ ] 14.3 Create `src/components/layout/Footer.tsx`: links, JEJEJO branding
  - [ ] 14.4 Create common components: `Button`, `Badge`, `Card`, `Modal`, `Input`, `Spinner`, `Skeleton` in `src/components/common/`
  - [ ] 14.5 Add wave SVG divider component with CSS animation for hero sections
  - **Requirement**: REQ-9, REQ-17, REQ-18

---

## Phase 8: Wallet Authentication

- [ ] 15. Implement Freighter wallet integration
  - [ ] 15.1 Create `src/context/WalletContext.tsx`: state for `publicKey`, `network`, `isConnected`; actions `connect`, `disconnect`
  - [ ] 15.2 Create `src/hooks/useWallet.ts`: exposes wallet context with typed return
  - [ ] 15.3 Implement `connect()`: calls `@stellar/freighter-api` `getPublicKey()`; stores in context and localStorage
  - [ ] 15.4 Implement `disconnect()`: clears context and localStorage wallet key
  - [ ] 15.5 Show truncated public key (`GXXX...YYYY`) in Navbar when connected
  - [ ] 15.6 Show network badge (Testnet/Mainnet) in header
  - [ ] 15.7 Guard contract-mutating pages with `<RequireWallet>` wrapper component that shows connect prompt if not connected
  - **Requirement**: REQ-8

---

## Phase 9: Wager Dashboard and Detail

- [ ] 16. Implement wager dashboard
  - [ ] 16.1 Create `src/lib/contracts.ts`: WagerContract client wrappers using Stellar SDK (`getWager`, `getPendingWagers`, `getActiveWagers`, `getParticipantWagers`)
  - [ ] 16.2 Create `src/hooks/useWagers.ts`: fetches and caches wager list with 30-second auto-refresh
  - [ ] 16.3 Create `src/components/wager/WagerCard.tsx`: event_metadata, category badge, stake, state badge, time ago, participants (truncated)
  - [ ] 16.4 Create filter sidebar: checkboxes for WagerState, EventCategory; token type dropdown; stake range inputs
  - [ ] 16.5 Implement client-side search filtering by event_metadata text
  - [ ] 16.6 Implement pagination (20 wagers per page) with prev/next controls
  - [ ] 16.7 Create `src/pages/Dashboard.tsx` composing stats bar + filter sidebar + wager grid
  - [ ] 16.8 Add stats bar: total active wagers, total volume (XLM/USDC), your open wagers count
  - **Requirement**: REQ-9

- [ ] 17. Implement wager detail page
  - [ ] 17.1 Create `src/pages/WagerDetail.tsx`: full wager info, state timeline, escrow balance, action buttons
  - [ ] 17.2 Show Deposit button if connected wallet is a participant and hasn't deposited
  - [ ] 17.3 Show Cancel button if wager is Pending and caller is a participant
  - [ ] 17.4 Show Raise Dispute button if wager is Disputed and caller is a participant and dispute window is open
  - [ ] 17.5 Show Finalize button if wager is Disputed, no dispute raised, and window has elapsed
  - [ ] 17.6 All action buttons call contract via Freighter-signed transaction; show loading state and success/error toast
  - **Requirement**: REQ-9 (AC-9.6)

---

## Phase 10: Create Wager Flow

- [ ] 18. Implement create wager form
  - [ ] 18.1 Create `src/pages/CreateWager.tsx` with form fields: opponent address, stake amount, token selector, event category, event metadata, event ID
  - [ ] 18.2 Implement client-side validation: stake > 0, event ID non-empty ≤ 64 chars, metadata ≤ 256 chars, opponent is valid Stellar address
  - [ ] 18.3 On submit: build and sign transaction via Freighter, submit to network, show success with wager ID and shareable link
  - [ ] 18.4 Map contract error codes to human-readable messages (e.g., `DuplicateEventId` → "This event ID has already been used for another wager")
  - [ ] 18.5 Add character counter on event_metadata input field
  - **Requirement**: REQ-10

---

## Phase 11: Profile Page

- [ ] 19. Implement user profile page
  - [ ] 19.1 Create `src/pages/Profile.tsx`: fetch participant's wager history via `getParticipantWagers`
  - [ ] 19.2 Compute and display stats: total wagers, wins, losses, draws, total volume per token, win rate %
  - [ ] 19.3 Create `src/components/profile/WagerHistory.tsx`: sortable table (date, event, stake, state, outcome)
  - [ ] 19.4 Support `/profile/:address` for viewing other wallets' profiles (read-only, no action buttons)
  - [ ] 19.5 Create `src/lib/export.ts` with `exportCSV(wagers)` and `exportJSON(wagers)` functions using `Blob` and `URL.createObjectURL`
  - [ ] 19.6 Add Export CSV and Export JSON buttons with correct filenames (`JEJEJO-history-{addr}-{date}.csv`)
  - **Requirement**: REQ-11, REQ-15

---

## Phase 12: Analytics Dashboard

- [ ] 20. Implement analytics dashboard
  - [ ] 20.1 Create `src/lib/indexer.ts`: HTTP client for off-chain indexer API (`/api/stats`, `/api/analytics/daily`, `/api/analytics/categories`)
  - [ ] 20.2 Create `src/pages/Analytics.tsx` with stats bar (total wagers, volume, resolution rate, dispute rate)
  - [ ] 20.3 Create `src/components/analytics/CategoryChart.tsx`: horizontal bar chart of wager count by EventCategory using Recharts
  - [ ] 20.4 Create `src/components/analytics/VolumeChart.tsx`: line chart of daily wager creation over last 30 days using Recharts
  - [ ] 20.5 All charts have accessible `aria-label` and data table fallback for screen readers
  - **Requirement**: REQ-12

---

## Phase 13: Notifications

- [ ] 21. Implement notification system
  - [ ] 21.1 Create `src/hooks/useNotifications.ts`: reads/writes notification list from localStorage keyed by wallet address
  - [ ] 21.2 Define notification types: `wager_invitation`, `deposit_reminder`, `wager_activated`, `result_submitted`, `dispute_raised`, `wager_completed`
  - [ ] 21.3 Poll for new wager events every 30 seconds and generate notifications for the connected wallet's wagers
  - [ ] 21.4 Create `src/components/notifications/NotificationBell.tsx`: bell icon with unread count badge (scale-in animation on new notification)
  - [ ] 21.5 Create `src/components/notifications/NotificationList.tsx`: dropdown list with mark-as-read and clear-all; clicking navigates to wager detail
  - **Requirement**: REQ-13

---

## Phase 14: Theming

- [ ] 22. Implement dark/light mode
  - [ ] 22.1 Create `src/context/ThemeContext.tsx`: state for `theme` (`dark` | `light`); reads from localStorage; falls back to `prefers-color-scheme`
  - [ ] 22.2 Apply theme class to `<html>` element; Tailwind's `darkMode: 'class'` strategy
  - [ ] 22.3 Implement theme toggle button in Navbar using a sun/moon icon
  - [ ] 22.4 Add CSS transition on `body` (`transition: background-color 200ms, color 200ms`)
  - [ ] 22.5 Audit all components for dark mode classes; ensure no unstyled elements in either theme
  - **Requirement**: REQ-14

---

## Phase 15: Settings Page

- [ ] 23. Implement settings page
  - [ ] 23.1 Create `src/pages/Settings.tsx` with sections: Network (testnet/mainnet selector), Display (theme), Notifications (toggle per type)
  - [ ] 23.2 All settings read from and write to localStorage on change (no save button needed)
  - [ ] 23.3 Add "Reset to Defaults" button that clears all localStorage settings keys and reloads defaults
  - **Requirement**: REQ-16

---

## Phase 16: Accessibility and Responsive Design

- [ ] 24. Accessibility pass
  - [ ] 24.1 Add `aria-label` to all icon-only buttons (theme toggle, notification bell, hamburger menu, export buttons)
  - [ ] 24.2 Audit color contrast in both themes using browser dev tools; fix any failing elements
  - [ ] 24.3 Verify full keyboard navigation: Tab order, Enter/Space on buttons, Escape to close modals
  - [ ] 24.4 Add `aria-live="polite"` regions for form validation errors and transaction status messages
  - [ ] 24.5 Add `role="status"` to loading spinners and skeleton components
  - [ ] 24.6 Test with VoiceOver (macOS) or NVDA (Windows) and fix announced content issues
  - **Requirement**: REQ-17

- [ ] 25. Responsive design pass
  - [ ] 25.1 Verify Dashboard layout at 320px, 768px, 1024px, 1440px
  - [ ] 25.2 Ensure Navbar hamburger menu works correctly on mobile
  - [ ] 25.3 Convert WagerHistory table to card-stack layout on screens < 768px
  - [ ] 25.4 Verify all tap targets are ≥ 44×44px on mobile (use browser mobile emulation)
  - [ ] 25.5 Test charts are readable and scrollable on mobile
  - **Requirement**: REQ-18

---

## Phase 17: Off-chain Indexer

- [ ] 26.* Scaffold off-chain event indexer (optional for MVP)
  - [ ] 26.1 Create `indexer/` directory with `package.json` (Node.js, TypeScript)
  - [ ] 26.2 Implement event subscription using Stellar RPC `getEvents` for WagerContract
  - [ ] 26.3 Store events in SQLite with tables: `wagers`, `events`, `participants`
  - [ ] 26.4 Implement REST API endpoints: `/api/stats`, `/api/wagers`, `/api/profile/:address`, `/api/analytics/daily`, `/api/analytics/categories`
  - [ ] 26.5 Add `indexer/README.md` with setup and run instructions
  - **Requirement**: REQ-12 (AC-12.5)
