# Using an LP Token or Child Token as the Dividend Token

Some dividend vaults weight rewards by LP tokens or child assets instead of the parent meme token alone.

## When to use this pattern

- You want liquidity providers rewarded from tax  
- You have a child receipt token that represents participation  
- Your product narrative is “provide liquidity / commit capital → earn tax flow”

## Design checklist

1. **Declare the asset** in launch schema and UI schema (symbol, decimals, approve target)  
2. **Document user steps** — add liquidity / stake / claim  
3. **Test fee-on-transfer interactions** if either asset charges fees  
4. **Test before and after parent graduation** — LP addresses and pool state may change meaning at listing  
5. **Protect accounting** against donations, dust, and rounding edge cases  
6. **Keep claim UX boring** — users should never need to understand splitter internals  

Official LP staking dividend factories are a concrete reference for this pattern on Cosm.
