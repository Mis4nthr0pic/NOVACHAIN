# NRC-31: Contract Admin Role Registry

**Status:** Draft  
**Category:** Developer Tooling  

## Abstract

Defines a standard registry for declaring and discovering admin roles held by token contracts. Every authority over a token must be publicly enumerable so that wallets, explorers, and risk tools can surface concentrated authority risks before a user interacts.

## Motivation

A token where one key can pause, blacklist, mint, and upgrade should be impossible to deploy without that risk being loud in every wallet. Today, authority structures are hidden in contract code. Users discover them only after harm. NOVA v0.5 section 39.3 requires that if a token has authorities, they must be public. This NRC provides the machine-readable format.

Centralized authority is not inherently wrong, but invisible centralized authority is. A registry lets wallets render a clear authority summary and lets protocols set policies about which role concentrations are acceptable.

## Specification

### AdminRole

```typescript
enum AdminRoleType {
    Mint,
    Pause,
    Blacklist,
    Upgrade,
    Fee,
    Oracle,
}

struct AdminRole {
    roleType: AdminRoleType;
    holder: address;
    rotationPolicy: string;
    timelockDuration: u64;
    capabilities: vec<string>;
    limitations: vec<string>;
}
```

### RoleRegistry

```typescript
struct RoleRegistry {
    contractAddress: address;
    roles: vec<AdminRole>;
    registeredAt: u64;
    updatedAt: u64;
}
```

### Registry Interface

```typescript
trait RoleRegistryStore {
    fn register(contractAddress: address, roles: vec<AdminRole>);
    fn getRoles(contractAddress: address) -> vec<AdminRole>;
    fn getHoldersByRole(contractAddress: address, roleType: AdminRoleType) -> vec<address>;
    fn hasConcentratedAuthority(contractAddress: address, holder: address) -> bool;
    fn updateRoles(contractAddress: address, roles: vec<AdminRole>);
}
```

### Concentrated Authority Detection

```typescript
fn hasConcentratedAuthority(contractAddress: address, holder: address) -> bool {
    let roles = getRoles(contractAddress);
    let held = roles.filter(|r| r.holder == holder);
    let sensitiveTypes = [AdminRoleType::Mint, AdminRoleType::Pause, AdminRoleType::Blacklist, AdminRoleType::Upgrade];
    let sensitiveHeld = held.filter(|r| sensitiveTypes.contains(r.roleType));
    return sensitiveHeld.len() >= 2;
}
```

### Wallet Display

Wallets and explorers must render admin roles for any token interaction:

```text
Token: NOVA
Admin Roles:
  Mint      nova1abc...  (timelock: 24h)
  Pause     nova1abc...  (timelock: 24h)
  Blacklist nova1abc...  (timelock: 24h)
  Upgrade   nova1def...  (timelock: 48h)
  Fee       nova1ghi...  (no timelock)

WARNING: Address nova1abc... holds Mint + Pause + Blacklist
```

### Deployment Validation

```typescript
fn validateDeployment(registry: vec<AdminRole>) -> bool {
    let holderCounts: map<address, u64> = {};
    for role in registry {
        if holderCounts.contains(role.holder) {
            holderCounts[role.holder] += 1;
        } else {
            holderCounts[role.holder] = 1;
        }
    }
    for (holder, count) in holderCounts {
        if count >= 3 {
            return false;
        }
    }
    return true;
}
```

## Rationale

Role types are enumerated rather than free-form strings so that wallets can build consistent UI and risk tooling can compare across tokens. Capabilities and limitations are strings to allow protocol-specific nuance. Timelock duration is first-class because it directly affects the urgency of responding to malicious role use.

## Security Considerations

- The registry itself does not enforce role behavior; it makes roles visible. On-chain enforcement is handled by NRC-36 token traits and the VM.
- A malicious deployer could register incomplete roles. Auditing tooling should cross-reference registry entries with verified contract behavior.
- Role rotation must emit events so that monitoring systems can detect changes.
- Concentrated authority warnings should not be dismissible in wallets for high-value interactions.
