# Migrated To DEX

## Overview

When a Cosm token reaches its bonding-curve milestone, Cosm **graduates** it:

1. Remaining curve inventory and quote reserve become DEX liquidity  
2. Token `status` moves from bonding (`Tradable`) to listed (`DEX`)  
3. Price discovery continues on the DEX pool  
4. Apps keep trading through **CosmPortal** — the button does not change  

For a typical 1B-supply token graduating at 800M circulating, about **200M tokens** plus the curve’s quote reserve seed the pool (subject to migration path and any migration fees).

---

## What changes for users

| Before graduation | After graduation |
|-------------------|------------------|
| Buy/sell against the bonding curve | Buy/sell against the DEX pool |
| Progress bar toward listing | “Listed / Graduated” state |
| No external pool yet | Pair / pool address available |
| Curve headroom rules | Normal DEX liquidity & price impact |

Graduation is a **protocol moment**, not a creator admin action. When the threshold is crossed, migration runs as part of Cosm’s lifecycle.

---

## What does **not** change for apps

Integrators should keep using CosmPortal:

- `quoteExactInput`  
- `swapExactInput`  

Graduation changes routing under the hood. Your product does **not** need a second trading stack, a different router ABI for casual users, or a “switch to PancakeSwap” forced flow.

> Power users may still want explorer links to the pair. Casual traders should stay on Portal so tax routing and protocol accounting remain consistent.

---

## Migration destinations

Depending on launch configuration, Cosm can migrate to:

| Destination | Typical fit |
|-------------|-------------|
| PancakeSwap **V2** style | Broad compatibility; natural fit for many **tax** tokens |
| PancakeSwap **V3** | Concentrated liquidity for non-tax markets when viable |
| PancakeSwap **Infinity** CL | Advanced CL path when configured |

**Tax tokens** are oriented toward V2-compatible migration because fee-on-transfer mechanics need pair-aware handling. Non-tax tokens may prefer V3 / Infinity, with fallback behavior when a path is not viable.

Exact path is chosen at launch / migrator configuration — terminals should **read** the resulting pool/pair from token state rather than assuming one venue for every market.

---

## Liquidity intuition

At the milestone, Cosm seeds DEX liquidity from:

- **Unsold inventory** still in the curve (commonly ~20% of max supply)  
- **Quote reserve** accumulated from buyers  

That is why curve progress matters: buyers are not only “aping a chart” — they are collectively funding the eventual DEX market.

---

## Tax tokens after listing

Graduation does **not** turn off tax.

- Portal-routed DEX trades continue to respect tax token rules  
- Splitter / dividend / vault funding continues  
- Pair links are useful for explorers; Portal remains the in-app swap path  

See [Cosm Tax Token](./tax-token/README.md).

---

## Detecting graduation in products

### Indexers

Watch for status transitions and migration-related events. Upsert:

| Field | Use |
|-------|-----|
| `status` | Bonding vs DEX |
| `pool` / pair | Explorer links |
| `graduatedAt` | “Listed X ago” copy |

Keep bonding history — “graduated at block / time” is useful narrative data.

### Frontends

1. When `status == DEX`, replace progress UI with listed UI  
2. Keep Buy/Sell on Portal  
3. Optionally show pair address + DEX deep link  
4. Keep tax / vault panels if the token is taxed  

---

## Edge cases worth handling

| Case | Product response |
|------|------------------|
| User’s quote was taken on curve, tx lands after migration | Re-quote; Portal routes to DEX path |
| Near-threshold buy partially fills curve headroom | Show net filled amount clearly |
| Tax token on V2 pair | Prefer Portal; warn if user tries external routers |
| Meta/image still loading | Listing state is independent of metadata |

---

## UX checklist for terminals

- [ ] Detect `status == DEX` (and/or migration events)  
- [ ] Replace “bonding progress” with “listed on DEX”  
- [ ] Keep quote → swap flow identical  
- [ ] Show pool/pair for power users  
- [ ] Keep tax / vault modules available when relevant  
- [ ] Do not call migrator contracts from end-user wallets  
