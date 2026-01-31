---
name: protocol
description: Expert MultiversX Protocol Knowledge. Use for understanding sharding, consensus, token standards (ESDT), and network parameters.
---

# MultiversX Protocol Expert

This skill encapsulates the deep theoretical and practical knowledge of the MultiversX network architecture. It is essential for architectural decisions, debugging complex failures, and understanding "why" things work the way they do.

## Core Concepts

1.  **Adaptive State Sharding**
    *   **Intra-shard vs. Cross-shard**: Transactions within the same shard are atomic and fast (6s). Cross-shard transactions are asynchronous and take longer (metadata + execution).
    *   **Account Location**: An address's shard is determined by the last bits of its public key.
    *   **Gas Implications**: Cross-shard calls consume more gas and require careful handling of intermediate states.

2.  **ESDT (Elrond Standard Digital Token)**
    *   **Native Tokens**: Tokens are first-class citizens, not smart contracts (unlike ERC-20).
    *   **Storage**: Token balances are stored directly in the user's account trie.
    *   **Roles**: `ESDTRoleLocalMint`, `ESDTRoleLocalBurn` allow contracts to manage supply without centralized minters.

3.  **Smart Contract Execution**
    *   **Arwen VM**: A high-performance WASM virtual machine.
    *   **Statelessness**: Contracts is stateless during execution until storage updates are committed (at the end).
    *   **Async Calls**: `async_call` does NOT return immediately. It schedules a call for a future block.

## 🛠️ Critical Knowledge & "Sharp Edges"

### 1. The Async Model
*   **Problem**: You cannot get the result of a cross-shard call in the same transaction.
*   **Solution**: Use Callbacks (`#[callback]`) to handle the result (success or failure) of an async operation.
*   **Warning**: State changes made before an async call are **committed** even if the remote call fails (unless you revert explicitly in the callback).

### 2. Gas Schedule
*   **Storage is Expensive**: Writing to storage is the highest cost.
*   **Bytes Matter**: The number of bytes in inputs and outputs directly affects gas cost.

### 3. Network Configuration
*   **Devnet / Testnet / Mainnet**: Distinct IDs (`D`, `T`, `1`).
*   **Epochs & Rounds**: Understanding finality time (approx 64s for full finality across shards).

## 📚 Expert Resources (Deep Dive)
*   **Protocol Experts**: `skills/expert/multiversx-protocol-experts/SKILL.md`
*   **Sharp Edges**: `skills/expert/mvx_sharp_edges/SKILL.md`
*   **Consult Docs**: `skills/expert/consult_mvx_docs/SKILL.md`

## ⚡ Key Constants

| Parameter | Value | Note |
| :--- | :--- | :--- |
| **Block Time** | 6 seconds | Per shard |
| **Max Gas/Block** | 1.5B | Target is lower |
| **Min Transaction Gas** | 50,000 | Basic transfer |
| **ESDT Transfer Gas** | ~300,000 | Depends on data |

**Design Rule**: Always assume your contract will be called from another shard. Design for asynchrony.


# Protocol Experts

---
name: multiversx-protocol-experts
description: Deep protocol knowledge for MultiversX architecture including sharding, consensus, ESDT standards, and cross-shard transactions. Use when reviewing protocol-level code, designing complex dApp architectures, or troubleshooting cross-shard issues.
---

# MultiversX Protocol Expertise

Deep technical knowledge of the MultiversX protocol architecture, utilized for reviewing protocol-level changes, designing complex dApp architectures involving cross-shard logic, and sovereign chain integrations.

## When to Use

- Reviewing protocol-level code changes
- Designing cross-shard smart contract interactions
- Troubleshooting async transaction issues
- Understanding token standard implementations
- Working with sovereign chain integrations
- Optimizing for sharded architecture

## 1. Core Architecture: Adaptive State Sharding

### Sharding Overview

MultiversX implements three types of sharding simultaneously:

| Sharding Type | Description | Implication |
|--------------|-------------|-------------|
| **Network** | Nodes distributed into shards | Each shard has subset of validators |
| **Transaction** | TXs processed by sender's shard | TX execution location is deterministic |
| **State** | Each shard maintains portion of state | Account data is shard-specific |

### Shard Assignment
Accounts are assigned to shards based on address:
```
Shard = last_byte_of_address % num_shards
```

**Practical implication**: Contract address determines which shard processes its transactions.

### The Metachain

The Metachain is a special coordinator shard that:
- Handles validator shuffling between shards
- Processes epoch transitions
- Manages system smart contracts (Staking, ESDT issuance)
- Notarizes shard block headers

**Important**: The Metachain does NOT execute general smart contracts.

```rust
// System contracts live on Metachain:
// - Staking contract
// - ESDT system smart contract
// - Delegation manager
// - Governance

// Regular contracts execute on shard determined by their address
```

## 2. Cross-Shard Transactions

### Transaction Flow

When sender and receiver are on different shards:

```
1. Sender Shard: Process TX, generate Smart Contract Result (SCR)
2. Metachain: Notarize sender block, relay SCR
3. Receiver Shard: Execute SCR, finalize transaction
```

### Atomicity Guarantees

**Key Property**: Cross-shard transactions are atomic but asynchronous.

```rust
// This cross-shard call is atomic
self.tx()
    .to(&other_contract)  // May be on different shard
    .typed(other_contract_proxy::OtherContractProxy)
    .some_endpoint(args)
    .async_call_and_exit();

// Either BOTH sides succeed, or BOTH sides revert
// But there's a delay between sender execution and receiver execution
```

### Callback Handling

```rust
#[endpoint]
fn cross_shard_call(&self) {
    // State changes HERE happen immediately (sender shard)
    self.pending_calls().set(true);

    self.tx()
        .to(&other_contract)
        .typed(proxy::Proxy)
        .remote_function()
        .callback(self.callbacks().on_result())
        .async_call_and_exit();
}

#[callback]
fn on_result(&self, #[call_result] result: ManagedAsyncCallResult<BigUint>) {
    // This executes LATER, back on sender shard
    // AFTER receiver shard has processed

    match result {
        ManagedAsyncCallResult::Ok(value) => {
            // Remote call succeeded
            self.pending_calls().set(false);
        },
        ManagedAsyncCallResult::Err(_) => {
            // Remote call failed
            // Original state changes are NOT auto-reverted!
            // Must manually handle rollback
            self.pending_calls().set(false);
            self.handle_failure();
        }
    }
}
```

## 3. Consensus: Secure Proof of Stake (SPoS)

### Validator Selection

```
Selection factors:
- Stake amount
- Validator rating (performance history)
- Random seed (from previous block)

Result: Deterministic but unpredictable validator selection
```

### Block Production

| Phase | Duration | Description |
|-------|----------|-------------|
| Propose | ~1s | Leader proposes block |
| Validate | ~1s | Validators verify and sign |
| Commit | ~1s | Block committed with BLS multi-sig |

### Finality

- **Intra-shard**: Instant finality (~6 seconds)
- **Cross-shard**: Finality after receiver shard processes (~12-18 seconds)

## 4. Token Standards (ESDT)

### Native Implementation

Unlike ERC-20, ESDT tokens are protocol-level, not smart contracts:

```rust
// ERC-20 (Ethereum): Contract tracks balances
mapping(address => uint256) balances;

// ESDT (MultiversX): Protocol tracks balances
// No contract needed for basic token operations
// Balances stored directly in account state
```

### ESDT Properties

| Property | Description | Security Implication |
|----------|-------------|---------------------|
| CanFreeze | Protocol can freeze assets | Emergency response capability |
| CanWipe | Protocol can wipe assets | Regulatory compliance |
| CanPause | Protocol can pause transfers | Circuit breaker |
| CanMint | Address can mint new tokens | Inflation control |
| CanBurn | Address can burn tokens | Supply control |
| CanChangeOwner | Ownership transferable | Admin key rotation |
| CanUpgrade | Properties can be modified | Flexibility vs immutability |

### ESDT Roles

```rust
// Roles are assigned per address per token
ESDTRoleLocalMint    // Can mint tokens
ESDTRoleLocalBurn    // Can burn tokens
ESDTRoleNFTCreate    // Can create NFT/SFT nonces
ESDTRoleNFTBurn      // Can burn NFT/SFT
ESDTRoleNFTAddQuantity   // Can add SFT quantity
ESDTRoleNFTUpdateAttributes  // Can update NFT attributes
ESDTTransferRole     // Required for restricted transfers
```

### Token Transfer Types

| Transfer Type | Function | Use Case |
|---------------|----------|----------|
| ESDTTransfer | Fungible token transfer | Simple token send |
| ESDTNFTTransfer | Single NFT/SFT transfer | NFT marketplace |
| MultiESDTNFTTransfer | Batch transfer | Contract calls with multiple tokens |

## 5. Built-in Functions

### Transfer Functions
```
ESDTTransfer@<token_id>@<amount>
ESDTNFTTransfer@<token_id>@<nonce>@<amount>@<receiver>
MultiESDTNFTTransfer@<receiver>@<num_tokens>@<token1_id>@<nonce1>@<amount1>@...
```

### Contract Call with Payment
```
MultiESDTNFTTransfer@<contract>@<num_tokens>@<token_data>...@<function>@<args>...
```

### Direct ESDT Operations
```
ESDTLocalMint@<token_id>@<amount>
ESDTLocalBurn@<token_id>@<amount>
ESDTNFTCreate@<token_id>@<initial_quantity>@<name>@<royalties>@<hash>@<attributes>@<uris>
```

## 6. Sovereign Chains & Interoperability

### Sovereign Chain Architecture

```
MultiversX Mainchain (L1)
    │
    ├── Sovereign Chain A (L2)
    │       └── Gateway Contract
    │
    └── Sovereign Chain B (L2)
            └── Gateway Contract
```

### Cross-Chain Communication

```rust
// Mainchain → Sovereign: Deposit flow
1. User deposits tokens to Gateway Contract on Mainchain
2. Gateway emits deposit event
3. Sovereign Chain validators observe event
4. Tokens minted on Sovereign Chain

// Sovereign → Mainchain: Withdrawal flow
1. User initiates withdrawal on Sovereign Chain
2. Sovereign validators create proof
3. Proof submitted to Mainchain Gateway
4. Tokens released on Mainchain
```

### Security Considerations

| Risk | Mitigation |
|------|------------|
| Bridge funds theft | Multi-sig validators, time locks |
| Invalid proofs | Cryptographic verification |
| Reorg attacks | Wait for finality before processing |
| Validator collusion | Slashing, stake requirements |

## 7. Critical Checks for Protocol Developers

### Intra-shard vs Cross-shard Awareness

```rust
// WRONG: Assuming synchronous execution
#[endpoint]
fn process_and_transfer(&self) {
    let result = self.external_contract().compute_value();  // May be async!
    self.storage().set(result);  // result may be callback, not value
}

// CORRECT: Handle async explicitly
#[endpoint]
fn process_and_transfer(&self) {
    self.tx()
        .to(&self.external_contract().get())
        .typed(proxy::Proxy)
        .compute_value()
        .callback(self.callbacks().handle_result())
        .async_call_and_exit();
}

#[callback]
fn handle_result(&self, #[call_result] result: ManagedAsyncCallResult<BigUint>) {
    match result {
        ManagedAsyncCallResult::Ok(value) => {
            self.storage().set(value);
        },
        ManagedAsyncCallResult::Err(_) => {
            // Handle failure
        }
    }
}
```

### Reorg Handling for Microservices

```typescript
// WRONG: Process immediately
async function handleTransaction(tx: Transaction) {
    await processPayment(tx);  // What if block is reverted?
}

// CORRECT: Wait for finality
async function handleTransaction(tx: Transaction) {
    // Wait for finality (configurable rounds)
    await waitForFinality(tx.hash, { rounds: 3 });

    // Or subscribe to finalized blocks only
    const status = await getTransactionStatus(tx.hash);
    if (status === 'finalized') {
        await processPayment(tx);
    }
}
```

### Gas Considerations for Cross-Shard

```rust
// Cross-shard calls consume gas on BOTH shards
// Sender: Gas for initiating call + callback execution
// Receiver: Gas for executing the called function

// Reserve enough gas for callback
const GAS_FOR_CALLBACK: u64 = 10_000_000;
const GAS_FOR_REMOTE: u64 = 20_000_000;

#[endpoint]
fn cross_shard_call(&self) {
    self.tx()
        .to(&other_contract)
        .typed(proxy::Proxy)
        .remote_function()
        .with_gas_limit(GAS_FOR_REMOTE)
        .callback(self.callbacks().on_result())
        .with_extra_gas_for_callback(GAS_FOR_CALLBACK)
        .async_call_and_exit();
}
```

## 8. MIP Standards Reference

Monitor MultiversX Improvement Proposals for standard implementations:

| MIP | Topic | Status |
|-----|-------|--------|
| MIP-2 | Fractional NFTs (SFT standard) | Implemented |
| MIP-3 | Dynamic NFTs | Implemented |
| MIP-4 | Token Royalties | Implemented |

## 9. Network Constants

| Constant | Value | Notes |
|----------|-------|-------|
| Max shards | 3 + Metachain | Current mainnet configuration |
| Round duration | ~6 seconds | Block time |
| Epoch duration | ~24 hours | Validator reshuffling period |
| Max TX size | 256 KB | Transaction data limit |
| Max gas limit | 600,000,000 | Per transaction |


# Protocol Experts (Expert)

---
name: mvx_protocol_experts
description: Expert knowledge of the MultiversX Protocol, Consensus (SPoS), Sharding, and Standard Implementations (MIPs).
---

# MultiversX Protocol Expertise

This skill encompasses deep knowledge of the core MultiversX protocol architecture, utilized for reviewing protocol-level changes, complex dApp architectures involving cross-shard logic, and sovereign chain integrations.

## 1. Core Architecture: Adaptive State Sharding
- **Sharding Types**:
    - **Network Sharding**: Nodes are distributed into shards.
    - **Transaction Sharding**: Transactions are processed by the shard containing the sender's account.
    - **State Sharding**: Each shard maintains a portion of the global state (account space).
- **Metachain**:
    - The coordinator shard.
    - Handles: Validator churning, Epoch/Round verification, Slashing, Header notarization.
    - **Does NOT** execute general smart contracts (except for system contracts like Staking, ESDT management).
- **Cross-Shard Transactions**:
    - **Async (Process -> Relay -> Execute)**: Sender Shard processes TX -> Generates internal transaction (Scratchpad) -> Relayer -> Receiver Shard executes.
    - **Atomicity**: Cross-shard is atomic but asynchronous.

## 2. Consensus: Secure Proof of Stake (SPoS)
- **Selection**: Deterministic selection of validators based on stake + rating.
- **BLS Signing**: Validators sign proposed blocks using BLS multi-signatures.
- **Finality**: Instant finality (within seconds) due to PBFT-like consensus within the shard.

## 3. Token Standards (ESDT)
- **Native Implementation**: Tokens are NOT smart contracts (unlike ERC-20). They are part of the protocol account state.
- **ESDT Properties**:
    - **CanFreeze**: Protocol can freeze assets if configured.
    - **CanWipe**: Protocol can wipe assets if configured.
    - **CanPause**: Protocol can pause transfers.
- **Roles**:
    - `ESDTRoleLocalMint`: Allows minting.
    - `ESDTRoleLocalBurn`: Allows burning.

## 4. Sovereign Chains & Interoperability
- **Sovereign Chains**: Dedicated blockchains extending MultiversX.
    - **Gateway Contract**: Handles bridging between Mainchain and Sovereign Chain.
    - **Settlement**: Sovereign Chains settle proofs to the MultiversX Mainchain (acting as Layer 1).
- **Interoperability Standards**:
    - **MIP-X**: Monitor [MIPs] for latest standard adoption.

## 5. Transaction Processing
- **Gas Schedule**: Operations have deterministic costs.
- **Built-in Functions**:
    - `ESDTTransfer`: Direct transfer.
    - `ESDTNFTTransfer`: Transfer of SFT/NFT.
    - `MultiESDTNFTTransfer`: Batch transfer (often used for Contract calls).

## 6. Critical Checks for Protocol Developers
- **Intra-shard vs Cross-shard**:
    - Always assume a call *might* be cross-shard unless verified co-location.
    - **Async**: If logic depends on the result of a call to another address, it MUST be an Async Call (Callbacks).
- **Reorg Handling**:
    - Microservices must handle block rollbacks (Standard: wait for 'Finalized' status, or handle re-org events).


# Sharp Edges

---
name: multiversx-sharp-edges
description: Catalog of non-obvious behaviors, gotchas, and platform-specific quirks in MultiversX that often lead to bugs. Use when debugging unexpected behavior, reviewing code for subtle issues, or learning platform-specific pitfalls.
---

# MultiversX Sharp Edges

A catalog of non-obvious behaviors, "gotchas," and platform-specific quirks that frequently lead to bugs in MultiversX smart contracts and dApps. Understanding these sharp edges is essential for writing correct code.

## When to Use

- Debugging unexpected contract behavior
- Reviewing code for subtle platform-specific issues
- Onboarding to MultiversX development
- Checking if a bug might be caused by a known quirk
- Preparing for security audits

## 1. Async Callbacks & Reverts

### The Sharp Edge
When an async call fails, the `#[callback]` is still executed, but state changes from the original transaction are **NOT automatically reverted**.

### The Problem
```rust
#[endpoint]
fn transfer_and_update(&self, recipient: ManagedAddress, amount: BigUint) {
    // State change happens IMMEDIATELY
    self.total_sent().update(|t| *t += &amount);

    // Async call to another contract
    self.tx()
        .to(&recipient)
        .egld(&amount)
        .callback(self.callbacks().on_transfer())
        .async_call_and_exit();
}

#[callback]
fn on_transfer(&self) {
    // If transfer FAILED, total_sent is STILL updated!
    // This is inconsistent state!
}
```

### The Solution
```rust
#[endpoint]
fn transfer_and_update(&self, recipient: ManagedAddress, amount: BigUint) {
    // DON'T update state before async call
    self.tx()
        .to(&recipient)
        .egld(&amount)
        .callback(self.callbacks().on_transfer(amount.clone()))
        .async_call_and_exit();
}

#[callback]
fn on_transfer(&self, amount: BigUint, #[call_result] result: ManagedAsyncCallResult<()>) {
    match result {
        ManagedAsyncCallResult::Ok(_) => {
            // Only update state on SUCCESS
            self.total_sent().update(|t| *t += &amount);
        },
        ManagedAsyncCallResult::Err(_) => {
            // Handle failure explicitly
            // Funds return to contract automatically
        }
    }
}
```

## 2. Gas Limits & Out of Gas (OOG)

### The Sharp Edge
OOG can leave cross-shard transactions in partial states.

### Cross-Shard OOG Scenario
```
1. Sender shard processes transaction (state changed)
2. Receiver shard runs out of gas
3. Receiver execution fails
4. Sender state changes PERSIST
5. Callback triggered with error
```

### The Solution
```rust
// Always reserve enough gas for callbacks
const CALLBACK_GAS: u64 = 10_000_000;

#[endpoint]
fn safe_cross_shard(&self) {
    self.tx()
        .to(&other_contract)
        .typed(proxy::Proxy)
        .function()
        .with_gas_limit(50_000_000)
        .callback(self.callbacks().handle_result())
        .with_extra_gas_for_callback(CALLBACK_GAS)
        .async_call_and_exit();
}
```

## 3. Storage Mappers vs Rust Types

### The Sharp Edge
`VecMapper` is NOT a `Vec`. They have fundamentally different memory models.

### The Problem
```rust
// VecMapper: Each element is a separate storage slot
// Accessing element = 1 storage read
// Iterating N elements = N storage reads
#[storage_mapper("users")]
fn users(&self) -> VecMapper<ManagedAddress>;

// If you load into a Vec, you load EVERYTHING into WASM memory
fn bad_function(&self) {
    let all_users: Vec<ManagedAddress> = self.users().iter().collect();
    // With 10,000 users = 10,000 storage reads + massive memory allocation
    // WILL run out of gas
}
```

### The Solution
```rust
// Paginate operations
fn process_users_paginated(&self, start: usize, count: usize) {
    let len = self.users().len();
    let end = (start + count).min(len);

    for i in start..end {
        let user = self.users().get(i + 1);  // VecMapper is 1-indexed!
        self.process_user(&user);
    }
}

// Or use appropriate mapper for the use case
// SetMapper for O(1) contains checks
// UnorderedSetMapper for efficient removal
```

## 4. Token Decimal Precision

### The Sharp Edge
ESDTs can have 0-18 decimals. Hardcoding decimal assumptions breaks contracts.

### The Problem
```rust
// WRONG: Assumes 18 decimals
fn convert_to_usd(&self, token_amount: BigUint) -> BigUint {
    let price = self.price().get();  // Price in 10^18
    &token_amount * &price / BigUint::from(10u64.pow(18))  // Assumes 18 decimals!
}
```

### The Solution
```rust
fn convert_to_usd(&self, token_amount: BigUint, token_decimals: u8) -> BigUint {
    let price = self.price().get();
    let decimal_factor = BigUint::from(10u64).pow(token_decimals as u32);
    &token_amount * &price / &decimal_factor
}

// Or require specific decimals
fn require_standard_decimals(&self, token_id: &TokenIdentifier) {
    let properties = self.blockchain().get_esdt_token_data(
        &self.blockchain().get_sc_address(),
        token_id,
        0
    );
    require!(properties.decimals == 18, "Token must have 18 decimals");
}
```

## 5. Upgradeability Pitfalls

### The Sharp Edge
`#[init]` is NOT called on upgrade. Only `#[upgrade]` runs.

### The Problem
```rust
// V1 contract
#[init]
fn init(&self) {
    self.version().set(1);
}

// V2 contract - added new storage
#[init]
fn init(&self) {
    self.version().set(2);
    self.new_feature_enabled().set(true);  // NEVER RUNS ON UPGRADE!
}

// After upgrade: version is still 1, new_feature_enabled is empty!
```

### The Solution
```rust
#[upgrade]
fn upgrade(&self) {
    // Initialize new storage here
    self.version().set(2);
    self.new_feature_enabled().set(true);

    // Migrate existing data if needed
    self.migrate_storage();
}
```

### Storage Layout Changes

**NEVER reorder struct fields:**
```rust
// V1
struct UserData {
    balance: BigUint,    // Encoded at position 0
    timestamp: u64,      // Encoded at position 1
}

// V2 - BREAKS EXISTING DATA
struct UserData {
    timestamp: u64,      // Now at position 0 - reads old balance bytes!
    balance: BigUint,    // Now at position 1 - reads old timestamp bytes!
    new_field: bool,     // This is fine (appended)
}
```

## 6. Block Info in Views

### The Sharp Edge
`get_block_timestamp()` and similar in `#[view]` functions may return different values off-chain vs on-chain.

### The Problem
```rust
#[view(isExpired)]
fn is_expired(&self) -> bool {
    let deadline = self.deadline().get();
    let current_time = self.blockchain().get_block_timestamp();
    // Off-chain simulation may return 0 or stale value!
    current_time > deadline
}
```

### The Solution
```rust
// Option 1: Don't rely on block info in views
#[view(getDeadline)]
fn get_deadline(&self) -> u64 {
    self.deadline().get()
    // Let client compare with their known current time
}

// Option 2: Accept timestamp as parameter for queries
#[view(isExpiredAt)]
fn is_expired_at(&self, check_time: u64) -> bool {
    let deadline = self.deadline().get();
    check_time > deadline
}
```

## 7. VecMapper Indexing

### The Sharp Edge
`VecMapper` is **1-indexed**, not 0-indexed like Rust `Vec`.

### The Problem
```rust
fn get_first_user(&self) -> ManagedAddress {
    self.users().get(0)  // PANIC! Index 0 doesn't exist
}
```

### The Solution
```rust
fn get_first_user(&self) -> ManagedAddress {
    require!(!self.users().is_empty(), "No users");
    self.users().get(1)  // First element is at index 1
}

fn iterate_users(&self) {
    for i in 1..=self.users().len() {  // 1 to len, inclusive
        let user = self.users().get(i);
        // process user
    }
}
```

## 8. EGLD vs ESDT Transfer Limitations

### The Sharp Edge
You cannot send EGLD and ESDT in the same transaction.

### The Problem
```rust
// This is IMPOSSIBLE on MultiversX
self.tx()
    .to(&recipient)
    .egld(&egld_amount)
    .single_esdt(&token_id, 0, &esdt_amount)  // Can't combine!
    .transfer();
```

### The Solution
```rust
// Separate transactions
self.tx().to(&recipient).egld(&egld_amount).transfer();
self.tx().to(&recipient).single_esdt(&token_id, 0, &esdt_amount).transfer();

// Or design around the limitation
// e.g., use wrapped EGLD (WEGLD) as ESDT
```

## 9. MapMapper Memory Model

### The Sharp Edge
`MapMapper` stores `4*N + 1` storage entries, making it very expensive.

### The Problem
```rust
// For 1000 users, this creates 4001 storage entries!
#[storage_mapper("balances")]
fn balances(&self) -> MapMapper<ManagedAddress, BigUint>;
```

### The Solution
```rust
// Use SingleValueMapper with address key when you don't need to iterate
#[storage_mapper("balance")]
fn balance(&self, user: &ManagedAddress) -> SingleValueMapper<BigUint>;

// Only use MapMapper when you MUST iterate over all entries
```

## 10. Require vs SC Panic

### The Sharp Edge
`require!` generates larger WASM than `sc_panic!` when the message is dynamic.

### The Problem
```rust
// Each unique string increases WASM size
require!(condition1, "Error message one");
require!(condition2, "Error message two");
require!(condition3, "Error message three");
```

### The Solution
```rust
// Use static error constants
const ERR_INVALID_AMOUNT: &str = "Invalid amount";
const ERR_UNAUTHORIZED: &str = "Unauthorized";

require!(amount > 0, ERR_INVALID_AMOUNT);
require!(caller == owner, ERR_UNAUTHORIZED);

// Reuse same constant for same error type
require!(amount1 > 0, ERR_INVALID_AMOUNT);
require!(amount2 > 0, ERR_INVALID_AMOUNT);
```

## Quick Reference: Common Gotchas

| Issue | Wrong | Right |
|-------|-------|-------|
| VecMapper index | `.get(0)` | `.get(1)` |
| Callback state | Update before async | Update in callback on success |
| Upgrade init | Rely on `#[init]` | Use `#[upgrade]` |
| Decimals | Hardcode `10^18` | Fetch from token properties |
| MapMapper | Use for per-user data | Use SingleValueMapper with key |
| Block info in view | Direct use | Pass as parameter |
| EGLD + ESDT | Same transaction | Separate transactions |
| Struct fields | Reorder | Only append |


# Sharp Edges (Expert)

---
name: mvx_sharp_edges
description: Identify weird behaviors in WASM, Gas limits, and async call failures specific to MultiversX.
---

# MultiversX Sharp Edges

This skill alerts you to non-obvious behaviors, "gotchas," and platform-specific quirks that often lead to bugs.

## 1. Async Callbacks & Reverts
- **The Edge**: When an Async Call fails, the `#[callback]` is executed.
- **The Sharp Logic**:
    - If you don't implement a callback, the funds return to the contract, but state changes in the original transaction are **NOT reverted** automatically.
    - *Mitigation*: You must manually revert state in the callback if the sub-call failed (or use the synchronous `back_transfers` carefully).

## 2. Gas Limits & Out of Gas (OOG)
- **The Edge**: OOG raises a specific error but can leave the system in a partial state if not handled.
- **The Sharp Logic**:
    - **Cross-shard OOG**: If the destination shard runs OOG, the transaction fails, but the sender shard has already processed the transfer. The callback is triggered with an error.

## 3. Storage Mappers vs Rust Types
- **The Edge**: `VecMapper` is NOT a `Vec`.
- **The Sharp Logic**:
    - `Vec` loads everything into WASM memory (RAM spike).
    - `VecMapper` loads nothing until you call `get(i)`.
    - *Bug*: Using `Vec` for a list of all users = OOG when userbase grows.

## 4. Token Decimal Precision
- **The Edge**: ESDTs can have 0-18 decimals.
- **The Sharp Logic**:
    - Hardcoding `10^18` for "One Token" is wrong.
    - Always fetch `decimals` or require standard 18-decimal tokens.

## 5. Upgradeability
- **The Edge**: `#[init]` is NOT called on upgrade. `#[upgrade]` is called.
- **The Sharp Logic**:
    - If you add a new storage mapper, you must initialize it in `#[upgrade]`.
    - Changing the storage layout of existing mappers (e.g. `struct` field order) corrupts data.

## 6. Block Info inside Views
- **The Edge**: `get_block_timestamp()` in a `#[view]` (off-chain simulation) might return 0 or a different value than on-chain.
- **The Sharp Logic**:
    - Don't rely on block info for critical off-chain view logic.


# Consult Docs

---
name: consult_mvx_docs
description: Access the global MultiversX documentation library to answer technical questions or verify implementation details.
---

# Consult MultiversX Documentation

You have access to a comprehensive library of MultiversX documentation, standards, and best practices stored in this repository at:
`antigravity/mvx_docs`

Note: Agents must resolve this path relative to the repository root (current workspace). If the agent runs outside this repo or without filesystem access, ensure the repo (including `antigravity/mvx_docs`) is mounted or provided in context.

## When to use this skill
- When you need to understand specific MultiversX protocols (ESDT, Smart Contracts, transactions).
- When you need to verify best practices for security, gas optimization, or architecture.
- When you need to look up details about a specific MIP (MultiversX Improvement Proposal).
- When you need to answer a user's question about the ecosystem using authoritative sources.

## How to use
Use standard file search tools to find relevant information within the documentation directory.

### 1. Find relevant files
Use `find_by_name`, `grep_search`, or your environment’s search to locate relevant documents.

```bash
# Example: Find docs related to ESDT tokens (repo-relative)
find_by_name(SearchDirectory="antigravity/mvx_docs", Pattern="*esdt*")
```

```bash
# Example: Search for "async call" inside the docs
grep_search(SearchPath="antigravity/mvx_docs", Query="async call")
```

### 2. Read content
Once you have identified relevant files, use `read_browser_url` (if it was a URL) or simply `view_file` to read the markdown content.

```bash
view_file(AbsolutePath="antigravity/mvx_docs/sc_async_calls.md")
```

### 3. Online fallback (if local docs unavailable)
If the local repository docs are not available in your runtime or appear outdated, consult the official MultiversX documentation site:

- Base URL: https://docs.multiversx.com/

Examples:
```bash
# Open a specific page
read_browser_url(URL="https://docs.multiversx.com/developers/sc-overview/")
```

Common direct links:
- Smart Contracts Overview: https://docs.multiversx.com/developers/smart-contracts
- sc-meta Tool: https://docs.multiversx.com/developers/meta/sc-meta
- Storage Mappers: https://docs.multiversx.com/developers/developer-reference/storage-mappers
- Annotations: https://docs.multiversx.com/developers/developer-reference/sc-annotations
- Payments: https://docs.multiversx.com/developers/developer-reference/sc-payments

Searching the docs site: The official site uses client-side search without a URL query parameter. To search, use an external search engine with a site filter, for example:

```text
site:docs.multiversx.com async call
site:docs.multiversx.com annotations payable
```

Note: Online access requires network permissions in your agent environment. Prefer local docs for reproducibility; use the official site (and site-filtered web search) when local docs are missing or stale.

## Best Practices
- Always verify your assumptions against these docs if you are unsure.
- Quote the documentation when explaining concepts to the user.
- If the docs seem outdated compared to the codebase you are working on, note that discrepancy to the user.


# Project Culture

---
name: multiversx-project-culture
description: Assess codebase quality and maturity based on documentation, testing practices, and code hygiene indicators. Use when evaluating project reliability, estimating audit effort, or onboarding to new codebases.
---

# Project Culture & Code Maturity Assessment

Evaluate the quality and reliability of a MultiversX codebase based on documentation presence, testing culture, code hygiene, and development practices. This assessment helps calibrate audit depth and identify areas of concern.

## When to Use

- Starting engagement with a new project
- Estimating audit scope and effort
- Evaluating investment or integration risk
- Providing feedback on development practices
- Prioritizing review focus areas

## 1. Documentation Quality

### Documentation Presence Checklist

| Item | Location | Status |
|------|----------|--------|
| README.md | Project root | [ ] Present [ ] Useful |
| Build instructions | README or BUILDING.md | [ ] Present [ ] Tested |
| API documentation | docs/ or inline | [ ] Present [ ] Complete |
| Architecture overview | docs/ or specs/ | [ ] Present |
| Deployment guide | README or DEPLOY.md | [ ] Present |

### MultiversX-Specific Documentation

| Item | Purpose | Status |
|------|---------|--------|
| `mxpy.json` | Standard build configuration | [ ] Present |
| `sc-config.toml` | Contract configuration | [ ] Present |
| `multiversx.yaml` | Additional config | [ ] Optional |
| `snippets.sh` | Interaction scripts | [ ] Helpful |
| `interaction/` | Deployment/call scripts | [ ] Very helpful |

### Specification Documents

| Document | Quality Indicator |
|----------|-------------------|
| Whitepaper | Formal specification of behavior |
| `specs/` directory | Detailed technical specs |
| MIP compliance docs | Standard adherence documentation |
| Security considerations | Threat model awareness |

### Documentation Quality Scoring

```
HIGH QUALITY:
- README explains purpose, build, test, deploy
- Architecture diagrams present
- API fully documented with examples
- Security model documented

MEDIUM QUALITY:
- README with basic instructions
- Some inline documentation
- Partial API coverage

LOW QUALITY:
- Minimal or no README
- No inline comments
- No architectural documentation
```

## 2. Testing Culture Assessment

### Test Presence

```bash
# Check for Rust unit tests
grep -r "#\[test\]" src/

# Check for Mandos/scenario tests
ls -la scenarios/

# Check for integration tests
ls -la tests/
```

### Mandos Scenario Coverage

| Coverage Level | Indicators |
|----------------|------------|
| **Excellent** | Every endpoint has scenario, edge cases tested, failure paths covered |
| **Good** | All endpoints have basic scenarios, some edge cases |
| **Minimal** | Only `deploy.scen.json` or few scenarios |
| **None** | No `scenarios/` directory |

### Test Quality Indicators

```rust
// HIGH QUALITY: Tests cover edge cases
#[test]
fn test_deposit_zero_amount() { }  // Boundary
#[test]
fn test_deposit_max_amount() { }   // Boundary
#[test]
fn test_deposit_wrong_token() { }  // Error case
#[test]
fn test_deposit_unauthorized() { } // Access control

// LOW QUALITY: Only happy path
#[test]
fn test_deposit() { }  // Basic only
```

### Continuous Integration

| CI Feature | Status |
|------------|--------|
| Automated builds | [ ] Present |
| Test execution | [ ] Present |
| Coverage reporting | [ ] Present |
| Lint/format checks | [ ] Present |
| Security scanning | [ ] Present |

### Simulation Testing

Look for:
- `mx-chain-simulator-go` usage
- Docker-based test environments
- Integration test scripts

## 3. Code Hygiene Assessment

### Linter Compliance

```bash
# Run Clippy
cargo clippy -- -W clippy::all

# Check formatting
cargo fmt --check
```

| Clippy Status | Interpretation |
|---------------|----------------|
| 0 warnings | Excellent hygiene |
| < 10 warnings | Good, minor issues |
| 10-50 warnings | Needs attention |
| > 50 warnings | Poor hygiene |

### Magic Numbers

```bash
# Find raw numeric literals
grep -rn "[^a-zA-Z_][0-9]\{2,\}[^a-zA-Z0-9_]" src/
```

**Bad:**
```rust
let seconds = 86400;  // What is this?
let fee = amount * 3 / 100;  // Magic 3%
```

**Good:**
```rust
const SECONDS_PER_DAY: u64 = 86400;
const FEE_PERCENT: u64 = 3;
const FEE_DENOMINATOR: u64 = 100;

let seconds = SECONDS_PER_DAY;
let fee = amount * FEE_PERCENT / FEE_DENOMINATOR;
```

### Error Handling

```bash
# Count unwrap usage
grep -c "\.unwrap()" src/*.rs

# Count expect usage
grep -c "\.expect(" src/*.rs

# Count proper error handling
grep -c "sc_panic!\|require!" src/*.rs
```

| Pattern | Quality Indicator |
|---------|-------------------|
| Mostly `require!` with messages | Good |
| Mixed `require!` and `unwrap()` | Needs review |
| Mostly `unwrap()` | Poor |

### Code Comments

| Aspect | Good Practice |
|--------|---------------|
| Complex logic | Has explanatory comments |
| Public APIs | Has doc comments |
| Assumptions | Documented inline |
| TODOs | Tracked, not ignored |

```rust
// GOOD: Complex logic explained
/// Calculates rewards using compound interest formula.
/// Formula: P * (1 + r/n)^(nt) where:
/// - P: principal
/// - r: annual rate (in basis points)
/// - n: compounding frequency
/// - t: time in years
fn calculate_rewards(&self, principal: BigUint, time: u64) -> BigUint {
    // ...
}

// BAD: No explanation for complex logic
fn calc(&self, p: BigUint, t: u64) -> BigUint {
    // Dense, unexplained calculation
}
```

## 4. Dependency Management

### Cargo.lock Presence

```bash
ls -la Cargo.lock
```

| Status | Interpretation |
|--------|----------------|
| Committed | Reproducible builds |
| Not committed | Version drift risk |

### Version Pinning

```toml
# GOOD: Specific versions
[dependencies.multiversx-sc]
version = "0.54.0"

# BAD: Wildcard versions
[dependencies.multiversx-sc]
version = "*"

# ACCEPTABLE: Caret (minor updates)
[dependencies.multiversx-sc]
version = "^0.54"
```

### Dependency Audit

```bash
# Check for known vulnerabilities
cargo audit
```

## 5. Maturity Scoring Matrix

### Score Calculation

| Category | Weight | High (3) | Medium (2) | Low (1) |
|----------|--------|----------|------------|---------|
| Documentation | 20% | Complete | Partial | Minimal |
| Testing | 30% | Full coverage | Basic coverage | Minimal |
| Code hygiene | 20% | Clean Clippy | Few warnings | Many issues |
| Dependencies | 15% | Pinned, audited | Pinned | Wildcards |
| CI/CD | 15% | Full pipeline | Basic | None |

### Interpretation

| Score | Maturity | Audit Focus |
|-------|----------|-------------|
| 2.5-3.0 | High | Business logic, edge cases |
| 1.5-2.4 | Medium | Broad review, verify basics |
| 1.0-1.4 | Low | Everything, assume issues exist |

## 6. Red Flags

### Immediate Concerns

| Red Flag | Risk |
|----------|------|
| No tests at all | Logic likely untested |
| Wildcard dependencies | Supply chain vulnerability |
| `unsafe` blocks without justification | Memory safety issues |
| Excessive `unwrap()` | Panic vulnerabilities |
| No README | Maintenance abandoned? |
| Outdated framework version | Known vulnerabilities |

### Yellow Flags

| Yellow Flag | Concern |
|-------------|---------|
| Few scenario tests | Limited coverage |
| Some Clippy warnings | Technical debt |
| Incomplete documentation | Knowledge silos |
| No CI/CD | Regression risk |

## 7. Assessment Report Template

```markdown
# Project Maturity Assessment

**Project**: [Name]
**Version**: [Version]
**Date**: [Date]
**Assessor**: [Name]

## Summary Score: [X/3.0] - [HIGH/MEDIUM/LOW] Maturity

## Documentation (Score: X/3)
- README: [Present/Missing]
- Build instructions: [Tested/Untested/Missing]
- Architecture docs: [Complete/Partial/Missing]
- API docs: [Complete/Partial/Missing]

## Testing (Score: X/3)
- Unit tests: [X tests found]
- Scenario tests: [X scenarios covering Y endpoints]
- Coverage estimate: [X%]
- Edge case coverage: [Good/Partial/Minimal]

## Code Hygiene (Score: X/3)
- Clippy warnings: [X warnings]
- Formatting: [Consistent/Inconsistent]
- Magic numbers: [X instances]
- Error handling: [Good/Needs work]

## Dependencies (Score: X/3)
- Cargo.lock: [Committed/Missing]
- Version pinning: [All/Some/None]
- Known vulnerabilities: [None/X found]

## CI/CD (Score: X/3)
- Build automation: [Yes/No]
- Test automation: [Yes/No]
- Security scanning: [Yes/No]

## Recommendations
1. [Highest priority improvement]
2. [Second priority]
3. [Third priority]

## Audit Focus Areas
Based on this assessment, the audit should prioritize:
1. [Area based on weaknesses]
2. [Area based on risk]
```

## 8. Improvement Recommendations by Level

### For Low Maturity Projects
1. Add basic README with build instructions
2. Create scenario tests for all endpoints
3. Fix all Clippy warnings
4. Pin dependency versions
5. Set up basic CI

### For Medium Maturity Projects
1. Expand test coverage to edge cases
2. Add architecture documentation
3. Document security considerations
4. Add coverage reporting
5. Implement security scanning

### For High Maturity Projects
1. Formal verification consideration
2. Fuzzing and property testing
3. External security audit
4. Bug bounty program
5. Incident response documentation


# Project Culture (Legacy)

---
name: project_culture
description: Assess code maturity based on MultiversX standards (docs presence, testing culture).
---

# Project Culture & Code Maturity Index

This skill helps you gauge the quality and reliability of a codebase based on "smells" and cultural indicators.

## 1. Documentation Quality
- **Docs Presence**: Is there a `README.md`? Does it explain *how* to build and test?
- **MultiversX Specifics**:
    - `mxpy.json`: Indicates use of standard build tools.
    - `snippets.sh` or `interaction/`: Scripts for deploying/interacting.
- **Specs**: Are there `.md` files in `specs/` or `whitepaper/`?

## 2. Testing Culture
- **Unit Tests**: Run `cargo test`. Are there tests? Do they cover edge cases or just "happy path"?
- **Integration Tests**: Look for `scenarios/` (Mandos).
    - *Gold Standard*: `scen.json` files covering every endpoint.
    - *Red Flag*: No scenarios, or only `deployment.scen.json`.
- **Simulation**: Is `mx-chain-simulator-go` mentioned in CI or scripts?

## 3. Code Hygiene
- **Linter**: Does `cargo clippy` pass?
- **Magic Numbers**: Are there raw integers (e.g., `86400`) instead of named constants (`SECONDS_PER_DAY`)?
- **Comments**: Are complex blocks explained?
- **Unwrap**: Excessive use of `unwrap()` instead of `sc_panic!` or checks.

## 4. Dependency Mangement
- **Lockfile**: Is `Cargo.lock` committed?
- **Version Pinning**: Are `multiversx-sc` versions pinned or wildcard `*`? (Wildcard is bad).

## Scoring (Mental Model)
- **High Maturity**: Full Mandos coverage, detailed specs, clean clippy, pinned versions. -> **Audit focus**: Business logic flaws.
- **Low Maturity**: No tests, spaghetti code, `unwrap()` everywhere. -> **Audit focus**: Basic safety, reentrancy, arithmetic overflow, DoS.


# Clarification Expert

---
name: multiversx-clarification-expert
description: Identify ambiguous requirements and ask targeted clarifying questions for MultiversX development. Use when user requests are vague, missing technical constraints, or have conflicting requirements.
---

# Clarification Expert

Identify ambiguity in user requests and ask targeted questions to unblock MultiversX development or auditing tasks. This skill prevents wasted effort from incorrect assumptions.

## When to Use

- User request is vague (e.g., "Make it secure")
- Missing technical constraints (e.g., "Add a token" without specifying standard)
- Conflicting requirements that need resolution
- Multiple valid implementation approaches exist
- Security implications require explicit user decisions

## Questioning Guidelines

### 1. Be Specific

Never ask open-ended questions. Always offer concrete choices.

**Bad:**
> "How should the token work?"

**Good:**
> "Should the token be:
> 1. A standard Fungible ESDT (simple transfers, 18 decimals)
> 2. A Semi-Fungible Token (SFT) with quantity per nonce
> 3. A Meta-ESDT with custom attributes
>
> If unsure, I recommend option 1 for simplicity."

### 2. Batch Related Questions

Group questions by topic to minimize back-and-forth.

**Example:**
> Before implementing the staking module, I need to clarify:
>
> **Token Configuration:**
> 1. Which token can be staked? (specific token ID or any ESDT?)
> 2. Is there a minimum stake amount?
>
> **Reward Mechanism:**
> 3. Fixed APY or dynamic based on total staked?
> 4. Reward distribution: claim-based or auto-compound?
>
> **Access Control:**
> 5. Can anyone stake, or only whitelisted addresses?

### 3. Propose Sensible Defaults

Always offer a recommended option when the user may not have strong preferences.

**Example:**
> "For the admin storage, I recommend using `SingleValueMapper` instead of `MapMapper` to save gas since you only need one admin. Shall I proceed with that approach?"

### 4. Explain Trade-offs

When choices have significant implications, explain the consequences.

**Example:**
> "For storing user balances, there are two approaches:
>
> **Option A: SingleValueMapper with address parameter**
> - More gas efficient for individual lookups
> - Cannot iterate over all users
>
> **Option B: MapMapper**
> - Can iterate over all users (useful for airdrops)
> - Higher gas cost (4N+1 storage slots)
>
> Which user enumeration requirement do you have?"

## Analysis Categories

### 1. Scope Definition
| Question | Why It Matters |
|----------|----------------|
| Full audit or specific module? | Determines depth vs breadth |
| New code or upgrade review? | Upgrade reviews need storage migration focus |
| Time-boxed or thorough? | Sets expectation for coverage |

### 2. Environment Context
| Question | Why It Matters |
|----------|----------------|
| Mainnet, Devnet, or Sovereign Chain? | Different security requirements |
| Existing deployment or new? | Upgrade compatibility concerns |
| Integration with other contracts? | Cross-contract interaction risks |

### 3. Risk Profile
| Question | Why It Matters |
|----------|----------------|
| TVL expectations? | High value = higher attacker incentive |
| User base: public or restricted? | Public = larger attack surface |
| Upgradeable or immutable? | Affects fix deployment strategy |

### 4. Technical Stack
| Question | Why It Matters |
|----------|----------------|
| Standard `multiversx-sc` or custom modules? | Custom code needs deeper review |
| Any `unsafe` blocks? | Requires manual memory safety verification |
| External dependencies? | Supply chain risk assessment |

## MultiversX-Specific Clarifications

### Token Standards
```
When user says "add a token", clarify:
- Fungible ESDT (standard fungible token)
- Non-Fungible Token (NFT) - unique items
- Semi-Fungible Token (SFT) - multiple of same nonce
- Meta-ESDT - fungible with metadata
```

### Storage Patterns
```
When user says "store user data", clarify:
- Per-user data: SingleValueMapper with address key
- Enumerable users: MapMapper or SetMapper
- Ordered data: VecMapper or LinkedListMapper
- Unique items: UnorderedSetMapper
```

### Access Control
```
When user says "admin only", clarify:
- Single owner (blockchain owner)
- Single admin (stored address)
- Multi-sig requirement
- Role-based (multiple permission levels)
- Time-locked operations
```

## Question Templates

### For New Feature Requests
> "To implement [FEATURE], I need to understand:
> 1. [CRITICAL_DECISION_1]
> 2. [CRITICAL_DECISION_2]
>
> My recommendation is [DEFAULT] because [REASON]. Should I proceed with this approach?"

### For Bug Reports
> "To fix this issue, I need to clarify:
> 1. Expected behavior: [WHAT_SHOULD_HAPPEN]
> 2. Current behavior: [WHAT_HAPPENS_NOW]
> 3. Reproduction steps: [HOW_TO_TRIGGER]
>
> Can you confirm these details?"

### For Security Reviews
> "Before auditing [CONTRACT], please confirm:
> 1. Threat model: Who are the trusted parties?
> 2. Invariants: What properties must always hold?
> 3. Known risks: Are there accepted trade-offs?
>
> This helps me focus on relevant attack vectors."

## Anti-Patterns to Avoid

- **Don't assume**: If something could go two ways, ask
- **Don't ask obvious questions**: "Is security important?" - always yes
- **Don't delay indefinitely**: If no response, state assumptions and proceed
- **Don't ask one question at a time**: Batch related questions together
- **Don't use jargon without explanation**: Clarify technical terms for non-experts


# Clarification Expert (Legacy)

---
name: clarification_expert
description: Expert at identifying underspecified requirements and asking high-value clarifying questions.
---

# Clarification Expert

This skill helps you identify ambiguity in user requests and ask targeted questions to unblock development or auditing tasks.

## When to Use
- The user's request is vague (e.g., "Make it secure").
- Missing technical constraints (e.g., "Add a token" but no standard specified).
- Conflicting requirements.

## Guidelines for Questions

### 1. Be Specific
Don't ask "What do you mean?". Ask "Do you want X or Y?".
*   *Bad*: "How should the token work?"
*   *Good*: "Should the token be a standard Fungible ESDT or a Semi-Fungible SFT with metadata?"

### 2. Batch Questions
Don't ask one question at a time. Group related questions into a numbered list.

### 3. Propose Defaults
Always offer a sensible default if the user doesn't know.
*   *Example*: "If you don't have a preference, I recommend using `SingleValueMapper` for the config storage to save gas. Shall I proceed with that?"

## Analysis Categories

1.  **Scope**: Is it a full audit or just a specific module?
2.  **Environment**: Mainnet, Devnet, or Sovereign Chain?
3.  **Risk Profile**: Is this a generic dApp or a high-value DeFi protocol?
4.  **Tech Stack**: Are we using standard `multiversx-sc` modules or custom unchecked arithmetic? (Rule: Always suggest standard modules).
