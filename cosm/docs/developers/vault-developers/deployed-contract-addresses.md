# Deployed Contract Addresses

> **Network:** BNB Smart Chain · Chain ID `56` · Protocol `cosm-v0.8.0`  
> Use these addresses in wallets, terminals, launchpads, and bots.

## Protocol entry points

| Contract | Address | What it is for |
|----------|---------|----------------|
| **CosmPortal** | `0x19a16516B187027EF778aEea4866FcFF65d5c03C` | Launch, quote, buy/sell (curve & DEX), token state |
| **CosmVaultPortal** | `0xB79a2cB9c0000fDb8ABb892e65F7d49FC04EA742` | Launch tax tokens **with** a Cosm Tax Vault |
| **CosmTriggerService** | `0x47748430d34b3575a74717a63eDB8798757D6830` | On-chain scheduling for strategies such as buybacks |
| **CosmTaxConverter** | `0xF390c921D5163D8A1eb07231518e1b8F1dB5b454` | Batch helpers for tax distribution operations |
| ProxyAdmin | `0x76b6C6DB5101265cE46c3A0377C9741E82dE633d` | Upgrade admin for protocol proxies (not an app entry) |

## Official Cosm Tax Vault factories

| Strategy | Address | `vaultType` |
|----------|---------|-------------|
| Split | `0x0CFF870676226485f85e5C9ee5334EB2Bbcc0a76` | `split` |
| Scheduled Buyback | `0x163292C2D316f6b6b6c65F7DfE152ec2D6983e97` | `scheduled-buyback` |
| Burn Dividend | `0x589D9d80Ac594b756f86DB29e0F09Fc0A7d5cc8A` | `burn-dividend` |
| LP Staking Dividend | `0xB29591DDafCbFfC810f6457AA2e534F12996BFF0` | `lp-staking-dividend` |
| Token Staking Dividend | `0x9A4C1fDd8Bb3ED03294634a0317F140147D9A88E` | `token-staking-dividend` |
| Rank Burn Dividend | `0xcD7Fe5f2B1fF33Cb83a9fe263897Ad37a2934f1e` | `rank-burn-dividend` |

## Supported quote tokens

| Symbol | Address |
|--------|---------|
| Native BNB | `0x0000000000000000000000000000000000000000` |
| USDT | `0x55d398326f99059fF775485246999027B3197955` |
| USDC | `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d` |
| USD1 | `0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d` |
| USDX | `0x95ffc15Ccfbf883B9eE2105F9F7587D6D43829C6` |

## Vanity address rules

Cosm token addresses are CREATE2 vanity addresses. The **low 16 bits** must match:

| Token kind | Suffix |
|------------|--------|
| Tax token | `0x0111` |
| Standard (non-tax) token | `0x0222` |

You can also read live values from Portal: `vanitySuffixTax()` and `vanitySuffixStandard()`.

## What not to hardcode

Per-token contracts are created at launch and must be read from chain:

- `taxSplitter`
- `dividend`
- vault instance
- DEX pair / pool after graduation

Do **not** put migrator, launcher facet, or implementation addresses into end-user app configs. Apps only need the proxies and official factories above.
