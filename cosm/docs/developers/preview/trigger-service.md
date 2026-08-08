# Cosm Trigger Service

CosmTriggerService is the on-chain scheduler used by strategies that need timed or conditional execution — most notably **scheduled buyback** vaults.

Address: `0x47748430d34b3575a74717a63eDB8798757D6830`

## What end users should see

Not “roles” and “callbacks” — outcomes:

- Next buyback window / readiness  
- Whether conditions are met  
- Recent buyback history if the vault exposes it  

## What operator dashboards do

1. Watch trigger request events  
2. When the request is ready **and** the vault reports ready, execute `trigger(requestId)`  
3. Ensure the executor can pay gas (vault fee design may help reimburse)

Consumer trading apps can ignore TriggerService entirely if they only buy/sell tokens. Vault products that promise automation should integrate readiness views even if a separate operator executes triggers.
