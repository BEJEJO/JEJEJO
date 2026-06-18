# JEJEJO Wager Protocol — Requirements Document

## Introduction

JEJEJO is a trustless prediction market and wager protocol built on Stellar Soroban. It transforms the Checkmate-Escrow chess wagering foundation into a general-purpose, two-party and multi-party wagering platform that supports any verifiable real-world event — sports, finance, esports, and custom categories.

JEJEJO preserves the battle-tested escrow lifecycle state machine, token allowlist, pause/unpause admin, two-step admin transfer, TTL extension, and oracle/result-store patterns from Checkmate-Escrow, while introducing a multi-oracle registry, event category metadata, multi-party pools, dispute windows, a React/TypeScript frontend with Freighter wallet integration, an analytics dashboard, user profiles, push-style notifications, dark/light mode UI, and CSV/JSON data export.

**Attribution**: This product is derived from Checkmate-Escrow, copyright (c) 2026 StellarCheckMate Contributors, licensed under the MIT License. All source files must retain the original MIT license header and acknowledge StellarCheckMate Contributors.

---

## Glossary

- **JEJEJO**: The product described in this document; a trustless prediction market and wager protocol on Stellar Soroban.
- **Protocol**: The set of Soroban smart contracts (WagerContract, OracleRegistry, OracleContract) that enforce wager rules on-chain.
- **Wager**: A stake agreement between two participants on the outcome of a real-world Event; replaces the chess-specific term "match".
- **Participant**: A Stellar account that has joined a Wager by depositing the required Stake.
- **Creator**: The Participant who calls `create_wager` and defines the Wager terms.
- **Pool**: The total tokens held in escrow for a Wager (sum of all Participant stakes).
- **Stake**: The token amount each Participant must deposit to activate a Wager.
- **Event**: A real-world occurrence (sports result, asset price, esports match, etc.) whose outcome resolves a Wager.
- **EventCategory**: A classification label for Events (`Sports`, `Finance`, `Esports`, `Custom`).
- **EventMetadata**: A string field attached to a Wager describing the underlying Event in human-readable form.
- **WagerState**: The lifecycle state of a Wager: `Pending`, `Active`, `Disputed`, `Completed`, or `Cancelled`.
- **OracleRegistry**: The Soroban contract that maintains the authorized set of oracle addresses per EventCategory.
- **OracleProvider**: An off-chain service registered in the OracleRegistry that is authorized to submit results for a specific EventCategory.
- **OracleContract**: A Soroban contract (one per OracleProvider) that stores verified Event results on-chain.
- **DisputeWindow**: The time period (in ledgers) after an oracle submits a result during which any Participant may raise a Dispute.
- **Dispute**: A formal on-chain challenge raised by a Participant within the DisputeWindow alleging the submitted result is incorrect.
- **Admin**: The privileged Stellar account that controls protocol-level configuration.
- **Frontend**: The React/TypeScript web application providing the JEJEJO user interface.
- **UserProfile**: A per-wallet view of a Participant's wager history, win rate, and volume.
- **Freighter**: The Stellar browser wallet extension used for transaction signing.
- **Token**: A Stellar SEP-41 token (XLM, USDC, or other allowlisted token) used as Stake currency.
- **Allowlist**: The set of Token addresses approved by the Admin for use in new Wagers.
- **TTL**: Time-to-live — ledgers before a Soroban storage entry expires.
- **Checkmate_Escrow**: The original open-source project from which JEJEJO is derived (MIT, StellarCheckMate Contributors).

---

## Requirements

### REQ-1: Attribution and License Preservation

**User Story**: As a user of JEJEJO, I want the project to clearly credit its open-source origins so that license obligations are transparently met.

**Acceptance Criteria**:
- AC-1.1: A `NOTICE` file exists at the repository root crediting "Checkmate-Escrow, copyright (c) 2026 StellarCheckMate Contributors" and linking to the original MIT license.
- AC-1.2: The `LICENSE` file retains the original MIT copyright notice for StellarCheckMate Contributors and adds JEJEJO contributors.
- AC-1.3: All Rust source files that contain substantial code derived from Checkmate-Escrow carry a `// Originally derived from Checkmate-Escrow (MIT) — StellarCheckMate Contributors` comment at the top.
- AC-1.4: The `README.md` includes an Acknowledgements section referencing the original project.

---

### REQ-2: Codebase Refactor — Rename and Restructure

**User Story**: As a developer, I want the codebase to reflect JEJEJO's identity and corrected architecture so that the code is maintainable and free of known bugs.

**Acceptance Criteria**:
- AC-2.1: The workspace is renamed from `Checkmate-Escrow` to `jejejo`.
- AC-2.2: The `contracts/escrow` crate is renamed to `contracts/wager` with the contract struct renamed to `WagerContract`.
- AC-2.3: The `contracts/oracle` crate is renamed to `contracts/oracle-store` with the contract struct renamed to `OracleStoreContract`.
- AC-2.4: A new `contracts/oracle-registry` crate is added containing the `OracleRegistry` contract.
- AC-2.5: The duplicate `DataKey::AllowedTokenCount` variant in `types.rs` is removed so the enum compiles without error.
- AC-2.6: `initialize()` in `WagerContract` returns `Result<(), Error>` and returns `Err(Error::AlreadyInitialized)` on re-init instead of silently returning.
- AC-2.7: `is_token_allowed` reads from `instance` storage (consistent with `add_allowed_token` which writes to `instance`).
- AC-2.8: `submit_result_with_oracle_record` passes the `caller` address to `submit_result` correctly.
- AC-2.9: Dead code (unused imports, unreachable arms) is removed from all contract files.
- AC-2.10: All existing tests pass after renaming, with no test regressions.

---

### REQ-3: Generalized Wager Lifecycle

**User Story**: As a user, I want to create a wager on any real-world event — not just chess — so that JEJEJO can be used for sports, esports, finance, and custom outcomes.

**Acceptance Criteria**:
- AC-3.1: `create_wager` accepts `participant1`, `participant2`, `stake_amount`, `token`, `event_id`, `event_category` (enum: Sports/Finance/Esports/Custom), and `event_metadata` (string, max 256 bytes).
- AC-3.2: `event_id` uniqueness is enforced on-chain; a duplicate returns `Error::DuplicateEventId`.
- AC-3.3: `event_metadata` max length of 256 bytes is enforced; exceeding it returns `Error::InvalidMetadata`.
- AC-3.4: The wager lifecycle states are `Pending → Active → Disputed → Completed` and `Pending → Cancelled`, matching the original escrow state machine extended with `Disputed`.
- AC-3.5: All existing lifecycle operations (`deposit`, `cancel_wager`, `expire_wager`, `submit_result`) work with the renamed types.
- AC-3.6: Events emitted use the namespace `wager` instead of `match` (e.g., `wager/created`, `wager/activated`, `wager/completed`, `wager/cancelled`, `wager/expired`).

---

### REQ-4: Multi-Oracle Registry

**User Story**: As an admin, I want to register different oracle providers for different event categories so that no single oracle is a universal point of failure.

**Acceptance Criteria**:
- AC-4.1: An `OracleRegistry` contract exists with `register_oracle(category, oracle_address)` and `remove_oracle(category, oracle_address)` — both admin-only.
- AC-4.2: `get_oracles_for_category(category)` returns the list of authorized oracle addresses for that category.
- AC-4.3: `is_oracle_authorized(category, oracle_address)` returns a bool — callable by anyone.
- AC-4.4: The `WagerContract` stores the `OracleRegistry` address at initialization instead of a single oracle address.
- AC-4.5: `submit_result` on `WagerContract` checks with `OracleRegistry.is_oracle_authorized(wager.event_category, caller)` before accepting a result.
- AC-4.6: Registering the same oracle for the same category twice is idempotent (no error, no duplicate).
- AC-4.7: Removing an oracle that is not registered returns `Error::OracleNotFound`.

---

### REQ-5: Dispute Window

**User Story**: As a participant, I want a time window after a result is submitted during which I can raise a dispute so that incorrect oracle results can be challenged before funds are paid out.

**Acceptance Criteria**:
- AC-5.1: After `submit_result` is called, the wager transitions to `Disputed` state (not immediately `Completed`) and a `dispute_deadline_ledger` is set to `current_ledger + DISPUTE_WINDOW_LEDGERS` (default: ~1 hour = 720 ledgers at 5s/ledger).
- AC-5.2: Any Participant of the wager may call `raise_dispute(wager_id)` during the dispute window; this stores the dispute on-chain and emits `wager/disputed`.
- AC-5.3: If no dispute is raised and the dispute window elapses, anyone may call `finalize_wager(wager_id)` to execute the payout and transition to `Completed`.
- AC-5.4: If a dispute is raised, the Admin resolves it via `resolve_dispute(wager_id, final_winner)` — admin only — which executes payout and transitions to `Completed`.
- AC-5.5: `raise_dispute` returns `Error::DisputeWindowClosed` if called after `dispute_deadline_ledger`.
- AC-5.6: `raise_dispute` returns `Error::AlreadyDisputed` if a dispute has already been raised for this wager.
- AC-5.7: `DISPUTE_WINDOW_LEDGERS` is configurable by the admin via `set_dispute_window(ledgers)`.

---

### REQ-6: Token Allowlist (Preserved and Fixed)

**User Story**: As an admin, I want to control which tokens can be staked in new wagers so that only audited and trusted assets are used.

**Acceptance Criteria**:
- AC-6.1: `add_allowed_token` and `remove_allowed_token` remain admin-only and update `instance` storage consistently (no storage tier mismatch).
- AC-6.2: `is_token_allowed` reads from the same `instance` storage tier that `add_allowed_token` writes to.
- AC-6.3: When no tokens are on the allowlist, any SEP-41 token is accepted (permissive default preserved).
- AC-6.4: Once at least one token is on the allowlist, only allowlisted tokens are accepted for new wagers.
- AC-6.5: `get_allowed_tokens` returns the current allowlist as an ordered vector.

---

### REQ-7: Admin Controls (Preserved and Extended)

**User Story**: As an admin, I want full emergency and configuration controls so that the protocol can be safely operated and upgraded.

**Acceptance Criteria**:
- AC-7.1: `pause` and `unpause` block/unblock `create_wager`, `deposit`, and `submit_result` when the contract is paused.
- AC-7.2: `cancel_wager` and `expire_wager` remain functional while the contract is paused.
- AC-7.3: Two-step admin transfer (`propose_admin` / `accept_admin`) is preserved.
- AC-7.4: `set_wager_timeout(ledgers)` allows the admin to configure the pending wager expiration window.
- AC-7.5: `set_dispute_window(ledgers)` allows the admin to configure the dispute window duration.
- AC-7.6: All admin operations emit events in the `admin` namespace.

---

### REQ-8: Frontend — Wallet Authentication

**User Story**: As a user, I want to connect my Freighter wallet to JEJEJO so that I can sign transactions without sharing my private key.

**Acceptance Criteria**:
- AC-8.1: The frontend has a "Connect Wallet" button that invokes the Freighter browser extension API.
- AC-8.2: Once connected, the user's Stellar public key is displayed (truncated) in the navigation bar.
- AC-8.3: All contract-mutating actions (create wager, deposit, cancel, raise dispute) require a connected wallet; unauthenticated users are shown a connect prompt.
- AC-8.4: "Disconnect Wallet" clears the session and returns the UI to the unauthenticated state.
- AC-8.5: The network (testnet/mainnet) is clearly displayed in the UI header.
- AC-8.6: Wallet connection state persists across page refreshes using localStorage.

---

### REQ-9: Frontend — Wager Dashboard

**User Story**: As a user, I want a dashboard that shows all open and recent wagers so that I can quickly find wagers to join or monitor.

**Acceptance Criteria**:
- AC-9.1: The dashboard displays a paginated list of wagers filterable by state (Pending, Active, Disputed, Completed, Cancelled).
- AC-9.2: Each wager card shows: event_metadata, event_category badge, stake amount, token symbol, state, and time since creation.
- AC-9.3: Wagers can be filtered by event category (Sports, Finance, Esports, Custom).
- AC-9.4: Wagers can be filtered by token type (XLM, USDC, etc.).
- AC-9.5: A search input filters wagers by event_metadata text (client-side).
- AC-9.6: Clicking a wager card opens a detail page with full wager information and available actions.
- AC-9.7: The dashboard auto-refreshes wager states every 30 seconds.

---

### REQ-10: Frontend — Create Wager Flow

**User Story**: As a user, I want a guided form to create a new wager so that I don't need to interact with the contract directly.

**Acceptance Criteria**:
- AC-10.1: A "Create Wager" form collects: opponent address, stake amount, token selection, event category, event metadata, and event ID.
- AC-10.2: Client-side validation surfaces errors before transaction submission (stake > 0, event ID non-empty, metadata ≤ 256 chars, opponent address valid).
- AC-10.3: After form submission, the transaction is signed via Freighter and submitted to the network.
- AC-10.4: Success state shows the new wager ID and a shareable link.
- AC-10.5: Error state shows a human-readable error message mapping contract error codes to plain English.

---

### REQ-11: Frontend — User Profile

**User Story**: As a user, I want to view my wager history and statistics so that I can track my performance on JEJEJO.

**Acceptance Criteria**:
- AC-11.1: The profile page is accessible at `/profile` and displays the connected wallet's wager history.
- AC-11.2: Statistics shown: total wagers, wins, losses, draws, total volume (per token), win rate (%).
- AC-11.3: Wager history is displayed in a sortable/filterable table (by date, state, stake amount).
- AC-11.4: Clicking a wager row navigates to the wager detail page.
- AC-11.5: Profile is view-only for disconnected users visiting `/profile/:address`.

---

### REQ-12: Frontend — Analytics Dashboard

**User Story**: As a user, I want to see platform-wide analytics so that I can understand JEJEJO's activity and health.

**Acceptance Criteria**:
- AC-12.1: The analytics page shows: total wagers created, total volume (per token), active wager count, resolution rate (completed / total), and dispute rate.
- AC-12.2: A bar chart shows wager count by event category.
- AC-12.3: A line chart shows daily wager creation over the last 30 days (indexed from contract events).
- AC-12.4: All charts are rendered using a client-side charting library (Recharts or Chart.js).
- AC-12.5: Analytics data is fetched from a lightweight off-chain indexer that reads contract events.

---

### REQ-13: Frontend — Notifications

**User Story**: As a participant, I want to receive in-app notifications when my wager state changes so that I never miss a deposit deadline or payout.

**Acceptance Criteria**:
- AC-13.1: A notification bell icon in the navigation bar shows an unread count badge.
- AC-13.2: Notifications are generated for: wager invitation (you are participant2 in a new wager), deposit reminder (you haven't deposited yet), wager activated (both deposited), result submitted (dispute window open), dispute raised, wager completed (payout executed).
- AC-13.3: Notifications are stored in localStorage per wallet address.
- AC-13.4: Clicking a notification navigates to the relevant wager detail page.
- AC-13.5: Notifications can be marked as read individually or all at once.

---

### REQ-14: Frontend — Dark/Light Mode

**User Story**: As a user, I want to toggle between dark and light mode so that the UI suits my preference and environment.

**Acceptance Criteria**:
- AC-14.1: A toggle button in the navigation bar switches between dark and light themes.
- AC-14.2: Theme preference is persisted in localStorage and restored on next visit.
- AC-14.3: The system's `prefers-color-scheme` media query is respected as the default if no preference is stored.
- AC-14.4: All UI components (cards, tables, charts, modals, forms) are fully styled in both themes with no unstyled elements.
- AC-14.5: Theme transition uses a CSS transition (200ms) to avoid jarring flips.

---

### REQ-15: Frontend — Data Export

**User Story**: As a user, I want to export my wager history so that I can use it for personal records or tax reporting.

**Acceptance Criteria**:
- AC-15.1: A "Export CSV" button on the profile page downloads a `.csv` file of the user's wager history.
- AC-15.2: A "Export JSON" button downloads a `.json` file of the same data.
- AC-15.3: Exported files include: wager_id, event_metadata, event_category, stake_amount, token, state, outcome, created_date, completed_date.
- AC-15.4: File names are `jejejo-history-{address}-{date}.csv` / `.json`.
- AC-15.5: Export works client-side with no server dependency (using Blob/URL.createObjectURL).

---

### REQ-16: Frontend — Settings Page

**User Story**: As a user, I want a settings page so that I can configure my preferences for network, notifications, and display.

**Acceptance Criteria**:
- AC-16.1: A settings page is accessible at `/settings`.
- AC-16.2: Settings include: network selection (testnet/mainnet), default token preference, notification toggles (per notification type), and theme preference.
- AC-16.3: All settings are persisted in localStorage.
- AC-16.4: A "Reset to Defaults" button clears all stored settings.

---

### REQ-17: Frontend — Accessibility

**User Story**: As a user with accessibility needs, I want JEJEJO's frontend to be accessible so that I can use it with assistive technologies.

**Acceptance Criteria**:
- AC-17.1: All interactive elements (buttons, links, inputs) have descriptive `aria-label` or visible text labels.
- AC-17.2: Color contrast ratios meet WCAG 2.1 AA minimums (4.5:1 for normal text, 3:1 for large text) in both themes.
- AC-17.3: Full keyboard navigation is supported — all interactive elements are reachable and operable via Tab/Enter/Space/Escape.
- AC-17.4: Focus indicators are visible and meet WCAG 2.1 AA contrast requirements.
- AC-17.5: All images and icons that convey meaning have `alt` text or `aria-label`.
- AC-17.6: The frontend is navigable and usable with a screen reader (VoiceOver/NVDA).
- AC-17.7: Form validation errors are announced to screen readers via `aria-live` regions.

---

### REQ-18: Frontend — Responsive Design

**User Story**: As a mobile user, I want JEJEJO to be fully usable on small screens so that I can manage wagers from my phone.

**Acceptance Criteria**:
- AC-18.1: The layout is responsive at breakpoints: 320px (mobile), 768px (tablet), 1024px (desktop), 1440px (wide).
- AC-18.2: Navigation collapses to a hamburger menu on mobile.
- AC-18.3: Wager cards stack vertically on mobile.
- AC-18.4: Tables switch to card-style stacked layout on screens < 768px.
- AC-18.5: All tap targets are at least 44×44px on mobile.

---

### REQ-19: Documentation

**User Story**: As a developer, I want comprehensive documentation so that I can understand, set up, and contribute to JEJEJO.

**Acceptance Criteria**:
- AC-19.1: `README.md` describes JEJEJO's purpose, quick start (build, test, deploy), environment variables, and links to all docs.
- AC-19.2: `docs/architecture.md` describes the three-contract architecture (WagerContract, OracleRegistry, OracleStoreContract), the frontend, and the off-chain indexer.
- AC-19.3: `docs/setup.md` provides step-by-step local development setup for contracts and frontend.
- AC-19.4: `CONTRIBUTING.md` describes branching strategy, commit conventions, PR process, and code style.
- AC-19.5: `CHANGELOG.md` records changes with version, date, and description starting from v1.0.0 (Checkmate-Escrow base) through the JEJEJO transformation.
- AC-19.6: `NOTICE` file credits the original Checkmate-Escrow project.

---

## Correctness Properties (for Property-Based Testing)

These properties must hold for all valid inputs and must be testable via property-based tests in the Rust contract test suite.

### P-1: Fund Conservation
For any sequence of `create_wager`, `deposit`, `cancel_wager`, `expire_wager`, `finalize_wager`, and `resolve_dispute` operations, the sum of token balances across all participants and the contract address must equal the initial total supply allocated to those participants. No tokens are created or destroyed.

### P-2: Wager State Machine Monotonicity
A wager's state transitions are monotone and irreversible: `Pending → Active → Disputed → Completed` and `Pending → Cancelled`. No wager may transition backward or skip states (e.g., `Pending → Completed` directly is invalid).

### P-3: Double Deposit Prevention
No participant may deposit more than once for the same wager. Calling `deposit` twice for the same (wager_id, participant) pair must always return `Error::AlreadyFunded`.

### P-4: Unauthorized Result Rejection
`submit_result` must reject any caller whose address is not authorized in the OracleRegistry for the wager's event category. This must hold for all wagers regardless of category or state.

### P-5: Dispute Window Enforcement
`raise_dispute` must succeed if and only if: (a) the wager is in `Disputed` state, (b) the caller is participant1 or participant2, and (c) the current ledger is ≤ `dispute_deadline_ledger`. All three conditions must be jointly necessary.

### P-6: Payout Completeness
After `finalize_wager` or `resolve_dispute` transitions a wager to `Completed`, `get_escrow_balance(wager_id)` must return 0 and the winning participant(s) must have received exactly `stake_amount * 2` (winner-takes-all) or `stake_amount` each (draw/refund).

### P-7: Event ID Uniqueness
No two wagers may share the same `event_id`. Attempting to create a second wager with a duplicate `event_id` must always return `Error::DuplicateEventId`, regardless of the state of the first wager.

### P-8: Pause Completeness
While the contract is paused, `create_wager`, `deposit`, and `submit_result` must all return `Error::ContractPaused` for any valid input. Read operations and cancellation must remain unaffected.

### P-9: Token Conservation on Cancel/Expire
When `cancel_wager` or `expire_wager` is called, each participant receives back exactly `stake_amount` if and only if they had previously deposited. Participants who had not deposited receive nothing.

### P-10: Oracle Registry Authorization Consistency
`is_oracle_authorized(category, address)` must return `true` if and only if the address was registered via `register_oracle(category, address)` and has not been subsequently removed via `remove_oracle(category, address)`.
