# Token Version Specification

“Version” in Cosm mostly describes **how a token was launched**, not how it is traded.

## Launch surfaces

| Surface | Typical product meaning |
|---------|-------------------------|
| `newTokenV7` | Standard (non-tax) token via Portal |
| `newTokenV6` | Tax token via Portal (no Cosm vault) |
| VaultPortal Cosm V6 | Tax token + vault in one flow |
| Flap-compatible vault layout | Same Path-B idea, alternate ABI packing for tooling compatibility |

## What stays stable for apps

Regardless of launcher version:

- Trading goes through CosmPortal  
- Vanity rules still apply  
- Status still moves bonding → DEX  
- Tax modules are still discovered from token state  

## Practical advice

Launchpads care about versioned launch methods.  
Wallets and terminals can usually ignore launcher version after indexing, as long as they read Portal state correctly.
