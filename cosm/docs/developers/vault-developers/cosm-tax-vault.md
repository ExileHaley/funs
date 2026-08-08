# Cosm Tax Vault

A **Cosm Tax Vault** is where trading tax becomes a product — not only a transfer to a wallet.

Creators attach a vault at launch (via CosmVaultPortal). As the token trades, a configured share of tax can fund the vault’s strategy. Terminals can then render a first-class vault page that holders understand.

---

## Official strategies (BSC)

| Strategy | `vaultType` | User-facing promise | Who acts |
|----------|-------------|---------------------|----------|
| **Split** | `split` | Share tax across team / partners by % | Recipients claim / receive per design |
| **Scheduled Buyback** | `scheduled-buyback` | Tax systematically buys the token back | Automated when ready (+ operator trigger) |
| **Burn Dividend** | `burn-dividend` | Burn to earn dividend weight | Holders burn / claim |
| **LP Staking Dividend** | `lp-staking-dividend` | Stake LP to earn dividends | LPs stake / claim |
| **Token Staking Dividend** | `token-staking-dividend` | Stake the token to earn dividends | Holders stake / claim |
| **Rank Burn Dividend** | `rank-burn-dividend` | Compete on a burn leaderboard | Holders burn / claim |

Factory addresses: [Deployed Contract Addresses](./deployed-contract-addresses.md).

---

## Why vaults matter for distribution

Without vaults, “tax token” is a weak differentiator. With vaults, terminals can package markets as:

- **Buyback tokens**  
- **Dividend / burn-to-earn tokens**  
- **Team-split treasury tokens**  
- **LP-rewarded tokens**  

That packaging improves discovery, retention, and creator storytelling.

---

## How tax reaches a vault

High level:

```text
Portal trade → tax rules → TaxSplitter → vault share → vault strategy
```

Exact timing depends on dispatch / deposit flows for that stack. Products should explain the **promise** (“tax funds buybacks”) and the **cadence** (“runs when conditions are met”), not the full internal call graph.

See [Tax Pipeline](../basic-and-mechanism/tax-token/tax-pipeline.md).

---

## Schema-driven experiences

Cosm vaults are designed to be **UI-schema friendly**:

| Schema | When |
|--------|------|
| Launch schema | Create-token form fields → `vaultData` |
| `vaultUISchema` | Token/vault detail page methods |

A terminal that renders schemas dynamically can support new official or partner strategies without shipping a custom screen for every factory.

Guide: [Building a Vault UI](./building-a-vault-ui.md).

---

## Design principles for vault authors

| Principle | Meaning |
|-----------|---------|
| **Readable** | A creator can explain it in one sentence |
| **Schema-friendly** | Launch + detail UIs need no bespoke ABI hardcoding |
| **Safe under tax** | Fee-on-transfer and delayed dispatch are normal |
| **Explicit automation** | If operators/triggers are required, say when actions run |
| **Stable identity** | Token↔vault addresses remain stable across upgrades |

---

## Choosing a strategy (creator guidance you can reuse)

| If the creator wants… | Suggest |
|-----------------------|---------|
| Pay multiple wallets from tax | Split |
| Support price with ongoing buys | Scheduled Buyback |
| Reward conviction via burning | Burn Dividend / Rank Burn |
| Reward liquidity providers | LP Staking Dividend |
| Reward stakers of the token | Token Staking Dividend |

---

## Related pages

- [Registered Vaults](../token-launcher-developers/registered-vaults.md)  
- [Launch Through VaultPortal](../token-launcher-developers/launch-through-vaultportal.md)  
- [Vault & VaultFactory Specification](./vault-and-vaultfactory-specification.md)  
- [Cosm Trigger Service](../preview/trigger-service.md)  
