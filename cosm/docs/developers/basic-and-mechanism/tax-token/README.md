# Cosm Tax Token

A **Cosm Tax Token** is a launchable token standard where buys and/or sells contribute a configured percentage to on-chain destinations — marketing wallets, holder dividends, or Cosm Tax Vaults.

Tax is a first-class product feature, not an afterthought:

- Rates are configured at launch  
- Accrual and distribution are handled by Cosm modules  
- Vault strategies can turn tax into buybacks, splits, or engagement games  

## In this section

* [PreBond Tax](./prebond-tax.md) — tax while the token is still on the bonding curve  
* [Tax Pipeline](./tax-pipeline.md) — splitter, dividend, converter, vault  
* [Tax Liquidation Mechanism](./tax-liquidation-mechanism.md) — how accrued tax becomes user-visible value  

## Two launch styles

| Style | Entry | Vault |
|-------|-------|-------|
| Tax without Cosm vault | CosmPortal | None |
| Tax with Cosm vault | CosmVaultPortal | Official / registered factory |

Both styles still trade through CosmPortal after launch.
