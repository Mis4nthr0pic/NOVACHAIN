# NOVA Documentation

## Bitcoin-Grade Security. Ethereum-Grade Programmability. Smart Contracts With the Footguns Removed.

**Version:** v0.2  
**Status:** Concept / architecture draft  
**Format:** Markdown documentation  
**Scope:** Updated from the original NOVA litepaper plus the new design standards discussed afterward.

---

## Table of Contents

1. [Signal](#1-signal)
2. [Quick Spec](#2-quick-spec)
3. [Why NOVA Exists](#3-why-nova-exists)
4. [Core Architecture](#4-core-architecture)
5. [Consensus: PoW + BlockDAG](#5-consensus-pow--blockdag)
6. [Aurora Finality](#6-aurora-finality)
7. [NovaVM](#7-novavm)
8. [Pulsar](#8-pulsar)
9. [Pulsar Safety Model](#9-pulsar-safety-model)
10. [Gas and Fees](#10-gas-and-fees)
11. [Tokenomics](#11-tokenomics)
12. [Accounts, Addresses, and Human Names](#12-accounts-addresses-and-human-names)
13. [Transactions and Receipts](#13-transactions-and-receipts)
14. [Events, Logs, and Bloom Filters](#14-events-logs-and-bloom-filters)
15. [Randomness](#15-randomness)
16. [Oracles and HTTP Attestations](#16-oracles-and-http-attestations)
17. [Precompiles](#17-precompiles)
18. [Quantum Resilience](#18-quantum-resilience)
19. [Anti-Blind-Signing Design](#19-anti-blind-signing-design)
20. [No Infinite Approvals](#20-no-infinite-approvals)
21. [On-Chain Contract Verification](#21-on-chain-contract-verification)
22. [Compressed Source Availability](#22-compressed-source-availability)
23. [On-Chain Runtime Upgrades](#23-on-chain-runtime-upgrades)
24. [Bridges](#24-bridges)
25. [Bootstrapping and Security Modes](#25-bootstrapping-and-security-modes)
26. [NRC Standards Index](#26-nrc-standards-index)
27. [Roadmap](#27-roadmap)
28. [What NOVA Does Not Claim](#28-what-nova-does-not-claim)
29. [Culture and Lore](#29-culture-and-lore)
30. [Closing](#30-closing)

---

# 1. Signal

Bitcoin gave the world money no government can stop.

Ethereum gave the world programs no company can shut down.

NOVA combines both with a third requirement:

> Smart contracts should be safer by construction.

Not safer because every developer remembered every checklist.

Not safer because every protocol hired the perfect auditor.

Not safer because users trusted another proxy, multisig, oracle, or middleware.

Safer because entire catastrophic bug classes are not expressible in the language.

NOVA is a permissionless Layer-1 blockchain with proof-of-work mining, sub-second blocks, deterministic finality, a deterministic WASM-based VM, and a new smart contract language called **Pulsar**.

The thesis is simple:

> The chain should remove the footguns.

---

# 2. Quick Spec

| Parameter | Value |
|---|---:|
| Name | NOVA |
| Ticker | NOVA |
| Smallest unit | centova = 10^-18 NOVA |
| Max supply | 42,000,000 NOVA |
| Initial reward | 0.1 NOVA / block |
| Halving interval | 210,000,000 blocks |
| Target block time | 400 ms |
| Halving duration | ~2.66 years |
| Emission lifetime | ~152 years |
| Consensus | PoW + BlockDAG + Aurora finality |
| VM | NovaVM, deterministic WASM subset |
| Language | Pulsar |
| Account model | Smart accounts from genesis |
| Premine | None |
| Foundation allocation | None at protocol level |

Geometric supply:

```text
2 * 0.1 * 210,000,000 = 42,000,000 NOVA
```

---

# 3. Why NOVA Exists

Smart contracts are asset-holding programs.

A normal bug crashes an app.

A smart-contract bug can drain a vault, brick a protocol, corrupt governance, lock user funds, or destroy years of trust in one transaction.

Ethereum proved smart contracts matter.

It also proved the danger of exposing developers to low-level power tools:

- Reentrancy
- `delegatecall` storage collisions
- Unsafe proxy upgrades
- `tx.origin` phishing
- `selfdestruct` griefing
- Manual access-control mistakes
- Oracle manipulation
- Unsafe randomness
- Stuck funds
- Assembly gas games
- Infinite token approvals
- Blind signing of opaque calldata

NOVA does not pretend all bugs can disappear.

But many of the worst historical bug classes should not be style-guide issues.

They should be impossible states.

---

# 4. Core Architecture

NOVA is a Rust workspace of independent crates, each replaceable:

```text
nova/
├── nova-node/         # binary: full node
├── nova-consensus/    # GHOSTDAG + Aurora finality
├── nova-pow/          # hashing, mining, difficulty
├── nova-vm/           # NovaVM deterministic WASM
├── nova-state/        # state database and commitments
├── nova-mempool/      # tx pool, encrypted mempool, fee market
├── nova-p2p/          # networking
├── nova-rpc/          # JSON-RPC + gRPC
├── nova-types/        # core types, serialization
├── nova-runtime/      # on-chain upgradeable runtime modules
├── pulsar-compiler/   # Pulsar -> NovaVM bytecode
└── pulsar-stdlib/     # safe primitives and audited modules
```

Key architectural choices:

- **Account model**, not UTXO.
- **BlockDAG**, not a single longest chain.
- **Execution separated from consensus.**
- **Versioned state commitments.**
- **Runtime modules can upgrade through NRC-09.**
- **Contracts compile to deterministic NovaVM bytecode.**
- **Protocol standards are defined through NRCs.**

---

# 5. Consensus: PoW + BlockDAG

Pure Nakamoto consensus at 400 ms blocks breaks down.

At that speed, normal network delay creates too many orphan blocks. Honest miners waste work. Effective security degrades.

NOVA keeps proof-of-work, but replaces the single longest chain with a BlockDAG.

Traditional chain:

```text
A -> B -> C -> D
```

BlockDAG:

```text
A ---> B -----> E
 \     \       /
  \--> C ----/
       \-- D
```

Blocks reference multiple known tips.

A deterministic GHOSTDAG-style ordering algorithm linearizes the DAG into one canonical execution order.

The goal:

> Include honest parallel work instead of throwing it away.

## 5.1 NovaHash

NOVA uses a memory-hard proof-of-work function inspired by RandomX.

Goal:

- Reduce specialized hardware advantage.
- Keep commodity hardware competitive longer.
- Avoid immediate mining centralization.

Non-goal:

- Pretending ASIC resistance lasts forever.

No proof-of-work chain should promise permanent ASIC resistance.

NovaHash is designed to slow specialization, not make it mathematically impossible.

## 5.2 Difficulty Adjustment

NOVA uses per-block difficulty adjustment with a smoothed moving average.

This avoids long adjustment windows and lets the network react quickly to hashrate changes.

---

# 6. Aurora Finality

Proof-of-work gives probabilistic finality.

NOVA adds **Aurora**, a deterministic finality gadget over PoW.

Aurora turns probabilistic PoW history into finalized checkpoints.

Every epoch:

1. Recent miners register finality keys.
2. Bonded miners become eligible for the finality committee.
3. Committee members automatically vote on checkpoint blocks.
4. A checkpoint with 2/3 quorum becomes finalized.

PoW gives open block production.

Aurora gives deterministic settlement.

Important:

> NOVA is not pure PoW. It is PoW block production plus bonded miner finality.

## 6.1 Automatic Voting

Aurora voting is automatic.

A miner does not manually click approve.

A bonded miner runs finality software. When selected for the committee, the node automatically signs valid checkpoint messages according to protocol rules.

```text
Miner runs NOVA node
        ↓
Miner registers Aurora finality key
        ↓
Miner posts slashable bond
        ↓
If selected, node watches checkpoints
        ↓
Node signs valid checkpoint
        ↓
2/3 quorum = finalized
```

## 6.2 Slashing

If a finality participant signs conflicting checkpoints or violates finality rules, the participant can be slashed.

Missed votes may lose rewards.

Double-signing is slashable.

## 6.3 Reward Split

Each block reward is split:

```text
80% -> PoW miner
20% -> Aurora finality committee
```

## 6.4 Quantum-Durable Finality

Aurora may use BLS signatures for speed, but long-term checkpoint durability should use post-quantum committee certificates.

BLS is a speed optimization.

Post-quantum checkpoint certificates are the long-term finality root.

---

# 7. NovaVM

NovaVM is a deterministic subset of WebAssembly.

Why WASM?

- Mature toolchains
- Sandboxing
- Performance
- Debuggers and profilers
- Easier compiler targets than custom bytecode

Consensus execution rules:

- No floating point in consensus execution.
- No threads.
- No nondeterministic host calls.
- Memory growth is metered and capped.
- Imports are whitelisted.
- Crypto uses fixed-cost precompiles.
- Modules are validated before deployment.

Different node implementations must run the same bytecode and produce the same state root.

> Determinism is not a feature. It is survival.

---

# 8. Pulsar

Pulsar is NOVA's smart contract language.

It looks familiar to TypeScript developers, but it is not JavaScript on-chain.

It is a safety-first contract language compiled to NovaVM bytecode.

Pulsar removes entire Solidity-style footguns:

- No `delegatecall`
- No `selfdestruct`
- No `tx.origin`
- No inline assembly
- No silent integer overflow
- No accidental payable fallback
- No untyped string-only revert model
- No storage-slot math
- No standing infinite approvals as the native token model

The language is opinionated.

That is the point.

---

# 9. Pulsar Safety Model

## 9.1 Piggy Bank Example

```typescript
// piggy_bank.pulsar
// Deposit any time. Withdraw only after unlock. Once broken, retired forever.

@invariant(self.totalSaved == self.balance)
contract PiggyBank {
    storage owner: address;
    storage unlockTime: u64;
    storage totalSaved: u256;
    storage broken: bool;

    event Deposited(from: address, amount: u256, newTotal: u256);
    event Broken(by: address, amount: u256);

    error TooEarly(unlockTime: u64, now: u64);
    error AlreadyBroken;
    error NotOwner(caller: address, owner: address);
    error EmptyPiggy;

    init(unlockTime: u64) {
        self.owner = msg.sender;
        self.unlockTime = unlockTime;
    }

    receive {
        if self.broken { revert AlreadyBroken; }
        self.totalSaved += msg.value;
        emit Deposited(msg.sender, msg.value, self.totalSaved);
    }

    pub fn breakPiggy() {
        if msg.sender != self.owner {
            revert NotOwner(msg.sender, self.owner);
        }
        if self.broken { revert AlreadyBroken; }
        if block.timestamp < self.unlockTime {
            revert TooEarly(self.unlockTime, block.timestamp);
        }
        if self.totalSaved == 0 { revert EmptyPiggy; }

        let amount: u256 = self.totalSaved;

        self.totalSaved = 0;
        self.broken = true;

        external {
            self.owner.transfer(amount);
        }

        emit Broken(msg.sender, amount);
    }

    pub view fn saved() -> u256 {
        return self.totalSaved;
    }
}
```

What this demonstrates:

- Explicit fund receipt through `receive`.
- Checked arithmetic.
- Typed errors.
- Contract invariant.
- Effects before interactions.
- External calls isolated inside `external { }`.

## 9.2 Reentrancy Model

Ethereum teaches:

```text
remember checks-effects-interactions
```

Pulsar enforces:

```text
external calls only inside external { }
storage writes after external calls are compile errors by default
```

Unsafe pattern:

```typescript
external {
    attacker.transfer(amount);
}

self.balance = 0; // compile error by default
```

Safe pattern:

```typescript
self.balance = 0;

external {
    user.transfer(amount);
}
```

The safe path is the normal path.

## 9.3 No Inline Assembly

Pulsar has no inline assembly.

Developers who need low-level control may target NovaVM directly, but such contracts are explicitly marked unsafe and do not receive Pulsar's language-level safety guarantees.

```text
Pulsar verified: yes/no
Unsafe NovaVM module: yes/no
Compiler guarantees: available/not available
```

## 9.4 Vulnerabilities Still Possible

Pulsar cannot prevent all bugs.

Example: missing access control.

```typescript
contract VulnerableVault {
    storage owner: address;
    storage totalDeposits: u256;

    error NotOwner(caller: address, owner: address);

    init() {
        self.owner = msg.sender;
    }

    receive {
        self.totalDeposits += msg.value;
    }

    // Vulnerable: anyone can become owner.
    pub fn setOwner(newOwner: address) {
        self.owner = newOwner;
    }

    pub fn withdrawAll() {
        if msg.sender != self.owner {
            revert NotOwner(msg.sender, self.owner);
        }

        let amount: u256 = self.totalDeposits;
        self.totalDeposits = 0;

        external {
            self.owner.transfer(amount);
        }
    }
}
```

This is not a reentrancy bug or arithmetic bug.

It is a business-logic/access-control bug.

NOVA still needs tests, audits, formal specs, and careful design.

## 9.5 Safety Table

| Bug Class | Legacy Exposure | NOVA / Pulsar Model |
|---|---|---|
| Reentrancy | Convention / modifier | Structural rule |
| Integer overflow | Historically dangerous | Checked by default |
| `delegatecall` collision | Proxy footgun | No `delegatecall` |
| `selfdestruct` | Griefing vector | Not supported |
| `tx.origin` phishing | Possible | `tx.origin` does not exist |
| Forgotten payable | Stuck funds | Explicit `receive` block |
| Storage layout corruption | Proxy risk | Typed migrations |
| String reverts | Hard to parse | Typed errors |
| Resource leaks | Easy | Linear resource types |
| Unsafe randomness | Common exploit vector | Native beacon |
| Sandwich attacks | Public mempool | Encrypted mempool |
| Infinite approvals | Common ERC20 UX failure | Capability-based token permissions |
| Blind signing | Opaque calldata | Typed intent signing |

---

# 10. Gas and Fees

One gas number is too blunt.

NOVA meters four resources independently:

| Dimension | Measures | Why It Exists |
|---|---|---|
| Compute | VM instructions | CPU cost |
| State R/W | Reads and writes | Database I/O |
| State growth | New persistent bytes | Long-term storage burden |
| Bandwidth | Calldata and logs | Network propagation |

This makes costs visible.

Post-quantum signatures become honestly priced:

```text
bigger signature -> bandwidth gas
expensive verify -> compute gas
larger metadata  -> state-growth gas
```

## 10.1 Are NOVA Contracts More Gas Intensive?

Pulsar contracts may do more raw compute than equivalent hand-optimized Solidity contracts because safety checks are on by default.

That does not automatically mean users pay more.

NOVA's philosophy:

> Spend cheap compute to save expensive mistakes.

The safe, readable path should not be more expensive than dangerous low-level tricks.

---

# 11. Tokenomics

NOVA has a hard cap:

```text
42,000,000 NOVA
```

No premine.

No protocol-level foundation allocation.

Issuance:

| Era | Block Range | Reward/Block | Issued | Cumulative |
|---:|---|---:|---:|---:|
| 1 | 0 - 210M | 0.10000 | 21,000,000 | 21,000,000 |
| 2 | 210M - 420M | 0.05000 | 10,500,000 | 31,500,000 |
| 3 | 420M - 630M | 0.02500 | 5,250,000 | 36,750,000 |
| 4 | 630M - 840M | 0.01250 | 2,625,000 | 39,375,000 |
| 5 | 840M - 1.05B | 0.00625 | 1,312,500 | 40,687,500 |

Fees:

```text
base fee     -> burned
priority tip -> miner
```

If usage is high, NOVA can become net-deflationary before issuance ends.

## 11.1 Daily Issuance in Era 1

At 400 ms blocks:

```text
86,400 seconds / 0.4 = 216,000 blocks/day
216,000 * 0.1 NOVA = 21,600 NOVA/day
```

Split:

```text
17,280 NOVA/day -> miners
4,320 NOVA/day  -> Aurora committee
```

With 100 perfectly equal miners:

```text
172.8 NOVA/day mining rewards per miner
43.2 NOVA/day Aurora rewards per equal participant
216 NOVA/day total if participating in both
```

---

# 12. Accounts, Addresses, and Human Names

NOVA has no permanent EOA/contract split.

Every account is a smart account.

An account can support:

- Passkeys
- Hardware wallets
- Session keys
- Spending limits
- Gas sponsorship
- Multisig
- Social recovery
- Key rotation
- Multiple signature schemes

A mnemonic is not the account.

A mnemonic is one possible credential.

The account is the smart account address.

## 12.1 Address Format

Recommended address format:

```text
nova1z4m8r7qk0n6v2x9h3c5twaepldjsyfgb1u84rmq2k0v6n9c3s5a
```

Internal size:

```text
32 bytes
```

Canonical user encoding:

```text
Bech32m
```

Network prefixes:

```text
nova1...   mainnet
tnova1...  testnet
dnova1...  devnet
```

Derivation:

```text
address = BLAKE3-256(
    "NOVA_ACCOUNT_V1" ||
    account_code_hash ||
    salt ||
    initial_auth_root
)
```

## 12.2 Human-Readable Aliases

Every account should have a deterministic readable alias derived from its address.

Example:

```text
nova1z4m8...c3s5a
= funny-horse-idaho-king
```

This alias is not a global username.

It is a human-readable fingerprint.

## 12.3 Global Names

Users can register global `.nova` names:

```text
alex.nova
cryptolar.nova
opensense.nova
vault.cryptolar.nova
dao.opensense.nova
```

Best wallet display:

```text
alex.nova
funny-horse-idaho-king
nova1z4m8...c3s5a
```

If no global name exists:

```text
funny-horse-idaho-king
nova1z4m8...c3s5a
```

---

# 13. Transactions and Receipts

A NOVA transaction hash should use a native transaction prefix:

```text
novatx1q4m7p9x2k6v8r3c5t0wzjhfn93ldqae7smyu6p4r8k2c0v5xg9
```

Internal hash:

```text
tx_hash = BLAKE3-256("NOVA_TX_V1" || canonical_transaction_bytes)
```

Developer hex:

```text
0x8f42c1e0a99b7d4d6c2e53f83a5b88dd934fb1e2b92a7126cc6f2dd5a7c9310e
```

## 13.1 Example Receipt

```text
NOVA TRANSACTION RECEIPT

Hash:
novatx1q4m7p9x2k6v8r3c5t0wzjhfn93ldqae7smyu6p4r8k2c0v5xg9

Type:
Native Transfer

Status:
Finalized

From:
funny-horse-idaho-king
nova1z4m8...c3s5a

To:
silent-orbit-purple-wolf
nova1p7sr...v2q9x

Amount:
25 NOVA

Fee:
0.0031 NOVA

Preconfirmation:
102 ms

Finalized:
Aurora checkpoint 70153

Gas:
compute:      1,220
state_read:   2
state_write:  2
state_growth: 0
bandwidth:    3,912 bytes
```

---

# 14. Events, Logs, and Bloom Filters

NOVA events should use Bloom filters, but not as the only event indexing system.

Block header:

```typescript
struct BlockHeader {
    logsRoot: bytes32;
    logsBloom: bytes;
}
```

Event log:

```typescript
struct EventLog {
    emitter: address;
    eventName: bytes32;
    topics: bytes32[];
    data: bytes;
}
```

Design:

```text
Bloom filter = fast search
Merkle logsRoot = cryptographic proof
Typed Pulsar ABI = decoding
```

## 14.1 Indexed Events

Pulsar syntax:

```typescript
event Transfer(
    indexed from: address,
    indexed to: address,
    amount: u256
);
```

Bloom inserts:

- Emitter address
- Event name hash
- Indexed topics

Bloom filters can have false positives.

They cannot prove an event occurred.

The `logsRoot` provides proof.

---

# 15. Randomness

NOVA randomness should not use:

```typescript
block.timestamp
blockhash
msg.sender
```

Those are predictable or manipulable.

NOVA uses a native randomness beacon derived from Aurora finality.

Flow:

```text
1. User submits transaction
2. Transaction is ordered into the BlockDAG
3. Aurora finalizes checkpoint
4. Committee produces randomness beacon
5. Beacon is post-processed
6. Contract reads randomness through safe API
```

Core property:

> The randomness exists only after the user action is committed.

## 15.1 Pulsar API

```typescript
let roll: u256 = random.uniform(1, 6, salt);
```

Do not encourage:

```typescript
random.beacon() % 6
```

because modulo can introduce bias.

## 15.2 Hacked Nodes

A normal hacked node cannot manipulate chain randomness.

It can lie to its local user interface, but the canonical chain remains unchanged.

A miner may influence inclusion or censorship.

An Aurora committee coalition may attempt liveness or withholding attacks.

The intended design prevents one node from secretly choosing the random result.

---

# 16. Oracles and HTTP Attestations

Pulsar contracts should never make raw HTTP requests during execution.

That would break deterministic consensus.

Bad:

```typescript
let price = http.get("https://api.exchange.com/btc");
```

Different nodes may get different answers.

Consensus dies.

Correct model:

```text
contract requests data
oracle nodes fetch off-chain
quorum signs result
result is posted on-chain
contract reads deterministic attestation
```

Rule:

> NOVA contracts cannot call the internet. NOVA contracts can consume signed attestations about internet data.

## 16.1 Oracle Price Object

```typescript
struct OraclePrice {
    value: u256;
    decimals: u8;
    updatedAt: u64;
    confidence: u256;
    sources: u16;
}
```

Example:

```typescript
let btc = oracle.price("BTC/USD");

if block.timestamp - btc.updatedAt > 60 {
    revert StalePrice(btc.updatedAt);
}

if btc.confidence < 95 {
    revert LowConfidence(btc.confidence);
}
```

## 16.2 BTC Oracle + Randomness Example

```typescript
@invariant(self.reserve == self.balance)
contract BtcRandomLottery {
    storage owner: address;
    storage ticketPrice: u256;
    storage reserve: u256;
    storage btcBullThreshold: u256;
    storage maxOracleAge: u64;
    storage paused: bool;

    event TicketPlayed(
        player: address,
        btcPrice: u256,
        roll: u256,
        won: bool,
        payout: u256
    );

    error Paused;
    error WrongTicketPrice(sent: u256, required: u256);
    error StaleOracle(updatedAt: u64, now: u64);
    error LowOracleConfidence(confidence: u256);
    error InsufficientReserve(needed: u256, available: u256);

    pub fn play(salt: bytes32) {
        if self.paused { revert Paused; }

        if msg.value != self.ticketPrice {
            revert WrongTicketPrice(msg.value, self.ticketPrice);
        }

        self.reserve += msg.value;

        let btc = oracle.price("BTC/USD");

        if block.timestamp - btc.updatedAt > self.maxOracleAge {
            revert StaleOracle(btc.updatedAt, block.timestamp);
        }

        if btc.confidence < 95 {
            revert LowOracleConfidence(btc.confidence);
        }

        let roll: u256 = random.uniform(1, 100, salt);

        let payout: u256 = 0;
        let won: bool = false;

        if btc.value >= self.btcBullThreshold {
            if roll <= 10 {
                won = true;
                payout = self.ticketPrice * 10;
            }
        } else {
            if roll <= 5 {
                won = true;
                payout = self.ticketPrice * 10;
            }
        }

        if won {
            if self.reserve < payout {
                revert InsufficientReserve(payout, self.reserve);
            }

            self.reserve -= payout;

            external {
                msg.sender.transfer(payout);
            }
        }

        emit TicketPlayed(msg.sender, btc.value, roll, won, payout);
    }
}
```

---

# 17. Precompiles

Precompiles should exist only for expensive, security-critical, consensus-sensitive, or widely reused operations.

Genesis candidates:

## Hashing

- SHA-256
- Keccak-256
- BLAKE3
- RIPEMD-160

## Signatures

- secp256k1
- Ed25519
- P-256
- ML-DSA
- SLH-DSA

## ZK / Proof Systems

- BN254
- BLS12-381
- KZG verification
- Modular exponentiation

## Proof Helpers

- Merkle proofs
- Sparse Merkle proofs
- MPT proofs

Do not make application logic a precompile.

No DEX math.

No JSON parsing.

No token standards.

No sorting helpers.

No AI magic.

---

# 18. Quantum Resilience

NOVA is not quantum-resilient because it picked one magic algorithm.

NOVA is quantum-resilient because it is crypto-agile from genesis.

Signature scheme registry:

```text
0x01  secp256k1
0x02  Ed25519
0x03  P-256
0x10  ML-DSA-44
0x11  ML-DSA-65
0x12  ML-DSA-87
0x20  SLH-DSA-SHA2-128s
0x21  SLH-DSA-SHA2-128f
0x30  Hybrid-secp256k1-ML-DSA-65
0x31  Hybrid-P256-ML-DSA-65
```

Address design:

```text
address = hash(account bytecode + salt + initial auth root)
```

Public keys can remain hidden until first use.

Migration path:

```text
classical -> hybrid -> PQ recommended -> PQ required for high-value accounts
```

## 18.1 Harvest Now, Decrypt Later

If mempool encryption uses only classical crypto, attackers can record ciphertext today and decrypt it years later with quantum computers.

NOVA should use hybrid encrypted mempool traffic from genesis:

```text
X25519 + ML-KEM-768
```

If classical crypto breaks later, ML-KEM protects the recorded traffic.

If ML-KEM has a future flaw, X25519 still protects against classical attackers today.

The attacker must break both layers.

---

# 19. Anti-Blind-Signing Design

NOVA should not treat signatures as approval of opaque bytes.

Standard accounts sign typed intents.

Wallets display deterministic state and asset diffs.

Smart-account policies can reject unknown calls, unlimited approvals, unsafe upgrades, and high-risk actions without timelocks.

## 19.1 Typed Intent Signing

Example transfer intent:

```typescript
struct TransferIntent {
    from: address;
    to: address;
    asset: AssetId;
    amount: u256;
    feeLimit: FeeLimit;
    validUntil: u64;
    nonce: u64;
}
```

Wallet display:

```text
Send:
100 NOVA

To:
alex.nova
funny-horse-idaho-king
nova1z4m8...c3s5a

Expires:
2 minutes

Max fee:
0.01 NOVA
```

## 19.2 Native Simulation

Before signing, wallets should receive deterministic simulation results:

```typescript
struct SimulationResult {
    stateDiff: StateDiff[];
    assetDiff: AssetDiff[];
    newApprovals: ApprovalDiff[];
    externalCalls: ExternalCall[];
    upgradeEffects: UpgradeEffect[];
    riskFlags: RiskFlag[];
}
```

If the wallet cannot explain the transaction, the account should not sign it.

---

# 20. No Infinite Approvals

NOVA should kill ERC20-style infinite approvals by design.

Ethereum model:

```text
approve(router, infinite)
swap later
hope router/frontend/token never gets compromised
```

NOVA-native token permissions should be:

- Bounded by amount
- Bounded by purpose
- Bounded by recipient
- Bounded by expiry
- Revocable
- One-time by default

## 20.1 Spend Authorization

```typescript
struct SpendAuth {
    owner: address;
    spender: address;
    token: address;
    amount: u256;
    recipient: address;
    purpose: bytes32;
    validAfter: u64;
    validUntil: u64;
    nonce: bytes32;
}
```

The DEX uses it once:

```typescript
token.transferWithAuth(auth, signature);
```

After use:

```text
nonce consumed
authorization dead
cannot be reused
cannot be increased
cannot become infinite
```

## 20.2 TokenSpend Capability

```typescript
capability TokenSpend {
    token: address;
    spender: address;
    maxAmount: u256;
    remainingAmount: u256;
    expiresAt: u64;
    allowedRecipient: option<address>;
    allowedFunction: option<selector>;
    revocable: bool;
}
```

Rule:

> No infinite approvals. No permanent router power. Every spend permission has a cap, purpose, and expiry.

---

# 21. On-Chain Contract Verification

NOVA should not depend on Etherscan-style centralized source verification.

This is **NRC-8**.

The chain stores verification records and cryptographic commitments.

The full source can live in IPFS, Arweave, Git, NOVA source blobs, or cache nodes.

The chain stores the proof anchors.

## 21.1 Verification Record

```typescript
struct VerificationRecord {
    contract: address;
    sourceRoot: bytes32;
    abiRoot: bytes32;
    compilerId: bytes32;
    compilerVersion: string;
    stdlibHash: bytes32;
    buildConfigHash: bytes32;
    bytecodeHash: bytes32;
    status: VerificationStatus;
}
```

Verification states:

```typescript
enum VerificationStatus {
    Unverified,
    SourceCommitted,
    ReproducibleBuildVerified,
    FormallyVerified
}
```

## 21.2 Verification Flow

```text
1. Developer writes Pulsar source
2. pulsarc compiles deterministically
3. Developer deploys bytecode with source/build commitments
4. Independent verifiers reproduce build
5. Output bytecode hash must match deployed bytecode hash
6. Verification status is written to NRC-8 registry
```

Decompilation may exist, but is not verification.

```text
Decompiled view = explanation
Verified source = reproducible source-to-bytecode match
```

---

# 22. Compressed Source Availability

NOVA can optionally store compressed source on-chain or in a protocol source blob layer.

This is **NRC-8S**, an extension to NRC-8.

Recommended model:

```text
readable Pulsar source
→ canonical archive
→ zstd compression
→ source blob
```

Do not use ugly-name minification as the source of truth.

Readable source compressed with zstd is preferred.

## 22.1 Why Not Ugly Names?

If code is minified:

```typescript
contract C{storage v:u256;event I(a:address,v:u256);error U;...}
```

Then NOVA needs a symbol map to recover readability:

```json
{
  "contract": { "C": "Counter" },
  "storage": { "v": "value" },
  "events": { "I": "Incremented" },
  "errors": { "U": "CounterUnderflow" }
}
```

The symbol map may eat most of the savings.

Better:

> Store readable canonical source, then compress bytes.

## 22.2 Storage Mode

```typescript
enum SourceStorageMode {
    CommitmentOnly,
    OnChainCompressed,
    ExternalContentAddressed
}
```

---

# 23. On-Chain Runtime Upgrades

NOVA should support Polkadot-style forkless runtime upgrades, but filtered through Bitcoin-grade paranoia.

This is **NRC-9**.

The idea:

```text
Node host = stable execution host
Runtime code = on-chain upgradeable logic
```

## 23.1 Upgradeable vs Non-Upgradeable

Safe candidates:

- Fee rules
- Gas schedule
- Precompile pricing
- Oracle registry parameters
- Name service rules
- Account standard versions
- Pulsar stdlib system modules
- Runtime host function table
- Aurora committee parameters
- Storage rent parameters
- Transaction type registry
- Signature scheme registry

Dangerous candidates:

- PoW validity rules
- BlockDAG ordering rules
- Supply schedule
- Emission cap
- Historical state transition rules
- Address format
- Core hashing commitments

Those require stricter constitutional hard fork or social consensus paths.

## 23.2 Runtime Package

```typescript
struct RuntimeUpgradePackage {
    runtimeVersion: u32;
    targetActivationEpoch: u64;

    wasmHash: bytes32;
    sourceRoot: bytes32;
    compilerId: bytes32;
    compilerVersion: string;
    buildConfigHash: bytes32;

    migrationHash: bytes32;
    specHash: bytes32;
    auditRoot: bytes32;

    safetyProfile: RuntimeSafetyProfile;
}
```

## 23.3 Upgrade Flow

```text
1. Proposal submitted
2. Runtime package published
3. Source and build metadata attached through NRC-8
4. Deterministic build reproduced by independent verifiers
5. Simulation against shadow state
6. Public challenge period
7. Governance/finality approval
8. Activation scheduled at epoch N
9. Nodes automatically switch runtime at epoch N
10. Post-upgrade monitoring window
```

## 23.4 Upgrade Classes

| Class | Type | Example | Delay |
|---|---|---|---|
| A | Parameter upgrade | Fee coefficients, oracle thresholds | 24h-72h |
| B | Runtime module upgrade | Account logic, name service | 7 days |
| C | VM/precompile upgrade | New precompile, host function | 14-30 days |
| D | Consensus/security upgrade | PoW, DAG, supply, state root | 30+ days / social consensus |

Core sentence:

> Forkless upgrades, filtered through Bitcoin-grade paranoia.

---

# 24. Bridges

NOVA should prefer light-client bridges.

No multisig pretending to be decentralization.

No MPC committee as magical security theater.

A bridge should verify consensus proofs.

But the honest rule is:

> A bridge is only as secure as the weaker chain.

wNOVA on Ethereum inherits Ethereum's cryptographic assumptions.

A NOVA bridge to a weaker chain inherits that chain's weakness.

There is no marketing fix for this.

---

# 25. Bootstrapping and Security Modes

If NOVA has only 3 miners, it is not secure.

It is a devnet.

With 3 miners:

```text
2 of 3 miners can dominate PoW
2 of 3 finality members can finalize
BFT tolerance is effectively zero
randomness is weak
bridges are dangerous
```

NOVA must define security modes.

## 25.1 Devnet Mode

```text
< 25 independent miners
no real value
no bridge
randomness experimental
Aurora optional or centralized
```

## 25.2 Bootstrap Mode

```text
25-99 independent miners
public testing
basic contracts
small-value experiments only
```

## 25.3 Early Mainnet Mode

```text
100+ independent miners
basic transfers and contracts
limited randomness/oracle features
no large canonical bridge yet
```

## 25.4 Full Security Mode

Activation thresholds may include:

```text
128+ bonded Aurora participants
stable finality participation for 90 days
no entity above 20% of hashrate
multiple client implementations
stable propagation
audited bridges
```

Honest sentence:

> NOVA's full security claims only apply after the network reaches sufficient miner diversity, committee size, hashrate, client diversity, and stable finality participation. Early devnet and testnet phases are explicitly not Bitcoin-grade secure.

---

# 26. NRC Standards Index

NRC means **NOVA Request for Comment**.

These standards define NOVA's core interfaces, wallet UX, verification model, upgrade model, and safety guarantees.

## Clean NRC Index

```text
NRC-1   Smart Account Interface
NRC-2   Signature Scheme Registry
NRC-3   Wallet Derivation and Recovery
NRC-4   Guardian Recovery
NRC-5   Session Keys and Scoped Permissions
NRC-6   Native Naming Standard
NRC-7   Typed Intent Signing
NRC-8   On-Chain Contract Verification
NRC-8S  Compressed Source Availability
NRC-9   On-Chain Runtime Upgrades
NRC-10  Token Capability Standard
NRC-11  Event Logs and Bloom Filters
NRC-12  Oracle and Data Attestation Standard
NRC-13  Native Randomness Standard
NRC-14  Address and Human Alias Standard
NRC-15  Transaction Hash and Receipt Standard
```

Most important safety stack:

```text
NRC-1  Smart accounts
NRC-5  Scoped permissions
NRC-7  Typed intent signing
NRC-8  Native verification
NRC-10 No infinite approvals
```

Full NRC documents are available in the [`nrcs/`](./nrcs/) directory.

---

# 27. Roadmap

## Phase 1: Months 0-6

- `nova-types`
- `nova-state`
- `nova-pow`
- Basic single-chain PoW devnet

## Phase 2: Months 6-12

- NovaVM deterministic WASM execution
- Raw WAT testing
- Basic transaction execution

## Phase 3: Months 12-18

- Pulsar v0 compiler
- ERC20-equivalent contracts
- Basic account abstraction
- Devnet launch

## Phase 4: Months 18-24

- BlockDAG
- Aurora finality
- Encrypted mempool
- RPC
- Public testnet

## Phase 5: Months 24-30

- Pulsar v1 safety features
- Formal tooling
- NRC-1 to NRC-15 draft implementations
- Wallets and explorer

## Phase 6: Months 30-36

- Stress tests
- Audits
- Multiple implementations
- Bug bounties
- Mainnet readiness review

## Phase 7: 36+

- Mainnet launch: **First Light**

---

# 28. What NOVA Does Not Claim

NOVA does not claim:

- No contract can ever be exploited.
- Quantum risk is solved forever.
- Bridges become magically safe.
- Oracle manipulation disappears.
- Governance cannot fail.
- Users cannot lose keys.
- Compiler bugs cannot exist.
- Runtime upgrades are risk-free.
- Three miners are enough for Bitcoin-grade security.

Remaining risks:

- Business logic errors
- Bad specifications
- Governance capture
- Oracle failures
- Bridge bugs
- Wallet compromise
- Social engineering
- Compiler bugs
- VM bugs
- Protocol implementation bugs
- Novel cryptographic attacks
- Miner centralization
- Finality committee concentration
- Unsafe runtime upgrades

The strongest honest claim:

> NOVA makes many catastrophic historical smart-contract bug classes structurally unreachable or significantly harder to express.

---

# 29. Culture and Lore

NOVA vocabulary:

| Name | Meaning |
|---|---|
| NOVA | Chain |
| Pulsar | Smart contract language |
| NovaVM | Execution environment |
| Aurora | Finality gadget |
| NovaHash | Proof-of-work function |
| centova | Smallest unit (10^-18 NOVA) |
| Gnov | Gas denomination |
| First Light | Mainnet launch |
| Big Bang | Genesis block |
| superNOVA | Major hard fork |

Cultural pitch:

> Bitcoin gave us unstoppable money.  
> Ethereum gave us unstoppable programs.  
> NOVA makes those programs harder to break.

Shorter:

> Bitcoin-grade settlement. Ethereum-grade apps. Smart contracts with the footguns removed.

---

# 30. Closing

NOVA is an attempt to combine three ideas:

1. Bitcoin's permissionless proof-of-work security.
2. Ethereum's programmable application layer.
3. A safety-first language and VM where catastrophic bug classes are removed by construction.

The hard part is not making another fast chain.

The hard part is making the safety claims real.

That requires:

- Formal specifications
- Deterministic VM rules
- Compiler verification
- Multiple implementations
- Audited crypto
- Adversarial testnets
- Permanent bug bounties
- Conservative engineering culture
- Honest bootstrapping claims
- Native verification
- No blind signing
- No infinite approvals
- Runtime upgrades with public challenge windows

NOVA's north star:

> The future of smart contracts should not depend on every developer remembering every footgun.

The chain should remove the footguns.

That is NOVA.
