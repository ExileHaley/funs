# Quick Start — Vault Builders

1. Define the strategy in one sentence (“split tax to N wallets”, “buy back every X hours”, …)  
2. Implement a VaultFactory + Vault instance pair  
3. Expose:
   - `vaultType`  
   - launch schema for `vaultData`  
   - `vaultUISchema` for runtime pages  
4. Register the factory on CosmVaultPortal  
5. Prove a full Path-B launch + tax funding + user action path  
6. Document whether the strategy needs automated operators  

Official factories are the reference for schemas, upgrade patterns, and UX expectations.
