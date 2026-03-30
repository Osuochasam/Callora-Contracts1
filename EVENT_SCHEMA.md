# Event Schema (Vault Contract)

Events emitted by the Callora vault contract for indexers and frontends. All topic/data types refer to Soroban/Stellar XDR values.

## Contract: Callora Vault

### `init`

Emitted when the vault is initialized.

| Field   | Location | Type   | Description           |
|---------|----------|--------|-----------------------|
| topic 0 | topics   | Symbol | `"init"`              |
| topic 1 | topics   | Address| vault owner           |
| data    | data     | i128   | initial balance       |

---

### `deposit`

Emitted when balance is increased via `deposit(amount)`.

| Field   | Location | Type   | Description   |
|---------|----------|--------|---------------|
| topic 0 | topics   | Symbol | `"deposit"`   |
| data    | data     | (i128, i128) | (amount, new_balance) |

---

### `deduct`

Emitted on each deduction: single `deduct(amount)` or each item in `batch_deduct(items)`.

| Field   | Location | Type   | Description   |
|---------|----------|--------|---------------|
| topic 0 | topics   | Symbol | `"deduct"`    |
| topic 1 | topics   | Address| caller        |
| topic 2 | topics   | Symbol | optional request_id (empty symbol if none) |
| data    | data     | (i128, i128) | (amount, new_balance) |

---

### `withdraw`

Emitted when the owner withdraws via `withdraw(amount)`.

| Field   | Location | Type   | Description   |
|---------|----------|--------|---------------|
| topic 0 | topics   | Symbol | `"withdraw"`  |
| topic 1 | topics   | Address| vault owner   |
| data    | data     | (i128, i128) | (amount, new_balance) |

---

### `withdraw_to`

Emitted when the owner withdraws to a designated address via `withdraw_to(to, amount)`.

| Field   | Location | Type   | Description   |
|---------|----------|--------|---------------|
| topic 0 | topics   | Symbol | `"withdraw_to"` |
| topic 1 | topics   | Address| vault owner   |
| topic 2 | topics   | Address| recipient `to` |
| data    | data     | (i128, i128) | (amount, new_balance) |

---

### `metadata_set`

Emitted when metadata is set for an offering via `set_metadata(offering_id, metadata)`.

| Field   | Location | Type   | Description   |
|---------|----------|--------|---------------|
| topic 0 | topics   | Symbol | `"metadata_set"` |
| topic 1 | topics   | String | offering_id   |
| topic 2 | topics   | Address| caller (owner/issuer) |
| data    | data     | String | metadata (IPFS CID or URI) |

---

### `metadata_updated`

Emitted when existing metadata is updated via `update_metadata(offering_id, metadata)`.

| Field   | Location | Type   | Description   |
|---------|----------|--------|---------------|
| topic 0 | topics   | Symbol | `"metadata_updated"` |
| topic 1 | topics   | String | offering_id   |
| topic 2 | topics   | Address| caller (owner/issuer) |
| data    | data     | (String, String) | (old_metadata, new_metadata) |

---

## Not yet implemented

- **OwnershipTransfer**: not present in current vault; would list old_owner, new_owner.
- **Pause**: not present in current vault; would indicate pause state change.

---

## Contract: Callora Settlement (`callora-settlement` v0.1.0)

### `payment_received`

Emitted by `receive_payment()` for every inbound payment regardless of `to_pool` mode.

| Field   | Location | Type    | Description                                                                 |
|---------|----------|---------|-----------------------------------------------------------------------------|
| topic 0 | topics   | Symbol  | `"payment_received"`                                                        |
| topic 1 | topics   | Address | `caller` — the vault or admin address that sent the payment                 |
| `from_vault` | data | Address | same as topic 1 (vault/admin caller)                                   |
| `amount`     | data | i128    | payment amount in USDC micro-units (stroops); always > 0                |
| `to_pool`    | data | bool    | `true` → credited to global pool; `false` → credited to a developer    |
| `developer`  | data | Option\<Address\> | `None` when `to_pool=true`; developer address when `to_pool=false` |

**Example — `to_pool = true` (global pool credit):**

```json
{
  "topics": ["payment_received", "GCALLER..."],
  "data": {
    "from_vault": "GCALLER...",
    "amount": 5000000,
    "to_pool": true,
    "developer": null
  }
}
```

**Example — `to_pool = false` (developer credit):**

```json
{
  "topics": ["payment_received", "GCALLER..."],
  "data": {
    "from_vault": "GCALLER...",
    "amount": 2500000,
    "to_pool": false,
    "developer": "GDEV..."
  }
}
```

---

### `balance_credited`

Emitted by `receive_payment()` **only** when `to_pool = false`. Follows the `payment_received` event for the same call.

| Field         | Location | Type    | Description                                          |
|---------------|----------|---------|------------------------------------------------------|
| topic 0       | topics   | Symbol  | `"balance_credited"`                                 |
| topic 1       | topics   | Address | `developer` — the address whose balance was updated  |
| `developer`   | data     | Address | same as topic 1                                      |
| `amount`      | data     | i128    | amount credited in this call (USDC micro-units)      |
| `new_balance` | data     | i128    | developer's cumulative balance after this credit     |

**Example:**

```json
{
  "topics": ["balance_credited", "GDEV..."],
  "data": {
    "developer": "GDEV...",
    "amount": 2500000,
    "new_balance": 7500000
  }
}
```

> **Note:** `balance_credited` is never emitted when `to_pool = true`. Indexers tracking developer earnings should subscribe to this event; indexers tracking total protocol revenue should subscribe to `payment_received` with `to_pool = true`.

### Version notes

| Version | Change |
|---------|--------|
| 0.1.0   | Initial settlement events: `payment_received`, `balance_credited` |

> If `PaymentReceivedEvent` or `BalanceCreditedEvent` structs gain new fields in future versions, a new row will be added here with the crate semver and a description of the added/changed fields.

---

## Contract: Callora Revenue Pool (`callora-revenue-pool`)

### `init`

Emitted when the revenue pool is initialized via `init(admin, usdc_token)`.

| Field   | Location | Type    | Description                        |
|---------|----------|---------|------------------------------------|
| topic 0 | topics   | Symbol  | `"init"`                           |
| topic 1 | topics   | Address | `admin` — the initial admin address |
| data    | data     | Address | `usdc_token` — USDC contract address |

---

### `distribute`

Emitted by `distribute(caller, to, amount)` after a successful USDC transfer from the contract to a recipient.

| Field   | Location | Type    | Description                                              |
|---------|----------|---------|----------------------------------------------------------|
| topic 0 | topics   | Symbol  | `"distribute"`                                           |
| topic 1 | topics   | Address | `to` — the recipient address that received the USDC      |
| data    | data     | i128    | `amount` — the amount transferred in USDC base units     |

**Access control:** only the stored admin may trigger this event.

**Pre-conditions checked before emit:**
- `amount > 0` (panics `"Amount must be positive"` otherwise)
- `contract_usdc_balance >= amount` (panics `"Insufficient contract balance"` otherwise)

**Example:**

```json
{
  "topics": ["distribute", "GRECIPIENT..."],
  "data": 5000000
}
```

---

### `batch_distribute`

Emitted once per payment entry by `batch_distribute(caller, payments)`. Each individual transfer emits its own event.

| Field   | Location | Type    | Description                                              |
|---------|----------|---------|----------------------------------------------------------|
| topic 0 | topics   | Symbol  | `"batch_distribute"`                                     |
| topic 1 | topics   | Address | `to` — the recipient for this payment entry              |
| data    | data     | i128    | `amount` — the amount for this entry in USDC base units  |

**Example (one entry in a batch):**

```json
{
  "topics": ["batch_distribute", "GDEV..."],
  "data": 3000000
}
```

---

### `receive_payment`

Emitted by `receive_payment(caller, amount, from_vault)` for logging inbound payment acknowledgements.

| Field        | Location | Type    | Description                                      |
|--------------|----------|---------|--------------------------------------------------|
| topic 0      | topics   | Symbol  | `"receive_payment"`                              |
| topic 1      | topics   | Address | `caller` — the admin address logging the payment |
| data tuple 0 | data     | i128    | `amount` — the logged amount                     |
| data tuple 1 | data     | bool    | `from_vault` — `true` if sourced from the vault  |

### Version notes

| Version | Change |
|---------|--------|
| 0.1.0   | Initial revenue pool events: `init`, `distribute`, `batch_distribute`, `receive_payment` |

