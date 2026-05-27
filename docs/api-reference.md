# IP Registry API Reference

Complete API documentation for the IP Registry smart contract.

---

## `commit_ip`

Timestamp a new IP commitment on-chain.

### Signature

```rust
pub fn commit_ip(env: Env, owner: Address, commitment_hash: BytesN<32>) -> u64
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `env` | `Env` | Soroban environment (injected automatically) |
| `owner` | `Address` | The address that owns the IP. Must authorize the transaction. |
| `commitment_hash` | `BytesN<32>` | 32-byte cryptographic hash: `sha256(secret \|\| blinding_factor)` |

### Returns

`u64` — The unique IP ID assigned to this commitment. IDs start at 1 and increment sequentially.

### Panics

| Error | Code | Condition |
|---|---|---|
| `ZeroCommitmentHash` | 2 | `commitment_hash` is all zeros |
| `CommitmentAlreadyRegistered` | 3 | `commitment_hash` already exists on-chain |
| Auth error | — | `owner` does not authorize the transaction |

### Authorization

Requires `owner.require_auth()` — the transaction must be signed by the owner's private key.

### Example (Rust SDK)

```rust
let owner = Address::from_string("GABC...");
let secret = BytesN::from_array(&env, &[/* 32 bytes */]);
let blinding_factor = BytesN::from_array(&env, &[/* 32 random bytes */]);

let mut preimage = Bytes::new(&env);
preimage.append(&secret.into());
preimage.append(&blinding_factor.into());
let commitment_hash: BytesN<32> = env.crypto().sha256(&preimage).into();

let ip_id = registry.commit_ip(&owner, &commitment_hash);
```

### Example (REST API)

**POST** `/ip/commit`

**Request Body:**
```json
{
  "owner": "GABC...",
  "commitment_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
}
```

**Response (200 OK):**
```json
1
```

---

## `batch_commit_ip`

Commit multiple IP hashes from the same owner in a single transaction. Reduces gas fees.

### Signature

```rust
pub fn batch_commit_ip(env: Env, owner: Address, hashes: Vec<BytesN<32>>) -> Vec<u64>
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `env` | `Env` | Soroban environment |
| `owner` | `Address` | Owner address (requires auth) |
| `hashes` | `Vec<BytesN<32>>` | Vector of commitment hashes to register |

### Returns

`Vec<u64>` — Vector of assigned sequential IP IDs.

### Panics

Same as `commit_ip` — panics if any hash is zero or already registered.

### Example (Rust SDK)

```rust
let hashes = Vec::from_array(&env, [hash1, hash2, hash3]);
let ip_ids = registry.batch_commit_ip(&owner, &hashes);
// ip_ids = [1, 2, 3]
```

### Example (REST API)

**POST** `/ip/batch`

**Request Body:**
```json
{
  "owner": "GABC...",
  "hashes": [
    "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "5feceb66ffc86f38d952786c6d696c79c2dbc239dd4e91b46729d73a27fb57e9"
  ]
}
```

**Response (200 OK):**
```json
[1, 2]
```

---

## `get_ip`

Retrieve an IP record by ID.

### Signature

```rust
pub fn get_ip(env: Env, ip_id: u64) -> IpRecord
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `env` | `Env` | Soroban environment |
| `ip_id` | `u64` | The unique identifier of the IP to retrieve |

### Returns

`IpRecord` struct:

```rust
pub struct IpRecord {
    pub ip_id: u64,
    pub owner: Address,
    pub commitment_hash: BytesN<32>,
    pub timestamp: u64,
    pub revoked: bool,
}
```

| Field | Type | Description |
|---|---|---|
| `ip_id` | `u64` | Unique identifier |
| `owner` | `Address` | Current owner's address |
| `commitment_hash` | `BytesN<32>` | The cryptographic commitment |
| `timestamp` | `u64` | Ledger timestamp when IP was committed |
| `revoked` | `bool` | Whether the IP has been revoked |

### Panics

| Error | Code | Condition |
|---|---|---|
| `IpNotFound` | 1 | IP record does not exist |

### Example (Rust SDK)

```rust
let record = registry.get_ip(&ip_id);
println!("Owner: {}", record.owner);
println!("Timestamp: {}", record.timestamp);
```

### Example (REST API)

**GET** `/ip/1`

**Response (200 OK):**
```json
{
  "ip_id": 1,
  "owner": "GABC...",
  "commitment_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "timestamp": 1713994200,
  "revoked": false
}
```

---

## `verify_commitment`

Verify that a secret and blinding factor match a stored commitment hash.

### Signature

```rust
pub fn verify_commitment(
    env: Env,
    ip_id: u64,
    secret: BytesN<32>,
    blinding_factor: BytesN<32>,
) -> bool
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `env` | `Env` | Soroban environment |
| `ip_id` | `u64` | The IP to verify |
| `secret` | `BytesN<32>` | The 32-byte secret used to create the commitment |
| `blinding_factor` | `BytesN<32>` | The 32-byte blinding factor used to create the commitment |

### Returns

`bool` — `true` if `sha256(secret || blinding_factor)` matches the stored commitment hash, `false` otherwise.

### Panics

| Error | Code | Condition |
|---|---|---|
| `IpNotFound` | 1 | IP record does not exist |

### Example (Rust SDK)

```rust
let is_valid = registry.verify_commitment(&ip_id, &secret, &blinding_factor);
if is_valid {
    println!("Commitment verified!");
}
```

### Example (REST API)

**POST** `/ip/verify`

**Request Body:**
```json
{
  "ip_id": 1,
  "secret": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "blinding_factor": "5feceb66ffc86f38d952786c6d696c79c2dbc239dd4e91b46729d73a27fb57e9"
}
```

**Response (200 OK):**
```json
{
  "valid": true
}
```

---

## `list_ip_by_owner`

List all IP IDs owned by an address.

### Signature

```rust
pub fn list_ip_by_owner(env: Env, owner: Address) -> Vec<u64>
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `env` | `Env` | Soroban environment |
| `owner` | `Address` | The address to list IPs for |

### Returns

`Vec<u64>` — Vector of all IP IDs owned by the address. Returns an empty vector if the address has no IPs.

### Panics

This function does not panic.

### Example (Rust SDK)

```rust
let ip_ids = registry.list_ip_by_owner(&owner);
for ip_id in ip_ids.iter() {
    let record = registry.get_ip(&ip_id);
    println!("IP {}: {}", ip_id, record.commitment_hash);
}
```

### Example (REST API)

**GET** `/ip/owner/GABC...`

**Response (200 OK):**
```json
{
  "ip_ids": [1, 2, 5]
}
```

---

## `transfer_ip`

Transfer IP ownership to a new address.

### Signature

```rust
pub fn transfer_ip(env: Env, ip_id: u64, new_owner: Address)
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `env` | `Env` | Soroban environment |
| `ip_id` | `u64` | The IP to transfer |
| `new_owner` | `Address` | The address that will become the new owner |

### Returns

This function does not return a value.

### Panics

| Error | Code | Condition |
|---|---|---|
| `IpNotFound` | 1 | IP record does not exist |
| Auth error | — | Current owner does not authorize the transaction |

### Authorization

Requires `record.owner.require_auth()` — the current owner must sign the transaction.

### Example (Rust SDK)

```rust
registry.transfer_ip(&ip_id, &new_owner);
```

### Example (REST API)

**POST** `/ip/transfer`

**Request Body:**
```json
{
  "ip_id": 1,
  "new_owner": "GDEF..."
}
```

**Response (200 OK):**
```json
{}
```

---

## `revoke_ip`

Revoke an IP record, marking it as invalid. Revoked IPs cannot be swapped.

### Signature

```rust
pub fn revoke_ip(env: Env, ip_id: u64)
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `env` | `Env` | Soroban environment |
| `ip_id` | `u64` | The IP to revoke |

### Returns

This function does not return a value.

### Panics

| Error | Code | Condition |
|---|---|---|
| `IpNotFound` | 1 | IP record does not exist |
| `IpAlreadyRevoked` | 4 | IP is already revoked |
| Auth error | — | Owner does not authorize the transaction |

### Authorization

Requires `record.owner.require_auth()` — only the current owner can revoke.

### Example (Rust SDK)

```rust
registry.revoke_ip(&ip_id);
```

### Example (REST API)

**POST** `/ip/revoke` (Note: Custom endpoint for revocation)

**Request Body:**
```json
{
  "ip_id": 1
}
```

**Response (200 OK):**
```json
{}
```

---

## `is_ip_owner`

Check if an address owns a specific IP.

### Signature

```rust
pub fn is_ip_owner(env: Env, ip_id: u64, address: Address) -> bool
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `env` | `Env` | Soroban environment |
| `ip_id` | `u64` | The IP to check |
| `address` | `Address` | The address to check for ownership |

### Returns

`bool` — `true` if the address owns the IP, `false` otherwise. Returns `false` if the IP does not exist.

### Panics

This function does not panic.

### Example

```rust
if registry.is_ip_owner(&ip_id, &address) {
    println!("Address owns this IP");
}
```

---

## Error Codes

| Error | Code | Description |
|---|---|---|
| `IpNotFound` | 1 | IP record does not exist |
| `ZeroCommitmentHash` | 2 | Commitment hash is all zeros |
| `CommitmentAlreadyRegistered` | 3 | Commitment hash already registered |
| `IpAlreadyRevoked` | 4 | IP is already revoked |
| `UnauthorizedUpgrade` | 5 | Caller is not admin (upgrade only) |

---

## Events

### `ip_commit`

Emitted when a new IP is committed.

**Topics:** `(symbol_short!("ip_commit"), owner: Address)`  
**Data:** `(ip_id: u64, timestamp: u64)`

---

## Storage Keys

| Key | Type | Description |
|---|---|---|
| `IpRecord(u64)` | Persistent | Stores IP record by ID |
| `OwnerIps(Address)` | Persistent | Maps owner → Vec of IP IDs |
| `NextId` | Persistent | Next available IP ID (monotonic counter) |
| `CommitmentOwner(BytesN<32>)` | Persistent | Maps commitment hash → owner (duplicate detection) |
| `Admin` | Persistent | Admin address for upgrades |

---

## TTL Management

All persistent storage entries are extended with a TTL of **~1 year** (6,307,200 ledgers at 5s/ledger).

See [TTL_MANAGEMENT.md](../TTL_MANAGEMENT.md) for details.

---

## Related Documentation

- [Commitment Scheme](commitment-scheme.md) — How to construct valid commitment hashes
- [Atomic Swap Flow](atomic-swap.md) — How to sell IP using atomic swaps
- [Security Considerations](security.md) — Best practices for secret management

---

## Issue #450: Anonymous IP Commitment

### `commit_ip_anonymous`

Commit an IP anonymously using a ZK-style nullifier. The owner's identity is not stored — only the nullifier and commitment hash are recorded.

```rust
pub fn commit_ip_anonymous(
    env: Env,
    nullifier: BytesN<32>,
    commitment_hash: BytesN<32>,
    anonymity_set_id: BytesN<32>,
) -> BytesN<32>
```

| Parameter | Type | Description |
|---|---|---|
| `nullifier` | `BytesN<32>` | Unique value derived from `sha256(secret \|\| nonce)` — prevents double-spend |
| `commitment_hash` | `BytesN<32>` | The IP commitment hash |
| `anonymity_set_id` | `BytesN<32>` | Group ID for the anonymity set this commitment joins |

**Returns:** The nullifier (for reference).

**Errors:** `ZeroCommitmentHash` (2) if either input is zero. `NullifierAlreadyUsed` (24) if the nullifier was already submitted.

### `get_anonymous_commitment`

```rust
pub fn get_anonymous_commitment(env: Env, nullifier: BytesN<32>) -> AnonymousCommitmentRecord
```

Retrieve an anonymous commitment record by its nullifier.

### `get_anonymity_set`

```rust
pub fn get_anonymity_set(env: Env, anonymity_set_id: BytesN<32>) -> Vec<BytesN<32>>
```

Returns all commitment hashes in the given anonymity set.

### `is_nullifier_used`

```rust
pub fn is_nullifier_used(env: Env, nullifier: BytesN<32>) -> bool
```

Returns `true` if the nullifier has already been used (double-spend check).

---

## Issue #451: Batch Revocation

### `batch_revoke_ip`

Revoke multiple IP commitments in a single transaction. Already-revoked IPs and IPs not owned by the caller are silently skipped.

```rust
pub fn batch_revoke_ip(env: Env, owner: Address, ip_ids: Vec<u64>) -> u32
```

| Parameter | Type | Description |
|---|---|---|
| `owner` | `Address` | Owner of the IPs — must authorize the transaction |
| `ip_ids` | `Vec<u64>` | List of IP IDs to revoke |

**Returns:** Number of IPs newly revoked in this call.

**Errors:** `EmptyBatchRevocation` (26) if `ip_ids` is empty.

**Authorization:** Requires `owner.require_auth()`.

---

## Issue #452: Conditional Verification

### `set_verification_condition`

Set a condition that must be met before an IP commitment can be verified.

```rust
pub fn set_verification_condition(env: Env, ip_id: u64, condition: VerificationCondition)
```

| Condition Variant | Description |
|---|---|
| `VerificationCondition::None` | No restriction — always verifiable |
| `VerificationCondition::TimeAfter(u64)` | Only verifiable after the given ledger timestamp |
| `VerificationCondition::TimeBefore(u64)` | Only verifiable before the given ledger timestamp |

**Authorization:** Requires the IP owner to authorize.

### `get_verification_condition`

```rust
pub fn get_verification_condition(env: Env, ip_id: u64) -> VerificationCondition
```

Returns the current condition for an IP. Returns `VerificationCondition::None` if none is set.

### `verify_commitment_conditional`

```rust
pub fn verify_commitment_conditional(
    env: Env,
    ip_id: u64,
    secret: BytesN<32>,
    blinding_factor: BytesN<32>,
) -> bool
```

Verifies the commitment only if the set condition is satisfied. Panics with `ConditionNotMet` (27) if the condition fails.

---

## Issue #453: IP Commitment Escrow

### `place_in_escrow`

Place an IP commitment into escrow. The IP will be transferred to the beneficiary once the release condition is met.

```rust
pub fn place_in_escrow(
    env: Env,
    ip_id: u64,
    beneficiary: Address,
    release_condition: VerificationCondition,
)
```

| Parameter | Type | Description |
|---|---|---|
| `ip_id` | `u64` | The IP to escrow |
| `beneficiary` | `Address` | Recipient when conditions are met |
| `release_condition` | `VerificationCondition` | Condition for release (use `None` for immediate release) |

**Errors:** `EscrowAlreadyExists` (29) if an escrow already exists for this IP.

**Authorization:** Requires the IP owner to authorize.

### `release_escrow`

```rust
pub fn release_escrow(env: Env, ip_id: u64)
```

Releases the escrowed IP to the beneficiary if the release condition is met. Anyone can call this once conditions are satisfied.

**Errors:** `EscrowNotFound` (30), `EscrowConditionsNotMet` (31).

### `cancel_escrow`

```rust
pub fn cancel_escrow(env: Env, ip_id: u64)
```

Cancels the escrow and returns control to the original owner. Only the original owner can cancel.

**Authorization:** Requires the original escrow owner to authorize.

### `get_escrow`

```rust
pub fn get_escrow(env: Env, ip_id: u64) -> EscrowRecord
```

Returns the escrow record for an IP. Panics with `EscrowNotFound` (30) if none exists.
