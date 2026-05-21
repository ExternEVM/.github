# ExternEVM

**Smart contracts that talk to the internet.**

ExternEVM is a custom blockchain runtime — built on a modified [Reth](https://github.com/paradigmxyz/reth) execution client — where Solidity contracts can call any external API during execution. No oracles. No off-chain relayers. No token subscriptions. Just a native precompile baked into the EVM that makes HTTP requests, parses JSON, and returns data directly to your contract.

```solidity
function getBitcoinPrice() external view returns (uint256) {
    ApiRequest memory req = ApiRequest({
        url: "https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd",
        method: "GET",
        headers: "",
        body: "",
        responsePath: "bitcoin.usd",
        responseType: 1
    });
    (bool ok, bytes memory out) = API_CALL.staticcall(abi.encode(req));
    require(ok, "API_CALL failed");
    return abi.decode(out, (uint256));
}
```

That contract returns the live Bitcoin price. Not from an oracle. Not from a Chainlink feed. From CoinGecko's API, fetched by the EVM itself during execution.

---

## How It Works

A custom precompile lives at address `0x00000000000000000000000000000000000000AA` inside a modified Reth node. When a Solidity contract calls that address via `staticcall`, it triggers native Rust code that:

1. Decodes the ABI-encoded `ApiRequest` struct from calldata
2. Validates the URL, method, headers, and body
3. Blocks private IPs, enforces timeouts, caps response size
4. Makes a real HTTP request using `reqwest` inside `tokio::task::block_in_place()`
5. Parses the JSON response
6. Extracts the value at the specified JSON path
7. ABI-encodes the result
8. Returns it to the contract

From the contract's perspective — it's a `staticcall`. From the node's perspective — it's an HTTP round-trip happening inside block execution.

```
Solidity Contract
    │ staticcall(0xAA, abi.encode(ApiRequest))
    ▼
Modified Reth EVM
    │ Routes to API_CALL precompile
    ▼
Native Rust Precompile
    │ HTTP request → JSON parse → ABI encode
    ▼
Contract receives abi.decode(out, (uint256))
    │ Uses live data in logic
    ▼
Done. No oracle. No middleware. No waiting.
```

---

## What's Been Built

### v1 — Single-Node Direct API Calls ✅

The foundation. One modified Reth node that executes real HTTP calls during EVM execution.

- Custom precompile at `0xAA` registered via `EvmFactory` trait injection
- Full ABI decoding/encoding with `alloy-sol-types`
- Live HTTP calls — Bitcoin price, ETH price, weather data, ISS coordinates, astronaut count
- JSON path extraction with dot notation and array indexing (`bitcoin.usd`, `properties.periods[0].temperature`)
- All 4 response types: `uint256`, `string`, `bool`, raw `bytes`
- URL validation, private IP blocking (`127.0.0.1`, `10.x.x.x`, `192.168.x.x`), redirect blocking
- 5-second timeout, 4KB request body limit, 32KB response cap
- Docker containerized — `docker compose up --build` runs the chain
- Foundry, MetaMask, and Remix compatible
- Chain ID: `22042004`

### v2 — Multi-Node Median Aggregation 🔨 In Progress

The leap from "one trusted node" to "multiple validators agreeing on reality."

- **Custom `extern/1` devp2p subprotocol** — nodes broadcast fetched API values to peers via RLP-encoded wire messages over RLPx
- **Median aggregation** for `uint256` responses — sort values from all validators, take the middle. One liar can't move the median
- **Majority vote** for `string` and `bool` responses — 2/3 agreement required
- **Deterministic request hashing** — `keccak256(url || method || body || responsePath || responseType)` for cross-node value correlation
- **Custom consensus layer binary** (`externevm-consensus`) — a standalone Rust binary implementing the Ethereum Engine API with JWT authentication
- **Round-robin PoA** proposer selection with 5-second slots and missed-slot recovery
- **3-node devnet** — each node runs Reth (EL) + `externevm-consensus` (CL), connected via p2p, producing blocks and exchanging API values independently

#### Architecture

```
┌──────────────────────────────────┐
│   externevm-consensus (CL)       │
│   Round-Robin PoA                │
│   ConsensusStrategy trait        │
│   v2: RoundRobin                 │
│   v4: StakeWeighted (future)     │
└───────────────┬──────────────────┘
                │ Engine API (HTTP + JWT)
                │ forkchoiceUpdatedV3
                │ getPayloadV3
                │ newPayloadV3
┌───────────────▼──────────────────┐
│   Modified Reth (EL)             │
│                                  │
│   0xAA  API_CALL precompile      │
│   extern/1  devp2p subprotocol   │
│   Protocol store (in-memory)     │
│                                  │
│   Future:                        │
│   0xA1  API_REQUEST              │
│   0xA2  API_READ                 │
└──────────────────────────────────┘
```

The EL/CL separation mirrors real Ethereum — Reth handles execution, the consensus binary handles block production. They communicate over the same Engine API that Lighthouse/Prysm use. Swapping consensus logic (round-robin → stake-weighted → BFT) means recompiling one binary. Reth stays untouched.

---

## The ApiRequest Struct

```solidity
struct ApiRequest {
    string url;            // Full URL
    string method;         // "GET" or "POST"
    bytes headers;         // JSON-encoded headers or empty
    bytes body;            // Request body for POST, empty for GET
    string responsePath;   // Dot-notation JSON path — "bitcoin.usd"
    uint8 responseType;    // 0 = bytes, 1 = uint256, 2 = string, 3 = bool
}
```

This struct is stable across all versions. The same contract works on v1 through v5. What changes underneath is how the data gets verified — not how you ask for it.

---

## Roadmap

### v2 — Multi-Node Median Aggregation *(in progress)*
Multiple validators fetch API data independently. Chain computes deterministic median. One malicious node can't move the result. Open submission model — sufficient for a permissioned validator set.

### v3 — Commit-Reveal Consensus
Validators commit `keccak256(value || salt)` first, then reveal. Prevents frontrunning, free-riding, and last-mover advantage. No one sees anyone's answer until everyone has committed. Requires ≥2/3 honest validators.

### v4 — Stake-Weighted Finalization + Slashing
Validators stake tokens to participate. Misbehavior (commit without reveal, hash mismatch, consecutive misses) gets slashed. Stake-weighted median gives more influence to validators with more skin in the game. Cost of attack becomes calculable.

### v5 — TEE Attestation + ZK Proof of Fetch
Hardware enclaves (SGX/Nitro) prove a validator actually made the HTTP request. TLSNotary provides software-based proof of fetch. ZK circuits prove JSON parsing was done correctly without revealing the raw response. Trust math, not people.

### Future — `solc` Modification
Replace the verbose `staticcall` pattern with native syntax:

```solidity
// Today (v1):
(bool ok, bytes memory out) = API_CALL.staticcall(abi.encode(req));
uint256 price = abi.decode(out, (uint256));

// Future:
uint256 price = api_call("https://api.coingecko.com/...", "bitcoin.usd");
```

Compiler lowers `api_call(...)` into the same precompile call — ABI encode, staticcall, decode — just without the boilerplate.

---

## Running Locally

```bash
# Clone with submodules
git clone --recursive https://github.com/ExternEVM/ExternEVM.git
cd ExternEVM

# Build modified Reth (~15-30 min first time)
cd reth && cargo build --release

# Start the node
cargo run --release -- node \
  --dev \
  --chain ../config/genesis.json \
  --http \
  --http.api eth,net,web3,debug,trace \
  --http.addr 0.0.0.0 \
  --http.port 8545

# Deploy and call (new terminal)
cd ../contracts && forge build
forge create src/ExternApiDemo.sol:ExternApiDemo \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

# Get live Bitcoin price from a smart contract
cast call $CONTRACT "getBitcoinPrice()(uint256)" --rpc-url http://127.0.0.1:8545
```

Or with Docker:

```bash
docker compose up --build
# Node running on port 8545
```

---

## The Question ExternEVM Asks

Every blockchain today uses oracles for external data — application-layer services that push data on-chain through smart contracts, flowing through the public mempool, operated by third parties, paid for with subscription tokens.

**Should external data access be a protocol-level primitive instead?**

Protocol-level means the precompile is part of the runtime — like `ecrecover` or `sha256`. Not a contract someone deployed. Not a service someone operates. Data finalization happens below the transaction layer, reducing MEV surface. Every contract on the chain has native access — no oracle deployment, no subscription, no middleware.

ExternEVM is building the answer.

---

## ⚠️ Warning

ExternEVM v1 is **single-node only**. Direct HTTP calls during EVM execution are non-deterministic and unsafe for multi-validator blockchains. This is an experimental research chain for development, demos, and prototyping. The multi-node path (v2+) replaces direct calls with validator aggregation and consensus-safe finalization.

---

## License

MIT
