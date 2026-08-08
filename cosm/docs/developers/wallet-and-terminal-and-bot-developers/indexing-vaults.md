# Indexing Vault Factories & Vaults

Vault-aware products feel premium because tax becomes interactive.

## Factories

Index official factories (and any later registered partners). Store:

- factory address  
- `vaultType` string (`split`, `scheduled-buyback`, …)  
- launch schema (for create-token forms)  

## Instances

When a token launches through CosmVaultPortal:

1. Factory deploys a vault instance  
2. VaultPortal stores `token → vault`  
3. Your indexer links both  

Read vault presence from VaultPortal (`getVault` / `tryGetVault`).  
**Do not** guess a vault from “beneficiary looks like a contract.”

## UI schema

Each vault can expose `vaultUISchema` — a declarative list of view/write methods. Terminals that render schema dynamically support new strategies without shipping an app update for every factory.

## Suggested vault card fields

- Strategy name (`vaultType`)  
- Short description from schema  
- TVL / pending rewards if the vault exposes them  
- Primary CTA (claim / burn / stake / view schedule)
