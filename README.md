### Repository structure

- **`proto/`**: Shared `.proto` schema (`secure_range_proof.proto`)
- **`server/`**: C++ server (Boost.Asio + trezor-crypto + nanopb)
- **`client/`**: Node.js TypeScript client (net + crypto + protobufjs)

---

## Protocol summary

### Authentication

- **Client → Server**: `ClientHello`
  - `serial_id`
  - `sig = ECDSA_sign( SHA256(serial_id) )`
- **Server → Client**: `ServerChallenge`
  - `nonce` (32 random bytes)
  - `server_sig = ECDSA_sign( SHA256(serial_id || nonce) )`
- **Client → Server**: `ClientResponse`
  - `sig = ECDSA_sign( SHA256(nonce) )`
- **Server** prints: `Client verified: serial_id=<serial>`

All signatures are **P1363 / IEEE** format (`r||s`, **64 bytes**).

### Range proof (ECC commitments)

Client proves a secret \(x\) lies in \([a,b]\) by proving both:

- \(x-a \ge 0\)
- \(b-x \ge 0\)

Using the “sum of four squares” fact and ECC commitments:

- Lower commitment: `c2 = (x-a)·G + r·H`
- Upper commitment: `c1 = (b-x)·G − r·H`
- Plus **4 sub-commitments** each for `(x-a)` and `(b-x)` using the squares decomposition

Server verifies:

- `sum(lower_commit[i]) == c2`
- `sum(upper_commit[i]) == c1`
- `c1 + c2 == (b-a)·G`
- `p1 = b·G − c1` equals `p2 = c2 + a·G`

> Note: this follows the prompt’s structure. It is a **demonstration** of the described protocol and is **not a production-grade ZK range proof**.

---

## Build & run (macOS)

### Clone the repo

```
git clone --recurse-submodules <repo-url>
```

### Prerequisites

Install tools via Homebrew:

```bash
brew update
brew install cmake boost protobuf python node
python3 -m pip install --user protobuf
```

### Build server (C++)

From repo root:

```bash
cmake -S . -B build
cmake --build build -j
./build/server/srp_server --port 9000 --config server/config/server.conf
```

### Run client (Node.js / TypeScript)

In a second terminal:

```bash
cd client
npm install
npm run dev -- --host 127.0.0.1 --port 9000 --config ./config/client.conf --bitlen 16 --requests 1
```

You should see:

- client: `[auth] ok: ...`
- server: `Client verified: serial_id=...`
- both sides: range-proof verification success


