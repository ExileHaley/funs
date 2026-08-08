# Token Migration

Migration is Cosm’s automatic graduation from bonding curve to DEX liquidity.

## For product UX

When migration completes:

1. Flip status chip to **Listed / DEX**  
2. Freeze bonding progress at 100% (or replace with listed state)  
3. Keep Buy/Sell buttons on Portal  
4. Add pair/pool explorer links  
5. Keep tax/vault modules visible when relevant  

## For indexers

Listen for migration / status-change signals and update:

- `status`  
- `pool` / pair  
- liquidity milestone timestamp  

Avoid deleting bonding history — traders like “graduated at” narratives.

## What apps should not do

- Call migrator contracts from end-user wallets  
- Assume every token migrates to the same DEX type  
- Disable Portal trading after listing  

Migration is a protocol action. Your job is to detect it and present it beautifully.
