# ExternEVM

A custom Reth-based EVM runtime where Solidity smart contracts call external APIs during execution — at the protocol level.

No oracle contracts. No off-chain relayers. No token subscriptions. Just a native EVM precompile that makes HTTP requests, parses JSON, and returns ABI-encoded data directly to your contract.

---

### What's working now (v1 — complete)

- Custom precompile at `0xAA` baked into a modified Reth execution client
- Live HTTP calls during EVM execution — Bitcoin prices, weather data, ISS coordinates, anything with a JSON API
- Full ABI decoding/encoding with alloy-sol-types
- URL validation, private IP blocking, timeout enforcement, redirect blocking
- All 4 response types: uint256, string, bool, raw bytes
- JSON path extraction with dot notation and array indexing
- Docker containerized — `docker compose up` and you have a running chain
- Foundry + MetaMask + Remix compatible

### What's in progress (v2 — multi-node)

- Custom `extern/1` devp2p subprotocol for cross-node value broadcasting
- RLP-encoded wire messages for peer-to-peer API value exchange
- Median aggregation (uint256) and majority vote (string/bool) across validators
- Deterministic request hashing for cross-node value correlation
- Custom consensus layer binary (`externevm-consensus`) implementing the Ethereum Engine API
- Round-robin PoA proposer selection with JWT-authenticated EL↔CL communication
- Multi-node devnet — 3 nodes syncing blocks, exchanging API values, computing median independently

### Roadmap

- **v2** — Multi-node median aggregation with custom CL over Engine API
- **v3** — Commit-reveal consensus — cryptographic hiding prevents frontrunning and free-riding
- **v4** — Stake-weighted finalization with slashing — economic security model
- **v5** — TEE attestation + ZK proof of fetch — trust math, not people

### Architecture
┌────────────────────────────┐
│   externevm-consensus      │  ← Custom CL binary (round-robin PoA)
│   Round-Robin → Stake PoS  │
└─────────────┬──────────────┘
│ Engine API (JWT auth)
│ forkchoiceUpdated / getPayload / newPayload
┌─────────────▼──────────────┐
│   Modified Reth (EL)       │
│   Precompile 0xAA — API_CALL (v1+)          │
│   extern/1 subprotocol — p2p values (v2+)   │
│   Future: 0xA1 API_REQUEST / 0xA2 API_READ  │
└────────────────────────────┘

### The question ExternEVM asks

Should external data access be a protocol-level primitive rather than an application-layer service?

We're building the answer.
