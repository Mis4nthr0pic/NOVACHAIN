# NOVA Example Contracts

Example smart contracts written in **Pulsar**, NOVA's safety-first contract language.

## Contracts

| Contract | File | Description |
|---|---|---|
| PiggyBank | `piggy_bank.pulsar` | Time-locked savings with explicit receive, invariants, and typed errors |
| Counter | `counter.pulsar` | Simple counter demonstrating basic Pulsar syntax |
| SimpleToken | `simple_token.pulsar` | Basic token following NRC-10 capability patterns |
| BtcRandomLottery | `btc_random_lottery.pulsar` | Oracle + randomness integration example |
| Escrow | `escrow.pulsar` | Buyer/seller escrow with arbiter dispute resolution |
| MultisigWallet | `multisig_wallet.pulsar` | m-of-n multisig with confirmation tracking |
| Timelock | `time_lock.pulsar` | Governance timelock for delayed execution |
| VulnerableVault | `vulnerable_vault.pulsar` | Demonstrates business-logic bugs Pulsar cannot prevent |

## Safety Features Demonstrated

- `external { }` blocks isolate outbound calls (reentrancy protection)
- Effects-before-interactions enforced structurally
- Checked arithmetic by default
- Typed errors instead of string reverts
- `@invariant` contract invariants
- Explicit `receive` blocks for fund acceptance
- No `delegatecall`, `selfdestruct`, `tx.origin`, or inline assembly
