# Tax Liquidation Mechanism

Tax does not always appear in every recipient wallet the instant a trade confirms. Cosm uses **explicit liquidation steps** so distribution stays safe, batched, and auditable.

---

## Why liquidation is a separate step

If every swap tried to fully settle every destination, trades would be heavier, more fragile, and harder to evolve. Instead:

1. Swaps accrue tax into the pipeline  
2. Distribution runs when someone dispatches / claims / triggers  
3. Vault strategies can add their own cadence (e.g. scheduled buybacks)  

This keeps the hot path (trading) fast while still making tax economically real.

---

## Typical flow

```text
Trades accrue tax
      │
      ▼
TaxSplitter.dispatch()     ◄── user / terminal / operator
      │
      ├─► marketing / treasury wallets
      ├─► dividend module funding
      └─► vault deposits / strategy funding
            │
            ├─► holders claim dividends
            └─► vault actions (claim / burn / stake / buyback trigger)
```

---

## Steps in plain language

| Step | Who usually does it | What users see |
|------|---------------------|----------------|
| Accrual | Automatic on trade | Tax % on confirm |
| `dispatch` | Anyone motivated / operators | Destinations receive funds |
| Dividend claim | Holders | “Claim” button + balance |
| Vault user actions | Holders / community | Burn / stake / claim |
| Triggered strategies | Operators when ready | “Buyback executed” / countdown |

---

## Scheduled strategies

Some vaults (notably **scheduled buyback**) need CosmTriggerService:

- Conditions + readiness must both be true  
- An authorized executor calls `trigger(requestId)`  
- Consumer apps should show **readiness / countdown**, not internal role names  

See [Cosm Trigger Service](../../preview/trigger-service.md).

---

## What to show each audience

| Audience | Helpful UI |
|----------|------------|
| Traders | Tax rate, net output, link to vault/rewards if relevant |
| Holders | Claimable balance + claim action |
| Creators | Destination breakdown + dispatch status |
| Vault users | Strategy status, schedules, primary action |
| Operators | Ready queues, gas/commission context (advanced dashboards) |

---

## Product anti-patterns

- Implying every trade instantly pays every recipient on-chain in the same tx (unless you know it does)  
- Exposing `DISPATCHER_ROLE` language in consumer copy  
- Hiding that buybacks depend on conditions/time  
- Forcing casual traders to understand splitter internals to buy a token  

---

## Builder checklist

- [ ] Explain accrual vs claim/dispatch in one short sentence on tax pages  
- [ ] Put claim CTAs where balances are non-zero  
- [ ] For automated vaults, show next-ready state  
- [ ] Keep operator tooling separate from the main trade screen  
