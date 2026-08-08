# PreBond Tax

“PreBond” means the token is still in bonding-curve status (`Tradable`) — before DEX graduation.

Tax during this phase matters: many tokens do most of their discovery trading on the curve. If your product only explains tax after listing, users will be surprised on the first buy.

---

## What traders should know

- Buys and sells still go through CosmPortal  
- Configured `buyTaxBps` / `sellTaxBps` reduce effective output  
- Tax value accrues toward splitter / destinations over time  
- Confirm screens should quote **net** output, not a tax-blind mid price  

Example confirm copy:

> You pay 1.00 USDT → you receive ~X TOKEN after 5% buy tax.

---

## What creators should know

During the curve phase, tax can already fund:

| Destination | Outcome |
|-------------|---------|
| Marketing / treasury | Team funding from early volume |
| Dividend module | Holder claims over time |
| Cosm vault (Path B) | Strategy funding from day one |

Graduation does not “start” tax — it **continues** tax under DEX routing.

---

## Bonding vs DEX tax UX

| Phase | Show tax? | Extra UI |
|-------|-----------|----------|
| Bonding | Yes | Progress + tax badge |
| DEX | Yes | Pair link + same tax badge |

Do not hide tax chips on bonding cards to make markets look “cleaner.”

---

## Interaction with first buys

Launch-time first buys (`quoteAmt`) are still trades. If buy tax is non-zero, the creator’s received token amount reflects tax rules. Simulate and display honestly on the launch success screen.

---

## Product tips

- Show tax badges early in discovery cards  
- Explain whether tax applies on buy, sell, or both  
- If a vault is attached, link vault from the token page even during bonding  
- Near graduation, re-quote carefully — headroom + tax both affect net out  

---

## Related

- [Tax Pipeline](./tax-pipeline.md)  
- [Tax Liquidation Mechanism](./tax-liquidation-mechanism.md)  
- [Bonding Curve](../bonding-curve.md)  
