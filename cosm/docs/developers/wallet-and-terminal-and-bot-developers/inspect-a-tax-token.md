# Inspect A Tax Token

Tax tokens need a richer detail page than standard tokens.

## Extra state to surface

- Buy tax / sell tax (bps → percent)  
- Destination breakdown (marketing / dividend / vault / other)  
- `taxSplitter` address + pending balances if useful  
- `dividend` claim entry  
- `vault` strategy type and deep link  
- After DEX: pair address and tax token pair/state views  

## Copy that helps users

Instead of raw bps only, show friendly strings:

- “5% buy tax · 5% sell tax”  
- “Tax funds a scheduled buyback vault”  
- “Holders can claim dividends from the dividend module”  

## Actions

| User goal | Action |
|-----------|--------|
| Trade | Portal swap (always) |
| Claim holder rewards | Dividend withdraw |
| Interact with strategy | Vault schema methods |
| Push pending tax | `dispatch` on splitter (advanced / ops) |

Keep advanced dispatch/ops behind an “Advanced” section in consumer apps.
