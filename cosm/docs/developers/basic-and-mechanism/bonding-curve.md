# Bonding Curve

## What users experience

Before a Cosm token is listed on a DEX, it trades on a **bonding curve** inside CosmPortal.

- **Buy** — send quote assets into the curve reserve; circulating supply increases  
- **Sell** — return tokens to the curve; receive quote assets back  
- **Price** — rises as more supply is purchased; discovery is continuous and fully on-chain  

There is no separate “pool UI” during this phase. The market *is* the curve. Creators, traders, wallets, and bots all interact with the same Portal surface.

---

## Why a bonding curve (not a normal AMM at launch)

| Bonding curve (Cosm pre-DEX) | Typical DEX pool |
|------------------------------|------------------|
| Liquidity path is protocol-defined | Anyone can add/remove liquidity |
| Progress toward a listing milestone | No built-in “graduate” moment |
| Inventory starts in the curve | Pool must be seeded up front |
| Great for fair discovery launches | Great for mature two-sided markets |

Cosm uses the curve for **discovery**, then [migrates to DEX](./list-on-dex.md) when the milestone is reached — so early trading and later liquidity are both first-class.

---

## Supply model

Cosm tokens use a fixed maximum supply:

| Parameter | Value |
|-----------|--------|
| Max supply | **1,000,000,000** tokens (`1e9` · 18 decimals) |
| Typical graduation threshold | **800,000,000** circulating (**80%**) |
| Remaining at graduation | **~200,000,000** tokens + quote reserve → DEX liquidity |

The exact threshold is stored per token as `dexSupplyThresh`. **Prefer reading it from Portal** rather than hardcoding, even though 800M is the common default.

### Mental picture

```text
Total supply: 1,000,000,000
                 │
                 ├─ sold on curve ──────────► circulating (toward 800M)
                 │
                 └─ remaining at threshold ─► seeded as DEX liquidity (~200M)
```

---

## How pricing works (intuition)

Cosm’s curve follows a constant-product style relationship with **virtual reserves**. Conceptually:

\[
(x + h)\,(y + r) = K
\]

| Symbol | Meaning |
|--------|---------|
| \(x\) | Token inventory still in the curve |
| \(y\) | Quote reserve paid in by buyers |
| \(h\) | Virtual token reserve (curve shape) |
| \(r\) | Virtual quote reserve (curve shape) |
| \(K\) | Invariant / virtual liquidity squared |

Under the hood Cosm does **not** rely on continuous mint/burn of max supply on every trade. Tokens begin in curve inventory; buys move supply into circulation until graduation.

### What products should do

- **Execution & confirmations** — always call Portal `quoteExactInput`  
- **Charts / offline sims** — fetch immutable curve parameters once per token and cache them  
- **Do not** ship a second pricing engine that bypasses Portal fees/tax/headroom rules  

---

## Multi-quote curves

Each token is bound to **one** quote asset at launch. That choice defines the market’s denomination forever.

| Quote | How traders pay on the curve |
|-------|------------------------------|
| **BNB** | Native `msg.value` |
| **USDT / USDC / USD1 / USDX** | ERC20 `approve` + Portal swap |
| **BNB → ERC20 quote** | Native one-click when `nativeToQuoteSwap` is enabled for that quote |

### Native → quote (BNB one-click)

For ERC20-quote markets, Cosm can accept BNB, swap it into the quote asset in-protocol, then buy on the curve.

- USDT / USDC / USD1 typically route via V2-style native→quote  
- USDX may use a mixed router path  
- **Always** read the on-chain flag (e.g. `nativeToQuoteSwapEnabled`) before advertising “Buy with BNB” on a stablecoin market  

If the flag is off, show only “Buy with USDT/USDC/…” — do not silently take BNB.

---

## Progress & market cards

Most UIs show progress toward graduation:

\[
\text{progress} \approx \frac{\text{circulatingSupply}}{\text{dexSupplyThresh}}
\]

Useful card fields while bonding:

| Field | Why users care |
|-------|----------------|
| Progress % | How close to DEX listing |
| Quote chip | What asset the market is priced in |
| Tax badge | Whether buys/sells fund tax destinations |
| 24h volume / trades | Activity (from your indexer) |
| Creator / meta | Trust & narrative |

When circulating supply reaches the threshold, the token migrates — see [Migrated To DEX](./list-on-dex.md).

---

## Headroom near graduation

Large buys near 100% may be **capped** so circulating supply does not freely overshoot the DEX threshold. Portal enforces remaining headroom.

Product tips:

- Re-quote immediately before submit  
- If output is lower than a naive mid-price, explain “final curve buy may be limited by listing threshold”  
- After graduation, switch copy from “progress” to “listed” — keep the same swap buttons  

---

## Launch-time first buy

Creators can bundle a first buy (`quoteAmt`) into the launch transaction to bootstrap charts and progress.

| Quote | Payment at launch |
|-------|-------------------|
| BNB | `msg.value` |
| ERC20 quote | Approve Portal for the first-buy amount |

Simulate if your UX promises a target progress (for example “launch near listing”).

---

## Tax tokens on the curve

Tax tokens use the **same** bonding-curve trading surface. Buy/sell tax is applied in-protocol; confirm screens should show **net** output after tax. Details: [Cosm Tax Token](./tax-token/README.md).

---

## Builder checklist

- [ ] Read `quoteToken`, `status`, `circulatingSupply`, `dexSupplyThresh` from Portal  
- [ ] Quote via Portal before every swap confirmation  
- [ ] Gate “Buy with BNB” on native→quote enablement  
- [ ] Show tax on tax tokens during bonding, not only after DEX  
- [ ] Handle headroom / partial fills near graduation gracefully  
