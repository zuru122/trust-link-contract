# TrustLink Contract — Architecture

## Overview

TrustLink is a Soroban smart contract on the Stellar network that implements a trustless escrow system between a **buyer** and one or more **payees**, mediated by a **resolver** in case of disputes. All state lives on-chain; no off-chain service is required for core escrow operations.

---

## Components

### 1. Contract Entry Points (`contracts/escrow/src/lib.rs`)

| Function | Caller | Description |
|---|---|---|
| `create_escrow` | First payee | Creates a new escrow in `Pending` state |
| `fund_escrow` | Buyer | Locks tokens into the contract, moves escrow to `Funded` |
| `confirm_delivery` | Buyer | Releases funds to seller on satisfied delivery |
| `raise_dispute` | Buyer | Moves escrow to `Disputed` with a 32-byte evidence hash |
| `resolve_dispute` | Resolver | Pays out to seller or refunds buyer based on dispute finding |
| `auto_release` | Anyone | Releases to seller once the shipping window has elapsed |
| `get_escrow` | Anyone | Read-only view of an escrow record |

### 2. Data Types (`EscrowData`)

```
EscrowData {
    seller:          Address        — party receiving funds on success
    buyer:           Option<Address>— set at fund time; None before funding
    resolver:        Address        — trusted third-party mediator
    token:           Address        — SEP-41 token contract address
    amount:          i128           — locked token amount
    shipping_window: u64            — seconds after funding before auto-release is allowed
    funded_at:       u64            — ledger timestamp recorded at fund time
    state:           EscrowState    — current lifecycle state
}
```

### 3. State Machine (`EscrowState`)

```mermaid
stateDiagram-v2
    [*] --> Pending: create_escrow (9 args)
    Pending --> Funded: fund_escrow
    Pending --> Canceled: cancel_escrow
    Funded --> Shipped: mark_shipped
    Funded --> Disputed: raise_dispute before dispute_deadline
    Shipped --> Disputed: raise_dispute before dispute_deadline
    Shipped --> Completed: confirm_delivery after dispute_deadline
    Funded --> Completed: auto_release after dispute_deadline + shipping window
    Shipped --> Completed: auto_release after dispute_deadline + shipping window
    Funded --> Completed: auto_release after delivered_at + release window
    Shipped --> Completed: auto_release after delivered_at + release window
    Disputed --> PendingFinalization: resolve_dispute
    PendingFinalization --> Disputed: appeal_dispute during appeal window
    PendingFinalization --> Completed: finalize_dispute(Release)
    PendingFinalization --> Refunded: finalize_dispute(Refund)
    Funded --> Expired: derived release eligibility
    Shipped --> Expired: derived release eligibility
    Expired --> Completed: auto_release
```

`Expired` is a derived eligibility condition, not a stored `EscrowState` in
the current contract. It is shown to make the time-based guard visible.

Valid transitions:

| From | To | Trigger |
|---|---|---|
| `Pending` | `Funded` | `fund_escrow` |
| `Funded` | `Completed` | `auto_release` after the dispute deadline and a release window |
| `Shipped` | `Completed` | `confirm_delivery` after the dispute deadline, or `auto_release` after a release window |
| `Funded` | `Disputed` | `raise_dispute` |
| `Shipped` | `Disputed` | `raise_dispute` |
| `Disputed` | `PendingFinalization` | `resolve_dispute` |
| `PendingFinalization` | `Disputed` | `appeal_dispute` during the appeal window |
| `PendingFinalization` | `Completed` or `Refunded` | `finalize_dispute` after the appeal window |

`Completed`, `Refunded`, and `Canceled` are terminal states. `Expired` is a
derived time condition and does not persist a separate state.

---

## Storage Layout

All storage uses Soroban **instance** storage (entries share the contract instance's TTL).

| `DataKey` | Type | Description |
|---|---|---|
| `EscrowCounter` | `u64` | Monotonically increasing counter; also the ID of the most-recently created escrow |
| `Escrow(id: u64)` | `EscrowData` | Full escrow record keyed by its numeric ID |

IDs start at `1`. The counter is read, incremented, and stored atomically inside `create_escrow`.

---

## Token Flow

TrustLink uses the [SEP-41](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md) token interface (`soroban_sdk::token::Client`).

```
fund_escrow:         buyer ──transfer──► contract
confirm_delivery:    contract ──transfer──► seller
auto_release:        contract ──transfer──► seller
resolve_dispute:     contract ──transfer──► seller  (if release_to_seller)
                     contract ──transfer──► buyer   (if refund)
```

The contract never holds more than one escrow's tokens per escrow ID. Multiple concurrent escrows each lock their own `amount` independently.

---

## Evidence Hash

`raise_dispute` accepts an `evidence_hash: Bytes` parameter that must be **exactly 32 bytes** (a SHA-256 digest of off-chain evidence). The hash is validated before any state change and emitted in the `raise_dispute` event for off-chain indexers.

---

## Events

| Topic | Data | Emitted by |
|---|---|---|
| `("create_escrow",)` | `escrow_id: u32` | `create_escrow` |
| `("fund_escrow",)` | `escrow_id: u32` | `fund_escrow` |
| `("confirm_delivery",)` | `escrow_id: u32` | `confirm_delivery` |
| `("raise_dispute",)` | `(escrow_id: u32, evidence_hash: Bytes)` | `raise_dispute` |
| `("resolve_dispute",)` | `(escrow_id: u32, release_to_seller: bool)` | `resolve_dispute` |
| `("auto_release",)` | `escrow_id: u32` | `auto_release` |

---

## Authorization Model

| Operation | Who must sign |
|---|---|
| `create_escrow` | `seller` |
| `fund_escrow` | `buyer` |
| `confirm_delivery` | `buyer` (retrieved from stored `EscrowData`) |
| `raise_dispute` | `buyer` (retrieved from stored `EscrowData`) |
| `resolve_dispute` | `resolver` (retrieved from stored `EscrowData`) |
| `auto_release` | No auth required — permissionless after window expires |
| `get_escrow` | No auth required — read-only |

---

## Cross-Contract Interactions

TrustLink calls one external contract: the **token contract** at `EscrowData.token`. All token interactions use `soroban_sdk::token::Client`, which conforms to the SEP-41 interface. No other cross-contract calls are made.

## Resolver schemes

See [docs/resolvers.md](docs/resolvers.md) and
[MULTI_RESOLVER_SCHEME.md](MULTI_RESOLVER_SCHEME.md) for single, M-of-N, and
fallback resolver behavior, including deadline gating and deadlock risks.

---

## Workspace Structure

```
trust-link-contract/
├── Cargo.toml                     — workspace manifest
├── ARCHITECTURE.md                — this file
├── README.md
└── contracts/
    └── escrow/
        ├── Cargo.toml
        └── src/
            ├── lib.rs             — contract implementation
            └── test.rs            — unit and integration tests
```
