# JEJEJO — Trustless Wager Protocol on Stellar

A trustless prediction market and wager protocol built on Stellar Soroban smart contracts. Participants stake tokens before an event, and the winner is automatically paid out the moment the result is verified on-chain — no middleman, no delays, no trust required.


## 🎯 What is JEJEJO?

JEJEJO combines real-world events with Stellar's fast settlement to create a fully on-chain wagering platform for sports, finance, esports, and custom outcomes.

Participants:

- Stake tokens into a Soroban escrow contract before an event begins
- Receive automatic payouts the instant the result is verified on-chain

A custom Oracle bridges external data sources to the smart contract, verifying event results and triggering payouts without any manual intervention.

This makes JEJEJO:

✅ Trustless (no platform can withhold or delay winnings)  
✅ Transparent (all stakes and payouts are verifiable on-chain)  
✅ Instant (Stellar's fast finality means payouts settle in seconds)  
✅ Accessible (anyone with a Stellar wallet can participate)

## 🚀 Features

- **Create a Wager**: Set stake amount, token address, event category, and event metadata
- **Flexible Token Support**: Any Stellar token address is accepted by default; once the admin adds at least one token via `add_allowed_token`, only allowlisted tokens are accepted for new wagers
- **Escrow Stakes**: Both participants deposit funds into the contract before the event starts
- **Oracle Integration**: Real-time result verification via category-specific oracle providers
- **Automatic Payouts**: Winner receives the full pot the moment the result is confirmed
- **Draw Handling**: Stakes are returned to both participants in the event of a draw
- **Admin Controls**: Pause/unpause, oracle registry, admin transfer, and wager timeout configuration
- **Transparent**: All escrow balances and payout history are verifiable on-chain

## 🗺️ Wager Lifecycle

Wagers move through the following states:

```
Pending ──► Active ──► Disputed ──► Completed
   │                                    ▲
   └──► Cancelled ◄─────────────────────
         (expire_wager / cancel_wager)
```

| State       | Description                                                  |
|-------------|--------------------------------------------------------------|
| `Pending`   | Wager created; awaiting deposits from both participants      |
| `Active`    | Both participants have deposited; awaiting oracle result     |
| `Disputed`  | Oracle result submitted; in dispute window                   |
| `Completed` | Finalized; payout executed                                   |
| `Cancelled` | Cancelled before activation, or expired after timeout        |

### Events Reference

| Topic (namespace / name)  | Emitted by          | Payload                                              |
|---------------------------|---------------------|------------------------------------------------------|
| `wager` / `init`          | `initialize`        | `(oracle_registry, admin_address)`                   |
| `admin` / `paused`        | `pause`             | `()`                                                 |
| `admin` / `unpaused`      | `unpause`           | `()`                                                 |
| `admin` / `xfer`          | `transfer_admin`    | `(old_admin, new_admin)`                             |
| `wager` / `created`       | `create_wager`      | `(wager_id, participant1, participant2, stake_amount)`|
| `wager` / `completed`     | `finalize_wager`    | `(wager_id, outcome)`                                |
| `wager` / `cancelled`     | `cancel_wager`      | `wager_id`                                           |
| `wager` / `expired`       | `expire_wager`      | `wager_id`                                           |
| `wager` / `disputed`      | `raise_dispute`     | `wager_id`                                           |

## 🛠️ Quick Start

### Prerequisites

- Rust (1.70+)
- Soroban CLI
- Stellar CLI

### Build

```bash
./scripts/build.sh
```

### Test

```bash
./scripts/test.sh
```

### Setup Environment

Copy the example environment file:

```bash
cp .env.example .env
```

Configure your environment variables in `.env`:

```env
# Network configuration
STELLAR_NETWORK=testnet
STELLAR_RPC_URL=https://soroban-testnet.stellar.org

# Contract addresses (after deployment)
CONTRACT_WAGER=<your-contract-id>
CONTRACT_ORACLE_REGISTRY=<your-contract-id>
CONTRACT_ORACLE_STORE=<your-contract-id>

# Frontend configuration
VITE_STELLAR_NETWORK=testnet
VITE_STELLAR_RPC_URL=https://soroban-testnet.stellar.org
```

Network configurations are defined in `environments.toml`:

- `testnet` — Stellar testnet
- `mainnet` — Stellar mainnet
- `futurenet` — Stellar futurenet
- `standalone` — Local development

### Deploy to Testnet

```bash
# Configure your testnet identity first
stellar keys generate deployer --network testnet

# Deploy
./scripts/deploy_testnet.sh
```

### Run Demo

Follow the step-by-step guide in `demo/demo-script.md`

## 📖 Documentation

- [Architecture Overview](docs/architecture.md)
- [Oracle Design](docs/oracle.md)
- [Threat Model & Security](docs/security.md)
- [Roadmap](docs/roadmap.md)
- [Deployment Guide](docs/deployment.md)

## 🎓 Smart Contract API

### Wager Management

```
create_wager(participant1, participant2, stake_amount, token, event_id, event_category, event_metadata) -> u64
get_wager(wager_id) -> Wager
cancel_wager(wager_id, caller) -> Result<(), Error>
expire_wager(wager_id) -> Result<(), Error>
get_participant_wagers(participant) -> Vec<u64>
get_pending_wagers() -> Vec<Wager>
get_active_wagers() -> Vec<Wager>
```

- `create_wager` must be authorized by `participant1`.
- `cancel_wager` may be called by either participant.
- `expire_wager` may be called by anyone once the wager timeout elapses.

### Escrow

```
deposit(wager_id, participant) -> Result<(), Error>
is_funded(wager_id) -> Result<bool, Error>
get_escrow_balance(wager_id) -> Result<i128, Error>
```

#### `is_funded` vs `get_escrow_balance`

These two functions answer different questions and are easy to confuse:

- **`is_funded(wager_id)`** — returns `true` only when *both* participants have deposited their stake (i.e. the wager has transitioned to `Active`). It reflects deposit flags, not token balances. Use this to gate event-start logic.

- **`get_escrow_balance(wager_id)`** — returns the total token amount currently held in escrow for the wager: `0`, `1×stake`, or `2×stake` depending on how many participants have deposited. Once a wager is `Completed` or `Cancelled` (funds already paid out or refunded), this always returns `0` regardless of on-chain token balances.

Examples:

| Scenario                                    | `is_funded` | `get_escrow_balance` |
|---------------------------------------------|-------------|----------------------|
| Only participant1 deposited                 | `false`     | `1 × stake_amount`   |
| Both participants deposited (Active)        | `true`      | `2 × stake_amount`   |
| Wager completed (payout done)               | `true`      | `0`                  |
| Wager cancelled (refunds done)              | `false`     | `0`                  |

### Oracle & Payouts

```
submit_result(wager_id, outcome, caller) -> Result<(), Error>
submit_result_with_oracle_record(wager_id, outcome, event_id) -> Result<(), Error>
raise_dispute(wager_id, caller) -> Result<(), Error>
finalize_wager(wager_id) -> Result<(), Error>
resolve_dispute(wager_id, final_outcome) -> Result<(), Error>
```

- `submit_result` is called by an oracle address authorized in the OracleRegistry for the wager's event category.
- `submit_result_with_oracle_record` stores the verified `event_id` for audit.
- After `submit_result`, the wager enters `Disputed` state; payout executes after the dispute window via `finalize_wager`.
- Admin may override via `resolve_dispute` if a dispute was raised.

## 🧪 Testing

Comprehensive test suite covering:

✅ Wager creation and configuration  
✅ Deposit validation and escrow locking  
✅ Oracle result submission and verification  
✅ Winner payout and draw refund logic  
✅ Dispute window and resolution  
✅ Cancellation and edge cases  
✅ Error handling and security checks

Run tests:

```bash
cargo test
```

## 🌍 Why This Matters

**The Problem**: Current event wagering and prediction markets rely entirely on centralized platforms. Participants have no guarantee their winnings will be paid out fairly or on time.

**The Solution**: By holding stakes in a Soroban smart contract and automating payouts via a verified Oracle, JEJEJO removes the need to trust any third party.

**Blockchain Benefits**:

- No platform can withhold or manipulate payouts
- Transparent stake and payout history for every wager
- Programmable rules enforced by smart contracts
- Accessible to anyone with a Stellar wallet

**Target Users**:

- Sports fans and esports viewers looking for trustless wagering
- Prediction market participants
- Communities wanting on-chain, verifiable outcomes
- Developers building on Stellar/Soroban

## 🗺️ Roadmap

- **v1.0 (Current)**: Token-allowlist escrow, multi-oracle registry, dispute window, basic wager flow
- **v1.1**: Analytics dashboard, off-chain event indexer
- **v2.0**: Multi-party pools, bracket payouts
- **v3.0**: Frontend UI with Freighter wallet integration
- **v4.0**: Mobile app, leaderboards, user profiles

See [docs/roadmap.md](docs/roadmap.md) for details.

## 🤝 Contributing

We welcome contributions! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

See our [Code of Conduct](CODE_OF_CONDUCT.md) and [Contributing Guidelines](CONTRIBUTING.md).

## 🌊 Drips Wave Contributors

This project participates in Drips Wave — a contributor funding program! Check out:

- [Wave Contributor Guide](docs/wave-guide.md) — How to earn funding for contributions
- [Wave-Ready Issues](https://github.com/issues?q=label%3Awave-ready) — Funded issues ready to tackle
- GitHub Issues labeled with `wave-ready` — Earn 100–200 points per issue

Issues are categorized as:

- `trivial` (100 points) — Documentation, simple tests, minor fixes
- `medium` (150 points) — Oracle helpers, validation logic, moderate features
- `high` (200 points) — Core escrow logic, Oracle integrations, security enhancements

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [Stellar Development Foundation](https://stellar.org) for Soroban
- Drips Wave for supporting public goods funding
