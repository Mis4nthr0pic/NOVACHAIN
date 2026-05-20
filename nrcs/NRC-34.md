# NRC-34: MEV Disclosure Standard

**Status:** Draft  
**Category:** Core Protocol  

## Abstract

Requires applications to publish structured disclosures about their ordering sensitivity. MEV risk must be visible to users, not hidden behind performance marketing.

## Motivation

NOVA v0.5 section 41.2 states that applications must disclose ordering assumptions. Today, users interact with DeFi protocols, DEXs, lending platforms, and governance systems without understanding how transaction ordering affects their outcomes. Sandwich attacks, liquidation sniping, governance sniping, and bridge withdrawal targeting all exploit ordering opacity.

Protocols that are sensitive to ordering should say so. Protocols that depend on oracle timing should say so. Protocols that run auctions should say so. This information must be machine-readable so that wallets, explorers, and risk tools can render it without requiring users to read documentation.

## Specification

### MEVDisclosure

```typescript
struct MEVDisclosure {
    contractAddress: address;
    sensitiveToOrdering: bool;
    dependsOnOracleOrdering: bool;
    allowsLiquidations: bool;
    hasAuctionMechanics: bool;
    mayLeakValueIfVisible: bool;
    orderingRiskDescription: string;
    mitigations: vec<string>;
    publishedAt: u64;
    updatedAt: u64;
}
```

### Disclosure Interface

```typescript
trait MEVDisclosureStore {
    fn publish(disclosure: MEVDisclosure);
    fn getDisclosure(contractAddress: address) -> option<MEVDisclosure>;
    fn updateDisclosure(disclosure: MEVDisclosure);
    fn getDisclosuresByRisk(riskField: string) -> vec<MEVDisclosure>;
}
```

### Display Format

Wallets and explorers must render MEV disclosure for any contract interaction:

```text
MEV Risk: nova1abc...

  Ordering sensitive:     Yes
  Oracle ordering:        Yes
  Liquidations:           Yes
  Auction mechanics:      No
  Value leak if visible:  Yes

  Description:
  "Swap execution depends on pool ordering. Large trades
   may be sandwiched if transactions are visible before
   inclusion."

  Mitigations:
  - Encrypted mempool (NRC-33)
  - Commit-reveal scheme
  - Slippage protection
```

### Compact Display

For transaction confirmation screens:

```text
MEV: Order-sensitive | Oracle-dependent | Liquidations | Value leak
```

### Missing Disclosure

When no MEV disclosure is published:

```text
MEV Risk: NOT DISCLOSED

  This contract has not published MEV risk disclosure.
  Assume ordering-sensitive until proven otherwise.
```

### Explorer Integration

Block explorers should tag contracts with MEV disclosure status:

```text
nova1abc...  DEX Pool  [Order-Sensitive] [Oracle-Dep] [Liquidations]
nova1def...  Lending    [Liquidations] [Value Leak]
nova1ghi...  Governance [Order-Sensitive]
```

## Rationale

Boolean fields capture the most common MEV vectors. The free-text description allows nuanced explanation that booleans cannot capture. Mitigations let protocols explain what they do about MEV, not just that it exists. Missing disclosure is displayed as a warning, not hidden, because the absence of disclosure is itself information.

## Security Considerations

- MEV disclosure is self-reported and cannot be verified on-chain in the general case. Tooling should flag inconsistencies between disclosed behavior and observed behavior.
- Protocols may understate their MEV sensitivity. Risk tools should default to assuming ordering sensitivity when disclosure is absent or incomplete.
- The orderingRiskDescription field should not be used to minimize risk. Wallets should render it verbatim without interpretation.
- MEV disclosure complements the encrypted mempool (NRC-33) but does not replace it. Disclosure makes risk visible; encryption reduces it.
