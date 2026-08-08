# Brief Overview of Cosm

Cosm is an on-chain token launch protocol on **BNB Smart Chain**. It helps creators launch tokens with transparent discovery, flexible quote markets, optional trading tax, and programmable tax vaults — then graduate liquidity to DEX venues when the bonding curve completes.

> This site is Cosm’s **public documentation** for creators and ecosystem builders.  
> See [Who This Documentation Is For](./who-this-is-for.md). Full index: [`llms.txt`](./llms.txt).

---

## What Cosm offers

#### A complete launch lifecycle

Every Cosm token follows a clear on-chain journey:

```text
Launch  →  Bonding curve discovery  →  Graduation threshold  →  DEX liquidity  →  Ongoing Portal trading
```

You do not leave the Cosm surface when a token graduates. **CosmPortal** remains the trading entry before and after listing.

#### Multi-quote markets

Creators can choose the market’s quote asset at launch:

| Quote | Typical use |
|-------|-------------|
| **BNB** | Native gas-token markets |
| **USDT / USDC / USD1** | Stablecoin-denominated launches |
| **USDX** | Cosm-supported quote markets |

Where enabled, traders can enter ERC20-quote markets with **BNB one-click buy**: Portal first converts BNB into the quote asset, then buys on the curve or DEX.

#### Non-tax and tax tokens

- **Standard tokens** — no creator trading tax; clean price discovery on the curve
- **Tax tokens** — configurable buy/sell tax that can fund marketing, dividends, buybacks, or Cosm Tax Vaults

#### Cosm Tax Vaults

Tax is not only a wallet transfer. Creators can attach a **vault strategy** so tax revenue becomes an on-chain product: split to recipients, scheduled buybacks, burn-to-earn dividends, staking dividends, and competitive rank-burn designs.

Vault-aware terminals can render richer pages by reading each vault’s declarative UI schema.

---

## Why builders integrate Cosm

| Benefit | Detail |
|---------|--------|
| One trading API | `quoteExactInput` / `swapExactInput` for curve **and** DEX |
| Discoverable launches | CREATE2 vanity suffixes + creation events |
| Quote flexibility | BNB and major stables, plus native→quote |
| Composable tax | Splitter, dividend, and vault destinations |
| Schema-driven vault UX | Launch forms and detail pages without hardcoding every strategy |

---

## Network snapshot

| Item | Value |
|------|--------|
| Chain | BNB Smart Chain |
| Chain ID | `56` |
| Protocol version | `cosm-v0.8.0` |
| Primary entry | CosmPortal `0x19a16516B187027EF778aEea4866FcFF65d5c03C` |
| Vault entry | CosmVaultPortal `0xB79a2cB9c0000fDb8ABb892e65F7d49FC04EA742` |

Full address tables: [Deployed Contract Addresses](./developers/deployed-contract-addresses.md).

---

## Start here

| If you want to… | Go to |
|-----------------|--------|
| Understand Cosm in one page | This overview |
| Learn curve → DEX → tax mechanics | [Basic & Mechanism](./developers/basic-and-mechanism/README.md) |
| Index / trade Cosm tokens | [Wallet & Terminal & Bot Builders](./developers/wallet-and-terminal-and-bot-developers/README.md) |
| Launch tokens in your app | [Token Launcher Builders](./developers/token-launcher-developers/README.md) |
| Build vault strategies | [Vault Builders](./developers/vault-developers/README.md) |

---

*Cosm — launch with curve precision, graduate with liquidity, and turn tax into strategy.*
