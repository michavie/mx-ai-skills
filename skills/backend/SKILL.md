---
name: backend
description: Expert MultiversX Backend Development (Go, Python, TypeScript). Use for microservices, off-chain logic, and blockchain interaction.
---

# MultiversX Backend Expert

This skill governs all server-side interactions with the MultiversX blockchain, including transaction processing, data indexing, and API integration using Go, Python, or Node.js.

## Core Capabilities

1.  **SDK Mastery**:
    *   **Go SDK (`mx-sdk-go`)**: High-performance, type-safe interaction. Preferred for microservices.
    *   **TypeScript/Node.js (`@multiversx/sdk-core`, NestJS)**: Ideal for API backends, serverless functions, and JS-native teams.
    *   **Python SDK (`mx-sdk-py`)**: Scripting, data analysis, and simple automation.

2.  **Blockchain Interaction**:
    *   **Observer Nodes**: Direct access to the network p2p layer.
    *   **Proxy API (Gateway)**: HTTP JSON API for standard queries.
    *   **Elasticsearch**: Querying historical data (transactions, logs, events).

3.  **Transaction Management**:
    *   **Construction**: Building complex transactions (ESDT transfer, contract calls).
    *   **Signing**: Handling private keys securely (PEM, Keyfile, Ledger).
    *   **Broadcasting**: Sending to the mempool and handling nonce gaps.

## 🛠️ Development Workflow

### 1. Choosing the Right Tool
*   **High Performance / Heavy Load**: Use **Go SDK**.
*   **Web Backends (NestJS, Express)**: Use **TypeScript SDK** (`@multiversx/sdk-core`, `sdk-nestjs`).
*   **Scripts / Data Science / AI**: Use **Python SDK**.

### 2. TypeScript / Node.js Backends
*   **Core Library**: `@multiversx/sdk-core` works in Node.js just like in the browser.
*   **NestJS Integration**: Use `@multiversx/sdk-nestjs` for dependency injection of:
    *   `ApiNetworkProvider` / `ProxyNetworkProvider`
    *   `CachingService` (Redis)
    *   `TransactionProcessor` (Monitoring)
*   **User Verification**: Use `NativeAuth` (server-side) to verify login tokens signed by the frontend.

### 3. Transaction Lifecycle (Backend)
When sending transactions from a backend service:
1.  **Nonce Management**: CRITICAL. You must track the account nonce locally to send high-throughput transactions. Do not rely solely on the API for every sync if speed matters.
2.  **Gas Simulation**: Always simulate transactions (cost calculation) before broadcasting to avoid "Out of Gas" errors.
3.  **Resubmission**: Implement logic to handle "stuck" transactions or dropped packets.

### 3. Data Ingestion
*   **Transaction Processor**: A service that listens to the blockchain for specific events (e.g., "SwapExecuted").
*   **Notifier**: Use MultiversX Notifier Service for push-based updates instead of polling.

## 📚 Expert Resources (Deep Dive)
*   **TS SDK Core Operations**: `skills/expert/mvx_sdk_js_core/SKILL.md` (Essential for Node.js)
*   **Go SDK Builders**: `skills/expert/mvx_sdk_go_builders/SKILL.md`
*   **Go SDK Data**: `skills/expert/mvx_sdk_go_data/SKILL.md`
*   **Python SDK Transactions**: `skills/expert/mvx_sdk_py_transactions/SKILL.md`
*   **Python Contracts**: `skills/expert/mvx_sdk_py_contracts/SKILL.md`

## ⚡ Common Operations (Go)

### Creating a Transaction
```go
tx := &data.Transaction{
    Nonce:    nonce,
    Value:    "1000000000000000000", // 1 EGLD
    Receiver: destinationAddress,
    Sender:   senderAddress,
    GasPrice: 1000000000,
    GasLimit: 50000,
    Data:     []byte("memo"),
    ChainID:  "D", // Devnet
    Version:  1,
}
```

### Signing (Go)
```go
err := interactor.SignTransaction(tx)
```

## ⚡ Common Operations (TypeScript / Node.js)

### Initializing a Provider (NestJS style)
```typescript
import { ApiNetworkProvider } from "@multiversx/sdk-core";

const provider = new ApiNetworkProvider("https://devnet-api.multiversx.com");
const networkConfig = await provider.getNetworkConfig();
```

### Verifying a Login Token (NativeAuth)
```typescript
import { NativeAuthServer } from "@multiversx/sdk-native-auth-server";

const server = new NativeAuthServer();
const result = await server.validate(accessToken);
// result.address, result.ttl, etc.
```


# Go SDK Builders

---
name: mvx_sdk_go_builders
description: Transaction construction and signing using Builders in Go SDK.
---

# MultiversX SDK-Go Builders

Using `TxBuilder` to construct and sign transactions.

## Transaction Builder

```go
import (
    "github.com/multiversx/mx-sdk-go/builders"
    "github.com/multiversx/mx-sdk-go/core"
    "github.com/multiversx/mx-sdk-go/data"
)

// Initialize
txBuilder, err := builders.NewTxBuilder(core.Marshalizer)

// Manual Transaction Construction
tx := &data.Transaction{
    Nonce:    nonce,
    Value:    "1000000000000000000", // 1 EGLD
    RcvAddr:  receiverAddressString,
    SndAddr:  senderAddressString,
    GasPrice: 1000000000,
    GasLimit: 50000,
    Data:     []byte("memo"),
    ChainID:  "D",
    Version:  1,
}

// Sign Transaction
err = txBuilder.ApplySignature(tx, privateKey)
// tx.Signature, tx.SenderUsername are updated
```

## Contract Builders (High Level)

There are specific builders for SC interactions (often generated code or custom implementations), but the core pattern is:

1. Create `data.Transaction` struct
2. Populate fields (Date = endpoint@arg1@arg2)
3. Sign with `TxBuilder`

## Token Transfers

ESDT transfers require specific data fields.

```go
// Native EGLD
tx.Value = "1000000000000000000"
tx.Data = []byte("")

// ESDT Transfer
// Function: ESDTTransfer
// Args: TokenIdentifier (hex), Amount (hex)
tx.Value = "0"
tx.Data = []byte("ESDTTransfer@" + hexTokenId + "@" + hexAmount)
```

## Relayed V3 Transactions

Constructing the inner transaction for a relayed call.

```go
// Inner Tx (User)
innerTx := &data.Transaction{...}
// Sign Inner Tx
signature, _ := userPrivateKey.Sign(innerTxBytes)

// Relayer Tx
relayerTx := &data.Transaction{
    Sender: relayerAddress,
    Data: []byte("relayedTx@" + innerTxBytesHex + "@" + signatureHex),
    ...
}
```

## Data Serialization

The SDK uses a `Marshalizer` interface (usually JSON).

```go
import "github.com/multiversx/mx-sdk-go/core"

bytes, err := core.Marshalizer.Marshal(tx)
```

## Best Practices

1. **Validate fields** before building
2. **Use correct ChainID** ('1', 'D', 'T')
3. **Handle Big Ints as strings** in `Value` field
4. **Gas Limit estimation** is manual or via proxy simulation


# Go SDK Core

---
name: mvx_sdk_go_core
description: Core network operations for MultiversX Go SDK - Proxy, VM Queries.
---

# MultiversX SDK-Go Core Operations

This skill covers the `blockchain` package and network interactions.

## Proxy (Network Client)

Setup the proxy to interact with the blockchain:

```go
import (
    "github.com/multiversx/mx-sdk-go/blockchain"
    "github.com/multiversx/mx-sdk-go/core/http"
)

// Configure Proxy
args := blockchain.ArgsProxy{
    ProxyURL:            "https://devnet-gateway.multiversx.com",
    Client:              http.NewHttpClientWrapper(nil, ""),
    SameScState:         false,
    ShouldBeSynced:      false,
    FinalityCheck:       false,
    CacheExpirationTime: time.Minute,
    EntityType:          core.Proxy,
}

proxy, err := blockchain.NewProxy(args)
```

## Reading Data

```go
// Get Account
address := data.NewAddressFromBech32String("erd1...")
account, err := proxy.GetAccount(ctx, address)
// account.Nonce, account.Balance

// Get Network Config
config, err := proxy.GetNetworkConfig(ctx)

// Get Transaction
tx, err := proxy.GetTransaction(ctx, "txHash", true)
```

## VM Queries (Smart Contract Reads)

Reading state from a smart contract without sending a transaction.

```go
argsQuery := blockchain.ArgsVmQueryGetter{
    Proxy:   proxy,
    Timeout: 10 * time.Second,
}
vmQueryGetter, err := blockchain.NewVmQueryGetter(argsQuery)

// Execute Query
response, err := vmQueryGetter.ExecuteQueryReturningBytes(
    contractAddress,
    "getFunctionName",
    [][]byte{arg1, arg2}, // Arguments as byte slices
)

// response is [][]byte
```

## Shard Coordination

Working with sharded addresses.

```go
// Create Coordinator
coordinator, err := blockchain.NewShardCoordinator(3, 0) // 3 shards, current 0

// Get Shard for Address
shardID := coordinator.ComputeId(address)
```

## Best Practices

1. **Reuse Proxy**: It maintains connection pools/caches
2. **Context**: Always pass context with timeout to avoid hanging
3. **VM Queries**: Preferred for reading state (instant, free)
4. **Error Handling**: Check for specific network errors


# Go SDK Data

---
name: mvx_sdk_go_data
description: Core data structures and types for MultiversX Go SDK.
---

# MultiversX SDK-Go Data Structures

These types define the core objects interacting with the blockchain.

## Transaction

`data.Transaction` is the main struct for all network interactions.

```go
type Transaction struct {
    Nonce            uint64 `json:"nonce"`
    Value            string `json:"value"`
    RcvAddr          string `json:"receiver"`
    SndAddr          string `json:"sender"`
    GasPrice         uint64 `json:"gasPrice,omitempty"`
    GasLimit         uint64 `json:"gasLimit,omitempty"`
    Data             []byte `json:"data,omitempty"`
    Signature        string `json:"signature,omitempty"`
    ChainID          string `json:"chainID"`
    Version          uint32 `json:"version"`
    Options          uint32 `json:"options,omitempty"`
    Guardian         string `json:"guardian,omitempty"`
    GuardianSignature string `json:"guardianSignature,omitempty"`
    Relayer          string `json:"relayer,omitempty"`
    RelayerSignature string `json:"relayerSignature,omitempty"`
}
```

## Address

```go
type AddressWithType struct {
    address []byte
    hrp     string // "erd" usually
}

// Methods
// NewAddressFromBech32String(bech32)
// NewAddressFromBytes(bytes)
// AddressAsBech32String()
```

## Account

```go
type Account struct {
    Address         string `json:"address"`
    Nonce           uint64 `json:"nonce"`
    Balance         string `json:"balance"` // BigInt string
    Username        string `json:"username"`
    CodeHash        string `json:"codeHash"`
    RootHash        string `json:"rootHash"`
    DeveloperReward string `json:"developerReward"`
}
```

## Network Config

```go
type NetworkConfig struct {
    ChainID                  string
    MinGasLimit              uint64
    MinGasPrice              uint64
    GasPerDataByte           uint64
    GasPriceModifier         float64
    AdditionalGasForTxSize   uint64
    ...
}
```

## VM Query

```go
type VmValueRequest struct {
    Address    string   `json:"scAddress"`
    FuncName   string   `json:"funcName"`
    CallValue  string   `json:"value"`
    CallerAddr string   `json:"caller"`
    Args       []string `json:"args"` // Hex encoded arguments
}

type VmValuesResponseData struct {
    ReturnData      []string `json:"returnData"` // Hex encoded return values
    ReturnCode      string   `json:"returnCode"`
    ReturnMessage   string   `json:"returnMessage"`
    GasRemaining    uint64   `json:"gasRemaining"`
    GasRefund       uint64   `json:"gasRefund"`
    OutputAccounts  map[string]*OutputAccount `json:"outputAccounts"`
}
```

## Best Practices

1. **Use `Value` as string**: To prevent overflow (Go's `int64` is too small for big token amounts).
2. **Bech32 addresses**: API expects bech32 strings, internal logic often uses bytes.
3. **Hex encoding**: `Data` field is often hex-encoded when checking explorers, but SDK expects bytes/string.
4. **Options bitmask**: Used for specialized txs (e.g. guarded).


# Go SDK Interactors

---
name: mvx_sdk_go_interactors
description: Components for interacting with the blockchain (Wallets, Transaction Interactors, Nonce Handlers) in Go.
---

# MultiversX SDK-Go Interactors

## Wallet Management

Loading keys and addresses.

```go
import "github.com/multiversx/mx-sdk-go/interactors"

wallet := interactors.NewWallet()

// Load PEM
privateKey, err := wallet.LoadPrivateKeyFromPemFile("wallet.pem")

// Load Keystore
privateKey, err := wallet.LoadPrivateKeyFromKeystoreFile("wallet.json", "password")

// Get Address
address, err := wallet.GetAddressFromPrivateKey(privateKey)
bech32Address := address.AddressAsBech32String()
```

## Transaction Nonce Handler (V3)

Automatically fetches and increments nonces. Recommended way to manage nonces.

```go
import "github.com/multiversx/mx-sdk-go/interactors/nonceHandlerV3"

// Initialize Handler
args := nonceHandlerV3.ArgsAddressNonceHandler{
    Proxy: proxy,
    Address: address,
}
handler, err := nonceHandlerV3.NewAddressNonceHandler(args)

// Fetch Initial Nonce
nonce, err := handler.GetNonce(ctx)

// Apply Nonce to Transaction & Increment
handler.ApplyNonceAndGasPrice(tx) // Sets tx.Nonce = current, then increments internal counter
```

## Transaction Interactor

High-level wrapper for sending transactions.

```go
import "github.com/multiversx/mx-sdk-go/interactors"

// Setup
txInteractor, err := interactors.NewTransactionInteractor(proxy, txBuilder)

// Send Transaction
txHash, err := txInteractor.SendTransaction(ctx, tx, privateKey)

// Send Multiple Transactions
txHashes, err := txInteractor.SendTransactions(ctx, txs, privateKey)
```

## Creating a Wallet Instance

```go
// Helper to create new keypair
privateKey, publicKey := wallet.GeneratePrivateKey()
```

## Best Practices

1. **Use NonceHandlerV3** for robust nonce management, especially in concurrent environments
2. **Use TransactionInteractor** to simplify signing + sending flow
3. **Secure Key Management** - avoid hardcoding private keys


# Python Native Contracts

---
name: mvx_sdk_py_contracts
description: Smart contract operations for MultiversX Python SDK.
---

# MultiversX SDK-Py Smart Contracts

This skill covers ABI loading, deployments, calls, and queries.

## ABI Loading

```python
from multiversx_sdk import Abi
from pathlib import Path

# Load from file
abi = Abi.load("contract.abi.json")
```

## Smart Contract Controller

```python
# Create controller with loaded ABI
controller = entrypoint.create_smart_contract_controller(abi=abi)
```

## Deployment

```python
bytecode = Path("contract.wasm").read_bytes()

tx = controller.create_transaction_for_deploy(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    bytecode=bytecode,
    gas_limit=60000000,
    arguments=[42]  # Typed automatically via ABI
)

tx_hash = entrypoint.send_transaction(tx)
parsed_outcome = controller.await_completed_deploy(tx_hash)
contract_address = parsed_outcome[0].contract_address
```

## Contract Calls

```python
# Call endpoint
tx = controller.create_transaction_for_execute(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    contract=contract_address,
    function="add",
    arguments=[10],
    gas_limit=5000000
)

# Call with token transfer (Transfer & Execute)
tx = controller.create_transaction_for_execute(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    contract=contract_address,
    function="deposit",
    gas_limit=10000000,
    native_amount=1000000000000000000
)
```

## VM Queries

Read-only calls to the contract state.

```python
# Using controller (parsed output)
result = controller.query_contract(
    contract=contract_address,
    function="getSum",
    arguments=[]
)
print(result[0])  # Typed output
```

## Without ABI (Manual Typing)

If ABI is not available, use manual type hinting (less recommended).

```python
from multiversx_sdk import SmartContractTransactionsFactory

factory = entrypoint.create_smart_contract_transactions_factory()

tx = factory.create_transaction_for_execute(
    sender=account.address,
    contract=contract_address,
    function="add",
    gas_limit=5000000,
    arguments=[42]  # Basic types inferred
)
```

## Upgrades

```python
new_bytecode = Path("contract_v2.wasm").read_bytes()

tx = controller.create_transaction_for_upgrade(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    contract=contract_address,
    bytecode=new_bytecode,
    gas_limit=60000000,
    arguments=[]
)
```

## Best Practices

1. **Always use ABI**: Ensures correct argument encoding/decoding
2. **Query for reads**: Free (no gas), instant response
3. **Wait for completion**: Use controller methods to get parsed results
4. **Gas limit**: Ensure enough gas for computation + storage


# Python SDK Core

---
name: mvx_sdk_py_core
description: Core SDK operations for MultiversX Python SDK - Entrypoints, Providers, and Network.
---

# MultiversX SDK-Py Core Operations

This skill covers the fundamental building blocks of the `multiversx-sdk` v2 library.

## Entrypoints

Network-specific clients that simplify common operations:

| Entrypoint | Network | Default URL |
|------------|---------|-------------|
| `DevnetEntrypoint` | Devnet | `https://devnet-api.multiversx.com` |
| `TestnetEntrypoint` | Testnet | `https://testnet-api.multiversx.com` |
| `MainnetEntrypoint` | Mainnet | `https://api.multiversx.com` |
| `LocalnetEntrypoint` | Local | `http://localhost:7950` |

```python
from multiversx_sdk import DevnetEntrypoint

# Default API
entrypoint = DevnetEntrypoint()

# Custom API
entrypoint = DevnetEntrypoint(url="https://custom-api.com")

# Proxy mode (gateway)
entrypoint = DevnetEntrypoint(url="https://devnet-gateway.multiversx.com", kind="proxy")
```

## Entrypoint Methods

| Method | Description |
|--------|-------------|
| `create_account()` | Create a new account instance |
| `create_network_provider()` | Get underlying network provider |
| `recall_account_nonce(address)` | Fetch current nonce from network |
| `send_transaction(tx)` | Broadcast single transaction |
| `send_transactions(txs)` | Broadcast multiple transactions |
| `get_transaction(tx_hash)` | Fetch transaction by hash |
| `await_completed_transaction(tx_hash)` | Wait for finality |

## Network Providers

Lower-level network access:

| Provider | Use Case |
|----------|----------|
| `ApiNetworkProvider` | Standard API queries (recommended) |
| `ProxyNetworkProvider` | Direct node interaction via gateway |

```python
# From entrypoint
provider = entrypoint.create_network_provider()

# Manual instantiation
from multiversx_sdk import ApiNetworkProvider

api = ApiNetworkProvider("https://devnet-api.multiversx.com", client_name="my-app")
```

## Provider Methods

| Method | Description |
|--------|-------------|
| `get_network_config()` | Chain ID, gas settings |
| `get_network_status()` | Current epoch, nonce |
| `get_block(block_hash)` | Fetch block data |
| `get_account(address)` | Account balance, nonce |
| `get_account_storage(address)` | Contract storage |
| `get_transaction(tx_hash)` | Transaction details |
| `query_contract(query)` | VM query (read-only) |

## Transaction Lifecycle

```python
# 1. Create account and sync nonce
account = Account.new_from_pem("wallet.pem")
account.nonce = entrypoint.recall_account_nonce(account.address)

# 2. Create transaction (via controller)
controller = entrypoint.create_transfers_controller()
tx = controller.create_transaction_for_transfer(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    receiver=receiver_address,
    native_amount=1000000000000000000
)

# 3. Send
tx_hash = entrypoint.send_transaction(tx)

# 4. Wait for completion
result = entrypoint.await_completed_transaction(tx_hash)
print(f"Status: {result.status}")
```

## Best Practices

1. **Use Entrypoints**: Simplifies configuration
2. **Sync nonce**: Before creating any transaction
3. **Handle exceptions**: Wrap network calls
4. **Use integers**: For amounts (18 decimals for EGLD)


# Python SDK Transactions

---
name: mvx_sdk_py_transactions
description: Transaction creation and token operations for MultiversX Python SDK.
---

# MultiversX SDK-Py Transactions & Tokens

This skill covers creating transactions using Controllers and Factories.

## Transfers

### Native EGLD Transfer
```python
controller = entrypoint.create_transfers_controller()

tx = controller.create_transaction_for_transfer(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    receiver=receiver_address,
    native_amount=1000000000000000000
)
```

### ESDT Token Transfer
```python
from multiversx_sdk import TokenTransfer

tx = controller.create_transaction_for_transfer(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    receiver=receiver_address,
    token_transfers=[
        TokenTransfer.fungible_from_amount("TOKEN-123456", 100.5, 18)
    ]
)
```

### NFT/SFT Transfer
```python
tx = controller.create_transaction_for_transfer(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    receiver=receiver_address,
    token_transfers=[
        TokenTransfer.nft_from_amount("NFT-123456", 1, 1)
    ]
)
```

## Token Management

### Issue Fungible Token
```python
controller = entrypoint.create_token_management_controller()

tx = controller.create_transaction_for_issuing_fungible(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    token_name="MyToken",
    token_ticker="MTK",
    initial_supply=1000000000000,
    num_decimals=6,
    can_freeze=False,
    can_wipe=True,
    can_pause=False,
    can_change_owner=True,
    can_upgrade=True,
    can_add_special_roles=True
)

tx_hash = entrypoint.send_transaction(tx)
outcome = controller.await_completed_issue_fungible(tx_hash)
print(outcome[0].token_identifier)
```

## Transaction Parsing

### Decoding Transaction Data
```python
from multiversx_sdk import TransactionDecoder

decoded = TransactionDecoder.decode_transaction(tx)
print(decoded.function)
print(decoded.arguments)
```

### Parsing Events
```python
tx_on_network = entrypoint.get_transaction(tx_hash)

for event in tx_on_network.logs.events:
    print(f"Event: {event.identifier}")
    for topic in event.topics:
        print(f"Topic: {topic.hex()}")
```

## Relayed Transactions (V3)

```python
tx = Transaction(
    sender=user_address,
    receiver=receiver_address,
    relayer=relayer_address,
    gas_limit=150000,  # +50k for relayed
    data=b"endpoint@args"
)

tx.signature = user_account.sign_transaction(tx)
tx.relayer_signature = relayer_account.sign_transaction(tx)

entrypoint.send_transaction(tx)
```

## Best Practices

1. **Use `TokenTransfer` helpers**: handles decimals correctly
2. **Check issuance outcome**: tokens get random suffix
3. **Relayed transactions**: sender and relayer must be in same shard
4. **Gas estimation**: Controllers generally handle basic estimation


# Python SDK Wallets

---
name: mvx_sdk_py_wallets
description: Wallet and key management for MultiversX Python SDK.
---

# MultiversX SDK-Py Wallets & Signing

This skill covers account management, key derivation, and transaction signing.

## Account Creation

```python
from multiversx_sdk import Account, Mnemonic, UserWallet, UserPem

# From PEM file (testing only)
account = Account.new_from_pem("wallet.pem")

# From keystore (production)
account = Account.new_from_keystore("wallet.json", "password")

# From secret key (hex string)
account = Account(secret_key=bytes.fromhex("..."))
```

## Mnemonic Operations

```python
# Generate new mnemonic
mnemonic = Mnemonic.generate()
words = mnemonic.get_words()

# Derive keys
secret_key = mnemonic.derive_key(0)  # Index 0
public_key = secret_key.generate_public_key()
address = public_key.to_address("erd")

# Get address from mnemonic
address = Mnemonic.to_address(mnemonic, 0)
```

## Wallet Storage

```python
# Save mnemonic to keystore
wallet = UserWallet.from_mnemonic(mnemonic.get_text(), "password")
wallet.save("wallet.json")

# Save secret key to keystore
wallet = UserWallet.from_secret_key(secret_key, "password")
wallet.save("wallet.json")

# Save to PEM (testing only)
pem = UserPem(label=address.to_bech32(), secret_key=secret_key)
pem.save("wallet.pem")
```

## Transaction Signing

```python
# Controller-created transactions are auto-signed
tx = controller.create_transaction_for_transfer(account, nonce, ...)

# Manual signing
tx.signature = account.sign_transaction(tx)
```

## Message Signing

```python
from multiversx_sdk import Message

message = Message("Hello".encode())
signature = account.sign_message(message)

# Verify
is_valid = account.verify_message(message, signature)
```

## NativeAuth

Authentication for dApps:

```python
from multiversx_sdk import NativeAuthClient, NativeAuthServer

# Client: Generate token
client = NativeAuthClient("https://app.example.com")
init_token = client.initialize()
# sign init_token...
access_token = client.get_token(address, init_token, signature)

# Server: Validate token
server = NativeAuthServer(accepted_origins=["https://app.example.com"])
result = server.validate(access_token)
# result.address, result.origin, result.block_hash
```

## Ledger

Requires `ledger-wallets` dependency.

```python
from multiversx_sdk import LedgerAccount

ledger = LedgerAccount(index=0)
print(ledger.address)
tx.signature = ledger.sign_transaction(tx)
```

## Security Best Practices

1. **Protect keystore passwords**
2. **Never expose PEM files** in production
3. **Validate addresses** before sending
4. **Use NativeAuth** for authentication
