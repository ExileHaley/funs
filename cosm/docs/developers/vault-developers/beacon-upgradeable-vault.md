# Building Upgradeable Vaults — Beacon Proxy Pattern

Official Cosm vaults separate **factory upgrades** from **strategy upgrades**:

| Layer | Pattern | Why |
|-------|---------|-----|
| Factory | Transparent proxy | Evolve creation / schema / registration logic |
| Vault instances | Beacon proxy | Upgrade all vaults of a type together |

## Product implications

- Token ↔ vault addresses stay stable across upgrades  
- Strategy bugfixes can roll out by vault type  
- Terminals should tolerate additive schema changes  

## Guidance for partners

If you ship a custom factory:

- Document who can upgrade  
- Prefer transparent communication when storage layouts change  
- Version your schemas so UIs can fall back gracefully
