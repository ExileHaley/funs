# Bonding Curve — Builders' Perspective

Think of the bonding curve as a Portal-routed market with a graduation cliff.

## Inputs and outputs

| Action | `inputToken` | `outputToken` |
|--------|--------------|---------------|
| Buy with quote | quote (or `0x0` for BNB / native→quote) | meme token |
| Sell for quote | meme token | quote (or `0x0` for BNB markets) |

## State you should track

| State | Use in UI |
|-------|-----------|
| `circulatingSupply` | Progress, mcap heuristics |
| `reserve` | Quote locked in curve |
| `dexSupplyThresh` | Graduation target |
| Curve params | Offline charts / simulations |

## Headroom

Near graduation, large buys may be partially filled up to the threshold. Always re-quote close to submission and handle reduced output gracefully in UX copy (“final curve buy may be capped at listing”).

## Do not rebuild Portal

Offline math is optional for charts. Execution should still go through Portal so fees, tax, and graduation guards stay correct.
