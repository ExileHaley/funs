# Registered Vaults

VaultPortal only launches against **registered** factories. Below are Cosm’s official strategies on BSC (`cosm-v0.8.0`).

---

## Official catalog

| Strategy | `vaultType` | Creator pitch | Typical `vaultData` |
|----------|-------------|---------------|---------------------|
| Split | `split` | Split tax to multiple recipients by percentage | Recipient list + bps |
| Scheduled Buyback | `scheduled-buyback` | Automatically buy back on a schedule / threshold | Trigger & buyback params |
| Burn Dividend | `burn-dividend` | Burn to earn dividend weight | Often empty |
| LP Staking Dividend | `lp-staking-dividend` | Stake LP to earn dividends | Often empty |
| Token Staking Dividend | `token-staking-dividend` | Stake the token to earn dividends | Often empty |
| Rank Burn Dividend | `rank-burn-dividend` | Competitive burn leaderboard + dividends | Optional min burn |

Factory addresses: [Deployed Contract Addresses](./deployed-contract-addresses.md).

---

## How launchpads should present them

Lead with **outcomes**, not contract names:

| Card title | Subtitle |
|------------|----------|
| Team Split | Share tax with partners by % |
| Auto Buyback | Turn tax into ongoing buys |
| Burn to Earn | Reward holders who burn |
| Stake to Earn | Reward stakers (token or LP) |
| Burn Ranked | Compete on the burn board |

Then reveal advanced parameters for power users.

---

## Registration model

- Only factories registered on CosmVaultPortal can be used in Path-B launches  
- Partner factories may appear later — treat the on-chain registry as source of truth  
- If a factory is not registered, hide it from the create form even if you know the address  

---

## Pairing with tax UX

Every vault launch is still a **tax token** launch. The create form should collect:

1. Token identity (name/symbol/meta)  
2. Tax rates  
3. Vault strategy + params  
4. Quote / first buy  

Do not let creators pick a vault while setting 0% tax with no path to fund it — validate that the economic story is coherent.

---

## Related

- [Launch Through VaultPortal](./launch-through-vaultportal.md)  
- [Cosm Tax Vault](../vault-developers/cosm-tax-vault.md)  
- [Building a Vault UI](../vault-developers/building-a-vault-ui.md)  
