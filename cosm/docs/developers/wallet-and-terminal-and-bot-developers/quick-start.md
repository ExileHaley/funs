# Quick Start — Wallet / Terminal / Bot

Get a minimal Cosm-aware product live quickly.

## 1. Configure chain contracts

On BNB Smart Chain (`56`), start with:

- CosmPortal — `0x19a16516B187027EF778aEea4866FcFF65d5c03C`
- CosmVaultPortal — `0xB79a2cB9c0000fDb8ABb892e65F7d49FC04EA742` (if you show vault launches)

Full tables: [Deployed Contract Addresses](./deployed-contract-addresses.md).

## 2. Discover new tokens

Index Portal creation events. For each candidate address:

- Confirm vanity suffix (`0x0111` tax / `0x0222` standard)  
- Store quote token, creator, timestamp, meta  
- Mark tax vs standard  

## 3. Render a market card

Read Portal token state and show at least:

- Name / symbol / image (from meta)  
- Quote asset  
- Status: bonding vs graduated  
- Progress (if bonding)  
- Tax badge (if taxed)

## 4. Enable trading

```text
quoteExactInput → show net output → user confirms → swapExactInput
```

Details: [Trade Tokens](./trade-tokens.md).

## 5. Add depth over time

- Vault pages for Path-B tokens  
- Dividend claim entry  
- Migration detection and pair links  
- Native→quote “Buy with BNB” when enabled  

You now have a Cosm terminal skeleton.
