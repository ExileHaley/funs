# Trade Tokens

All Cosm trading — bonding curve **and** graduated DEX — should go through **CosmPortal**.

| Item | Value |
|------|--------|
| Portal | `0x19a16516B187027EF778aEea4866FcFF65d5c03C` |
| Chain | BNB Smart Chain (`56`) |

One integration covers the full lifecycle. Status changes; your swap button should not.

---

## End-to-end flow

```text
Read token (quote, status, tax flags)
        │
        ▼
quoteExactInput  ──►  show net out + fees/tax
        │
        ▼
approve / permit (if ERC20 input)
        │
        ▼
swapExactInput   ──►  refresh balances
```

Always re-quote immediately before the user confirms. Launch markets move quickly.

---

## Get a quote

Call `quoteExactInput`:

| Field | Meaning |
|-------|---------|
| `inputToken` | Asset spent (`address(0)` = native BNB) |
| `outputToken` | Asset received |
| `inputAmount` | Exact input in smallest units |

### Important quoting notes

- The method may **not** be marked `view`. Call it with `eth_call` / client simulation — **do not** send a transaction just to quote  
- For bonding tokens, Portal prices the curve (including headroom near graduation)  
- For graduated tokens, Portal prices the migration pool path  
- Tax tokens: treat the quote as **net-oriented** — show tax in the confirm sheet anyway for clarity  

---

## Swap

Call `swapExactInput` (payable):

| Field | Meaning |
|-------|---------|
| `inputToken` | Same as quote |
| `outputToken` | Same as quote |
| `inputAmount` | Exact input |
| `minOutputAmount` | Slippage floor derived from your quote |
| `permitData` | Optional ERC-2612 blob; use `0x` if you `approve` |

### Slippage

Example policy (tune per product risk):

- Stable markets / larger size → tighter (0.5%–1%)  
- New bonding launches → wider (1%–5%)  
- Always compute `minOut` from a **fresh** quote, not a stale card mid-price  

---

## Buy patterns

### 1) Market quote is BNB

| Param | Value |
|-------|--------|
| `inputToken` | `0x000…000` |
| `outputToken` | meme token |
| `msg.value` | `inputAmount` |

No ERC20 approve for the input.

### 2) Market quote is USDT / USDC / USD1 / USDX

| Param | Value |
|-------|--------|
| Approve | quote token → Portal for `inputAmount` |
| `inputToken` | quote token address |
| `outputToken` | meme token |
| `msg.value` | `0` |

### 3) Buy an ERC20-quote market with BNB (one-click)

Only when native→quote is **enabled** for that quote (check token/quote config, e.g. `nativeToQuoteSwapEnabled`):

| Param | Value |
|-------|--------|
| `inputToken` | `0x000…000` |
| `outputToken` | meme token |
| `msg.value` | BNB amount to spend |

Portal swaps BNB → quote, then buys the token. If the flag is false, hide this path or the transaction will revert (`NativeToQuoteSwapNotSupported`).

---

## Sell patterns

| Param | Value |
|-------|--------|
| Approve / permit | meme token → Portal |
| `inputToken` | meme token |
| `outputToken` | quote token (`0x0` if quote is BNB) |
| `msg.value` | `0` |

Selling directly to BNB on an ERC20-quote market is not the default consumer path — prefer selling to the market’s quote asset unless you deliberately support extra routing.

### Optional permit

`permitData` can pack an EIP-2612 permit so sells skip a separate approve tx when the token supports it. If you do not implement permit, empty bytes + classic `approve` is fine.

---

## Scenario matrix

| User intent | Token status | `inputToken` | `outputToken` | Value / approve |
|-------------|--------------|--------------|---------------|-----------------|
| Buy with BNB | Bonding/DEX, quote=BNB | `0x0` | token | `msg.value` |
| Buy with USDT | Bonding/DEX, quote=USDT | USDT | token | approve USDT |
| Buy with BNB | Bonding/DEX, quote=USDT, native→quote on | `0x0` | token | `msg.value` |
| Sell for BNB | Bonding/DEX, quote=BNB | token | `0x0` | approve token |
| Sell for USDT | Bonding/DEX, quote=USDT | token | USDT | approve token |

---

## Tax tokens & fee-on-transfer

Tax tokens may deliver fewer tokens than a naive mid-price suggests.

**Do:**

- Quote through Portal  
- Show buy/sell tax % on the confirm sheet  
- Set `minOutputAmount` from the Portal quote with slippage  
- Keep post-trade “Vault / Rewards” entry points visible  

**Don’t:**

- Promise “zero fee” on a taxed market  
- Route casual users to raw DEX routers that ignore Cosm tax accounting  
- Cache quotes across long confirmation delays without refresh  

---

## After graduation

Keep the **same** methods. Optionally:

- Show pair/pool for explorers  
- Change badge from “Bonding” to “Listed”  
- Keep native→quote if still enabled  

See [Token Migration](./token-migration.md) and [Migrated To DEX](../basic-and-mechanism/list-on-dex.md).

---

## Errors worth mapping to friendly copy

| On-chain style failure | User-facing hint |
|------------------------|------------------|
| Slippage / min out | “Price moved — try again” |
| Insufficient allowance | “Approve the quote token first” |
| Insufficient value | “Not enough BNB attached” |
| Native→quote disabled | “This market doesn’t support Buy with BNB” |
| Curve headroom / threshold | “Listing threshold reached — re-quote” |
| Global pause / switch | “Trading temporarily unavailable” |

Exact revert names vary; simulate before send when UX allows.

---

## Minimal client flow (illustrative)

```ts
// 1) quote
const expected = await portal.quoteExactInput({
  inputToken: USDT,
  outputToken: token,
  inputAmount: amountIn,
});

// 2) approve quote once (ERC20 path)
await usdt.approve(PORTAL, amountIn);

// 3) swap with slippage
const minOut = (expected * 99n) / 100n; // example 1%
await portal.swapExactInput({
  inputToken: USDT,
  outputToken: token,
  inputAmount: amountIn,
  minOutputAmount: minOut,
  permitData: "0x",
});
```

```ts
// BNB buy (quote = BNB, or native→quote enabled)
await portal.swapExactInput(
  {
    inputToken: zeroAddress,
    outputToken: token,
    inputAmount: bnbIn,
    minOutputAmount: minOut,
    permitData: "0x",
  },
  { value: bnbIn },
);
```

Adapt ABI / viem / ethers details to your stack. The Portal surface is what matters.

---

## Product checklist

- [ ] One Buy/Sell module for bonding **and** DEX  
- [ ] Fresh quote on confirm  
- [ ] Correct approve target = Portal  
- [ ] Native→quote gated by on-chain flag  
- [ ] Tax disclosure on tax tokens  
- [ ] Friendly mapping for common failures  
