# NRC-8S: Compressed Source Availability

**Status:** Draft  
**Category:** Developer Tooling  
**Extends:** NRC-8  

## Abstract

Optional extension to NRC-8 that defines how compressed source code is stored on-chain or in NOVA source blobs.

## Motivation

On-chain source storage makes contracts permanently readable without external dependencies. However, raw source can be expensive. Compressed source balances readability with cost.

## Specification

### Storage Mode

```typescript
enum SourceStorageMode {
    CommitmentOnly,              // only hash stored on-chain
    OnChainCompressed,           // zstd-compressed source on-chain
    ExternalContentAddressed,    // IPFS, Arweave, Git, etc.
}
```

### Recommended Compression

```text
readable Pulsar source
→ canonical archive (tar)
→ zstd compression
→ source blob
```

### Why Not Minification

Minified source:

```typescript
contract C{storage v:u256;event I(a:address,v:u256);error U;...}
```

Requires a symbol map to recover readability:

```json
{
  "contract": { "C": "Counter" },
  "storage": { "v": "value" },
  "events": { "I": "Incremented" },
  "errors": { "U": "CounterUnderflow" }
}
```

The symbol map often consumes most of the savings from minification.

Preferred approach:

> Store readable canonical source, then compress bytes with zstd.

### Source Blob

```typescript
struct SourceBlob {
    sourceRoot: bytes32;
    data: bytes;           // zstd-compressed
    compressionAlgorithm: CompressionAlgorithm;
    originalSize: u32;
    compressedSize: u32;
}

enum CompressionAlgorithm {
    Zstd,
    None,
}
```

### Gas Pricing

Source blobs are priced by:

- Bandwidth gas (calldata size)
- State growth gas (persistent storage size)

Compression reduces both.

## Rationale

Readable source + compression is better than minified source + symbol maps. Developers, auditors, and users can read the actual code without a reverse-mapping step.

## Security Considerations

- Source blob must hash to the committed `sourceRoot`
- Decompression must be deterministic
- Size limits prevent abuse
