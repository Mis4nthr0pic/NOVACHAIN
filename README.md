# NOVA Documentation

**Bitcoin-grade settlement. Ethereum-grade programmability. Smart contracts with the footguns removed.**

**Version:** v0.6  
**Status:** Concept / architecture draft  
**Scope:** Full design document for NOVA, including base protocol architecture, Pulsar, NovaVM, accounts, safety standards, bridge/oracle/custody risk models, human-visible signing security, token safety, governance safety, enforcement layers, formal verification, monitoring, quantum enforcement, and alpha scope.

---

## Table of contents

1. [Signal](#1-signal)
2. [Quick spec](#2-quick-spec)
3. [Why NOVA exists](#3-why-nova-exists)
4. [Design philosophy](#4-design-philosophy)
5. [Enforcement layers](#5-enforcement-layers)
6. [Alpha scope](#6-alpha-scope)
7. [Core architecture](#7-core-architecture)
8. [Consensus: PoW + BlockDAG](#8-consensus-pow--blockdag)
9. [Aurora finality](#9-aurora-finality)
10. [NovaVM](#10-novavm)
11. [Pulsar](#11-pulsar)
12. [Pulsar safety model](#12-pulsar-safety-model)
13. [Gas and fees](#13-gas-and-fees)
14. [Tokenomics](#14-tokenomics)
15. [Accounts, addresses, and human names](#15-accounts-addresses-and-human-names)
16. [Transactions and receipts](#16-transactions-and-receipts)
17. [Events, logs, and bloom filters](#17-events-logs-and-bloom-filters)
18. [Randomness](#18-randomness)
19. [Oracles and HTTP attestations](#19-oracles-and-http-attestations)
20. [Precompiles](#20-precompiles)
21. [Quantum resilience](#21-quantum-resilience)
22. [Anti-blind-signing design](#22-anti-blind-signing-design)
23. [No infinite approvals](#23-no-infinite-approvals)
24. [On-chain contract verification](#24-on-chain-contract-verification)
25. [Compressed source availability](#25-compressed-source-availability)
26. [On-chain runtime upgrades](#26-on-chain-runtime-upgrades)
27. [Bridges](#27-bridges)
28. [Bootstrapping and security modes](#28-bootstrapping-and-security-modes)
29. [Human-visible security](#29-human-visible-security)
30. [Threat model from historical crypto failures](#30-threat-model-from-historical-crypto-failures)
31. [Account security levels](#31-account-security-levels)
32. [Custody account standard](#32-custody-account-standard)
33. [App contract upgrade safety](#33-app-contract-upgrade-safety)
34. [Oracle safety standard](#34-oracle-safety-standard)
35. [Bridge risk standard](#35-bridge-risk-standard)
36. [Verified app manifest](#36-verified-app-manifest)
37. [Contract and protocol risk labels](#37-contract-and-protocol-risk-labels)
38. [Contract safety metadata](#38-contract-safety-metadata)
39. [Safer token standard](#39-safer-token-standard)
40. [Governance safety standard](#40-governance-safety-standard)
41. [MEV and encrypted mempool](#41-mev-and-encrypted-mempool)
42. [NRC standards index](#42-nrc-standards-index)
43. [Roadmap](#43-roadmap)
44. [What NOVA does not claim](#44-what-nova-does-not-claim)
45. [Culture and lore](#45-culture-and-lore)
46. [Closing](#46-closing)

---

# 1. Signal

Bitcoin gave the world money no government can stop.

Ethereum gave the world programs no company can shut down.

NOVA combines both with a third requirement:

**Smart contracts should be safer by construction — and where full construction safety is impossible, they must be provably bounded, visible, and monitorable.**

Not safer because every developer remembered every checklist.

Not safer because every protocol hired the perfect auditor.

Not safer because users trusted another proxy, multisig, oracle, bridge, frontend, or middleware layer.

Safer because entire catastrophic bug classes are not expressible in the language.

Safer because dangerous permissions expire.

Safer because upgrades are visible.

Safer because signers agree on what they are signing.

Safer because wallets explain risk before asking for approval.

Safer because tokens cannot hide dangerous behavior.

Safer because governance proposals expose what they can actually do.

NOVA is a permissionless Layer 1 blockchain with proof-of-work mining, BlockDAG ordering, deterministic Aurora finality, a deterministic WASM-based VM (NovaVM), and a new safety-first smart contract language called Pulsar.

NOVA treats historical crypto failures as design requirements.

The thesis is simple:

**The chain should remove the footguns.**

---

# 2. Quick spec

| Parameter             | Value                                                   |
| --------------------- | ------------------------------------------------------- |
| Name                  | NOVA                                                    |
| Ticker                | NOVA                                                    |
| Smallest unit         | centova = 10^-18 NOVA                                   |
| Max supply            | 42,000,000 NOVA                                         |
| Initial reward        | 0.1 NOVA / block                                        |
| Halving interval      | 210,000,000 blocks                                      |
| Target block time     | 400 ms                                                  |
| Halving duration      | ~2.66 years                                             |
| Emission lifetime     | ~152 years                                              |
| Consensus             | PoW + BlockDAG + Aurora finality                        |
| VM                    | NovaVM, deterministic WASM subset                       |
| Language              | Pulsar                                                  |
| Account model         | Smart accounts from genesis                             |
| Native safety model   | Typed intents, capability permissions, account policies, formal verification |
| Premine               | None                                                    |
| Foundation allocation | None at protocol level                                  |
| Formal verification   | Pulsar Prover (mandatory for high-value)                |
| Privacy               | Optional shielded intents (hybrid ZK)                   |
| Monitoring            | On-chain anomaly detection hooks                        |

Geometric supply:

```txt
2 * 0.1 * 210,000,000 = 42,000,000 NOVA
```

---

# 3. Why NOVA exists

Smart contracts are asset-holding programs.

A normal bug crashes an app.

A smart contract bug can drain a vault, brick a protocol, corrupt governance, lock user funds, or destroy years of trust in one transaction.

Ethereum proved smart contracts matter.

It also proved the danger of exposing developers and users to low-level power tools:

* Reentrancy
* `delegatecall` storage collisions
* Unsafe proxy upgrades
* `tx.origin` phishing
* `selfdestruct` griefing
* Manual access-control mistakes
* Oracle manipulation
* Unsafe randomness
* Stuck funds
* Assembly gas games
* Infinite token approvals
* Blind signing of opaque calldata
* Compromised frontends
* Signer deception
* Bridge trust assumptions hidden behind marketing
* Governance attacks
* Admin-key compromise
* Tokens with hidden behavior
* MEV and ordering manipulation

NOVA does not pretend all bugs can disappear.

But many of the worst historical bug classes should not be style-guide issues.

They should be impossible states.

And where they cannot be made impossible, they should at least become slower, louder, more visible, and harder to trigger by accident.

---

# 4. Design philosophy

NOVA should not make every risky pattern impossible.

Some systems need upgradeability, pausable tokens, bridges, governance vetoes, custom custody flows, or unusual token behavior.

The rule is:

```txt
safe defaults should be easy
unsafe choices should be explicit
hidden risk should be rejected
```

NOVA does not try to remove human judgment.

It tries to remove hidden danger.

A protocol can choose to be upgradeable.

A token can choose to be pausable.

A bridge can choose a multisig trust model.

A DAO can choose an emergency veto.

But those choices should be visible to wallets, explorers, protocols, auditors, voters, signers, and users.

Danger should not hide behind generic labels like:

```txt
contract interaction
approve
execute
proposal #42
unknown token
```

NOVA's cultural bias is conservative:

```txt
make safe behavior the default
make dangerous behavior noisy
make irreversible actions slow
make signatures meaningful
make contracts explain themselves
```

**Formal where possible, visible & bounded everywhere else.**

---

# 5. Enforcement layers

NOVA separates safety into three layers.

Not every safety rule belongs in consensus.

Some rules must be enforced by the chain.

Some rules belong inside smart account policy.

Some rules belong in wallets, explorers, and applications.

## 5.1 Consensus-enforced safety

Consensus-enforced rules are rejected by the chain if violated.

Examples:

* Invalid signatures
* Invalid state transitions
* Invalid token trait behavior when runtime-checkable
* Invalid capability use
* Invalid randomness API use
* Invalid bytecode
* Invalid deterministic execution
* Invalid supply schedule
* Invalid finality vote

This is the strongest enforcement layer.

If consensus rejects it, it cannot happen on-chain.

## 5.2 Account-enforced safety

Account-enforced rules are enforced by smart account policies.

Examples:

* Spending limits
* New recipient cooldowns
* Multisig thresholds
* Hardware signer requirements
* Recovery delays
* Upgrade delays
* Capability warm-up periods
* Independent simulation attestation requirements

This lets different accounts choose different risk profiles.

A burner wallet should not behave like an exchange cold wallet.

## 5.3 Interface-enforced safety

Interface-enforced rules are warnings and displays shown by wallets, explorers, and apps.

Examples:

* Risk labels
* Bridge trust warnings
* Token trait warnings
* Governance proposal effect previews
* Frontend manifest mismatch warnings
* Contract safety metadata display

This layer cannot stop every attack, but it can prevent users and signers from flying blind.

## 5.4 The enforcement rule

A safety feature is stronger when it moves closer to consensus.

But not every risk belongs in consensus.

NOVA should be clear about what is:

```txt
chain-enforced
account-enforced
wallet-enforced
socially-enforced
not enforceable
```

This makes the safety model honest.

## 5.5 AI-Assisted & Runtime Monitoring Layer

A fourth layer supplements static rules with adaptive monitoring.

* AI-assisted formal proof generation + human audit loop.
* On-chain anomaly detection (unusual velocity, oracle spikes, capability abuse patterns) that can trigger automatic pauses or alerts.
* Continuous fuzzing/symbolic execution results published as on-chain metadata.
* This layer catches emergent behaviors that static rules cannot yet cover.

This is not a replacement for consensus, account, or interface enforcement.

It is an adaptive safety net for unknown unknowns.

---

# 6. Alpha scope

NOVA begins as a research and study project.

The first goal is not to launch a production network.

The first goal is to use historical crypto failures as design input and build small prototypes that test whether better defaults are possible.

The first alpha does not try to build the full chain.

It focuses on the safety model:

1. Typed intent object
2. Capability-based token permissions
3. Token traits registry
4. Risk label generator
5. Simple Pulsar examples
6. Multisig intent agreement prototype
7. Custody account policy simulator
8. Hack-class threat matrix
9. Governance proposal intent prototype
10. Verified app manifest draft
11. Pulsar Prover MVP
12. Monitoring simulator

Alpha success means:

```txt
people can understand the model
small prototypes prove the concepts
failure modes are mapped clearly
NRCs start becoming concrete
```

Alpha does not need:

```txt
mainnet
token launch
production security claims
bridge launch
real-value custody
```

The study path comes first.

The chain comes later.

---

# 7. Core architecture

NOVA is a Rust workspace of independent crates, each replaceable:

```txt
nova/
├── nova-node/         # binary: full node
├── nova-consensus/    # GHOSTDAG-style ordering + Aurora finality
├── nova-pow/          # hashing, mining, difficulty
├── nova-vm/           # NovaVM deterministic WASM
├── nova-state/        # state database and commitments
├── nova-mempool/      # tx pool, encrypted mempool, fee market
├── nova-p2p/          # networking
├── nova-rpc/          # JSON-RPC + gRPC
├── nova-types/        # core types, serialization
├── nova-runtime/      # on-chain upgradeable runtime modules
├── nova-accounts/     # smart accounts, policies, capabilities
├── nova-intents/      # typed intent model and simulation outputs
├── nova-risk/         # risk labels and safety metadata
├── nova-governance/   # proposal intents, timelocks, simulations
├── nova-tokens/       # token traits, token capabilities, behavior locks
├── nova-formal/       # Pulsar Prover, invariant checker
├── nova-monitoring/   # anomaly detection
├── nova-zk/           # shielded intents
├── pulsar-compiler/   # Pulsar -> NovaVM bytecode
└── pulsar-stdlib/     # safe primitives and audited modules
```

Key architectural choices:

* Account model, not UTXO.
* BlockDAG, not a single longest chain.
* Execution separated from consensus.
* Versioned state commitments.
* Runtime modules can upgrade through NRC-09.
* Contracts compile to deterministic NovaVM bytecode.
* Protocol standards are defined through NRCs.
* Wallet safety is part of the protocol design, not an afterthought.

## 7.1 Supply chain security

NOVA is a Rust workspace with dozens of crates and hundreds of external dependencies.

A compromise in an obscure dependency (serialization library, crypto primitive, encoding crate) can bypass every safety guarantee built into the protocol.

Defenses:

* **Pinned dependencies**: All crates use strict `Cargo.lock` pinning. No floating versions in production builds.
* **Dependency auditing**: Every dependency is audited before inclusion. Audit results are recorded in the repository.
* **Reproducible dependency builds**: Full dependency tree must build deterministically. Two independent builders must produce byte-identical binaries.
* **Vulnerability monitoring**: Automated continuous scanning of all dependencies for known vulnerabilities. Critical vulns trigger immediate patches and node update alerts.
* **Minimal dependency surface**: Every external dependency must justify its inclusion. Prefer Rust standard library and in-house implementations for security-critical paths.
* **Dependency sandboxing**: Dependencies do not get access to filesystem, network, or environment variables at compile time unless explicitly granted.

---

Pure Nakamoto consensus at 400 ms blocks breaks down.

At that speed, normal network delay creates too many orphan blocks. Honest miners waste work. Effective security degrades.

NOVA keeps proof-of-work, but replaces the single longest chain with a BlockDAG.

Traditional chain:

```txt
A -> B -> C -> D
```

BlockDAG:

```txt
A ---> B -----> E
 \     \       /
  \--> C ----/
       \-- D
```

Blocks reference multiple known tips.

A deterministic GHOSTDAG-style ordering algorithm linearizes the DAG into one canonical execution order.

The goal:

**Include honest parallel work instead of throwing it away.**

## 8.1 NovaHash

NOVA uses a memory-hard proof-of-work function inspired by RandomX.

Goals:

* Reduce specialized hardware advantage.
* Keep commodity hardware competitive longer.
* Avoid immediate mining centralization.

Non-goal:

* Pretending ASIC resistance lasts forever.

No proof-of-work chain should promise permanent ASIC resistance.

NovaHash is designed to slow specialization, not make it mathematically impossible.

## 8.2 Difficulty adjustment

NOVA uses per-block difficulty adjustment with a smoothed moving average.

This avoids long adjustment windows and lets the network react quickly to hashrate changes.

## 8.3 Enhanced consensus monitoring

* Orphan rate monitoring + automatic difficulty compensation.
* Optional ghost-tip weighting during bootstrap mode to reduce wasted work.
* Aurora committee reputation scoring based on participation history.

## 8.4 Eclipse attack and network partitioning resistance

At 400 ms block times, NOVA requires fast, reliable network propagation. An attacker with a botnet could attempt to eclipse Aurora committee members or dominant miners, feeding them a distorted view of the DAG.

Defenses:

* **Diversity requirement**: Nodes must maintain connections to a minimum number of diverse peers across independent network segments. Sybil-resistant peer selection based on proof-of-work identity or long-lived reputation.
* **Partition detection**: If a node detects that its view of the DAG diverges significantly from its peers' views, it enters a degraded mode that refuses to finalize transactions until partition resolution.
* **Checkpoint cross-validation**: Aurora finality checkpoints are independently verified by non-committee nodes. A forged checkpoint cannot survive cross-validation.
* **Network partition recovery**: If the network partitions, each partition continues operating but refuses to finalize. Mergers are handled conservatively with the heaviest-weight DAG winning, followed by Aurora re-finalization.

---

# 9. Aurora finality

Proof-of-work gives probabilistic finality.

NOVA adds Aurora, a deterministic finality gadget over PoW.

Aurora turns probabilistic PoW history into finalized checkpoints.

Every epoch:

* Recent miners register finality keys.
* Bonded miners become eligible for the finality committee.
* Committee members automatically vote on checkpoint blocks.
* A checkpoint with 2/3 quorum becomes finalized.

PoW gives open block production.

Aurora gives deterministic settlement.

Important:

**NOVA is not pure PoW. It is PoW block production plus bonded miner finality.**

## 9.1 Automatic voting

Aurora voting is automatic.

A bonded miner runs finality software. When selected for the committee, the node automatically signs valid checkpoint messages according to protocol rules.

```txt
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

## 9.2 Slashing

If a finality participant signs conflicting checkpoints or violates finality rules, the participant can be slashed.

Missed votes may lose rewards.

Double-signing is slashable.

Partial slashing applies to liveness failures: missed votes result in proportional reward reduction rather than full slashing.

## 9.3 Reward split

Each block reward is split:

```txt
80% -> PoW miner
20% -> Aurora finality committee
```

## 9.4 Quantum-durable finality

Aurora may use BLS signatures for speed, but long-term checkpoint durability should use post-quantum committee certificates.

BLS is a speed optimization.

Post-quantum checkpoint certificates are the long-term finality root.

## 9.5 Post-quantum fallback enforcement

Post-quantum fallback certificates become mandatory after 2030.

Classical-only finality signatures are deprecated on a defined timeline.

L3+ accounts must rotate to hybrid or post-quantum signature schemes within migration windows defined by NRC-42.

## 9.6 Cryptographic implementation security

Aurora's signature aggregation and checkpoint scheme relies on correct cryptographic implementations. A bug in the math can allow forged checkpoints.

Defenses:

* **Conservative crypto**: NOVA uses well-established, heavily reviewed cryptographic primitives. Novel or experimental schemes are not used for consensus-critical paths.
* **Multi-implementation verification**: All consensus-critical cryptographic operations (signature verification, checkpoint aggregation, hash computations) have at least two independent implementations. Both must agree on every result. A disagreement halts finality.
* **Constant-time implementations**: All cryptographic operations are implemented in constant time to prevent timing side-channels.
* **Formal verification of critical paths**: The signature aggregation and verification logic is a target for formal verification. The math must be proven correct, not just tested.
* **Crypto audit track**: All cryptographic implementations undergo independent security audit by specialized cryptography auditors before mainnet inclusion.

---

# 10. NovaVM

NovaVM is a deterministic subset of WebAssembly.

Why WASM?

* Mature toolchains
* Sandboxing
* Performance
* Debuggers and profilers
* Easier compiler targets than custom bytecode

Consensus execution rules:

* No floating point in consensus execution.
* No threads.
* No nondeterministic host calls.
* Memory growth is metered and capped.
* Imports are whitelisted.
* Crypto uses fixed-cost precompiles.
* Modules are validated before deployment.

NovaVM supports formal symbolic execution hooks for the Pulsar Prover pipeline, allowing verification conditions to be checked against execution traces.

Different node implementations must run the same bytecode and produce the same state root.

## 10.1 VM safety and host call sandboxing

The gap between Pulsar's language guarantees and actual NovaVM execution is a critical trust boundary.

NOVA treats this gap as a first-class security surface.

**Compiler soundness**: The Pulsar compiler must preserve the language's safety guarantees when lowering to WASM bytecode. If the compiler inserts incorrect barrier instructions for `external { }` blocks, or fails to enforce checked arithmetic at the WASM level, Pulsar's safety guarantees become illusions.

Defenses:

* **Reference interpreter**: A verified reference interpreter runs alongside the compiler output. If the reference interpreter and the compiled WASM disagree on any execution, the contract is rejected.
* **Bytecode verification**: Before deployment, NovaVM validates that compiled bytecode respects Pulsar's safety rules at the WASM level — not just at the source level. This includes verifying that state writes cannot occur after external calls, and that arithmetic operations include overflow checks.
* **Compiler testing pipeline**: Every compiler release is tested against a corpus of adversarial Pulsar programs designed to find soundness gaps. Differential fuzzing compares compiler output against the reference interpreter.

**Host call sandboxing**: Precompiles and host functions execute outside the WASM sandbox with elevated privileges. A single memory safety bug in a precompile could enable VM escape.

Defenses:

* **Strict sandbox boundary**: Host calls communicate with WASM only through copy-in/copy-out buffers. No shared memory. No raw pointer passing.
* **Memory-safe Rust**: All precompiles are implemented in safe Rust. Unsafe code blocks in precompiles are prohibited without exception and a separate audit track.
* **Per-precompile audit**: Every precompile undergoes independent security audit before mainnet inclusion. Audit results are published on-chain.
* **Precompile fuzzing**: Continuous fuzzing of all precompile implementations with AddressSanitizer and MemorySanitizer enabled in CI.
* **Input validation**: Host functions validate all inputs before processing. No precompile trusts WASM-generated values without bounds checking.

## 10.2 WASM runtime hardening

A zero-day in the WASM runtime itself could allow a carefully crafted contract to escape the sandbox and access node memory directly.

Defenses:

* **No JIT compilation**: NovaVM does not use Just-In-Time compilation. All WASM is executed through a verified interpreter or ahead-of-time compiled with bounds checks preserved. JIT introduces an entire class of speculative execution and code generation attacks that NOVA refuses to accept.
* **Interpreted-only fallback**: If a node detects unexpected behavior in the AOT compiler path, it falls back to pure interpretation. Performance loss is acceptable; consensus failure is not.
* **Sandbox escape response**: If a sandbox escape is ever discovered, the protocol has a pre-planned response: halt block production, patch the runtime through NRC-9 emergency procedure, and re-validate all deployed contracts against the patched runtime.
* **Continuous WASM runtime fuzzing**: The WASM interpreter is continuously fuzzed with AddressSanitizer and MemorySanitizer. Any memory violation, no matter how small, is treated as a critical security bug.
* **Multi-runtime verification**: At least two independent WASM runtime implementations must agree on execution results. A disagreement halts the node and alerts the network.

Determinism is not a feature.

It is survival.

---

# 11. Pulsar

Pulsar is NOVA's smart contract language.

It looks familiar to TypeScript developers, but it is not JavaScript on-chain.

It is a safety-first contract language compiled to NovaVM bytecode.

Pulsar removes entire Solidity-style footguns:

* No `delegatecall`
* No `selfdestruct`
* No `tx.origin`
* No inline assembly
* No silent integer overflow
* No accidental payable fallback
* No untyped string-only revert model
* No storage-slot math
* No standing infinite approvals as the native token model

The language is opinionated.

That is the point.

---

# 12. Pulsar safety model

## 12.1 Piggy bank example

```rust
// piggy_bank.pulsar
// Deposit any time. Withdraw only after unlock. Once broken, retired forever.

@invariant(self.totalSaved <= self.balance)
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

* Explicit fund receipt through `receive`.
* Checked arithmetic.
* Typed errors.
* Contract invariant.
* Effects before interactions.
* External calls isolated inside `external { }`.
* Safe invariant: uses `<=` not `==` with `self.balance` (see section 12.11).

## 12.11 Invariant anti-pattern: strict balance equality

A critical anti-pattern in smart contract design is using strict equality with `self.balance`:

```rust
// DANGEROUS — bricks the contract permanently
@invariant(self.totalSaved == self.balance)
```

Attack vector:

* An attacker pre-funds the contract address before deployment. The contract is born with `self.balance > 0` but `self.totalSaved == 0`. The invariant fails. Every function reverts. All funds are locked forever.
* A miner sends block rewards to the contract address. `self.balance` increases without any `receive` call. The invariant breaks.
* Any protocol-level forced transfer mechanism creates the same risk.

The cost of the attack is 1 centova. The damage is permanent fund lock.

Correct pattern:

```rust
// SAFE — allows unexpected balance increases without bricking
@invariant(self.totalSaved <= self.balance)
```

This guarantees the contract never tracks more than it holds, but does not break if the balance is higher than expected.

Pulsar's compiler should warn when an invariant uses `==` with `self.balance` or any field that can be influenced by external balance changes.

## 12.2 Reentrancy model

Ethereum teaches:

```txt
remember checks-effects-interactions
```

Pulsar enforces:

```txt
external calls only inside external { }
storage writes after external calls are compile errors by default
```

Unsafe pattern:

```rust
external {
    attacker.transfer(amount);
}

self.balance = 0; // compile error by default
```

Safe pattern:

```rust
self.balance = 0;

external {
    user.transfer(amount);
}
```

The safe path is the normal path.

## 12.3 No inline assembly

Pulsar has no inline assembly.

Developers who need low-level control may target NovaVM directly, but such contracts are explicitly marked unsafe and do not receive Pulsar's language-level safety guarantees.

```txt
Pulsar verified: yes/no
Unsafe NovaVM module: yes/no
Compiler guarantees: available/not available
```

## 12.4 Vulnerabilities still possible

Pulsar cannot prevent all bugs.

Example: missing access control.

```rust
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

NOVA still needs tests, audits, formal specs, bug bounties, and careful design.

## 12.5 Safety table

| Bug class                 | Legacy exposure         | NOVA / Pulsar model                |
| ------------------------- | ----------------------- | ---------------------------------- |
| Reentrancy                | Convention / modifier   | Structural rule                    |
| Integer overflow          | Historically dangerous  | Checked by default                 |
| `delegatecall` collision  | Proxy footgun           | No `delegatecall`                  |
| `selfdestruct`            | Griefing vector         | Not supported                      |
| `tx.origin` phishing      | Possible                | `tx.origin` does not exist         |
| Forgotten payable         | Stuck funds             | Explicit `receive` block           |
| Storage layout corruption | Proxy risk              | Typed migrations                   |
| String reverts            | Hard to parse           | Typed errors                       |
| Resource leaks            | Easy                    | Linear resource types              |
| Unsafe randomness         | Common exploit vector   | Native beacon                      |
| Sandwich attacks          | Public mempool          | Encrypted mempool design           |
| Infinite approvals        | Common ERC20 UX failure | Capability-based token permissions |
| Blind signing             | Opaque calldata         | Typed intent signing               |
| Token misbehavior         | Hidden trait surprises  | Token traits standard              |

## 12.6 Formal Verification Pipeline (Pulsar Prover)

Pulsar ships with a built-in specification language for formal verification.

Specifications are attached to functions and contracts:

```rust
@invariant(self.totalSaved <= self.balance)
@spec "withdraw only after unlock and owner approval"
pub fn breakPiggy() { ... }
```

The `pulsar-prover` toolchain (Lean4 + AI backend) generates and checks verification conditions against the contract's NovaVM bytecode.

Verification labels:

```txt
Standard           — compiled, basic safety checks pass
Formally Verified  — invariants and specs proven by pulsar-prover
High-Assurance     — formally verified + independent audit + reproducible build
```

High-assurance contracts require `proven: true` in their NRC-8 verification record (NRC-40).

Formal verification is not mandatory for all contracts.

It is mandatory for contracts that hold large values, manage governance, control bridges, or custody user funds.

The verification pipeline is part of the alpha scope (Phase 0) and integrates with the `nova-formal` crate.

## 12.7 Prover soundness and integrity

The Pulsar Prover is itself a security-critical component. A compromised or buggy prover that issues false proofs is worse than no prover — it gives false confidence.

**Soundness bugs**: If the prover can be tricked into issuing a valid proof for a contract that violates its specifications, the "Formally Verified" label becomes a weapon against users.

Defenses:

* **Proof checking, not just proof generation**: The chain does not trust proof generation. It runs proof checking — a simpler, more auditable process that verifies a proof is valid. Proof checking is much easier to get right than proof generation.
* **Independent prover implementations**: At least two independent implementations of the prover must agree on the verification result. A proof is only accepted if both provers independently confirm it.
* **AI output is advisory**: AI-assisted proof generation produces candidate proofs. Every AI-generated proof is verified by a deterministic proof checker. The AI cannot skip the checking step.
* **Prover version pinning**: Proofs declare which prover version generated them. If a prover version is later found to have a soundness bug, all proofs from that version are automatically downgraded from "Formally Verified" to "Standard" in the NRC-8 registry.
* **Adversarial prover testing**: The prover is continuously tested against a corpus of intentionally incorrect contracts that should fail verification. If any incorrect contract passes, the prover is halted and all affected proofs are flagged.

## 12.9 Historical compiler bug precedents

The idea that a compiler can silently break language-level safety guarantees is not theoretical. It has happened repeatedly.

| Component NOVA | Risk | Historical precedent | NOVA defense |
|---|---|---|---|
| Pulsar `external { }` enforcement | Compiler fails to insert reentrancy barrier at WASM level | **Vyper `@nonreentrant` bug** (2023): compiler failed to generate correct locks, Curve Finance drained for millions | Reference interpreter + bytecode verification (section 10.1) |
| Pulsar optimizer | Optimizer removes safety checks to save gas | **Solidity Yul optimizer bugs**: optimizer incorrectly removed storage writes and memory checks | Optimizer must preserve all safety checks — optimizer passes are verified, not trusted (NRC-45) |
| Pulsar Prover | Under-constrained formal model misses a condition | **Circom under-constrained circuits**: compiler omitted constraints, allowing invalid proofs | Independent prover implementations + proof checking (section 12.7) |
| NovaVM runtime | WASM sandbox escape via crafted bytecode | **Multiple WASM engine CVEs**: V8, SpiderMonkey, Wasmer all had sandbox escapes | No JIT, interpreted fallback, continuous fuzzing (section 10.2) |

The pattern is consistent: **language guarantees that survive to the source level but not to the execution level are illusions.**

NOVA's response: every safety guarantee must be verified at the execution level, not just the source level.

## 12.10 Optimizer safety rules

The Pulsar optimizer must never remove or weaken a safety check for performance.

Rules:

* **Checked arithmetic is non-negotiable**: The optimizer cannot replace checked add/sub/mul with unchecked equivalents, even if it can prove the values fit. Proof is not runtime safety.
* **`external { }` barriers are structural**: The optimizer cannot reorder instructions across an external call boundary, merge state writes with external calls, or eliminate the barrier between them.
* **Invariant checks are preserved**: `@invariant` checks cannot be optimized away, even if the optimizer believes they always pass.
* **Optimization whitelist**: Only optimizations that have been formally proven sound against the Pulsar semantics are enabled. "Looks correct" is not sufficient.
* **Differential testing**: Every optimizer pass is tested by comparing optimized and unoptimized output against the reference interpreter. Any disagreement disables that optimization pass.

---

# 13. Gas and fees

One gas number is too blunt.

NOVA meters four resources independently:

| Dimension    | Measures             | Why it exists            |
| ------------ | -------------------- | ------------------------ |
| Compute      | VM instructions      | CPU cost                 |
| State R/W    | Reads and writes     | Database I/O             |
| State growth | New persistent bytes | Long-term storage burden |
| Bandwidth    | Calldata and logs    | Network propagation      |
| Proof verify | Formal verification  | Proof checking cost      |

This makes costs visible.

Post-quantum signatures become honestly priced:

```txt
bigger signature -> bandwidth gas
expensive verify -> compute gas
larger metadata  -> state-growth gas
```

## 13.1 Are NOVA contracts more gas intensive?

Pulsar contracts may do more raw compute than equivalent hand-optimized Solidity contracts because safety checks are on by default.

That does not automatically mean users pay more.

NOVA's philosophy:

**Spend cheap compute to save expensive mistakes.**

The safe, readable path should not be more expensive than dangerous low-level tricks.

---

# 14. Tokenomics

NOVA has a hard cap:

```txt
42,000,000 NOVA
```

No premine.

No protocol-level foundation allocation.

Issuance:

| Era | Block range  | Reward/block |     Issued | Cumulative  |
| --- | -----------: | -----------: | ---------: | ----------: |
| 1   |     0 - 210M |      0.10000 | 21,000,000 |  21,000,000 |
| 2   |  210M - 420M |      0.05000 | 10,500,000 |  31,500,000 |
| 3   |  420M - 630M |      0.02500 |  5,250,000 |  36,750,000 |
| 4   |  630M - 840M |      0.01250 |  2,625,000 |  39,375,000 |
| 5   | 840M - 1.05B |      0.00625 |  1,312,500 |  40,687,500 |

Fees:

```txt
base fee     -> burned
priority tip -> miner
```

If usage is high, NOVA can become net-deflationary before issuance ends.

## 14.1 Daily issuance in era 1

At 400 ms blocks:

```txt
86,400 seconds / 0.4 = 216,000 blocks/day
216,000 * 0.1 NOVA = 21,600 NOVA/day
```

Split:

```txt
17,280 NOVA/day -> miners
4,320 NOVA/day  -> Aurora committee
```

With 100 perfectly equal miners:

```txt
172.8 NOVA/day mining rewards per miner
43.2 NOVA/day Aurora rewards per equal participant
216 NOVA/day total if participating in both
```

---

# 15. Accounts, addresses, and human names

NOVA has no permanent EOA/contract split.

Every account is a smart account.

An account can support:

* Passkeys
* Hardware wallets
* Session keys
* Spending limits
* Gas sponsorship
* Multisig
* Social recovery
* Key rotation
* Multiple signature schemes
* Policy-based signing
* Capability permissions
* Signer risk profiles

A mnemonic is not the account.

A mnemonic is one possible credential.

The account is the smart account address.

## 15.1 Address format

Recommended address format:

```txt
nova1z4m8r7qk0n6v2x9h3c5twaepldjsyfgb1u84rmq2k0v6n9c3s5a
```

Internal size:

```txt
32 bytes
```

Canonical user encoding:

```txt
Bech32m
```

Network prefixes:

```txt
nova1...   mainnet
tnova1...  testnet
dnova1...  devnet
```

Derivation:

```txt
address = BLAKE3-256(
    "NOVA_ACCOUNT_V1" ||
    account_code_hash ||
    salt ||
    initial_auth_root
)
```

## 15.2 Human-readable aliases

Every account should have a deterministic readable alias derived from its address.

Example:

```txt
nova1z4m8...c3s5a
= funny-horse-idaho-king
```

This alias is not a global username.

It is a human-readable fingerprint.

## 15.3 Global names

Users can register global `.nova` names:

```txt
alex.nova
cryptolar.nova
opensense.nova
vault.cryptolar.nova
dao.opensense.nova
```

Best wallet display:

```txt
alex.nova
funny-horse-idaho-king
nova1z4m8...c3s5a
```

If no global name exists:

```txt
funny-horse-idaho-king
nova1z4m8...c3s5a
```

---

# 16. Transactions and receipts

A NOVA transaction hash should use a native transaction prefix:

```txt
novatx1q4m7p9x2k6v8r3c5t0wzjhfn93ldqae7smyu6p4r8k2c0v5xg9
```

Internal hash:

```txt
tx_hash = BLAKE3-256("NOVA_TX_V1" || canonical_transaction_bytes)
```

Developer hex:

```txt
0x8f42c1e0a99b7d4d6c2e53f83a5b88dd934fb1e2b92a7126cc6f2dd5a7c9310e
```

## 16.1 Example receipt

```txt
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

# 17. Events, logs, and bloom filters

NOVA events should use bloom filters, but not as the only event indexing system.

Block header:

```rust
struct BlockHeader {
    logsRoot: bytes32;
    logsBloom: bytes;
}
```

Event log:

```rust
struct EventLog {
    emitter: address;
    eventName: bytes32;
    topics: bytes32[];
    data: bytes;
}
```

Design:

```txt
Bloom filter = fast search
Merkle logsRoot = cryptographic proof
Typed Pulsar ABI = decoding
```

## 17.1 Indexed events

Pulsar syntax:

```rust
event Transfer(
    indexed from: address,
    indexed to: address,
    amount: u256
);
```

Bloom inserts:

* Emitter address
* Event name hash
* Indexed topics

Bloom filters can have false positives.

They cannot prove an event occurred.

The `logsRoot` provides proof.

---

# 18. Randomness

NOVA randomness should not use:

```txt
block.timestamp
blockhash
msg.sender
```

Those are predictable or manipulable.

NOVA uses a native randomness beacon derived from Aurora finality.

Flow:

```txt
1. User submits transaction
2. Transaction is ordered into the BlockDAG
3. Aurora finalizes checkpoint
4. Committee produces randomness beacon
5. Beacon is post-processed
6. Contract reads randomness through safe API
```

Core property:

**The randomness exists only after the user action is committed.**

## 18.1 Pulsar API

```rust
let roll: u256 = random.uniform(1, 6, salt);
```

Do not encourage:

```rust
random.beacon() % 6
```

Modulo can introduce bias.

## 18.2 Hacked nodes

A normal hacked node cannot manipulate chain randomness.

It can lie to its local user interface, but the canonical chain remains unchanged.

A miner may influence inclusion or censorship.

An Aurora committee coalition may attempt liveness or withholding attacks.

The intended design prevents one node from secretly choosing the random result.

## 18.3 DAG grinding resistance

A sophisticated attacker with significant hashrate may attempt to "grind" the BlockDAG structure — selectively including or excluding blocks to influence which transactions are ordered before the Aurora checkpoint, thereby biasing the randomness beacon.

Defenses:

* **VDF post-processing**: The raw beacon output passes through a Verifiable Delay Function before becoming usable randomness. This prevents any party from evaluating the beacon output faster than the VDF delay, even if they can influence the DAG structure.
* **Multi-source entropy mixing**: The beacon combines entropy from multiple independent sources (Aurora committee signatures, block hash commitments, VDF output) using a collision-resistant combiner. Controlling one source does not control the output.
* **Min-seed-distance rule**: The protocol enforces a minimum distance between when a transaction is committed and when its randomness is revealed. Grinding that changes DAG ordering within this window does not affect already-committed transactions.
* **Grinding cost analysis**: The economic cost of grinding should always exceed the expected value of the bias. This is monitored continuously and VDF difficulty is adjusted if grinding becomes economically viable.

---

# 19. Oracles and HTTP attestations

Pulsar contracts should never make raw HTTP requests during execution.

That would break deterministic consensus.

Bad:

```rust
let price = http.get("https://api.exchange.com/btc");
```

Different nodes may get different answers.

Consensus dies.

Correct model:

```txt
contract requests data
oracle nodes fetch off-chain
quorum signs result
result is posted on-chain
contract reads deterministic attestation
```

Rule:

**NOVA contracts cannot call the internet. NOVA contracts can consume signed attestations about internet data.**

## 19.1 Oracle price object

```rust
struct OraclePrice {
    value: u256;
    decimals: u8;
    updatedAt: u64;
    confidence: u256;
    sources: u16;
    twapWindow: u64;
    liquidityDepth: u256;
    deviationBps: u32;
}
```

Example:

```rust
let btc = oracle.price("BTC/USD");

if block.timestamp - btc.updatedAt > 60 {
    revert StalePrice(btc.updatedAt);
}

if btc.confidence < 95 {
    revert LowConfidence(btc.confidence);
}

if btc.deviationBps > 500 {
    revert PriceMovedTooFar(btc.deviationBps);
}
```

## 19.2 BTC oracle + randomness example

```rust
@invariant(self.reserve <= self.balance)
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

# 20. Precompiles

Precompiles should exist only for expensive, security-critical, consensus-sensitive, or widely reused operations.

Genesis candidates:

Hashing:

```txt
SHA-256
Keccak-256
BLAKE3
RIPEMD-160
```

Signatures:

```txt
secp256k1
Ed25519
P-256
ML-DSA
SLH-DSA
```

ZK / proof systems:

```txt
BN254
BLS12-381
KZG verification
Modular exponentiation
```

Proof helpers:

```txt
Merkle proofs
Sparse Merkle proofs
MPT proofs
```

Do not make application logic a precompile.

No DEX math.

No JSON parsing.

No token standards.

No sorting helpers.

No AI magic.

---

# 21. Quantum resilience

NOVA is not quantum-resilient because it picked one magic algorithm.

NOVA is quantum-resilient because it is crypto-agile from genesis.

Signature scheme registry:

```txt
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

```txt
address = hash(account bytecode + salt + initial auth root)
```

Public keys can remain hidden until first use.

Migration path:

```txt
classical -> hybrid -> PQ recommended -> PQ required for high-value accounts
```

## 21.1 Harvest now, decrypt later

If mempool encryption uses only classical crypto, attackers can record ciphertext today and decrypt it years later with quantum computers.

NOVA should use hybrid encrypted mempool traffic from genesis:

```txt
X25519 + ML-KEM-768
```

If classical crypto breaks later, ML-KEM protects the recorded traffic.

If ML-KEM has a future flaw, X25519 still protects against classical attackers today.

The attacker must break both layers.

## 21.2 Post-Quantum Migration Enforcement

NRC-2 now includes mandatory deprecation timelines for classical-only signature schemes.

Migration requirements:

* L3+ accounts must rotate to hybrid or post-quantum signature schemes within defined windows.
* On-chain quantum readiness score tracks each account's migration status.
* Accounts that do not migrate within the window receive visible warnings in wallets and explorers.
* After the migration deadline, classical-only signatures are downgraded in security level but remain valid for existing funds.

The quantum readiness score is part of NRC-42 and is displayed alongside account security levels.

---

# 22. Anti-blind-signing design

NOVA should not treat signatures as approval of opaque bytes.

Standard accounts sign typed intents.

Wallets display deterministic state and asset diffs.

Smart-account policies can reject unknown calls, unlimited approvals, unsafe upgrades, and high-risk actions without timelocks.

## 22.1 Typed intent signing

Example transfer intent:

```rust
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

```txt
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

## 22.2 Native simulation

Before signing, wallets should receive deterministic simulation results:

```rust
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

But simulation alone is not enough.

If the interface is compromised, it can lie about the transaction and the simulation result.

High-value accounts require independent simulation attestation. That model is defined later in this document.

---

# 23. No infinite approvals

NOVA should kill ERC20-style infinite approvals by design.

Ethereum model:

```txt
approve(router, infinite)
swap later
hope router/frontend/token never gets compromised
```

NOVA-native token permissions should be:

* Bounded by amount
* Bounded by purpose
* Bounded by recipient
* Bounded by expiry
* Revocable
* One-time by default

## 23.1 Spend authorization

```rust
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

```rust
token.transferWithAuth(auth, signature);
```

After use:

```txt
nonce consumed
authorization dead
cannot be reused
cannot be increased
cannot become infinite
```

## 23.2 TokenSpend capability

```rust
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

**No infinite approvals. No permanent router power. Every spend permission has a cap, purpose, and expiry.**

---

# 24. On-chain contract verification

NOVA should not depend on Etherscan-style centralized source verification.

This is NRC-8.

The chain stores verification records and cryptographic commitments.

The full source can live in IPFS, Arweave, Git, NOVA source blobs, or cache nodes.

The chain stores the proof anchors.

## 24.1 Verification record

```rust
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

```rust
enum VerificationStatus {
    Unverified,
    SourceCommitted,
    ReproducibleBuildVerified,
    FormallyVerified
}
```

## 24.2 Verification flow

```txt
1. Developer writes Pulsar source
2. pulsarc compiles deterministically
3. Developer deploys bytecode with source/build commitments
4. Independent verifiers reproduce build
5. Output bytecode hash must match deployed bytecode hash
6. Verification status is written to NRC-8 registry
```

Decompilation may exist, but is not verification.

```txt
Decompiled view = explanation
Verified source = reproducible source-to-bytecode match
```

---

# 25. Compressed source availability

NOVA can optionally store compressed source on-chain or in a protocol source blob layer.

This is NRC-8S, an extension to NRC-8.

Recommended model:

```txt
readable Pulsar source
→ canonical archive
→ zstd compression
→ source blob
```

Do not use ugly-name minification as the source of truth.

Readable source compressed with zstd is preferred.

## 25.1 Why not ugly names?

If code is minified:

```rust
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

**Store readable canonical source, then compress bytes.**

## 25.2 Storage mode

```rust
enum SourceStorageMode {
    CommitmentOnly,
    OnChainCompressed,
    ExternalContentAddressed
}
```

---

# 26. On-chain runtime upgrades

NOVA should support Polkadot-style forkless runtime upgrades, but filtered through Bitcoin-grade paranoia.

This is NRC-9.

The idea:

```txt
Node host = stable execution host
Runtime code = on-chain upgradeable logic
```

## 26.1 Upgradeable vs non-upgradeable

Safe candidates:

* Fee rules
* Gas schedule
* Precompile pricing
* Oracle registry parameters
* Name service rules
* Account standard versions
* Pulsar stdlib system modules
* Runtime host function table
* Aurora committee parameters
* Storage rent parameters
* Transaction type registry
* Signature scheme registry

Dangerous candidates:

* PoW validity rules
* BlockDAG ordering rules
* Supply schedule
* Emission cap
* Historical state transition rules
* Address format
* Core hashing commitments

Those require stricter constitutional hard fork or social consensus paths.

## 26.2 Runtime package

```rust
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

## 26.3 Upgrade flow

```txt
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

## 26.4 Upgrade classes

| Class | Type                       | Example                             | Delay                       |
| ----- | -------------------------- | ----------------------------------- | --------------------------- |
| A     | Parameter upgrade          | Fee coefficients, oracle thresholds | 24h-72h                     |
| B     | Runtime module upgrade     | Account logic, name service         | 7 days                      |
| C     | VM/precompile upgrade      | New precompile, host function       | 14-30 days                  |
| D     | Consensus/security upgrade | PoW, DAG, supply, state root        | 30+ days / social consensus |

Core sentence:

**Forkless upgrades, filtered through Bitcoin-grade paranoia.**

---

# 27. Bridges

NOVA should prefer light-client bridges.

No multisig pretending to be decentralization.

No MPC committee as magical security theater.

A bridge should verify consensus proofs.

But the honest rule is:

**A bridge is only as secure as the weaker chain.**

wNOVA on Ethereum inherits Ethereum's cryptographic assumptions.

A NOVA bridge to a weaker chain inherits that chain's weakness.

There is no marketing fix for this.

## 27.1 Bridge risk display

Every bridge should expose:

* Bridge type
* Proof model
* Signer count
* Upgrade authority
* Daily mint cap
* Single transaction cap
* Challenge period
* Last verified proof
* Emergency pause authority
* Worst-case trust assumption

Example wallet label:

```txt
Bridge type:
Multisig bridge

Trust assumption:
3 of 5 signers can mint wrapped assets

Daily mint cap:
500,000 NOVA

Upgrade authority:
Instant upgrade by bridge admin multisig

Risk:
High
```

A bridge should not be allowed to hide its trust model behind vague language.

---

# 28. Bootstrapping and security modes

If NOVA has only 3 miners, it is not secure.

It is a devnet.

With 3 miners:

```txt
2 of 3 miners can dominate PoW
2 of 3 finality members can finalize
BFT tolerance is effectively zero
randomness is weak
bridges are dangerous
```

NOVA must define security modes.

## 28.1 Devnet mode

```txt
< 25 independent miners
no real value
no bridge
randomness experimental
Aurora optional or centralized
```

## 28.2 Bootstrap mode

```txt
25-99 independent miners
public testing
basic contracts
small-value experiments only
```

## 28.3 Early mainnet mode

```txt
100+ independent miners
basic transfers and contracts
limited randomness/oracle features
no large canonical bridge yet
```

## 28.4 Full security mode

Activation thresholds may include:

* 128+ bonded Aurora participants
* stable finality participation for 90 days
* no entity above 20% of hashrate
* multiple client implementations
* stable propagation
* audited bridges

Honest sentence:

**NOVA's full security claims only apply after the network reaches sufficient miner diversity, committee size, hashrate, client diversity, and stable finality participation. Early devnet and testnet phases are explicitly not Bitcoin-grade secure.**

---

# 29. Human-visible security

NOVA's safety model cannot stop at the smart contract language.

Pulsar can remove many contract-level footguns. Typed intents can make transactions easier to understand. Verification can prove which code is deployed.

But none of that is enough if the human signing the transaction is shown a lie.

Modern crypto attacks increasingly target the space between what the signer sees and what the chain executes. A signer may believe they are approving a treasury transfer while the actual transaction upgrades contract code, changes permissions, or grants control to an attacker.

NOVA treats this as a first-class security problem.

The signing layer must be designed so that the intent displayed to the human, the intent signed by each key, and the action executed by the chain are cryptographically bound together.

The goal:

**Bad contracts should be harder to write, and bad signatures should be harder to obtain.**

## 29.1 Canonical intent hash

Every high-risk transaction should produce a canonical intent object.

This object describes what the transaction means, not just the raw bytes being submitted.

Example:

```txt
Intent type:
Treasury transfer

From:
cold treasury

To:
hot wallet

Amount:
10,000 NOVA

Code changes:
none

New permissions:
none

Upgrade effects:
none

Oracle changes:
none

Bridge effects:
none
```

The canonical intent is hashed and signed alongside the transaction.

If the transaction bytes and the human-readable intent do not match, the account rejects the signature.

A wallet should not ask the user to sign opaque calldata.

It should ask the user to sign a specific, structured claim about what will happen.

## 29.2 Cross-signer intent agreement

Multisig security should not only require multiple signatures.

It should require multiple signers to agree on the same intent.

In a NOVA multisig, every signer must sign the same canonical intent hash.

If one signer approves a transfer intent and another unknowingly signs an upgrade intent, the transaction is invalid.

The chain should reject multisig transactions when the signer intent hashes diverge.

A multisig should not mean:

```txt
five people signed some bytes
```

It should mean:

```txt
five people signed the same human-readable action
```

## 29.3 Independent simulation attestation

Transaction simulation is useful, but simulation alone is not enough.

If the wallet UI is compromised, it can lie about the transaction and also lie about the simulation result.

NOVA should support independent simulation attestation for high-value accounts.

Possible models:

* A second device verifies the canonical intent and simulation result through an independent channel.
* A hardware signing device renders the transaction meaning from the canonical intent directly.
* Multiple independent simulators produce signed simulation results.
* The smart account requires a valid simulation attestation before accepting signatures above a threshold.

Core rule:

**The same compromised screen should not be trusted to display both the question and the answer.**

For low-value accounts, normal wallet simulation may be enough.

For treasuries, exchanges, bridges, DAOs, and protocol admin accounts, independent verification should be mandatory.

## 29.4 Code-change visibility

Any transaction that changes executable code must be displayed as a code-change event.

This includes:

* proxy upgrades
* smart account code rotation
* runtime module swaps
* contract migration
* permission changes that enable future upgrades
* stdlib module replacement

Code changes should never be hidden inside a generic contract interaction.

Wallets must display:

* Old code hash
* New code hash
* Verification status of the new code
* Compiler version
* Source availability
* Audit status
* Activation delay
* Upgrade authority
* Rollback path, if any

Example:

```txt
THIS TRANSACTION CHANGES CODE

Old code:
0xabc...

New code:
0xdef...

New code status:
Unverified

Activation:
Immediate

Risk:
Critical
```

For high-value accounts, immediate upgrades to unverified code should be rejected by default unless the account policy explicitly allows them.

## 29.5 Hardware attestation for high-value accounts

NOVA smart accounts should be able to require hardware-backed signing for high-value actions.

A custody account may enforce:

* only registered hardware devices can sign
* device attestation must be present
* firmware version must be approved
* signatures from normal browser wallets are rejected
* large transfers require hardware confirmation
* upgrades require hardware confirmation
* new signer registration requires delay

This does not remove all supply-chain risk, but it reduces the chance that malware on a laptop can silently approve a catastrophic transaction.

The signing device must display the canonical intent itself.

The laptop may lie.

The signing device should not.

## 29.6 Hardware + Biometric Attestation Extensions

NOVA supports multi-factor intent signing beyond traditional hardware wallets:

* FIDO2/WebAuthn attestation for consumer devices.
* Hardware enclave attestation (TEE) for server-side signing.
* Optional biometric hash attestation (stored as a device credential, never transmitted).
* Multi-factor intent signing: combination of hardware + biometric + knowledge factors.
* "Secure Element Required" flag for L4 accounts — rejects signatures from devices without a secure element.

Biometric data never leaves the device.

The chain only sees an attestation that a biometric check was performed, not the biometric data itself.

Device health checks can verify:

* Firmware version is not known-compromised.
* Secure element is present and active.
* Device has not been flagged by manufacturer revocation lists.

## 29.7 Hardware diversity and side-channel resistance

Relying on a single hardware vendor or TEE implementation creates a single point of failure.

If one secure element or TEE has a side-channel vulnerability, every L4 account depending on it is exposed.

Defenses:

* **Multi-vendor requirement**: L4 accounts should require signatures from devices produced by at least two independent hardware vendors. A vulnerability in one vendor's implementation does not compromise the account.
* **No single-TEE trust model**: No security-critical operation should depend on a single TEE. If TEE attestation is used, it must be combined with an independent channel (second device, hardware wallet from different manufacturer, or independent software verifier).
* **Side-channel mitigation awareness**: The threat model explicitly acknowledges that hardware side-channel attacks (cache timing, power analysis, electromagnetic leakage) exist. The protocol does not pretend they are impossible.
* **Hardware revocation response**: If a hardware vendor discloses a vulnerability, affected attestation keys are flagged. L4 accounts using compromised hardware enter a mandatory rotation period with reduced limits until new devices are registered.
* **Intent display on signing device**: The canonical intent must be rendered and confirmed on the signing device's own screen. The host computer's display is never trusted as the sole source of transaction meaning.

---

# 30. Threat model from historical crypto failures

NOVA should treat the history of crypto hacks as a requirements document.

Not all hacks are the same.

Some are language bugs.

Some are bad protocol economics.

Some are oracle failures.

Some are bridge failures.

Some are key-management failures.

Some are governance failures.

Some are frontend or signing deception.

Some are token misbehavior.

A useful security model does not pretend one abstraction solves all of them.

It asks:

* Can this failure mode be made impossible?
* Can it be made harder?
* Can it be slowed down?
* Can it be made visible?
* Can it be limited by caps?
* Can users and signers be warned before it happens?

## 30.1 Failure mode matrix

| Hack class               | Can NOVA prevent it? | NOVA defense                                          | Remaining risk                                                     |
| ------------------------ | -------------------- | ----------------------------------------------------- | ------------------------------------------------------------------ |
| Reentrancy               | Mostly               | Pulsar external-call rules                            | Compiler bugs, unsafe NovaVM modules                               |
| Integer overflow         | Mostly               | Checked arithmetic                                    | Unsafe low-level modules                                           |
| Infinite approval drain  | Mostly               | Capability permissions                                | User signs malicious bounded permission                            |
| Blind signing            | Partially            | Typed intents, risk labels                            | Compromised displays, social engineering                           |
| Signer deception         | Partially            | Canonical intent hash, multisig intent agreement      | Malware, hardware compromise                                       |
| Oracle manipulation      | Partially            | TWAP, confidence, source quorum, circuit breakers     | Market-wide manipulation, bad config                               |
| Bridge exploit           | Partially            | Light-client preference, caps, risk registry          | Weak external chain, bridge implementation bug                     |
| Governance attack        | Partially            | Voting delay, execution timelock, proposal simulation | Governance capture, bribery, apathy                                |
| Admin key compromise     | Partially            | Account security levels, timelocks, signer rotation   | Bad human operations                                               |
| Frontend compromise      | Partially            | Verified app manifest, frontend attestation           | Domain compromise, signing-key compromise                          |
| Custody compromise       | Partially            | Custody account standard, withdrawal delays           | Internal failure, social engineering                               |
| Bad upgrade              | Partially            | Upgrade timelock, verification, risk diff             | Legitimate governance approves malicious upgrade                   |
| Unsafe randomness        | Mostly               | Aurora randomness beacon                              | Committee withholding or liveness attack                           |
| Flash-loan manipulation  | Partially            | Oracle delay, state-change limits, TWAP               | Bad protocol design                                                |
| Malicious token behavior | Partially            | Token traits standard, protocol rejection             | Tokens lying about traits, supply-chain attacks on token contracts |
| MEV extraction           | Partially            | Encrypted mempool, MEV disclosure                     | Ordering power, censorship, private orderflow abuse                |
| Compiler soundness bug   | Partially            | Reference interpreter, bytecode verification, diff fuzzing | WASM-level reentrancy barrier failure, optimizer removing checks  |
| Prover soundness bug     | Partially            | Independent provers, proof checking, version pinning  | False "Formally Verified" label on malicious contract             |
| WASM sandbox escape      | Partially            | No JIT, interpreted fallback, multi-runtime verification | Zero-day in WASM engine allowing node memory access               |
| DAG randomness grinding  | Partially            | VDF post-processing, multi-source entropy, min-seed-distance | Miner coalition biasing beacon via DAG structure manipulation      |
| Supply chain compromise  | Partially            | Pinned deps, dependency auditing, reproducible builds | Backdoor in obscure Rust dependency                               |
| Eclipse / network partition | Partially         | Diverse peers, partition detection, checkpoint cross-validation | Botnet isolating Aurora committee or miners                        |
| Crypto implementation bug | Partially           | Multi-implementation verification, constant-time, formal verification of critical paths | Forged Aurora checkpoint via buggy signature aggregation           |
| Hardware side-channel    | Partially            | Multi-vendor requirement, hardware revocation response, no single-TEE trust | Key extraction from secure element via power analysis              |

## 30.2 Design principle

NOVA should not claim:

```txt
This can never be hacked.
```

The honest claim is stronger:

```txt
This system makes common historical failure modes structurally harder, more visible, more bounded, or impossible by default.
```

## 30.3 Expanded threat entries

The threat model is updated to include emerging attack surfaces:

* **AI-generated attacks**: AI-assisted vulnerability discovery and exploit generation. Mitigated by formal verification (Pulsar Prover) and continuous fuzzing.
* **Formal-tool supply-chain risk**: Compromised verification tooling that produces false proofs. Mitigated by independent prover implementations and reproducible verification.
* **Monitoring evasion**: Attackers who learn anomaly detection patterns and avoid triggering them. Mitigated by non-deterministic monitoring thresholds and independent monitoring nodes.
* **Post-quantum migration apathy**: Users and protocols that delay migrating to quantum-safe cryptography. Mitigated by NRC-42 enforcement timelines and on-chain quantum readiness scores.
* **Compiler optimizer removing safety checks**: Inspired by the Solidity Yul optimizer bugs. The Pulsar optimizer must never remove checked arithmetic, external-call barriers, or invariant checks. Differential testing catches regressions.
* **Under-constrained formal model**: Inspired by Circom under-constrained circuits. The Pulsar-to-Lean4 translation may omit constraints, allowing false proofs. Mitigated by independent prover implementations and adversarial prover testing.
* **Supply chain attacks on dependencies**: A backdoor in an obscure Rust crate used by `nova-types` or `nova-vm` could bypass all protocol-level security. Mitigated by pinned dependencies, auditing, and reproducible builds (section 7.1).
* **WASM runtime zero-day**: A sandbox escape in the WASM interpreter could allow crafted contracts to access node memory. Mitigated by no-JIT policy, interpreted fallback, and continuous runtime fuzzing (section 10.2).
* **Eclipse attacks on Aurora committee**: A botnet isolating committee members to feed them a distorted DAG view. Mitigated by diverse peer requirements and partition detection (section 8.4).
* **Cryptographic implementation flaws**: A bug in signature aggregation math could allow forged checkpoints. Mitigated by multi-implementation verification and constant-time implementations (section 9.6).

---

# 31. Account security levels

Not every account needs the same security model.

A burner wallet should not behave like an exchange cold wallet.

A DAO treasury should not behave like a game account.

NOVA accounts can declare security levels.

## 31.1 Security levels

| Level | Use case                     | Default policy                                            |
| ----- | ---------------------------- | --------------------------------------------------------- |
| L0    | Burner / test account        | Single signer, low friction                               |
| L1    | Normal user                  | Typed intents, bounded permissions                        |
| L2    | Active trader / builder      | Session keys, limits, device registry                     |
| L3    | DAO / protocol admin         | Multisig, timelocks, simulation attestation               |
| L4    | Treasury / exchange / bridge | Hardware attestation, withdrawal delay, challenge windows |

## 31.2 Example L4 policy

```txt
Transfers below 10,000 NOVA:
2 of 3 operational signers

Transfers above 10,000 NOVA:
3 of 5 treasury signers
24 hour delay
independent simulation required

Transfers to a new recipient:
48 hour delay

Contract upgrade:
4 of 5 policy signers
7 day delay
verified code required

Recovery key change:
14 day delay
hardware attestation required
```

## 31.3 New recipient cooldown

High-value accounts should treat new recipients as risk events.

```txt
Known recipient:
normal policy

New recipient:
delay required
extra signer required
monitoring alert emitted
cancel window available
```

This reduces the chance that a single bad signature drains an account instantly.

## 31.4 Capability warm-up

New permissions should be treated as risk events.

When a high-value account grants a new spending capability, the capability does not immediately reach full power.

During warm-up:

* low-value uses are allowed
* high-value uses are delayed
* unusual recipients trigger alerts
* the capability can be canceled without cost

A capability that has never been used should not be allowed to drain an account in a single transaction.

Combined with new recipient cooldowns and large transfer delays, this prevents the most damaging path:

```txt
attacker tricks signer
new capability is granted
new recipient is created
full balance drains in one transaction
```

That sequence should fail at every step.

---

# 32. Custody account standard

Exchanges, treasuries, bridges, DAOs, and large protocols should not write custody logic from scratch.

NOVA should provide a canonical custody account standard.

A custody account may include:

* daily withdrawal limits
* weekly withdrawal limits
* new recipient cooldowns
* large transfer delays
* separate operational keys and policy keys
* separate upgrade keys and withdrawal keys
* mandatory multisig for high-value actions
* public challenge windows
* pause-on-anomaly rules
* signer rotation delays
* recovery delays
* hardware signer requirements

This makes the safe custody pattern the default pattern.

Teams can still design custom custody systems, but they should have to explain why they are rejecting the standard one.

## 32.1 Pause-on-anomaly

A custody account can pause automatically when abnormal behavior appears.

Example triggers:

* transfer above historical range
* new recipient above threshold
* multiple failed signer attempts
* new signer added
* policy changed
* simulation mismatch
* frontend attestation mismatch
* bridge mint above daily average

Pausing should not be a magic security button.

It should be a bounded emergency mode with clear rules.

## 32.2 Separate powers

Operational keys should move funds within limits.

Policy keys should change limits.

Upgrade keys should change code.

Recovery keys should restore access.

No single key should do everything.

---

# 33. App contract upgrade safety

NOVA's protocol runtime upgrades use a conservative process.

Application contracts that hold significant value should follow the same philosophy.

A high-value app contract upgrade should move through a visible flow:

```txt
1. Upgrade proposed
2. New source code published
3. Reproducible build verified
4. Simulation result published
5. Risk diff generated
6. Public challenge window begins
7. Timelock expires
8. Upgrade activates
9. Monitoring period begins
```

The upgrade should expose:

* what code changes
* what permissions change
* what storage changes
* what assets can move
* what users are affected
* whether withdrawal behavior changes
* whether admin powers change
* whether oracle logic changes
* whether bridge logic changes

A contract should not be considered safe just because it is verified.

The upgrade path must also be safe.

## 33.1 Upgrade risk classes

| Class    | Example                                                               | Suggested delay |
| -------- | --------------------------------------------------------------------- | --------------- |
| Low      | Metadata update                                                       | 0-24h           |
| Medium   | Parameter change                                                      | 24-72h          |
| High     | Logic upgrade                                                         | 7 days          |
| Critical | Upgrade affects withdrawals, minting, bridge, oracle, or admin powers | 14-30 days      |

## 33.2 Unsafe upgrade labels

Wallets and explorers should label:

* Immutable
* Upgradeable with delay
* Upgradeable after verification
* Upgradeable instantly by multisig
* Upgradeable instantly by single key
* Unknown upgrade path

Instant upgrade by a single key should be treated as critical risk.

---

# 34. Oracle safety standard

Price is one of the most dangerous inputs in DeFi.

NOVA oracle standards should force protocols to make price assumptions explicit.

Required oracle fields:

* price
* decimals
* updatedAt
* confidence
* source count
* TWAP window
* liquidity depth
* deviation from previous price
* circuit breaker status

Protocols should be able to require:

* minimum liquidity
* maximum price deviation per block
* minimum oracle age
* maximum oracle age
* multi-source quorum
* fallback oracle
* emergency stale mode

Unsafe oracle usage should be visible in the verifier:

```txt
Oracle risk: HIGH
Reason: single source, no TWAP, no deviation limit
```

## 34.1 Same-transaction oracle use

Protocols should avoid updating a price and using that same update for borrowing, minting, or liquidation in the same transaction.

Risky:

```txt
manipulate price
update oracle
borrow against manipulated value
exit
```

Safer:

```txt
update price
wait for confirmation window
use price after delay
```

---

# 35. Bridge risk standard

Bridge hacks are among the largest failures in crypto history.

NOVA should treat bridges as dangerous until proven otherwise.

A bridge must declare:

* bridge type
* source chain
* destination chain
* proof model
* signer set
* upgrade authority
* mint authority
* pause authority
* daily cap
* transaction cap
* challenge period
* exit path
* known assumptions

## 35.1 Bridge types

| Type         | Trust model                                  |
| ------------ | -------------------------------------------- |
| Light client | Verifies source-chain consensus proofs       |
| Optimistic   | Assumes fraud proofs during challenge window |
| ZK bridge    | Verifies validity proof of source state      |
| Multisig     | Trusts signer threshold                      |
| MPC          | Trusts distributed signer threshold          |
| Custodial    | Trusts custodian                             |

NOVA should not let all of these look equally safe.

Light-client and ZK bridges should declare their formal verification status. Bridges that have been formally verified against their specification receive a higher trust rating in the bridge risk registry (NRC-27).

## 35.2 Wrapped asset mint limits

Wrapped assets should support:

* maxDailyMint
* maxTxMint
* maxTotalSupply
* emergencyPause
* slowExitMode
* proofDelay

A bridge failure should not be able to inflate wrapped assets infinitely in one transaction.

---

# 36. Verified app manifest

A verified contract is not enough if users interact with a compromised frontend.

NOVA should support verified app manifests.

An app manifest can declare:

* official domains
* frontend build hashes
* IPFS or Arweave hashes
* contract addresses
* allowed methods
* expected intent types
* risk policy
* frontend signing key

When a user signs through a frontend, the frontend may sign the same human-readable intent it displays.

The smart account can then check:

* Was this intent produced by an official frontend?
* Is this method expected for this app?
* Is this contract address registered?
* Does the displayed action match the submitted transaction?

This does not make frontend compromise impossible.

It does make spoofed or modified interfaces easier to detect.

## 36.1 Wallet warnings

Wallets should warn when:

* frontend domain is not official
* frontend build hash is unknown
* contract address is not in app manifest
* method is not expected for this app
* intent type does not match app policy
* frontend signature is missing for high-risk action

## 36.2 Frontend signing key management

A manifest's frontend signing key is itself a security surface.

The manifest must declare:

* key rotation policy
* key revocation path
* emergency revocation authority
* whether key changes require timelock

A stolen frontend signing key should not be a silent compromise.

Frontend key rotation events should be visible in the app manifest history, signed by a higher-authority key registered on-chain.

---

# 37. Contract and protocol risk labels

NOVA should not only make contracts safer.

It should make danger visible.

Wallets and explorers should label risk in plain language.

Examples:

* This transaction transfers funds.
* This transaction grants spending permission.
* This transaction changes executable code.
* This transaction upgrades a contract.
* This transaction changes an oracle.
* This transaction changes bridge permissions.
* This transaction adds a new signer.
* This transaction removes a guardian.
* This transaction disables a delay.
* This transaction allows future withdrawals.

High-risk actions should be impossible to hide inside vague labels like `contract interaction`.

The wallet should explain what matters before the user signs.

## 37.1 Example wallet screen

```txt
You are signing:

Action:
Deposit 100 NOVA into Vault X

Expected result:
You receive vault shares

Risks:
Vault is upgradeable
Oracle is single source
Withdrawals have 24h delay
Admin can pause withdrawals
No formal verification found

Permission created:
Vault can spend up to 100 NOVA
Expires in 10 minutes
One-time use
```

That is the opposite of blind signing.

---

# 38. Contract safety metadata

For NOVA to be serious about removing footguns, verification cannot only mean source code is uploaded.

Contracts should publish safety metadata.

Example:

```rust
struct ContractSafetyMetadata {
    verifiedSource: bool;
    reproducibleBuild: bool;
    formalInvariants: bool;
    fuzzTests: bool;
    auditReports: AuditReport[];
    bugBounty: BugBountyInfo;
    upgradeable: bool;
    upgradeDelay: u64;
    adminRoles: AdminRole[];
    oracleDependencies: OracleDependency[];
    bridgeDependencies: BridgeDependency[];
    knownRisks: string[];
}
```

A verified contract should show:

* Tests: yes/no
* Fuzzing: yes/no
* Formal invariants: yes/no
* Audit: yes/no
* Bug bounty: yes/no
* Known risks: listed/not listed

## 38.1 Formal invariants

Pulsar should make invariants normal:

```rust
@invariant(totalAssets >= totalShares)
@invariant(reserve == balance)
@invariant(userDebt <= collateralValue * maxLTV)
```

The toolchain should run:

* unit tests
* property tests
* fuzz tests
* symbolic checks
* invariant checks
* state machine tests
* differential tests

---

# 39. Safer token standard

Many hacks come not from broken contracts, but from broken tokens.

ERC777 callbacks created reentrancy through normal-looking transfers.

Fee-on-transfer tokens broke vault accounting.

Rebasing tokens made share math wrong.

Tokens with hidden mint authority inflated supply.

Tokens with hidden pause authority froze users.

Tokens with hidden blacklist authority broke liquidations.

NOVA should not copy ERC20 blindly.

A NOVA token standard should make dangerous behavior visible.

This is NRC-36.

## 39.1 Token traits

Every token must declare its behavioral traits:

```rust
struct TokenTraits {
    rebasing: bool;
    feeOnTransfer: bool;
    externalCallback: bool;
    mintable: bool;
    pausable: bool;
    blacklistable: bool;
    oracleDependent: bool;
    fixedSupply: bool;
    paymaster: bool;
    formallyVerified: bool;
}
```

Protocols can reject tokens with traits they do not handle.

A vault that does not support fee-on-transfer should reject fee-on-transfer tokens at the source.

An AMM that does not support rebasing should reject rebasing tokens at the source.

A lending protocol that does not support callbacks should reject callback tokens at the source.

## 39.2 Token trait enforcement

The token does not get to lie about itself.

But this only works if NOVA defines how traits are enforced.

Trait enforcement requires runtime hooks.

Examples:

* If a token declares `externalCallback: false`, the VM can reject token execution paths that attempt external calls during transfer.
* If a token declares `fixedSupply: true`, the token standard rejects mint paths after deployment.
* If a token declares `feeOnTransfer: false`, transfer results must preserve exact sent and received amounts.
* If a token declares `pausable: false`, the token cannot expose a pause path.
* If a token declares `blacklistable: false`, the token cannot branch transfer validity on an address blacklist.

Some traits are fully enforceable by the runtime.

Others are only detectable through verification, simulation, and monitoring.

The standard must label each trait as:

```txt
consensus-enforced
runtime-detectable
verification-dependent
metadata-only
```

A token that violates a runtime-enforceable trait should fail execution.

A token that violates a declared but non-consensus trait should be quarantined by protocol policy and flagged by wallets and explorers.

## 39.3 Authority declarations

If a token has authorities, they must be public:

* mint authority
* pause authority
* blacklist authority
* upgrade authority
* fee authority
* oracle authority

Each authority declares:

* who holds it
* how it rotates
* how it is timelocked
* what it can do
* what it cannot do

A token where one key can pause, blacklist, mint, and upgrade should be impossible to deploy without that risk being loud in every wallet and every protocol.

## 39.4 Behavior locks

A token may declare permanent locks:

* mint authority renounced forever
* pause authority renounced forever
* upgrade authority renounced forever
* blacklist authority renounced forever

Locks must be on-chain and verifiable.

If a token declares mint authority is renounced and the chain detects an attempt to mint, the transaction is invalid.

This makes fixed supply token mean something.

## 39.5 Wallet display

When a wallet sees a token, it should show:

```txt
USDX
A stablecoin

Traits:
mintable
pausable
blacklistable
oracle dependent

Authorities:
mint - 3 of 5 multisig
pause - single key
blacklist - single key
upgrade - none

Risk:
Single key can freeze your balance.
Single key can blacklist your address.
Issuer can mint unlimited new supply.
```

A user signing an approval for that token should see those risks before the approval is granted.

## 39.6 Protocol trait policy

Every DeFi protocol should publish a trait policy:

```rust
struct ProtocolTraitPolicy {
    acceptsRebasing: bool;
    acceptsFeeOnTransfer: bool;
    acceptsCallback: bool;
    acceptsPausable: bool;
    acceptsBlacklistable: bool;
    requiresFixedSupply: bool;
    minimumAuthorityDelay: u64;
}
```

Token additions to a protocol are checked against the policy.

Mismatches are rejected at the source, not after the accounting has already broken.

This turns "we never noticed it was a fee-on-transfer token" from an excuse into a deployment-time error.

---

# 40. Governance safety standard

Governance is one of the largest attack surfaces in crypto.

A governance system cannot be safe if voters cannot understand proposals.

NOVA should not enforce one governance model.

But NOVA should require that governance actions be visible.

This is NRC-29.

## 40.1 Proposal surface

Every on-chain governance proposal must publish its effect surface:

* assets that can move
* contracts that can be upgraded
* oracle sources that can change
* permissions that can be granted or revoked
* tokens that can be minted
* admin roles that can change
* bridges that can be reconfigured

A vote should not say:

```txt
Proposal #42
```

A vote should say:

```txt
Proposal #42:
Can move 80% of treasury.
Can upgrade vault logic.
Can mint unlimited token X.
```

A voter who cannot see what they are voting on should not be expected to vote responsibly.

## 40.2 Proposal intent hash

Governance proposals should also have canonical proposal intent hashes.

A voter should not sign or vote on arbitrary execution bytes.

They should vote on a structured description of the proposal's effects:

* assets moved
* contracts upgraded
* roles changed
* tokens minted
* oracles changed
* bridges reconfigured
* delays changed
* veto powers changed

If the proposal bytes do not match the proposal intent, execution fails.

This connects governance safety to NOVA's core typed intent model.

## 40.3 Voting power borrowing

Governance attacks often use temporary voting power.

```txt
Borrow large supply through a flash loan.
Vote.
Repay.
Profit.
```

NOVA's governance standard should support voting checkpoints:

* voting power snapshot taken before proposal
* borrowed tokens cannot vote
* voting weight cannot be transferred during voting window
* new tokens cannot be minted into a vote

This is a protocol-level convention, not a chain-level rule.

But governance contracts that ignore this should be visibly flagged.

## 40.4 Proposal simulation

Every executable proposal should publish a deterministic simulation. For proposals that affect high-value contracts, a formal proof of safety properties may also be required:

* state diff if executed
* asset movements
* upgrade effects
* permission changes
* formal verification of invariant preservation (optional, mandatory for critical proposals)

**Trojan proposal defense**: A sophisticated attacker may craft a governance proposal that passes formal verification but contains a subtle logical flaw. Defenses:

* Formal verification of a proposal is necessary but not sufficient for approval.
* All proposals undergo independent human review by at least two auditors who were not involved in writing the proposal.
* The challenge period includes adversarial testing: security researchers are explicitly encouraged to attempt to break the proposal.
* Proposals that modify consensus-critical code (Class D) require social consensus, not just governance vote, precisely because formal verification alone cannot be trusted to catch every flaw.

* state diff if executed
* asset movements
* upgrade effects
* permission changes

Voters and watchers should see what the proposal does, not what it claims to do.

## 40.5 Execution timelock

High-impact proposals should not execute immediately on passing.

A timelock window allows:

* discovery of malicious effects
* emergency response
* exit by affected users
* veto by guardian roles, if defined

Instant execution after a passing vote should be a critical risk flag in every governance display.

## 40.6 Quorum and emergency veto

Governance contracts should declare:

* quorum floor
* proposal threshold
* veto authority, if any
* emergency response authority
* pause authority during active exploit

Mechanics are protocol-level decisions.

Visibility of those mechanics is chain-level required.

---

# 41. MEV and encrypted mempool

NOVA does not pretend MEV disappears.

The goal is to reduce avoidable extraction and make ordering assumptions explicit.

NOVA's encrypted mempool design should answer:

* Who can decrypt pending transactions?
* When are transactions revealed?
* Can miners reorder after reveal?
* How are failed decryptions handled?
* How are private orderflow and searchers disclosed?
* What happens if the decryption committee censors?
* What metadata remains visible before reveal?

## 41.1 Encrypted mempool goals

An encrypted mempool should reduce:

* sandwich attacks
* copy-trading of pending transactions
* liquidation sniping from visible calldata
* governance proposal sniping
* bridge withdrawal targeting

It cannot eliminate:

* validator ordering power after reveal
* censorship
* cross-domain MEV
* private orderflow abuse
* latency games

## 41.2 MEV disclosure standard

Applications should disclose ordering assumptions.

Example:

```txt
This protocol is sensitive to transaction ordering.
This protocol depends on oracle update ordering.
This protocol allows liquidations.
This protocol has auction mechanics.
This protocol may leak value if transactions are visible before execution.
```

MEV risk should be visible, not hidden behind performance marketing.

---

# 42. NRC standards index

NRC means **NOVA Request for Comment**.

These standards define NOVA's core interfaces, wallet UX, verification model, upgrade model, and safety guarantees.

## 42.1 Core NRCs

```txt
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

## 42.2 Security extension NRCs

```txt
NRC-16  Canonical Intent Hash
NRC-17  Multisig Intent Agreement
NRC-18  Independent Simulation Attestation
NRC-19  Code Change Intent Standard
NRC-20  Account Security Levels
NRC-21  Custody Account Standard
NRC-22  Application Contract Upgrade Standard
NRC-23  Verified App Manifest
NRC-24  Hardware Signer Attestation
NRC-25  Capability Warm-Up Periods
NRC-26  Wallet Risk Label Standard
NRC-27  Bridge Risk Registry
NRC-28  Wrapped Asset Mint Limits
NRC-29  Safe Governance Standard
NRC-30  Oracle Safety Standard
NRC-31  Contract Admin Role Registry
NRC-32  Contract Safety Metadata
NRC-33  Encrypted Mempool
NRC-34  MEV Disclosure Standard
NRC-35  Frontend Integrity Registry
NRC-36  Token Traits Standard
NRC-37  Governance Proposal Intent Hash
NRC-38  Enforcement Layer Classification
```

## 42.3 Formal verification & monitoring NRCs

```txt
NRC-39  Pulsar Formal Specification Language
NRC-40  Pulsar Prover & High-Assurance Certification
NRC-41  On-Chain Anomaly Detection & Monitoring Hooks
NRC-42  Post-Quantum Migration Standard
NRC-43  Shielded Intent Privacy (Optional)
NRC-44  AI-Assisted Verification Guidelines
NRC-45  Compiler Soundness & Supply Chain Security
```

Most important safety stack:

```txt
NRC-1   Smart accounts
NRC-7   Typed intent signing
NRC-10  No infinite approvals
NRC-16  Canonical intent hash
NRC-17  Multisig intent agreement
NRC-18  Independent simulation attestation
NRC-20  Account security levels
NRC-21  Custody account standard
NRC-23  Verified app manifest
NRC-26  Wallet risk labels
NRC-29  Safe governance standard
NRC-32  Contract safety metadata
NRC-36  Token traits standard
NRC-37  Governance proposal intent hash
NRC-39  Pulsar formal specification language
NRC-40  Pulsar Prover & high-assurance certification
NRC-41  On-chain anomaly detection & monitoring hooks
NRC-45  Compiler soundness & supply chain security
```

---

# 43. Roadmap

## Phase 0: Study alpha

* Threat model from historical crypto failures
* Typed intent object
* Capability-based token permission prototype
* Token traits registry prototype
* Risk label generator
* Simple Pulsar examples
* Multisig intent agreement prototype
* Custody account policy simulator
* Governance proposal intent prototype
* Verified app manifest draft
* Pulsar Prover MVP
* Monitoring simulator

## Phase 1: Months 0-6

* `nova-types`
* `nova-state`
* `nova-pow`
* Basic single-chain PoW devnet
* Initial account model
* NRC draft repository

## Phase 2: Months 6-12

* NovaVM deterministic WASM execution
* Raw WAT testing
* Basic transaction execution
* Basic typed intent object

## Phase 3: Months 12-18

* Pulsar v0 compiler
* ERC20-equivalent contracts
* Basic account abstraction
* Capability permissions prototype
* Token traits standard draft
* Devnet launch

## Phase 4: Months 18-24

* BlockDAG
* Aurora finality
* Encrypted mempool
* RPC
* Public testnet

## Phase 5: Months 24-30

* Pulsar v1 safety features
* Formal tooling
* Pulsar Prover integration
* NRC-1 to NRC-15 draft implementations
* Wallets and explorer
* Risk labels v0
* Monitoring hooks v0

## Phase 6: Months 30-36

* Human-visible signing security
* Canonical intent hash
* Multisig intent agreement
* Independent simulation attestation
* Custody account standard
* Verified app manifests
* Governance safety reference module
* Token traits enforcement in stdlib
* Formal verification gates for high-value contracts
* Post-quantum migration tooling

## Phase 7: 36+

* Stress tests
* Audits
* Multiple implementations
* Bug bounties
* Formal verification audit of core protocol
* Post-quantum readiness review
* Mainnet readiness review
* Mainnet launch: **First Light**

---

# 44. What NOVA does not claim

NOVA does not claim:

* No contract can ever be exploited.
* Quantum risk is solved forever.
* Bridges become magically safe.
* Oracle manipulation disappears.
* Governance cannot fail.
* Users cannot lose keys.
* Compiler bugs cannot exist.
* Runtime upgrades are risk-free.
* Three miners are enough for Bitcoin-grade security.
* Wallet screens cannot lie.
* Humans cannot be socially engineered.
* Tokens always behave as they declare.
* MEV disappears.
* Formal verification eliminates all bugs.
* The compiler always preserves language guarantees at the WASM level.
* The WASM runtime has no sandbox escape vulnerabilities.
* Hardware signing devices have no side-channel vulnerabilities.
* Dependencies have no supply-chain compromises.
* Signature aggregation has no implementation bugs.

Remaining risks:

* Business logic errors
* Bad specifications
* Governance capture
* Oracle failures
* Bridge bugs
* Wallet compromise
* Frontend compromise
* Social engineering
* Compiler bugs
* VM bugs
* Protocol implementation bugs
* Novel cryptographic attacks
* Miner centralization
* Finality committee concentration
* Unsafe runtime upgrades
* Bad custody operations
* Signer device compromise
* Hardware supply-chain risk
* Token contract supply-chain risk
* MEV and censorship
* Compiler soundness gaps (language guarantees not preserved at WASM level)
* WASM runtime sandbox escapes
* Prover soundness bugs (false "Formally Verified" labels)
* Dependency supply-chain compromises
* Cryptographic implementation errors
* Network eclipse and partitioning attacks
* Hardware side-channel and TEE vulnerabilities
* Optimizer removing safety checks for performance

The strongest honest claim:

**NOVA makes many catastrophic historical smart-contract, signing, custody, oracle, bridge, governance, token, and MEV-related failure modes structurally unreachable, significantly harder to express, harder to hide, more bounded, or slower to execute — and where full prevention is impossible, provably bounded and continuously monitored.**

---

# 45. Culture and lore

NOVA vocabulary:

| Name        | Meaning                    |
| ----------- | -------------------------- |
| NOVA        | Chain                      |
| Pulsar      | Smart contract language    |
| NovaVM      | Execution environment      |
| Aurora      | Finality gadget            |
| NovaHash    | Proof-of-work function     |
| centova     | Smallest unit, 10^-18 NOVA |
| Gnov        | Gas denomination           |
| First Light | Mainnet launch             |
| Big Bang    | Genesis block              |
| superNOVA   | Major hard fork            |

Cultural pitch:

```txt
Bitcoin gave us unstoppable money.
Ethereum gave us unstoppable programs.
NOVA makes those programs harder to break.
Prove it.
```

Formal verification and reproducible builds are cultural requirements, not optional extras.

Shorter:

```txt
Bitcoin-grade settlement.
Ethereum-grade apps.
Smart contracts with the footguns removed.
```

Security pitch:

```txt
NOVA should not only make bad contracts harder to write.
It should make bad signatures harder to obtain.
It should make bad tokens harder to hide.
It should make bad governance harder to execute.
```

---

# 46. Closing

NOVA is an attempt to combine six ideas:

1. Bitcoin's permissionless proof-of-work security.
2. Ethereum's programmable application layer.
3. A safety-first language and VM where catastrophic bug classes are removed by construction.
4. A human-visible signing model where dangerous actions cannot hide behind opaque bytes.
5. A protocol surface where tokens, oracles, bridges, governance, custody, and upgrades declare their risks before users touch them.
6. Accessible formal verification that gives mathematical confidence where it matters most.

The hard part is not making another fast chain.

The hard part is making the safety claims real.

That requires:

* Formal specifications
* Deterministic VM rules
* Compiler verification
* Multiple implementations
* Audited crypto
* Adversarial testnets
* Permanent bug bounties
* Conservative engineering culture
* Honest bootstrapping claims
* Native verification
* No blind signing
* No infinite approvals
* Runtime upgrades with public challenge windows
* Canonical intent hashes
* Multisig intent agreement
* Independent simulation attestation
* Verified app manifests
* Risk labels in wallets and explorers
* Token traits declared and enforced
* Governance actions made visible before they execute
* MEV assumptions disclosed instead of hidden
* Formal verification accessible to developers, not just researchers
* On-chain monitoring that catches what static rules miss
* Post-quantum migration built into the protocol timeline

NOVA's north star:

**The future of smart contracts should not depend on every developer remembering every footgun.**

The future of custody should not depend on every signer trusting whatever their screen shows them.

The future of DeFi should not depend on every protocol guessing how every token behaves.

The chain should remove the footguns.

The wallet should expose the danger.

The signer should understand the action.

The protocol should know what kind of token it is holding.

The formal tools should give mathematical confidence where it matters most.

The monitoring layer should watch what humans miss.

That is NOVA.
