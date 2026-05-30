# ExternEVM

**Smart contracts that talk to the internet.**

ExternEVM is a custom blockchain protocol — execution layer built on a modified [Reth](https://github.com/paradigmxyz/reth), consensus layer built from scratch over the [Ethereum Engine API](https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md) — where Solidity contracts can call any external API during execution. No oracles. No off-chain relayers. No token subscriptions. Just a native precompile baked into the EVM that makes HTTP requests, parses JSON, and returns data directly to your contract.

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

In a multi-node deployment, each validator fetches independently, broadcasts via a custom devp2p subprotocol (`extern/1`), and the precompile returns the **median** of all submissions — tolerating minority Byzantine behavior at the protocol level.

```
Solidity Contract
    │ staticcall(0xAA, abi.encode(ApiRequest))
    ▼
Modified Reth EVM
    │ Routes to API_CALL precompile
    ▼
Native Rust Precompile
    │ HTTP fetch → broadcast to peers → collect values → compute median
    ▼
Contract receives abi.decode(out, (uint256))
    │ Aggregated result from multiple validators
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

### v2 — Multi-Node Consensus + Median Aggregation ✅

The leap from "one trusted node" to "multiple validators agreeing on reality."

**Execution Layer (Modified Reth):**

- **Protocol store** — thread-safe in-memory storage (`Arc<RwLock<>>` + `LazyLock` singleton) for pending requests, validator submissions, and finalized results. 15 unit tests passing.
- **Median aggregation** for `uint256` — sort values from all validators, take the middle. One liar can't move the median.
- **Majority vote** for `string` and `bool` — 2/3 agreement required for finalization.
- **Custom `extern/1` devp2p subprotocol** — nodes broadcast fetched API values to peers via RLP-encoded wire messages over RLPx. `ProtocolHandler` → `ConnectionHandler` → bidirectional `Stream` implementation.
- **Deterministic request hashing** — `keccak256(url ‖ 0xFF ‖ method ‖ 0xFF ‖ responsePath ‖ 0xFF ‖ responseType)` for cross-node value correlation.

**Consensus Layer (`externevm-consensus`):**

- **Standalone Rust binary** — zero Reth dependencies, pure HTTP client using `reqwest` + `serde_json` + `jsonwebtoken`.
- **Round-Robin PoA** proposer selection — validators take turns producing blocks every 5 seconds, following [EIP-225](https://eips.ethereum.org/EIPS/eip-225) (Clique) conceptually but implemented as a separate CL binary over the Engine API.
- **Engine API V3** — `forkchoiceUpdatedV3`, `getPayloadV3`, `newPayloadV3` per the [Cancun spec](https://github.com/ethereum/execution-apis/blob/main/src/engine/cancun.md). JWT authentication per the [Engine API auth spec](https://github.com/ethereum/execution-apis/blob/main/src/engine/authentication.md).
- **`ConsensusStrategy` trait** — the swap point for future consensus upgrades. v2 implements `RoundRobin`. v4 will implement `StakeWeighted`. Same Engine API, same binary structure, different selection logic.
- **3-node devnet confirmed working** — round-robin block production across 3 validators, all nodes accepting each other's blocks via Engine API.

#### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     ExternEVM Protocol                        │
│                                                               │
│  ┌───────────────────────┐  Engine API   ┌─────────────────┐ │
│  │   Execution Layer     │◄─(EIP-3675)──►│ Consensus Layer │ │
│  │   (Modified Reth)     │  JWT Auth     │                 │ │
│  │                       │  Port 8551    │ externevm-      │ │
│  │  0xAA API_CALL        │               │ consensus       │ │
│  │  extern/1 subproto    │               │                 │ │
│  │  Protocol store       │               │ Round-Robin PoA │ │
│  │  Median aggregation   │               │ 5-second slots  │ │
│  │                       │               │ Fork choice     │ │
│  └───────────┬───────────┘               └────────┬────────┘ │
│              │ eth/68 + extern/1                   │          │
│              └─────────────┬───────────────────────┘          │
│                            │                                  │
│                    ┌───────▼────────┐                         │
│                    │  Peer Nodes    │                         │
│                    │  (same stack)  │                         │
│                    └────────────────┘                         │
└──────────────────────────────────────────────────────────────┘
```

The EL/CL separation follows [EIP-3675](https://eips.ethereum.org/EIPS/eip-3675) (The Merge). This mirrors the Lighthouse↔Reth / Prysm↔Geth architecture of production Ethereum — but with custom consensus logic and native external data access. Swapping consensus (round-robin → stake-weighted → BFT) means recompiling one binary. Reth stays untouched.

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

### v1 — Single-Node Direct API Calls ✅
Foundation. One Reth node, one precompile, live HTTP during execution.

### v2 — Multi-Node Median Aggregation ✅
Custom consensus layer (round-robin PoA via Engine API). Multiple validators fetch independently, broadcast via `extern/1` devp2p subprotocol, chain computes median. 3-node devnet confirmed working.

### v3 — Commit-Reveal Consensus
Validators commit `keccak256(value || salt)` first, then reveal. Prevents frontrunning, free-riding, and last-mover advantage. No one sees anyone's answer until everyone has committed. Requires ≥2/3 honest validators.

### v4 — Stake-Weighted Finalization + Slashing
Validators stake tokens to participate. Misbehavior (commit without reveal, hash mismatch, consecutive misses) gets slashed. Stake-weighted median gives more influence to validators with more at risk. Cost of attack becomes calculable. `ConsensusStrategy` trait swaps from `RoundRobin` to `StakeWeighted`.

### v5 — TEE Attestation + ZK Proof of Fetch
Hardware enclaves (SGX/Nitro) prove a validator actually made the HTTP request and received the response they claim. TLSNotary provides software-based proof of fetch. ZK circuits prove JSON parsing was done correctly without revealing the raw response. Trust math, not people.

### Future — `solc` Modification
Replace the verbose `staticcall` pattern with native syntax:

```solidity
// Today:
(bool ok, bytes memory out) = API_CALL.staticcall(abi.encode(req));
uint256 price = abi.decode(out, (uint256));

// Future:
uint256 price = api_call("https://api.coingecko.com/...", "bitcoin.usd");
```

Compiler lowers `api_call(...)` into the same precompile call — ABI encode, staticcall, decode — just without the boilerplate.

---

## Running Locally

### Single-Node (v1 mode)

```bash
git clone --recursive https://github.com/ExternEVM/ExternEVM.git
cd ExternEVM

cd reth && cargo build --release
cargo run --release -- node \
  --dev \
  --chain ../config/genesis.json \
  --http --http.api eth,net,web3,debug,trace \
  --http.addr 0.0.0.0 --http.port 8545

# Deploy and call (new terminal)
cd ../contracts && forge build
forge create src/ExternApiDemo.sol:ExternApiDemo \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

cast call $CONTRACT "getBitcoinPrice()(uint256)" --rpc-url http://127.0.0.1:8545
```

### Multi-Node Devnet (v2 mode)

Requires 6 terminals — 3 Reth nodes (EL) + 3 consensus nodes (CL). Each node runs both binaries communicating over the Engine API with shared JWT authentication.

```bash
# Build both layers
cd reth && cargo build --release && cd ..
cd consensus && cargo build && cd ..
openssl rand -hex 32 > config/jwt.hex

# Start Node 1 EL
EXTERNEVM_VALIDATOR_ADDRESS=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 \
cargo run --release -- node \
  --chain ../config/genesis.json \
  --http --http.api eth,net,web3,debug,trace,admin \
  --http.addr 0.0.0.0 --http.port 8545 \
  --authrpc.port 8551 --authrpc.jwtsecret ../config/jwt.hex \
  --port 30303 --discovery.port 30303 --discovery.v5.port 9200 \
  --datadir /tmp/externevm-node1

# Start Node 1 CL
cargo run -- \
  --validator 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 \
  --validators 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266,0x70997970C51812dc3A010C7d01b50e0d17dc79C8,0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC \
  --el-auth-url http://127.0.0.1:8551 --el-rpc-url http://127.0.0.1:8545 \
  --all-el-auth-urls http://127.0.0.1:8551,http://127.0.0.1:8552,http://127.0.0.1:8553 \
  --jwt-secret ../config/jwt.hex --slot-time 5
```

Nodes 2 and 3 use different ports (`8546`/`8552`/`30304`/`9201` and `8547`/`8553`/`30305`/`9202`) and connect to Node 1 via `--trusted-peers`. See the [full devnet guide](./README.md) in the repo.

### Docker

```bash
docker compose up --build
# Single-node on port 8545
```

---

## References

| Specification | How ExternEVM Uses It |
|---------------|----------------------|
| [EIP-3675 — The Merge](https://eips.ethereum.org/EIPS/eip-3675) | EL/CL separation — consensus binary talks to Reth via Engine API |
| [EIP-225 — Clique PoA](https://eips.ethereum.org/EIPS/eip-225) | Round-robin proposer selection design (reimplemented over Engine API) |
| [EIP-4399 — PREVRANDAO](https://eips.ethereum.org/EIPS/eip-4399) | Deterministic per-slot randomness in payload attributes |
| [Engine API — Paris](https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md) | `forkchoiceUpdated`, `newPayload`, `getPayload` core methods |
| [Engine API — Cancun](https://github.com/ethereum/execution-apis/blob/main/src/engine/cancun.md) | V3 methods with `parentBeaconBlockRoot` field |
| [Engine API — Auth](https://github.com/ethereum/execution-apis/blob/main/src/engine/authentication.md) | JWT HS256 authentication between EL and CL |
| [Chainlink Whitepaper v2](https://research.chain.link/whitepaper-v2.pdf) | Application-layer oracle design (comparison point) |
| [TLSNotary](https://tlsnotary.org/) | Software-based proof of TLS fetch (v5 research reference) |

---

## The Question ExternEVM Asks

Every blockchain today uses oracles for external data — application-layer services that push data on-chain through smart contracts, flowing through the public mempool, operated by third parties, paid for with subscription tokens.

**Should external data access be a protocol-level primitive instead?**

Protocol-level means the precompile is part of the runtime — like `ecrecover` or `sha256`. Not a contract someone deployed. Not a service someone operates. Data finalization happens below the transaction layer, reducing MEV surface. Every contract on the chain has native access — no oracle deployment, no subscription, no middleware.

ExternEVM is building the answer.

---

## ⚠️ Experimental Protocol

ExternEVM is research software. The v2 devnet uses round-robin PoA — suitable for development, demonstration, and protocol research. The roadmap progresses through commit-reveal schemes ([v3](#roadmap)), stake-weighted consensus with slashing ([v4](#roadmap)), and TEE/ZK-assisted verification ([v5](#roadmap)).

---

## License

MIT

---

Made with 💖 by [Prateush Sharma](https://github.com/prateushsharma)
