# Vault & VaultFactory Specification

## Responsibilities

| Component | Responsibility |
|-----------|----------------|
| **VaultFactory** | Creates vaults, declares `vaultType`, exposes launch schema, manages upgrades |
| **Vault** | Receives tax value, runs strategy logic, exposes user/operator actions |
| **Schemas** | Tell terminals how to render launch forms and detail pages |

## Lifecycle

```text
Factory registered on VaultPortal
        │
        ▼
Creator launches via VaultPortal with vaultData
        │
        ▼
Factory.newVault → BeaconProxy instance
        │
        ▼
Tax token launched & linked
        │
        ▼
Trades fund tax → vault logic → user claims / buybacks / splits
```

## Deployment pattern (official style)

- Factory proxy: Transparent upgradeable proxy  
- Vault instances: Beacon proxies sharing one implementation per `vaultType`  

This lets Cosm upgrade a strategy family coherently while keeping each token’s vault address stable.

## Compatibility bar

A vault is “Cosm-ready” when:

- VaultPortal can launch it without custom off-chain glue  
- A schema-driven terminal can render its main actions  
- Tax funding cannot brick the vault under normal trading
