# Launch Token Through Portal

Use **CosmPortal** when the creator does not need a Cosm Tax Vault, or when launching a standard token.

| Item | Value |
|------|--------|
| Portal | `0x19a16516B187027EF778aEea4866FcFF65d5c03C` |
| Standard launch | `newTokenV7` |
| Tax launch (no Cosm vault) | `newTokenV6` |

For vault strategies, see [Launch Through VaultPortal](./launch-through-vaultportal.md).

---

## Standard tokens — `newTokenV7`

Best for clean markets without creator trading tax.

### Typical configuration

- Name / symbol  
- Meta URI (image & socials)  
- Quote: BNB or a supported stable (USDT / USDC / USD1 / USDX)  
- Optional first buy (`quoteAmt`) to bootstrap progress  
- Migrator preference (V2 / V3 / Infinity family — product default)

### Vanity

Predicted token address low 16 bits must be **`0x0222`**.

---

## Tax tokens without Cosm vault — `newTokenV6`

Best for “tax to wallet / dividend” launches.

### Typical configuration

- Buy/sell tax bps  
- Destination shares (marketing, dividend, etc.)  
- Quote + optional first buy  
- Meta / vanity (**`0x0111`**)  
- **No** vault factory selection  

On this path Cosm does **not** attach an official vault. `vault` on token state stays empty/zero.

---

## First buy (`quoteAmt`)

Creators often buy at launch to seed charts and progress.

| Quote | How payment works |
|-------|-------------------|
| BNB | `msg.value` covers the first buy |
| ERC20 quote | Approve Portal for `quoteAmt`; usually `msg.value = 0` |

### Product tips

- Simulate before broadcast if you promise a target progress %  
- Near-graduation first buys need careful headroom messaging  
- Show the creator their expected token balance after launch  

---

## Salt & vanity search

Cosm launches use CREATE2 salts so addresses are recognizable.

| Kind | Suffix |
|------|--------|
| Standard | `0x0222` |
| Tax | `0x0111` |

Implementation sketch for launchpads:

1. Read `standardTokenImpl` / `taxTokenImpl` from Portal  
2. Iterate salts until `predictDeterministicAddress` matches the suffix  
3. Ensure the predicted address has empty code  
4. Pass that salt into the launch params  

Keep this invisible in UX — creators should not type salts.

---

## Approvals & value

| Scenario | Approve | `msg.value` |
|----------|---------|-------------|
| Standard/tax, quote=BNB, with/without first buy | — | first buy (+ fees if any) |
| ERC20 quote, `quoteAmt = 0` | usually none | `0` |
| ERC20 quote, `quoteAmt > 0` | quote → Portal | `0` (typical) |

---

## Simulate → launch → share

Recommended launchpad pipeline:

```text
Validate form
   → find vanity salt
   → simulate launch
   → request wallet tx
   → wait for receipt
   → open token page (trade via Portal)
```

On failure, surface simulation errors in human language (invalid tax math, bad quote, vanity mismatch, insufficient allowance).

---

## After launch

- Deep-link trading via Portal immediately  
- Index the creation event for your feed  
- Remind creators that graduation is threshold-driven, not an admin button  
- If tax was enabled, show where tax goes (wallet / dividend)  

---

## Checklist

- [ ] Correct method: V7 standard / V6 tax  
- [ ] Vanity suffix matches token kind  
- [ ] Quote asset supported  
- [ ] First-buy payment path correct  
- [ ] Simulation before broadcast  
- [ ] Token page ready with Portal trade widget  
