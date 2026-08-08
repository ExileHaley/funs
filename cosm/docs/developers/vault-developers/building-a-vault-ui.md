# Building a Vault UI

Vault pages turn tax from a percentage into an experience. Cosm’s schema approach lets one terminal support many strategies.

---

## Detail page recipe

```text
Resolve vault via VaultPortal
        │
        ▼
Read vaultType + vaultUISchema
        │
        ├─ view methods  → status cards / charts
        └─ write methods → wallet actions
        │
        ▼
Keep Portal trade module on the same token page
```

### Step by step

1. From the token, call VaultPortal `tryGetVault` / `getVault`  
2. If missing, do not show a vault tab (Path A token)  
3. Read `vaultUISchema`  
4. Render **view** methods as read-only cards (balances, readiness, rankings, …)  
5. Render **write** methods as CTAs (claim, burn, stake, …)  
6. Preserve method order from the schema when possible  

---

## Launch form recipe

Used on create-token (VaultPortal) flows:

1. Creator selects a factory / `vaultType`  
2. Load the factory’s launch schema fields  
3. Validate locally (bps sums, address lists, ranges)  
4. ABI-encode `vaultData`  
5. Simulate the full VaultPortal launch  
6. Submit and route to token + vault pages  

See [Launch Through VaultPortal](../token-launcher-developers/launch-through-vaultportal.md).

---

## Layout suggestions

### Mobile-first token + vault

```text
[ Trade ]
[ Vault status summary ]
[ Primary vault action ]
[ More vault methods ]
[ Tax explainer ]
```

### Desktop

```text
Trade (left)     │  Vault panel (right)
Stats / About    │  Schema methods stack
```

---

## Copy guidelines

| Do | Don’t |
|----|-------|
| “Claim rewards” | “Call withdrawDividends” |
| “Buyback ready in 02:14” | “TRIGGER_ROLE pending” |
| “Burn to increase weight” | Dump raw bps math without context |
| Show tax % near vault promise | Hide tax while advertising rewards |

---

## Strategy-specific UI hints

| vaultType | Highlight |
|-----------|-----------|
| `split` | Recipient list / claimable by user |
| `scheduled-buyback` | Countdown / readiness / last buyback |
| `burn-dividend` | Burn amount input + claim |
| `*-staking-dividend` | Stake / unstake / claim + approvals |
| `rank-burn-dividend` | Leaderboard + burn CTA |

Use schema labels first; fall back to these hints when schema text is thin.

---

## Approvals

Many write methods need ERC20 approvals (token, LP, etc.).

- Detect allowance before prompting  
- Request exact or reasonable infinite allowance per your wallet policy  
- Re-read vault view state after success  

---

## UX do’s and don’ts

**Do**

- Order components the way the schema orders methods  
- Show pending vs claimable clearly  
- Link back to the parent token  
- Empty-state the vault tab when `vault` is zero  

**Don’t**

- Hardcode one vault ABI for all strategies  
- Require users to know operator role names  
- Block trading because vault RPC failed — degrade gracefully  

---

## Related

- [Cosm Tax Vault](./cosm-tax-vault.md)  
- [Vault & VaultFactory Specification](./vault-and-vaultfactory-specification.md)  
- [Indexing Vaults](../wallet-and-terminal-and-bot-developers/indexing-vaults.md)  
