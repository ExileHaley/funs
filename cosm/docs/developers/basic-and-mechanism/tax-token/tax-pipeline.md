# Tax Pipeline

Cosm breaks tax handling into composable modules so products can explain the journey clearly — from a trader’s swap to a creator’s treasury, a holder’s claim, or a vault strategy.

---

## End-to-end picture

```text
Trader swaps on CosmPortal (curve or DEX)
              │
              ▼
        Tax token rules
     (buyTaxBps / sellTaxBps)
              │
              ▼
          TaxSplitter
              │
     ┌────────┼────────────┐
     ▼        ▼            ▼
 Marketing  Dividend     Vault / other
 wallets     claims      strategies
```

Tax is assessed on eligible buys/sells. Value accrues into the pipeline, then moves to destinations through explicit distribution steps (see [Tax Liquidation Mechanism](./tax-liquidation-mechanism.md)).

---

## Modules

### TaxSplitter

The hub that receives and routes tax.

- Holds accrued value between distributions  
- `dispatch()` pushes funds to destinations by basis-point shares  
- May be called by users, terminals, or operators  

**Product tip:** show “pending / last dispatch” only in advanced or creator views; casual traders care more about tax % and net output.

### Dividend

Optional holder claim surface.

- When configured, eligible holders withdraw from the dividend module  
- Not every tax token enables dividends  
- Claim UX should be one obvious button with a clear balance  

### TaxConverter

Operational helper for batch distribution flows.

- Useful for dashboards / operators  
- Normal trading apps usually never call it  
- Do not put Converter in the primary Buy/Sell path  

### Vault

When a token launches through VaultPortal, a share of tax can fund a **Cosm Tax Vault**:

- Split, scheduled buyback, burn/staking/rank dividend strategies  
- Terminals render vault pages via `vaultUISchema`  
- See [Cosm Tax Vault](../../vault-developers/cosm-tax-vault.md)  

---

## Destination design (creator perspective)

At launch, creators allocate tax outcomes. Conceptually:

| Destination style | User promise |
|-------------------|--------------|
| Marketing / treasury wallet | Team receives a share |
| Dividend module | Holders claim over time |
| Cosm vault | Tax powers a strategy page |

Good launch UIs force creators to **name the promise** (“buybacks”, “holder rewards”, “team split”) before showing raw bps fields.

---

## Bonding vs DEX

| Phase | Tax still applies? | Trading surface |
|-------|--------------------|-----------------|
| Bonding (`Tradable`) | Yes (PreBond) | Portal curve |
| Graduated (`DEX`) | Yes | Portal DEX routing |

Graduation does not disable tax. It changes where liquidity lives, not whether tax rules exist.

---

## How apps should resolve addresses

Always read **per token** from Portal / VaultPortal:

| Field | Use |
|-------|-----|
| `taxSplitter` | Dispatch / pending tax views |
| `dividend` | Claim entry (if non-zero) |
| `vault` | Strategy page (Path B only) |
| tax bps fields | Badges & confirm copy |

Never assume a global splitter or a single vault address for all markets.

---

## Terminal information architecture

Recommended token detail tabs/sections:

1. **Trade** — Portal buy/sell  
2. **Tax** — rates + short explanation of where tax goes  
3. **Rewards** — dividend claim (if any)  
4. **Vault** — strategy UI (if Path B)  
5. **Pool** — pair/explorer after graduation  

Keep Buy/Sell dominant; tax education should not bury the trade CTA.

---

## Builder checklist

- [ ] Resolve splitter / dividend / vault per token  
- [ ] Show buy & sell tax on confirm  
- [ ] Link vault/dividend only when addresses are present  
- [ ] Don’t invent vaults from beneficiary heuristics  
- [ ] Keep Converter/ops out of consumer trade flows  
