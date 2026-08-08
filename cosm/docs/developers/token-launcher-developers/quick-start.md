# Quick Start — Token Launchers

## Decide the launch path

| Creator goal | Path | Contract |
|--------------|------|----------|
| Non-tax token | Portal | `newTokenV7` |
| Tax token, simple destinations | Portal | `newTokenV6` |
| Tax token + Cosm vault strategy | VaultPortal | `newCosmTokenV6WithVault` |

## Build the form

Minimum fields:

- Name, symbol  
- Meta (image / social URI)  
- Quote token  
- Optional first buy amount  
- For tax: buy/sell tax and destinations  
- For vault: factory + strategy parameters  

## Behind the form

1. Find a CREATE2 salt matching vanity (`0x0222` or `0x0111`)  
2. Encode launch params (and `vaultData` when needed)  
3. Simulate the transaction  
4. Request ERC20 approvals if the first buy uses a stable quote  
5. Send the launch tx  
6. Index the new token and route the creator to its page  

Addresses: [Deployed Contract Addresses](./deployed-contract-addresses.md)
