# Index Token Created Events

Discovery begins with on-chain launch events — not off-chain APIs.

## What to index

Subscribe to creation/launch events from:

- **CosmPortal** — standard launches and tax launches without Cosm vault  
- **CosmVaultPortal** — tax + vault launches  

Persist a normalized market record, for example:

| Field | Why it matters |
|-------|----------------|
| `token` | Primary key |
| `quoteToken` | Payment asset & routing |
| `isTax` / tax bps | Badges and fee copy |
| `creator` | Profile / trust signals |
| `meta` | Name image social |
| `blockNumber` / `txHash` | Ordering & dedupe |
| `vault` (optional) | Strategy markets |

## Vanity validation

Cosm expects CREATE2 vanity addresses. After indexing, verify the low 16 bits:

| Kind | Expected suffix |
|------|-----------------|
| Tax | `0x0111` |
| Standard | `0x0222` |

Reject or flag mismatches — they are not valid Cosm launches for this deployment.

## Enrichment

Events give you the skeleton. Enrich asynchronously with Portal reads:

- circulating supply / progress  
- status  
- splitter / dividend / vault  
- native→quote enabled flag  

## Reorgs & idempotency

Indexers should upsert by `token` (and vault by `token`) so chain reorgs or retries do not create duplicates in the product feed.
