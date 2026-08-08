# Inspect A Token

Token detail pages are where Cosm products feel trustworthy. Read on-chain state first; decorate with metadata second.

---

## Primary reads

| Call | Best for |
|------|----------|
| `getToken(token)` | Detail pages — full tax, splitter, dividend, pool, vault fields |
| Safe list variants (V8/V9-style) | Feeds and scanners — compact structs + flags like native→quote |
| Token ERC20 views | `name`, `symbol`, `decimals`, `balanceOf` |
| VaultPortal `tryGetVault` / `getVault` | Confirm Path-B vault presence |

---

## Fields every product should understand

| Field | Meaning | UI use |
|-------|---------|--------|
| `status` | Bonding (`Tradable`) vs graduated (`DEX`) | Badge + layout switch |
| `quoteToken` | Payment asset | Quote chip; buy path |
| `circulatingSupply` | Sold supply | Progress / stats |
| `dexSupplyThresh` | Graduation target | Progress denominator |
| `reserve` | Quote in curve (bonding) | Advanced stats |
| `buyTaxBps` / `sellTaxBps` | Tax rates | Badges + confirm |
| `taxSplitter` | Tax hub | Advanced / creator |
| `dividend` | Claim module | Rewards tab |
| `vault` | Cosm vault (Path B) | Vault tab |
| `pool` / pair | DEX venue after listing | Explorer links |
| native→quote flag | BNB one-click eligibility | Toggle visibility |

---

## Display patterns

### Bonding card

- Progress bar to graduation  
- Quote chip (BNB / USDT / …)  
- Tax chip if taxed  
- Primary CTA: **Buy** via Portal  

### Graduated card

- “Listed” chip  
- Pair shortcut for power users  
- Same **Buy** CTA via Portal  
- Optional liquidity / mcap stats from your indexer  

### Tax card extras

- Rate summary (“5% buy · 5% sell”)  
- “Where tax goes” one-liner  
- Entry to Rewards / Vault if modules exist  

---

## Suggested detail page structure

```text
Header (name, image, status, quote, tax)
   │
   ├─ Trade module (Portal quote + swap)
   ├─ Stats (progress or listed metrics)
   ├─ Rewards (dividend)      [if configured]
   ├─ Vault (schema UI)       [if Path B]
   └─ About (meta, socials, contract links)
```

Keep Trade above the fold. Education modules scroll below.

---

## Safety & trust

- Valid vanity ≠ endorsement  
- Meta is creator-controlled — sandbox images/links  
- Show contract addresses with copy buttons  
- Warn on extreme tax if your product policy requires it  

---

## Refresh policy

| Data | Suggested refresh |
|------|-------------------|
| Quote preview | On amount change + before confirm |
| Balances | After tx + on wallet focus |
| Progress / status | On new blocks / trades |
| Meta image | Cache aggressively |

---

## Related

- [Inspect A Tax Token](./inspect-a-tax-token.md)  
- [Trade Tokens](./trade-tokens.md)  
- [Parse Token Meta](./parse-token-meta.md)  
- [Indexing Vaults](./indexing-vaults.md)  
