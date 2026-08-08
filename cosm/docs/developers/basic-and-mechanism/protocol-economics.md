# Protocol Economics

Cosm separates fee layers so creators and apps can explain costs honestly.

## Layers

| Layer | Who it serves | Where users see it |
|-------|---------------|--------------------|
| **Curve / protocol fee** | Protocol sustainability on bonding-curve trades | Slightly less output vs a zero-fee curve |
| **Buy tax / sell tax** | Creator economy on tax tokens | Lower proceeds; funds destinations |
| **Tax destinations** | Marketing, dividends, vaults, treasury | Vault pages, claim pages, recipient wallets |
| **Migration fees** | Optional graduation economics | Applied when liquidity is seeded on DEX |
| **Operator commission** | Optional subsidy for tax/trigger operators | Usually invisible to casual traders |

## What to show in UI

For traders:

- Expected output after fees/tax  
- Tax rate badges on tax tokens  
- Clear “you pay / you receive” breakdown on confirm  

For creators:

- Protocol fee vs creator tax (do not conflate them)  
- Where tax goes (wallet vs dividend vs vault strategy)  
- Whether a vault requires ongoing operators (e.g. scheduled buyback)

## Reading live configuration

Fee profiles and bps are on-chain and can evolve by governance/admin action. Products should read Portal / token state at runtime instead of shipping frozen percentage tables in the client.
