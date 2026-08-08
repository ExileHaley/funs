# Portal vs VaultPortal

Cosm exposes two launch entries. Choosing the right one is the main product decision at create-token time. Trading after launch almost always goes through **CosmPortal** either way.

| Contract | Address |
|----------|---------|
| CosmPortal | `0x19a16516B187027EF778aEea4866FcFF65d5c03C` |
| CosmVaultPortal | `0xB79a2cB9c0000fDb8ABb892e65F7d49FC04EA742` |

---

## CosmPortal — the protocol hub

**CosmPortal** is home base for Cosm markets:

- Launch **standard** tokens (`newTokenV7`)  
- Launch **tax** tokens without attaching a Cosm Tax Vault (`newTokenV6`)  
- Quote and trade every Cosm token (curve and DEX)  
- Read token state for apps and terminals  

If the creator only needs a market — taxed or not — with simple destinations (marketing wallet, dividend module), Portal is enough.

---

## CosmVaultPortal — tax + vault in one flow

**CosmVaultPortal** is for creators who want tax revenue to power a **strategy vault**:

- Pick a registered vault factory (split, buyback, dividend styles, …)  
- Configure vault parameters (`vaultData`)  
- Launch the tax token and vault together  
- Terminals later render the vault via UI schema  

This path is for “tax is a product,” not only “tax is a wallet transfer.”

---

## Side-by-side

| | CosmPortal | CosmVaultPortal |
|--|------------|-----------------|
| Standard (non-tax) token | Yes — `newTokenV7` | No (use Portal) |
| Tax token without Cosm vault | Yes — `newTokenV6` | No |
| Tax token **with** Cosm vault | No | Yes |
| Who trades after launch | Portal | Portal (under the hood) |
| Best for | Most launchpads & simple tax | Strategy / vault products |

---

## Mental model

```text
Creator wants a simple market
        └─► CosmPortal
              ├─ newTokenV7  (standard)
              └─ newTokenV6  (tax, no Cosm vault)

Creator wants tax → vault strategy
        └─► CosmVaultPortal
              ├─ factory creates vault
              └─ Portal launches tax token
                    └─ vault linked on-chain
```

---

## Path A vs Path B (product language)

You may see “Path A / Path B” in Cosm materials:

| Name | Meaning | Entry |
|------|---------|-------|
| **Path A** | Tax (or standard) launch **without** Cosm vault attachment | Portal |
| **Path B** | Tax launch **with** Cosm vault from a registered factory | VaultPortal |

Rules of thumb for UI copy:

- **“Launch a token”** → Portal  
- **“Launch a tax token with buyback / dividend / split vault”** → VaultPortal  

---

## What gets written on-chain

| Launch style | `vault` on token state | Typical modules |
|--------------|------------------------|-----------------|
| Standard via Portal | Empty / zero | — |
| Tax Path A via Portal | Empty / zero | TaxSplitter ± Dividend |
| Tax Path B via VaultPortal | Vault instance address | TaxSplitter + Vault (± Dividend) |

Apps must **not** invent a vault because a marketing address “looks like a contract.” Only trust VaultPortal / Path-B wiring.

---

## Decision guide for launchpad PMs

1. Does the creator want a vault strategy users can open as a page?  
   - **No** → Portal  
   - **Yes** → VaultPortal + factory picker  
2. Is the token taxed?  
   - **No** → Portal `newTokenV7`  
   - **Yes** → Portal or VaultPortal per step 1  
3. After launch, where does trading live?  
   - **Always Portal** in the trading UI  

---

## Related guides

- [Launch Through Portal](../token-launcher-developers/launch-through-portal.md)  
- [Launch Through VaultPortal](../token-launcher-developers/launch-through-vaultportal.md)  
- [Cosm Tax Vault](../vault-developers/cosm-tax-vault.md)  
- [Trade Tokens](../wallet-and-terminal-and-bot-developers/trade-tokens.md)  
