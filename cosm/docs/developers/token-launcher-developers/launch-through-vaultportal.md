# Launch Token Through VaultPortal

Use **CosmVaultPortal** when tax should power a **Cosm Tax Vault** strategy.

| Item | Value |
|------|--------|
| VaultPortal | `0xB79a2cB9c0000fDb8ABb892e65F7d49FC04EA742` |
| Recommended method | `newCosmTokenV6WithVault` |

This is **Path B**: vault instance + tax token launched together, then mapped on-chain.

---

## When to send creators here

| Creator intent | Use VaultPortal? |
|----------------|------------------|
| “Split tax to my team” | Yes — `split` |
| “Tax should buy back the token” | Yes — `scheduled-buyback` |
| “Holders burn/stake for rewards” | Yes — dividend family |
| “Just a tax to my wallet” | No — use [Portal Path A](./launch-through-portal.md) |
| “No tax at all” | No — Portal `newTokenV7` |

---

## End-to-end flow

```text
1. Creator picks a vault factory (vaultType)
2. UI loads factory launch schema
3. Creator fills strategy fields → encode vaultData
4. UI assembles token params (name, tax, quote, salt, meta, …)
5. Find vanity salt (tax suffix 0x0111)
6. simulate full VaultPortal transaction
7. broadcast
8. factory deploys vault → Portal launches tax token → mapping stored
9. Open token page + vault page
```

---

## Recommended method

`newCosmTokenV6WithVault` — Cosm’s launch layout for tax + vault.

A Flap-compatible layout may also exist for shared tooling. Product teams should **standardize on one encoding** in their UI to avoid dual form bugs.

---

## Local validation by strategy

Examples (illustrative — always re-check with simulation):

| vaultType | Local checks |
|-----------|--------------|
| `split` | 1–10 recipients; non-zero unique addresses; bps sum to `10000` |
| `scheduled-buyback` | Trigger / amount / interval parameters in allowed ranges |
| `burn-dividend` | Often empty `vaultData` |
| `lp-staking-dividend` | Often empty `vaultData` |
| `token-staking-dividend` | Often empty `vaultData` |
| `rank-burn-dividend` | Empty or optional `minBurnAmount` |

Local validation is UX. Chain rules are final.

---

## Quote constraints

Official vault templates are oriented around **BNB-quote** Path-B launches.

If your product wants “USDT market + Cosm vault”, verify that combination is supported before enabling it in the form. Do not assume every Portal quote works with every vault factory.

---

## What must succeed in simulation

A good preflight covers:

- `vaultData` decode / factory `newVault` construction  
- Tax param validity  
- Vanity / salt availability  
- Payment value / allowances  
- Portal tax launch portion of the flow  

If simulation fails, do not offer “force send.”

---

## After launch — creator success screen

Show at least:

| Item | Why |
|------|-----|
| Token address | Share / add to wallet |
| Vault address | Strategy page |
| Tax summary | Buy/sell % + promise (“funds buybacks”) |
| Trade CTA | Portal buy (creator or community) |
| Next steps | How rewards/buybacks appear over time |

If the strategy needs operators (buybacks), set expectations in plain language:

> “Buybacks run when the vault’s conditions are met.”

Avoid role jargon in consumer copy.

---

## Indexer notes

On success, persist:

- `token`  
- `vault`  
- `factory` / `vaultType`  
- tax bps  
- creation tx / time  

Terminals that index VaultPortal + factory events can deep-link vault cards immediately.

---

## Checklist

- [ ] Factory registered on VaultPortal  
- [ ] Launch schema rendered & encoded correctly  
- [ ] Tax vanity `0x0111`  
- [ ] Quote supported for this vault path  
- [ ] Full simulation green  
- [ ] Token + vault pages both ready  
