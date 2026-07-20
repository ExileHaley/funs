# Cosm 前端对接 API

> BSC 主网 · Chain ID `56` · 版本 `cosm-v0.7.0-flap-align`  
> Vanity 后缀（代币地址**低 16 bit**须匹配）：有税 `0x0111` · 无税 `0x0222`（读 `vanitySuffixTax` / `vanitySuffixStandard`）

## 合约地址（前端调用）

> **2026-07-21 主网部署** · `cosm-v0.7.0-flap-align` · 旧 2026-07-20 批次（`0xE5884D…` / `0x79cb79…` 等）已废弃。  
> 下表为前端需**直接 call** 的固定地址；impl / facet / migrator 等由协议内部使用，**勿写进前端配置**（完整清单见 `deployments/bsc-56.json`）。

### 协议入口

| 合约 | 地址 | 用途 |
|------|------|------|
| CosmPortal | `0xaC11A6ee36Ed4a7A6A0F2aEe7F54aF0c841B3234` | 发币、曲线/DEX 买卖、状态查询、迁移 |
| CosmVaultPortal | `0x39BcdA6cfF9a4807B7a4571D94DD675b5E306e60` | 有税路径 B（带金库发币） |
| CosmTriggerService | `0x22EE465D493EfF231693fC337f92b69149Dd14a0` | 链上调度器；scheduled-buyback 金库通过它预约/回调回购 |

### 发币后按 token 读取（勿写死）

| 来源 | 字段 | 前端用途 |
|------|------|----------|
| `Portal.getToken(token)` | `taxSplitter` | `dispatch()` 分配税 |
| `Portal.getToken(token)` | `dividend` | 持币分红 claim |
| `VaultPortal.getVault(token)` | `vault` | 金库页读写（`vaultUISchema` / claim 等） |
| 代币合约 `token` | — | `approve(Portal)` 卖出；金库写操作 approve |

Vanity salt 搜址：`Portal.standardTokenImpl()` / `Portal.taxTokenImpl()`（view，非固定地址）。

### 金库工厂（VaultPortal 已注册 official）

| 工厂 | 地址 | vaultType |
|------|------|-----------|
| Split | `0x88bb9377096DfADcC1cb8A58A28139E048C3CC03` | `split` |
| Scheduled Buyback | `0xA75ad6018654B2ce0227c4Ab17c093d7550cb30A` | `scheduled-buyback` |
| Burn Dividend | `0xbc7600BBEb37147BE7b052B0bdAffFcf3A77615c` | `burn-dividend` |
| LP Staking Dividend | `0x698f4d7a19dD9288Faf8de71A818a3faC7e8Ede4` | `lp-staking-dividend` |
| Token Staking Dividend | `0xD1568fF4AEe3F557771b5cCD53988560f17CDc50` | `token-staking-dividend` |
| Rank Burn Dividend | `0xcdfc393E60432a631512cF7a38c32e9D44eB4AC0` | `rank-burn-dividend` |

### Quote 白名单

发币时选定的 `quoteToken` 锁定全生命周期（曲线 / 协议费 / 税 / 迁移 LP）。

| 代币 | quoteToken | decimals | 支付 |
|------|------------|----------|------|
| BNB | `0x0000000000000000000000000000000000000000` | 18 | `msg.value` |
| USDT | `0x55d398326f99059fF775485246999027B3197955` | 18 | 先 `approve(Portal, amount)` |
| USDC | `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d` | 18 | 同上 |
| USD1 | `0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d` | 18 | 同上 |
| USDX | `0x95ffc15Ccfbf883B9eE2105F9F7587D6D43829C6` | 18 | 同上 |

| 发币路径 | 可用 quote |
|----------|-----------|
| 无税 `Portal.newTokenV7` | 上表全部 |
| 有税路径 A `Portal.newTokenV6`（无金库） | 上表全部 |
| 有税路径 B `VaultPortal.*`（带金库） | **仅 BNB** |

---

## 金库 Schema 使用指南

> **结论：** Schema **只**在 **有税 · 带 Vault（路径 B）** 时用——**发币前**读 `vaultDataSchema` 配参数，**发币后**读 `vaultUISchema` 渲染金库详情页。  
> 无税 `newTokenV7`、有税路径 A（税进钱包）**没有金库**，**不需要** Schema。

| Schema | 挂在哪 | 何时读 | 干什么 |
|--------|--------|--------|--------|
| `vaultDataSchema()` | **金库工厂**（§金库工厂地址表） | 发币前 · 用户选玩法后 | 动态渲染发币表单 → `abi.encode` → `vaultData` |
| `vaultUISchema()` | **金库实例**（`VaultPortal.getVault(token).vault`） | 发币后 · 代币详情「金库 Tab」 | 动态渲染读数 + 操作按钮（含 approve 提示） |

Schema 是链上 UI 元数据，**不参与签名**；业务校验在 `onBeforeLaunch` / `newVault` / 各写方法里。

### 什么时候不用 Schema

| 场景 | 用什么 |
|------|--------|
| 无税发币 | `Portal.newTokenV7`，无 vault |
| 有税 · 税进钱包（路径 A） | `Portal.newTokenV6` + `TaxSplitter.dispatch` + 可选 `CosmDividend` |
| 列表 / K 线 / 曲线买卖 | `Portal.getTokenV8Safe` / `quoteExactInput` / `swapExactInput` |
| scheduled-buyback 回购执行 | **keeper** 监听 `CosmTriggerRequested`，`isRequestReady(id)` 且 vault `getStatus().ready` 时调 `TriggerService.trigger(requestId)`（`TRIGGER_ROLE`） |

### 步骤 A — 发币页（`vaultDataSchema`）

路径 B 入口：`VaultPortal.newTokenV6WithVault` · quote **仅 BNB** · `mktBps > 0`。

1. **展示玩法列表** — 硬编码 §金库工厂 六个地址，或读 `VaultPortal` 工厂注册事件。
2. **用户选中工厂** → `factory.vaultDataSchema()`  
   返回 `{ description, fields[], isArray }`；按 `fields` 顺序渲染控件（`isArray=true` 时支持动态增删行，如 split 收款人）。  
   **仅表单发现，不做业务校验。**
3. **（可选）约束提示** → `factory.tokenCreationPolicies()`，在 UI 展示如「仅 BNB」等 advisory 文案；**不强制执行**。
4. **编码 + 本地校验 `vaultData`** → 用户填完表单，`abi.encode` 成 `bytes vaultData`（各工厂编码见 [§工厂地址与 vaultData](#工厂地址与-vaultdata)），并按 schema / 文档规则在本地检查（如 split：1–10 人、地址去重、bps 合计 10000）。
5. **提交前校验** — 见下节 [§路径 B 提交前校验](#路径-b-提交前校验newtokenv6withvault)；**通过后再发币**。
6. **发币** — 连同代币 meta、salt、税率等调 `newTokenV6WithVault`（建议先 `simulateContract`）：

```typescript
const schema = await factory.read.vaultDataSchema();
const vaultData = encodeVaultDataFromSchema(schema, userForm); // 类型/顺序须与链上一致

const params = {
  name, symbol, meta, salt,
  quoteToken: zeroAddress,
  quoteAmt, buyTaxBps, sellTaxBps,
  mktBps: 10_000, /* 路径 B 须 >0，营销税进金库 */
  deflationBps: 0, dividendBps: 0, lpBps: 0,
  vaultFactory: factoryAddress,
  vaultData,
  /* …minimumShareBalance, taxDuration 等见 VaultPortal 结构体 */
};

// 建议：simulateContract 通过后再 write（见 §路径 B 提交前校验）
await publicClient.simulateContract({
  address: vaultPortalAddress,
  abi: vaultPortalAbi,
  functionName: "newTokenV6WithVault",
  args: [params],
  account: userAddress,
  value: quoteAmt,
});

await vaultPortal.write.newTokenV6WithVault([params], { value: quoteAmt });
```

链上：`VaultPortal` → `factory.newVault(..., vaultData)` 创建金库实例 → `Portal.newTokenV6` 发币 → 写入 `vaults[token]`。

### 路径 B 提交前校验（`newTokenV6WithVault`）

发币前按顺序做以下检查；**第 3、4 步为必做**，第 5 步强烈建议，其余可在 UI 层提前拦截无效提交。

```
vaultDataSchema()           → 表单结构（非校验）
tokenCreationPolicies()     → UI 约束提示（可选）
onBeforeLaunch(bytes)       → 链上预检税率/分配/quote（必做）
本地 vaultData 规则          → 按玩法校验编码内容（必做）
simulateContract            → 模拟整笔发币，含 vaultData（强烈建议）
newTokenV6WithVault         → 正式提交
```

#### 各方法职责

| 方法 | 作用 | 能否代替提交前校验 |
|------|------|-------------------|
| `vaultDataSchema()` | 表单字段结构 | ❌ 仅 UI |
| `tokenCreationPolicies()` | 约束文案提示 | ❌ 仅 advisory |
| **`onBeforeLaunch(bytes)`** | **税率 / 四路分配 / quote 链上预检** | ✅ 必调 |
| 本地 `vaultData` 规则 | `vaultData` 格式与玩法业务规则 | ✅ 必做 |
| **`simulateContract(newTokenV6WithVault)`** | **含 `vaultData` 的完整模拟** | ✅ 强烈建议 |

> **`onBeforeLaunch` 不校验 `vaultData`**。`vaultData` 只在交易执行时由 `factory.newVault(..., vaultData)` decode 并校验；格式错误会导致整笔交易 revert。因此须 **本地规则 + simulateContract** 双保险。

#### 1. 工厂 / Portal 层预检

```typescript
// 工厂是否已注册且启用（未注册工厂 permissionless，但 UI 通常只展示已注册项）
const finfo = await vaultPortal.read.getVaultFactory([factoryAddress]);
if (finfo.registered && !finfo.enabled) throw new Error("factory disabled");

// 工厂权限（TIME_DEPENDENT 等）
const [policy] = await vaultPortal.read.getFactoryPolicy([factoryAddress]);
// policy: 0=OPEN · 1=TIME_DEPENDENT · 2=DISABLED

// quote 是否被该工厂支持（路径 B 应为 address(0)）
const quoteOk = await factory.read.isQuoteTokenSupported([quoteToken]);
if (!quoteOk) throw new Error("quote not supported");

// salt 低 16 bit 须为 0x0111（taxTokenImpl）
const predicted = await portal.read.predictTokenAddress([true, salt]);
if (Number(BigInt(predicted) & 0xffffn) !== 0x0111) throw new Error("invalid tax salt");
```

#### 2. `onBeforeLaunch` — 与 VaultPortal 内部一致

`CosmVaultPortal` 发币前会 staticcall 工厂 `onBeforeLaunch(abi.encode(LaunchValidationDataV1))`；前端应构造**相同 payload** 预检：

```typescript
import { encodeAbiParameters, parseAbiParameters } from "viem";

const validationData = encodeAbiParameters(
  parseAbiParameters(
    "uint8 tokenVersion, address quoteToken, uint16 buyTaxRate, uint16 sellTaxRate, uint16 vaultBps, uint16 deflationBps, uint16 dividendBps, uint16 lpBps, address dividendToken, uint256 minimumShareBalance"
  ),
  [
    6, // COSM_VERSION_TAX_V3
    quoteToken,
    buyTaxBps,
    sellTaxBps,
    mktBps, // = vaultBps
    deflationBps,
    dividendBps,
    lpBps,
    dividendToken,
    minimumShareBalance,
  ]
);

const [ok, reason] = await factory.read.onBeforeLaunch([validationData]);
if (!ok) throw new Error(reason ?? "onBeforeLaunch failed");
// 例：quote 非 BNB → "vault requires native BNB quote"
```

`LaunchValidationDataV1` 字段定义见 `ICosmVaultFactory`（与上表 `parseAbiParameters` 顺序一致）。

VaultPortal 自身还会在发币时检查：`mktBps > 0`、`mktBps + deflationBps + dividendBps + lpBps = 10000`、`buyTaxBps` 或 `sellTaxBps` 至少一侧 > 0。

#### 3. 本地 `vaultData` 规则（按玩法）

| vaultType | 本地须检查 |
|-----------|------------|
| `split` | 1–10 个 `Recipient`、地址非零且去重、`bps` 合计 10000 |
| `scheduled-buyback` | 触发/回购参数在工厂允许范围内 |
| `burn-dividend` / `lp-staking-dividend` / `token-staking-dividend` | 空 `0x` |
| `rank-burn-dividend` | 空 `0x` 或 `abi.encode(uint256 minBurnAmount)` |

完整编码见 [§工厂地址与 vaultData](#工厂地址与-vaultdata)。

#### 4. 整笔模拟（强烈建议）

模拟通过 ≈ 正式交易可通过（含 `vaultData` decode、`newVault` 构造、`Portal.newTokenV6`）：

```typescript
await publicClient.simulateContract({
  address: vaultPortalAddress,
  abi: vaultPortalAbi,
  functionName: "newTokenV6WithVault",
  args: [params],
  account: userAddress,
  value: quoteAmt,
});
```

链上实际顺序（供对照）：`predictTokenAddress` → `onBeforeLaunch` → `factory.newVault(..., vaultData)` → `Portal.newTokenV6` → 写入 `vaults[token]`。

### 步骤 B — 金库详情页（`vaultUISchema`）

1. **判断有无金库** → `VaultPortal.tryGetVault(token)`；`found === false` 则隐藏金库 Tab（路径 A / 无税币）。
2. **读操作说明** → `vault.vaultUISchema()`  
   返回 `{ vaultType, description, methods[] }`；按 `methods` **顺序**渲染（view 在前、write 在后）。
3. **填数据（推荐）** — 优先调 schema 里的聚合 view：
   - `vault.getStatus()` — 池子总览
   - `vault.getUserInfo(connectedAccount)` — 个人可领 / 质押 / 算力等
4. **写操作** — 遍历 `methods[]` 中 `isWriteMethod === true` 的项：
   - 按 `inputs[]` 渲染参数表单
   - 按 `approvals[]` 在写之前 approve（如 `taxToken` → `token.approve(vault, amount)`；LP 质押 → approve LP）
   - 调 `vault.write[method.name](args)`
5. **税入账（与 Schema 并行，所有有税币）** — 买卖产生的税先进 `TaxSplitter` 账本，须单独提供按钮：

```typescript
const { found, info } = await vaultPortal.read.tryGetVault([token]);
if (!found) return;

const vault = getContract({ address: info.vault, abi: vaultAbi });
const ui = await vault.read.vaultUISchema();

for (const m of ui.methods) {
  if (!m.isWriteMethod) {
    // 例：getStatus() / getUserInfo(user)
    await vault.read[m.name](...);
  } else {
    // 例：burn / stake / claim — 先看 m.approvals
    for (const a of m.approvals) {
      if (a.tokenType === "taxToken") await taxToken.write.approve([info.vault, amount]);
      if (a.tokenType === "lpToken") await lpToken.write.approve([info.vault, amount]);
    }
    await vault.write[m.name](args);
  }
}

// 税推进金库（路径 B：market === vault）
const splitter = (await portal.read.getToken([token])).taxSplitter;
await splitter.write.dispatch();
```

**lp-staking：** pair 在发币时与 `CosmTaxToken.mainPool` 相同（CREATE2 预测），迁移后即可 stake LP，**无需** `setPair`。

### 六个 vaultType 速览

| vaultType | 发币 `vaultData` | 详情页 UISchema 重点 | 写操作谁调 |
|-----------|------------------|----------------------|------------|
| `split` | `Recipient[]` | `claim` / `dispatch` | 用户 |
| `scheduled-buyback` | 触发/回购参数 | 只读 `getStatus` / 倒计时 | keeper → `TriggerService.trigger` |
| `burn-dividend` | 空 | `burn` / `claim` | 用户 |
| `token-staking-dividend` | 空 | `stake` / `withdraw` / `claim` | 用户 |
| `lp-staking-dividend` | 空 | 同 staking | 用户 |
| `rank-burn-dividend` | `minBurnAmount`（可空） | `burn` / `claim` + Top10 | 用户 |

完整 ABI 字段见 [§金库实例方法](#金库实例方法完整列表) · 结构体定义见 [§金库 UI Schema 类型](#金库-ui-schema-类型vaultdataschema--vaultuischema)。

---

## 常见问题 · 前端速查

> 固定入口：**发币/买卖/查询** → `Portal` · **带金库发币** → `VaultPortal` · 地址见 [§合约地址](#合约地址前端调用)。

### 1. 无税代币 — 创建入口

| 项 | 值 |
|----|-----|
| **合约** | `CosmPortal` proxy |
| **方法** | `newTokenV7(LaunchParams p) payable` |
| **关键字段** | `isTaxed = false` · `buyTaxBps/sellTaxBps = 0` · `beneficiary = 0`（V7 会自动设为发币人） |
| **salt** | 对 `Portal.standardTokenImpl()` 搜 vanity **`0x0222`** |
| **quote** | BNB / USDT / USDC / USD1 / USDX 均可 |
| **首买** | `quoteAmt > 0` 时：BNB 走 `msg.value`；ERC20 先 `approve(Portal, quoteAmt)` |
| **迁移** | 发币时 `migratorType`：`0`=PCS V3 · `3`=PCS Infinity（**V7 默认 Infinity**） |

```typescript
// 伪代码
await portal.write.newTokenV7([launchParams], { value: quoteIsBnb ? quoteAmt : 0n });
```

---

### 2. 有税 · 不带 Vault（路径 A）— 创建入口

| 项 | 值 |
|----|-----|
| **合约** | `CosmPortal` proxy |
| **方法** | `newTokenV6(LaunchParams p) payable` |
| **关键字段** | `isTaxed = true` · `beneficiary ≠ 0`（**营销税钱包**）· 买卖税率至少一侧 >0 |
| **salt** | 对 `Portal.taxTokenImpl()` 搜 vanity **`0x0111`** |
| **quote** | 五档 quote 均可 |
| **tax 分配** | `LaunchParams.tax`（`TaxAllocation`）：`mktBps/deflationBps/dividendBps/lpBps` 合计 10000 |
| **金库** | 无；`getToken(token).vault == 0` |
| **税到账** | 曲线/DEX 上**累账**；须调 `TaxSplitter.dispatch()` 才打到 mkt/分红/LP |

```typescript
await portal.write.newTokenV6([{
  ...meta, salt, quoteToken, quoteAmt,
  beneficiary: marketerWallet,
  buyTaxBps: 500, sellTaxBps: 500,
  isTaxed: true,
  tax: { mktBps: 8000, deflationBps: 0, dividendBps: 2000, lpBps: 0, ... },
  migratorType: 0, // 税币固定 V2 迁移
}], { value });
```

---

### 3. 有税 · 带 Vault（路径 B）— 创建入口

| 项 | 值 |
|----|-----|
| **合约** | `CosmVaultPortal` proxy（**不是** Portal） |
| **方法** | `newTokenV6WithVault(NewTokenV6WithVaultParams p) payable` |
| **quote** | **仅 BNB**（`quoteToken = 0`） |
| **mkt** | `mktBps` **必须 >0**（营销份额进金库，不进钱包） |
| **内部流程** | 工厂 `newVault` → `Portal.newTokenV6` → 写入 `vaults[token]` |

```typescript
await vaultPortal.write.newTokenV6WithVault([{
  name, symbol, meta, salt,
  quoteToken: zeroAddress,
  quoteAmt, buyTaxBps, sellTaxBps,
  mktBps: 10_000, deflationBps: 0, dividendBps: 0, lpBps: 0,
  minimumShareBalance, dividendMode, dividendToken, converter,
  antiFarmerDuration, taxDuration,
  vaultFactory: VAULT_FACTORY.split,
  vaultData: encodedFormData,
}], { value: quoteAmt });
```

#### 3.1 带 Vault 时，Schema 怎么用？

完整步骤见 **[§金库 Schema 使用指南](#金库-schema-使用指南)**（发币前 `vaultDataSchema` · 发币后 `vaultUISchema` · 税 `dispatch`）。

#### 3.2 Vault 地址存在哪？怎么查？

| 存储位置 | 读法 | 说明 |
|----------|------|------|
| **VaultPortal 映射（权威）** | `VaultPortal.getVault(token)` 或 `tryGetVault(token)` | 路径 B 发币后写入；`VaultInfo.vault` 即实例地址 |
| **Portal TokenState** | `Portal.getToken(token).vault` | 与上同步；路径 A 恒为 `0` |
| **TaxToken 链上** | `TaxToken(token).vault()` | 路径 B 与金库相同；`TaxSplitter.market` 也指向金库 |
| **TaxSplitter** | `taxSplitter.market()` | 营销税 dispatch 目标 |

```typescript
// 推荐：先判断有没有金库
const { found, info } = await vaultPortal.read.tryGetVault([tokenAddress]);
if (found) {
  const vault = info.vault;           // 金库实例
  const factory = info.vaultFactory;  // 创建工厂
  const isOfficial = info.isOfficial;
}

// 轻量列表页也可：
const vault = await portal.read.getVault([tokenAddress]); // 无则 0
```

---

### 4. 曲线阶段交易（`status = Tradable`）

| 项 | 值 |
|----|-----|
| **合约** | `CosmPortal` proxy |
| **估价** | `quoteExactInput(QuoteParams)` |
| **成交** | `swapExactInput(SwapParams) payable` |
| **插件币** | `extensionID ≠ 0` 时必须 `swapExactInputV3` + `extensionData` |

**QuoteParams / SwapParams**

```typescript
// 买入（BNB 计价）
{ inputToken: zeroAddress, outputToken: token, inputAmount: bnbIn }

// 卖出
{ inputToken: token, outputToken: quoteToken, inputAmount: tokenAmt,
  minOutputAmount: slippageMin, permitData: optionalPermit }
```

| quote | 买入 | 卖出 |
|-------|------|------|
| BNB | `swapExactInput{ value: amount }` | 收到 BNB 到 `msg.sender` |
| ERC20 | 先 `approve(Portal)` · `msg.value=0` | 先 `token.approve(Portal)` |

**普通币 V7 曲线费**：默认 `bondingCurveFeeBps=125`（1.25%）打给 `beneficiary`（发币人），不是全局 `feeReceiver`。

---

### 5. 迁移后交易（`status = DEX`）— 仍走 Portal

| 项 | 值 |
|----|-----|
| **入口** | **同一组** `quoteExactInput` / `swapExactInput`（或 extension 用 V3） |
| **内部** | Portal 根据 token 类型转发 PCS，**前端无需换 Router** |
| **dexId**（`getTokenV8Safe`） | `2` 税币 → PCS **V2** · `3` 普通 → **V3** · `4` 普通 → **Infinity CL** |

```
迁移前 (Tradable)     迁移后 (DEX)
─────────────────     ─────────────────────────────
Portal 曲线 AMM   →   Portal → PCS V2 / V3 / Infinity
quoteExactInput       quoteExactInput  （同一 ABI）
swapExactInput        swapExactInput
```

| 代币类型 | 迁移目标 | 用户也可直连 |
|----------|----------|--------------|
| 有税 | PCS V2 pair | PCS V2 Router（SupportingFeeOnTransfer） |
| 无税 V3 | PCS V3 pool | PCS V3 SwapRouter |
| 无税 Infinity | Infinity CL + GoPlus 锁 LP | Infinity Universal Router |

**税币 DEX 阶段**：仍可能收池子税；大额卖出触发 `liquidationThreshold` 清算。无 token 侧 `dispatchTax`，与 Flap 一致。

**可选**：高级用户迁移后可直连 PCS；站内统一走 Portal 即可覆盖报价 + 路由。

---

### 6. 其他常见入口

| 需求 | 合约 · 方法 |
|------|-------------|
| 代币详情（列表） | `Portal.getTokenV8Safe(token)` |
| 代币详情（完整） | `Portal.getToken(token)` |
| 预测地址 | `Portal.predictTokenAddress(isTaxed, salt)` |
| 手动迁移 | `Portal.migrateToDex(token)`（通常买入达阈值自动迁） |
| 持币分红领取 | `CosmDividend(dividend).withdrawDividends()` · `dividend = getToken(token).dividend` |
| 支持的 quote 列表 | `Portal.getSupportedQuoteTokens()` |
| 金库工厂列表 | `VaultPortal` 事件 / 硬编码 `VAULT_FACTORY` 常量 |
| LP 手续费（beneficiary） | `Portal.claim(token)` / `delegateClaim(token)`（Infinity 锁仓 LP） |

---

## 前端调用路由（详细索引）

| 场景 | 合约 · 方法 | 说明 |
|------|-------------|------|
| 发普通币 | `Portal.newTokenV7` | `isTaxed=false` |
| 发税收币 · 税进钱包 | `Portal.newTokenV6` | 路径 A，`beneficiary`=钱包 |
| 发税收币 · mkt 进金库 | `VaultPortal.newTokenV6WithVault` | 路径 B；内部再调 Portal |
| 列表 / 行情 | `Portal.getTokenV8Safe` | 高频轻量 |
| 详情扩展 | `Portal.getToken` | `taxSplitter` / `dividend` / `vault` / `feeProfile` |
| 曲线 / DEX 报价买卖 | `Portal.quoteExactInput` / `swapExactInput` | `Tradable`→曲线；`DEX`→Portal 转发 PCS V2/V3/**Infinity** |
| 手动迁移 | `Portal.migrateToDex` | 通常自动迁；漏迁时点 |
| 迁移后也可直连 PCS | PancakeSwap Router | 可选；站内统一走 Portal 即可 |
| 持币分红领取 | `CosmDividend.withdrawDividends` | 地址=`getToken.dividend` |
| 金库玩法 | 各 Vault 实例方法 | 地址=`VaultPortal.getVault(token).vault` |
| 金库表单自描述 | `factory.vaultDataSchema` / `vault.vaultUISchema` | 动态表单 |
| 延期回调 | `TriggerService.requestTrigger` / `trigger` | Keeper 持 `TRIGGER_ROLE` |

### 状态机

```
Invalid(0) ──发币成功──► Tradable(1) ──达阈值/手动迁移──► DEX(2)
                            │                              │
                       Portal 曲线买卖         Portal → PCS（税 V2 · 普通默认 Infinity / 可选 V3）
```

`circulatingSupply >= 800_000_000 ether` 可迁移；曲线阶段最后一笔买入可自动触发。

---

## CosmTypes（前端对接必读）

源码：`contracts/CosmTypes.sol`。ABI 编码顺序与下表字段顺序一致。  
常量：`COSM_VERSION_TAX_V3 = 6` · `COSM_VERSION_STANDARD_V3 = 7`。

### `TokenStatus` — 代币生命周期（对齐 Flap）

| 值 | 名称 | 前端含义 |
|----|------|----------|
| `0` | `Invalid` | 未发行 / 未知地址 |
| `1` | `Tradable` | 曲线阶段，买卖走 Portal 曲线 |
| `2` | `InDuel` | 占位（不用） |
| `3` | `Killed` | 占位（不用） |
| `4` | `DEX` | 已迁移，买卖仍走 Portal（内部转发 PCS V2 / V3 / Infinity） |
| `5` | `Staged` | 占位（不用） |

### `FeeProfile` — 协议费档位（`getToken.feeProfile`）

| 值 | 名称 | 曲线买卖协议费 | 迁移费 LIQUIDITY/RESERVE |
|----|------|----------------|--------------------------|
| `0` | `FEE_GLOBAL_DEFAULT` | `protocolFeeBps` / `protocolSellFeeBps`（默认各 100=1%） | `liquidityFeeBps` / `reserveFeeBps`（默认 0） |
| `1` | `FEE_FLAPSALE_V0` | 买卖各固定 `100` | `0` |
| `2` | `FEE_ZERO` | `0` | `0` |

发币默认 `0`；仅 owner 可改。列表页一般不展示；详情可从 `getToken` 读。

---

### `TaxAllocation` — 税收四路（嵌在 `LaunchParams.tax`）

**规则：** `mktBps + deflationBps + dividendBps + lpBps` 必须 = `10000`；或**全 0**（Portal 自动 `mktBps=10000`）。  
**普通币 V7：** 整结构填 0。

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `mktBps` | `uint16` | 营销份额（万分比）。路径 A → `beneficiary` 钱包；路径 B → 金库。路径 B 表单须 >0 |
| `deflationBps` | `uint16` | 回购销毁份额 → TaxSplitter 回购并烧本币 |
| `dividendBps` | `uint16` | 持币分红份额 → 部署 CosmDividend；填 `0` 则无分红合约 |
| `lpBps` | `uint16` | 加 LP 份额 → 迁移后由 TaxSplitter 加流动性 |
| `minimumShareBalance` | `uint256` | 分红最低持仓（代币最小单位）。`dividendBps>0` 时建议 ≥ `10000e18`；持仓低于此份额记 0 |
| `dividendMode` | `uint8` | 分红发放币种：`0`=quote（BNB 时为 WBNB 再 unwrap）· `1`=本税币 · `2`=其他 ERC20 |
| `dividendToken` | `address` | 仅 `dividendMode=2` 必填；mode 0/1 填 `address(0)`。mode2 发币前调 `Portal.hasDividendLiquidity(token)` |
| `converter` | `address` | 仅 `dividendMode=2` 必填；Case3 MEV 兑换员，调 `TaxSplitter.dispatch` 兑 quote→分红币。mode 0/1 填 `0` |
| `antiFarmerDuration` | `uint256` | 迁移后 anti-farmer 秒数。`0`=跳过，直接仅主池收税；上限 365 天 |
| `taxDuration` | `uint256` | 收税总时长（秒，含 anti-farmer）。`0`=永不过期；若 >0 须 ≥ `antiFarmerDuration`；上限约 100 年 |
| `mktBps2/3/4` | `uint16` | 从 `mktBps` 划出；合计 ≤ `mktBps`；默认 0 |
| `market2/3/4` | `address` | 对应收款地址；bps>0 时必填 |

```solidity
struct TaxAllocation {
    uint16 mktBps;               // 营销总额（含 mkt1–4）
    uint16 deflationBps;
    uint16 dividendBps;
    uint16 lpBps;
    uint256 minimumShareBalance;
    uint8 dividendMode;
    address dividendToken;
    uint256 antiFarmerDuration;
    uint256 taxDuration;
    uint16 mktBps2;              // 从 mktBps 划出
    uint16 mktBps3;
    uint16 mktBps4;
    address market2;
    address market3;
    address market4;
    address converter;           // Case3：mode=2 必填
}
```

---

### `LaunchParams` — `Portal.newTokenV6` / `newTokenV7` 入参

| 字段 | 类型 | 前端备注 | V7 普通币 | 路径 A（无金库） | 路径 B（VaultPortal 内部组装，勿手填调 Portal） |
|------|------|----------|-----------|------------------|--------------------------------------------------|
| `name` | `string` | ERC20 `name()` | 必填 | 必填 | 必填 |
| `symbol` | `string` | ERC20 `symbol()` | 必填 | 必填 | 必填 |
| `meta` | `string` | 元数据 URI（IPFS/HTTPS）→ `metaURI()`；图标/描述写 JSON，不单独上链 | 可空 | 可空 | 可空 |
| `salt` | `bytes32` | CREATE2 salt；预测地址**低 16 bit**须匹配 vanity，否则 `VanityMismatch` | `Portal.standardTokenImpl()` + `0x0222` | `Portal.taxTokenImpl()` + `0x0111` | 同左（tax） |
| `quoteToken` | `address` | 曲线支付币；`address(0)`=BNB。发币后锁定全生命周期 | 白名单五选一 | 白名单五选一 | **仅 BNB** |
| `quoteAmt` | `uint256` | 发币同时首买量（最小单位）。`0`=只发不买 | 按需 | 按需 | 按需 |
| `beneficiary` | `address` | 税收 mkt / V7 LP 费领取方 | 可 `0`（默认=发币人，供 `claim`） | 用户钱包 | Vault 地址（Portal 内部写入） |
| `buyTaxBps` | `uint16` | 买入税（万分比，100=1%） | `0` | 与 sell 至少一侧 >0 | 同左 |
| `sellTaxBps` | `uint16` | 卖出税（万分比） | `0` | 同上 | 同上 |
| `isTaxed` | `bool` | 须与调用方法一致 | `false` | `true` | `true` |
| `tax` | `TaxAllocation` | 见上表 | 全 0 | 四路合计 10000（或全 0） | 同左 |
| `migratorType` | `MigratorType` | **对齐 Flap**：`0`=V3 · `1`=V2 · `2`=V4(禁用) · `3`=Infinity；V7 请显式传 `3`；税币强制 `1` | `3`（Infinity）或 `0`（V3） | 忽略（强制 V2） | 忽略（强制 V2） |

**支付：**

- BNB：`msg.value >= quoteAmt`（多余会退）
- ERC20：先 `quote.approve(Portal, quoteAmt)`，`msg.value` 须为 `0`

```solidity
enum MigratorType { V3_MIGRATOR, V2_MIGRATOR, V4_UNI_MIGRATOR, PCS_INFINITY_CL_MIGRATOR } // = Flap 0/1/2/3

struct LaunchParams {
    string name;
    string symbol;
    string meta;
    bytes32 salt;
    address quoteToken;      // 0 = BNB
    uint256 quoteAmt;        // 0 = 不首买
    address beneficiary;     // V7=0；路径A=钱包
    uint16 buyTaxBps;
    uint16 sellTaxBps;
    bool isTaxed;
    TaxAllocation tax;
    MigratorType migratorType; // V7 显式 3=Infinity 或 0=V3（Flap 数值）
}
```

---

### `QuoteParams` — `Portal.quoteExactInput`（估价；曲线 / DEX）

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `inputToken` | `address` | 买入 = `quoteToken`；卖出 = meme 代币地址 |
| `outputToken` | `address` | 买入 = meme；卖出 = `quoteToken` |
| `inputAmount` | `uint256` | 输入数量（最小单位） |

返回值已扣协议费与买卖税。不改链上状态。

---

### `SwapParams` — `Portal.swapExactInput`（成交）

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `inputToken` | `address` | 同 QuoteParams |
| `outputToken` | `address` | 同 QuoteParams |
| `inputAmount` | `uint256` | 实际支付量 |
| `minOutputAmount` | `uint256` | 滑点保护；不足 revert `Slippage`。建议用 quote 结果 × (1−滑点) |
| `permitData` | `bytes` | 可选 EIP-2612；`abi.encode(deadline, v, r, s)`，value=`inputAmount`；空=走 approve |

**支付：**

- 买入 BNB：`msg.value >= inputAmount`
- 买入 ERC20：`approve(Portal, inputAmount)` 或填 `permitData`
- 卖出：`meme.approve` 或 `permitData`，`msg.value=0`
- `status` 为 `Tradable(1)` 或 `DEX(4)` 均可

```solidity
struct QuoteParams {
    address inputToken;
    address outputToken;
    uint256 inputAmount;
}

struct SwapParams {
    address inputToken;
    address outputToken;
    uint256 inputAmount;
    uint256 minOutputAmount; // 滑点下限
    bytes permitData;        // 可选 EIP-2612；空 bytes
}

struct SaltLockEntry {
    address locker;      // 0 = 未锁定
    uint8 tokenVersion;  // 6 税 / 7 普通
    bool isUsed;         // 发币后 true
}
```

---

### `TokenState` — `Portal.getToken`（详情 / 索引）

列表高频请用 `getTokenV8Safe`；需要分红入口、拆分器、金库、费档位时用本结构。

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `status` | `TokenStatus` | 对齐 Flap：`0` Invalid · `1` Tradable · `4` DEX（2/3/5 占位不用） |
| `tokenVersion` | `uint8` | `6`=CosmTaxToken · `7`=CosmToken |
| `quoteToken` | `address` | 发币锁定的 quote；BNB=`0` |
| `reserve` | `uint128` | 曲线 quote 储备；迁移后清零 |
| `circulatingSupply` | `uint128` | 曲线流通供给；≥ 该 token 的 `dexSupplyThresh` 可迁移 |
| `buyTaxBps` | `uint256` | 买入税率（万分比） |
| `sellTaxBps` | `uint256` | 卖出税率（万分比） |
| `beneficiary` | `address` | mkt 收款方（钱包或金库） |
| `progress` | `uint256` | Flap Wad：`circulatingSupply * 1e18 / dexSupplyThresh`；迁完=`1e18` |
| `tax` | `TaxAllocation` | 发币锁定的四路配置（只读展示） |
| `taxSplitter` | `address` | CosmTaxSplitter；普通币为 `0` |
| `dividend` | `address` | CosmDividend；未开分红为 `0`。领取调其 `withdrawDividends` |
| `pool` | `address` | 迁移后池；Tax=V2 pair · 普通 V3=pool · Infinity=poolId 截断；未迁=`0` |
| `vault` | `address` | 路径 B 金库；路径 A / 普通=`0`。亦可 `VaultPortal.tryGetVault` |
| `feeProfile` | `FeeProfile` | 协议费档位，见上表 |

```solidity
struct TokenState {
    TokenStatus status;
    uint8 tokenVersion;          // 6 tax / 7 standard
    address quoteToken;
    uint128 reserve;
    uint128 circulatingSupply;
    uint256 buyTaxBps;
    uint256 sellTaxBps;
    address beneficiary;
    uint256 progress;            // 0..1e18 (Wad)
    TaxAllocation tax;
    address taxSplitter;
    address dividend;            // 领分红入口
    address pool;
    address vault;               // 路径 B
    FeeProfile feeProfile;
}
```

---

### `TokenStateV8Safe` — `Portal.getTokenV8Safe`（列表 / 行情）

**不含** `taxSplitter` / `dividend` / `vault` / `feeProfile` / 完整 `tax`；需要时再调 `getToken`。

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `status` | `uint8` | 同 TokenStatus：`0/1/4`（DEX） |
| `reserve` | `uint256` | 曲线 quote 储备 |
| `circulatingSupply` | `uint256` | 流通供给 |
| `price` | `uint256` | 当前曲线单价（quote wei / 1 token）；DEX 阶段用阈值供给估算 |
| `tokenVersion` | `uint8` | `6` / `7` |
| `r` / `h` / `k` | `uint256` | 曲线虚拟参数；一般 UI 不展示，高级图表可用 |
| `dexSupplyThresh` | `uint256` | 发币 `dexThresh` 解析值；默认 `800_000_000e18`，进度条分母 |
| `quoteTokenAddress` | `address` | quote；BNB=`0` |
| `nativeToQuoteSwapEnabled` | `bool` | Flap：`true`=ERC20 quote（USDT/USDC/USD1）可用 BNB 一键买（先 native→quote）；BNB quote 为 `false`（直接 `msg.value`） |
| `extensionID` | `bytes32` | 非零时须用 `swapExactInputV3`（带 `extensionData`）；旧 `swapExactInput` revert |
| `buyTaxRate` | `uint256` | 同 `buyTaxBps` |
| `sellTaxRate` | `uint256` | 同 `sellTaxBps` |
| `pool` | `address` | 迁移后池；未迁=`0` |
| `progress` | `uint256` | Wad；`1e18`=可迁/已迁 |
| `lpFeeProfile` | `uint8` | Flap 占位；Cosm 恒 `0`，可忽略 |
| `dexId` | `uint8` | 迁移后交易入口提示：`2`=PCS V2（税币）· `3`=PCS V3 · `4`=PCS Infinity CL（普通币默认） |

```solidity
struct TokenStateV8Safe {
    uint8 status;
    uint256 reserve;
    uint256 circulatingSupply;
    uint256 price;
    uint8 tokenVersion;
    uint256 r;
    uint256 h;
    uint256 k;
    uint256 dexSupplyThresh;
    address quoteTokenAddress;
    bool nativeToQuoteSwapEnabled; // true → ERC20 quote 可用 BNB 一键买
    bytes32 extensionID;           // 插件 ID；0=无
    uint256 buyTaxRate;
    uint256 sellTaxRate;
    address pool;
    uint256 progress;
    uint8 lpFeeProfile;            // 恒 0
    uint8 dexId;                   // 2=V2 · 3=V3 · 4=Infinity
}
```

---

### VaultPortal 结构体（路径 B / 金库，非 CosmTypes 文件但前端常用）

#### `NewTokenV6WithVaultParams` — `VaultPortal.newTokenV6WithVault`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `name` / `symbol` / `meta` | `string` | 同 LaunchParams |
| `salt` | `bytes32` | 须用 `Portal.taxTokenImpl()` 搜到的 `0x0111` salt |
| `quoteToken` | `address` | **仅** `address(0)`（BNB） |
| `quoteAmt` | `uint256` | 首买；BNB 时 `msg.value`；ERC20 仍 approve **Portal** |
| `buyTaxBps` / `sellTaxBps` | `uint16` | 至少一侧 >0 |
| `mktBps` | `uint16` | **必须 >0**（mkt 进金库） |
| `deflationBps` / `dividendBps` / `lpBps` | `uint16` | 与 `mktBps` 合计 = 10000 |
| `minimumShareBalance` | `uint256` | 同 TaxAllocation |
| `dividendMode` | `uint8` | 同 TaxAllocation |
| `dividendToken` | `address` | 同 TaxAllocation |
| `converter` | `address` | 同 TaxAllocation（mode=2 必填） |
| `antiFarmerDuration` | `uint256` | 同 TaxAllocation |
| `taxDuration` | `uint256` | 同 TaxAllocation |
| `vaultFactory` | `address` | 已注册且 `enabled` 的工厂（见工厂地址表） |
| `vaultData` | `bytes` | 按该工厂 `vaultDataSchema()` 编码 |

#### `VaultInfo` — `getVault` / `tryGetVault`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `vault` | `address` | 金库实例；未绑定=`0` |
| `isOfficial` | `bool` | 发币时快照：是否官方工厂 |
| `riskLevel` | `RiskLevel` | `0` UNVERIFIED · `1` LOW · `2` MEDIUM · `3` HIGH |
| `category` | `VaultCategory` | `0` NONE · `1` SPLIT · `2` BUYBACK · `3` DIVIDEND · `4` AIRDROP（legacy） · `5` GAME |
| `vaultFactory` | `address` | 创建该金库的工厂 |

#### `VaultFactoryInfo` — `getVaultFactory`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `registered` | `bool` | 未注册不可选 |
| `enabled` | `bool` | 禁用不可发币 |
| `official` | `bool` | UI「官方」标签 |
| `riskLevel` | `RiskLevel` | 风险展示 |
| `category` | `VaultCategory` | 分类筛选 |

### 金库 UI Schema 类型（`vaultDataSchema` / `vaultUISchema`）

> 使用场景与逐步对接见 **[§金库 Schema 使用指南](#金库-schema-使用指南)**。

| 类型 | 字段（顺序固定） | 备注 |
|------|------------------|------|
| `FieldDescriptor` | `name` · `fieldType` · `description` · `decimals` | 表单控件；`decimals` 用于金额展示 |
| `VaultDataSchema` | `description` · `fields[]` · `isArray` | 发币 `vaultData` 表单；`isArray=true` 表示字段数组（如 split 收款人） |
| `ApproveAction` | `tokenType` · `amountFieldName` | 写方法前需 approve；`tokenType` 如 `taxToken`/`lpToken` |
| `VaultMethodSchema` | `name` · `description` · `inputs` · `outputs` · `approvals` · `isInputArray` · `isOutputArray` · `isWriteMethod` | 金库页按钮；`isWriteMethod=false` 为 view |
| `VaultUISchema` | `vaultType` · `description` · `methods[]` | 整页操作自描述 |
| `FactoryPolicy` | `target` · `operator` · `value` · `description` | 发币约束提示；如 `quoteToken eq address(0)` |

---

## 发币流程

### 1. 本地找 vanity salt

CREATE2 部署者 = **Portal**；低 16 bit 须匹配，否则 `VanityMismatch`。

- V7 → `await portal.read.standardTokenImpl()` + `0x0222`
- V6 / WithVault → `await portal.read.taxTokenImpl()` + `0x0111`
- 两种 salt **不通用**；每个 salt 链上只能成功用一次
- **本地 keccak 搜**；不要循环 RPC 调 `predictTokenAddress`

```typescript
import {
  type Address, type Hex, concat, getContractAddress, keccak256, pad, toHex,
} from "viem";

const PROXY_PREFIX = "0x3d602d80600a3d3981f3363d3d373d3d3d363d73" as const;
const PROXY_SUFFIX = "0x5af43d82803e903d91602b57fd5bf3" as const;

function initCodeHash(impl: Address): Hex {
  return keccak256(concat([PROXY_PREFIX, impl.toLowerCase() as Hex, PROXY_SUFFIX]));
}

export function findVanitySalt(opts: {
  portal: Address;
  impl: Address;
  vanitySuffix?: number;
  max?: number;
}): { salt: Hex; address: Address } {
  const suffix = opts.vanitySuffix ?? 0x0111;
  const max = opts.max ?? 2_000_000;
  const bytecodeHash = initCodeHash(opts.impl);
  for (let i = 0; i < max; i++) {
    const salt = pad(toHex(i), { size: 32 });
    const address = getContractAddress({
      opcode: "CREATE2", from: opts.portal, salt, bytecodeHash,
    });
    if (Number(BigInt(address) & 0xffffn) === suffix) {
      return { salt, address };
    }
  }
  throw new Error("vanity salt not found");
}
```

```bash
python3 tools/find_vanity.py --predict --portal 0xaC11A6ee36Ed4a7A6A0F2aEe7F54aF0c841B3234 --tax
python3 tools/find_vanity.py --predict --portal 0xaC11A6ee36Ed4a7A6A0F2aEe7F54aF0c841B3234
```

### 2. 支付再发币

| quote | 方式 |
|-------|------|
| BNB | `msg.value >= quoteAmt` |
| ERC20 | `approve(Portal, quoteAmt)`，`msg.value=0` |

路径 B 同样 approve **Portal**（不是 VaultPortal）。

路径 B 提交前校验见 **[§路径 B 提交前校验](#路径-b-提交前校验newtokenv6withvault)**。链上执行顺序：`predictTokenAddress` → `onBeforeLaunch` → 工厂 `newVault` → `Portal.newTokenV6` → 写入 `vaults[token]`。

---

## CosmPortal 方法

地址（proxy）：`0xaC11A6ee36Ed4a7A6A0F2aEe7F54aF0c841B3234`

### 发币（用户）

| 方法 | 返回 | 备注 |
|------|------|------|
| `newTokenV7(LaunchParams p) payable` | `address token` | 普通币；`isTaxed` 必须 false |
| `newTokenV6(LaunchParams p) payable` | `address token` | 路径 A；`beneficiary`≠0；税率至少一侧 >0 |

### 曲线交易（用户）

| 方法 | 返回 | 备注 |
|------|------|------|
| `quoteExactInput(QuoteParams)` | `uint256 outputAmount` | 曲线/DEX 估价（非 view；DEX 调 Quoter） |
| `swapExactInputV3(ExactInputV3Params) payable` | `uint256 outputAmount` | 同 swapExactInput + **`extensionData`** → 插件 `onTrade` |
| `swapExactInput(SwapParams) payable` | `uint256 outputAmount` | extension 代币不可用（须 V3） |
| `migrateToDex(address token)` | `address pool` | 流通量须已达阈值 |
| `claim(address token)` | `(tokenAmount, ethAmount)` | **仅 beneficiary**；Infinity GoPlus 收 LP 费打给调用方 |
| `delegateClaim(address token)` | `(tokenAmount, quoteAmount)` | **owner / guardian / roller** 代领至 beneficiary |

### 查询（前端高频）

| 方法 | 返回 | 备注 |
|------|------|------|
| `getTokenV8Safe(address) view` | `TokenStateV8Safe` | 列表默认 |
| `getToken(address) view` | `TokenState` | 详情 / 分红 / 金库 |
| `getVault(address) view` | `address` | 路径 B 金库；无则 `0` |
| `predictTokenAddress(bool isTaxed, bytes32 salt) view` | `address` | 复核 salt；勿暴力搜 |
| `isQuoteTokenSupported(address) pure` | `bool` | 发币表单预检 |
| `getSupportedQuoteTokens() pure` | `address[]` | 含 BNB=`0` |
| `hasDividendLiquidity(address otherToken) view` | `bool` | mode2 分红币校验 |
| `enableTaxOnBondingCurve() pure` | `bool` | 恒 `true` |
| `version() pure` | `string` | `cosm-v0.7.0-flap-align` |
| `vanitySuffixFor(bool isTaxed) view` | `uint16` | 推荐取后缀 |
| `getFeeRate() view` | `(buy, sell)` | 全局默认买卖协议费 |
| `getMigrationFeeRate() view` | `(liquidity, reserve)` | 迁移费；默认 `0,0` |
| `feeExempt(address) view` | `bool` | 交易者是否豁免协议费 |
| `bitFlags() view` | `uint256` | 熔断位图：`1`=全局 · `2`=V7 · 默认 `3` |
| `isSpammerBlocked(address) view` | `bool` | 发币黑名单 |
| `getSaltLock(bytes32) view` | `SaltLockEntry` | vanity salt 预占；`locker=0`=未锁 |
| `getLockedSaltsByUserAndVersion(user, ver, offset, limit) view` | `(salts, entries, total)` | 分页查用户已锁 salt |
| `saltLockFee() view` | `uint256` | 预占费用（BNB）；默认 `0.01 ether` |
| `getLocks(address) view` | `uint256[]` | GoPlus lockId（Infinity 或 V3；通常 1 个） |
| `infinityLockId(address) view` | `uint256` | Infinity GoPlus lockId |
| `v3LockId(address) view` | `uint256` | PCS V3 GoPlus lockId |
| `roller() view` | `address` | 可 `delegateClaim` 的运营地址 |

### Salt 预占（用户 / vanity 市场）

| 方法 | 备注 |
|------|------|
| `lockSalt(bytes32 salt, uint8 tokenVersion) payable` | `tokenVersion`=`6` 税 / `7` 普通；`msg.value == saltLockFee`；地址须已匹配 vanity。他人已锁则发币 `CosmSaltLockedByAnotherUser` |

### 只读配置

| 方法 | 含义 |
|------|------|
| `TOTAL_SUPPLY()` | `1e9 ether` |
| `DEX_SUPPLY_THRESH()` | `8e8 ether` | 默认阈值常量；单 token 实际阈值见 `getTokenV8Safe.dexSupplyThresh` |
| `defaultTaxConverter()` | `address` | Case3 发币默认 converter |
| `swapRegistry()` | `address` | Case3 PCS V2 路径注册表 |
| `BPS()` | `10000` |
| `vanitySuffixTax()` / `vanitySuffixStandard()` | `0x0111` / `0x0222` |
| `protocolFeeBps()` / `protocolSellFeeBps()` | 默认买/卖协议费 |
| `liquidityFeeBps()` / `reserveFeeBps()` | 迁移费 |
| `standardTokenImpl()` / `taxTokenImpl()` | CREATE2 模板 |
| `feeReceiver()` / `pcsRouter()` / `nonce()` | 配置 / 统计 |
| `migratorV2()` / `migratorV3()` / `migratorInfinity()` | Tax→V2 · 普通→V3/Infinity |
| `taxSplitterImpl()` / `dividendImpl()` | 模板 |
| `guardian()` / `vaultPortal()` | 熔断 guardian · 路径 B 入口 |
| `owner()` | 业务 owner（非 ProxyAdmin） |

### Owner / Guardian 运营（前端一般不调）

`setFeeReceiver` · `setProtocolFeeRates` · `setProtocolFeeBps` · `setMigrationFeeRates` · `setFeeProfile` · `setFeeExemption` · `setBitFlags` · `halt` · `haltNewTokenV7` · `setGuardian` · `setSaltLockFee` · `setVaultPortal` · `setSpammerBlockedBatch` · `registerExtension` · `setDefaultTaxConverter` · `setSwapRegistry` · `changeMarketWallet` · `setTaxWalletConfig` · `setTaxConverter` · `setTokenBeneficiary` · `excludeAddressFromDividends` · `setTokenDividendToken` · `transferOwnership`

熔断：`halt()` 清零 flags（owner/guardian）；`haltNewTokenV7()` 仅停 V7；升级后若未 init 过 CB 则视为全开，直至首次 `setBitFlags`/`halt`。

`setTaxWalletConfig(token, mkt1, mkt2, bps2, mkt3, bps3, mkt4, bps4)`：在 TaxSplitter 总 `mktBps` 内切分 mkt1–4（Flap `setWalletConfig`）；未配时仅 mkt1=`beneficiary`。

### 事件

| 事件 | 用途 |
|------|------|
| `TokenCreated(ts, creator, nonce, token, name, symbol, meta, tokenVersion)` | 刷新列表 |
| `TokenBought` / `TokenSold` | 成交 / 价格 |
| `TokenProgressChanged(token, progress)` | 进度条（Wad `0..1e18`，对齐 Flap） |
| `TokenMigrated(ts, token, pool, tokenLiquidity, quoteLiquidity, migratorKind)` | 切 PCS；`migratorKind`：2=V2 · 3=V3 · 4=Infinity |
| `MigrationFeesPaid(token, reserveFee, liquidityFee, tokensBurned)` | 迁移抽成（费率为 0 时可能不发） |
| `BitFlagsChanged(oldFlags, newFlags)` | 熔断 |
| `CosmSaltLocked(locker, salt, tokenAddress, tokenVersion, ts)` | salt 预占 |
| `SpammerBlocked(spammer, blocked)` | 发币黑名单 |
| `TokenBeneficiaryChanged` / `MarketWalletChanged` | mkt 收款方变更 |

---

## CosmVaultPortal 方法

地址（proxy）：`0x39BcdA6cfF9a4807B7a4571D94DD675b5E306e60`

| 方法 | 返回 | 调用方 | 备注 |
|------|------|--------|------|
| `newTokenV6WithVault(NewTokenV6WithVaultParams) payable` | `token` | 用户 | 路径 B；`MAGIC_DIVIDEND_COMPUTED` 需 v2.3+ 工厂 |
| `newTokenV7WithVault(NewTokenV7WithVaultParams) payable` | `token` | 用户 | Flap V7 + feeConfigs + vault · extension · dexThresh |
| `getVault(address taxToken) view` | `VaultInfo` | 前端 | 已知路径 B |
| `tryGetVault(address taxToken) view` | `(found, VaultInfo)` | 前端 | 不确定时用 |
| `getVaultFactory(address factory) view` | `VaultFactoryInfo` | 前端 | 选工厂标签 |
| `getFactoryPolicy(factory) view` | `(policy, policyData, description)` | 前端 | `0=OPEN · 1=TIME_DEPENDENT · 2=DISABLED` |
| `getVaultCategory` / `getFactoryCategory` | `VaultCategory` | 前端 | 展示标签 |
| `portal() view` | `CosmPortal` | 前端 | 关联 Portal |
| `registerVaultFactory(...)` | — | **owner** | 部署注册；默认 policy=OPEN |
| `disableFactory` / `setTimeDependentPolicy` / `resetFactoryPolicy` | — | **owner** | 工厂权限 |
| `setVaultCategory` / `setFactoryCategory` | — | **owner** | 重打标签 |
| `refreshTokenVault(token)` | — | 任意 | 同步缓存 vault ← TaxSplitter.market |

事件：`CosmTaxVaultTokenCreated` · `VaultFactoryRegistered` · `FactoryPermissionPolicySet` · `VaultCategoryUpdated` · `FactoryCategoryUpdated` · `TokenVaultRefreshed`

---

## 金库工厂（Vault Factory）

> Schema 发币/详情步骤见 **[§金库 Schema 使用指南](#金库-schema-使用指南)**。

前端用工厂做**表单发现**（`vaultDataSchema`）；`newVault` **仅** VaultPortal 内部调用。

| 方法 | 返回 | 前端用途 |
|------|------|----------|
| `vaultType() pure` | `string` | 玩法名 |
| `vaultDataSchema() pure` | `VaultDataSchema` | 动态生成发币表单并 `abi.encode` |
| `isQuoteTokenSupported(address) view` | `bool` | 全部工厂仅 BNB=`address(0)` |
| `factorySpecVersion() pure` | `string` | 当前 `"v2.2"` |
| `tokenCreationPolicies() pure` | `FactoryPolicy[]` | UI 约束提示（如 quote=BNB）；**不强制执行** |
| `onBeforeLaunch(bytes) view` | `(bool, string)` | 发币前链上预检（税率/分配/quote）；前端**必调**，见 [§路径 B 提交前校验](#路径-b-提交前校验newtokenv6withvault) |
| `newVault(...)` | `address` | **勿直接调** |

### 工厂地址与 vaultData

| vaultType | 工厂地址 | vaultData 编码 |
|-----------|----------|----------------|
| `split` | `0x88bb9377096DfADcC1cb8A58A28139E048C3CC03` | `abi.encode(Recipient[])` · `Recipient{address recipient; uint16 bps}` · 1–10 人去重 · bps 合计 10000 |
| `scheduled-buyback` | `0xA75ad6018654B2ce0227c4Ab17c093d7550cb30A` | `abi.encode(triggerMode, buybackMode, intervalSeconds, minBnbAmount, maxBnbPerTrigger[, firstExecutableAt])` · trigger:`0=time 1=amount+interval 2=both` · buyback:`0=token 1=LP` · `firstExecutableAt` 可选 unix 秒 |
| `burn-dividend` | `0xbc7600BBEb37147BE7b052B0bdAffFcf3A77615c` | 空 `0x` |
| `lp-staking-dividend` | `0x698f4d7a19dD9288Faf8de71A818a3faC7e8Ede4` | 空 `0x`（pair 发币时 CREATE2 预测，同 `token.pair()`） |
| `token-staking-dividend` | `0xD1568fF4AEe3F557771b5cCD53988560f17CDc50` | 空 `0x` |
| `rank-burn-dividend` | `0xcdfc393E60432a631512cF7a38c32e9D44eB4AC0` | `abi.encode(uint256 minBurnAmount)`；也可空=`0` |

---

## 金库实例方法（完整列表）

地址：`VaultPortal.getVault(token).vault`（先 `tryGetVault` 确认 `found`）。  
收税均为被动入账（`receive`），无需主动调用。  
前端聚合首选：`getStatus()`（池子）+ `getUserInfo(user)`（个人）；写操作见各表「操作」小节。

### 通用（全部 vaultType）

| 方法 | 类型 | 返回 | 备注 |
|------|------|------|------|
| `factory()` | view | `address` | 创建工厂 |
| `taxToken()` | view | `address` | 关联税币 |
| `creator()` | view | `address` | 发币人 |
| `quoteToken()` | view | `address` | 恒 BNB=`0` |
| `vaultType()` | view | `string` | 如 `"split"` |
| `description()` | view | `string` | 动态文案 |
| `vaultUISchema()` | view | `VaultUISchema` | 渲染操作区 |

---

### `split` — 分配税收（账本领取）

税 BNB 入账只按 bps **记账**，不自动打款。用户调 `claim`；`dispatch` 可选（代全员推送）。

**UISchema：** `dispatch` · `claim`

#### 只读

| 方法 | 返回 | 备注 |
|------|------|------|
| `getStatus()` | `SplitStatus` | **首选** · 见下表 |
| `getUserInfo(address user)` | `UserInfo` | **首选** · 含 claimable / isRecipient |
| `claimable(address user)` | `uint256` | 含未记账余额近似 |
| `getRecipientsInfo()` | `RecipientInfo[]` | 全员账本 |
| `recipients(i)` | `(address, uint16 bps)` | 列表项 |
| `userBalances(user)` | `(uint128 accumulated, uint128 claimed)` | 原始账本 |
| `recipientCount()` / `TOTAL_BPS()` | … | 人数 / `10000` |
| `totalDistributed()` / `totalClaimed()` | `uint256` | 累计入账 / 已领 |

`SplitStatus`：`vaultBalance` · `totalDistributed` · `totalClaimed` · `totalClaimable` · `uncredited` · `recipientCount` · `recipients[]`  
`RecipientInfo`：`recipient` · `bps` · `accumulated` · `claimed` · `claimable`  
`UserInfo`：`bps` · `accumulated` · `claimed` · `claimable` · `isRecipient`

#### 操作

| 方法 | 调用方 | 前置 | 备注 |
|------|--------|------|------|
| `claim(address user)` | 用户 / 代领 | 无 | 领取 `accumulated−claimed`；内部先 `accrue` |
| `dispatch()` | 任何人 / keeper | 无 | 可选：代全部收款人推送 |
| `accrue()` | 任何人 | 无 | 把直转未记账余额记入账本 |

事件：`CosmSplitVaultDistributed` · `CosmSplitVaultClaimed` · `CosmSplitVaultDispatched`

---

### `scheduled-buyback` — 定时回购销毁

**UISchema：** `canTrigger` · `getStatus` · `countdownSeconds`  
**Keeper（对齐 Flap）：** 金库在 `receive()` / 回调末尾自动 `requestTrigger`；服务端持 **`CosmTriggerService.TRIGGER_ROLE`**，监听 `CosmTriggerRequested`，**`isRequestReady(id)` 且 `vault.getStatus().ready===true`** 时调 `TriggerService.trigger(requestId)` → 金库 `trigger(requestId)` 执行回购。  
**注意：** 无 `OPERATOR_ROLE` / 无直调 `vault.trigger()`；每次预约消耗 `getFee()` BNB（默认 **0.0002 BNB**），从金库余额扣除。  
**FAILED 回调：** 监听 `CosmTriggerExecuted(success=false)` → 任何人可调 `retryTrigger(requestId)`（OOG 等）。  
手动充值：转 BNB 到 vault（`Deposited`，并尝试预约下一轮）。

#### 只读

| 方法 | 返回 | 备注 |
|------|------|------|
| `getStatus()` | `BuybackStatus` | **首选** · 一次读齐页面 |
| `canTrigger()` | `bool` | 回购条件是否满足（不含 Trigger 预约费） |
| `countdownSeconds()` | `uint256` | 距下次时间窗；已到点为 0 |
| `nextTriggerAt()` / `nextSpendBnb()` | `uint256` | 下次时间 / 下次花费 |
| `triggerMode()` / `buybackMode()` | `uint8` | 0–2 / 0–1 |
| `intervalSeconds()` / `minBnbAmount()` / `maxBnbPerTrigger()` | `uint256` | 配置 |
| `firstExecutableAt()` / `lastTriggeredAt()` | `uint256` | 首次可执行 / 上次触发 |
| `totalBurned()` / `totalBnbSpent()` / `triggerCount()` | `uint256` | 累计销毁 / 花费 / 次数 |
| `pendingRequestId()` | `uint256` | 当前 TriggerService 预约 ID；0=无 |
| `triggerService()` / `router()` | `address` | 调度器 / PCS V2 Router |

`BuybackStatus`：`triggerMode` · `buybackMode` · `intervalSeconds` · `minBnbAmount` · `maxBnbPerTrigger`(0=无上限) · `lastTriggeredAt` · `nextTriggerAt` · `countdownSeconds` · `vaultBnb` · `nextSpendBnb` · `totalTokensBurned` · `totalBnbSpent` · `triggerCount` · `pendingRequestId` · `ready` · `buybackModeLabel` · `triggerModeLabel` · `executionPathLabel`

#### 操作

| 方法 | 调用方 | 前置 | 备注 |
|------|--------|------|------|
| `trigger(uint256 requestId)` | **`CosmTriggerService` 回调** | `canTrigger()==true` 时执行回购 | 实现 `ITriggerReceiver` |
| `receive()` / 转 BNB | 用户 / 税分 | — | 入账并 `_tryScheduleTrigger` |

事件：`Deposited` · `TriggerScheduled` · `ScheduledBuyback`

---

### `burn-dividend` — 燃烧分红

燃烧本币进黑洞累加算力；税 BNB 按 `accRewardPerShare` 分红。**claim 后算力不清零**。

**UISchema：** `burn` · `claim`（burn 前 `approve(vault, amount)` taxToken）

#### 只读

| 方法 | 返回 | 备注 |
|------|------|------|
| `getStatus()` | `BurnStatus` | **首选** |
| `getUserInfo(address user)` | `UserInfo` | **首选** · `pending`/`shareBps` |
| `pendingReward(address user)` | `uint256` | 待领 BNB |
| `totalBurned()` / `accRewardPerShare()` / `pendingBnb()` | … | 池子 |
| `totalRewardsIn()` / `totalClaimed()` / `participantCount()` | … | 统计 |
| `burned(user)` / `rewardDebt(user)` / `claimed(user)` | `uint256` | 个人 |

`BurnStatus`：`vaultBnb` · `totalBurned` · `pendingBnb` · `accRewardPerShare` · `totalRewardsIn` · `totalClaimed` · `participantCount`  
`UserInfo`：`burned` · `rewardDebt` · `pending` · `claimed` · `shareBps`

#### 操作

| 方法 | 调用方 | 前置 | 备注 |
|------|--------|------|------|
| `burn(uint256 amount)` | **用户** | `approve(vault, amount)` | 转死地址；累加算力 |
| `claim()` | **用户** | — | → `uint256` 领取 BNB；算力保留 |

事件：`Burned` · `Claimed`

---

### `token-staking-dividend` — 本币质押分红

**UISchema：** `stake` · `withdraw` · `claim`（stake 前 `approve(vault, amount)` taxToken）

#### 只读

| 方法 | 返回 | 备注 |
|------|------|------|
| `getStatus()` | `StakeStatus` | **首选** |
| `getUserInfo(address user)` | `UserInfo` | **首选** |
| `pendingReward(address user)` | `uint256` | 待领 BNB |
| `totalStaked()` / `accRewardPerShare()` / `pendingBnb()` | … | 池子 |
| `totalRewardsIn()` / `totalClaimed()` / `participantCount()` | … | 统计 |
| `staked(user)` / `rewardDebt(user)` / `claimed(user)` | `uint256` | 个人 |

`StakeStatus`：`vaultBnb` · `totalStaked` · `pendingBnb` · `accRewardPerShare` · `totalRewardsIn` · `totalClaimed` · `participantCount`  
`UserInfo`：`staked` · `rewardDebt` · `pending` · `claimed` · `shareBps`

#### 操作

| 方法 | 调用方 | 前置 | 备注 |
|------|--------|------|------|
| `stake(uint256 amount)` | **用户** | `approve(vault, amount)` | 质押本币 |
| `withdraw(uint256 amount)` | **用户** | — | 解押；超额则截断 |
| `claim()` | **用户** | — | → `uint256` 领 BNB |

事件：`Staked` · `Withdrawn` · `Claimed`

---

### `lp-staking-dividend` — LP 质押分红

发币时 factory 已用 CREATE2 预测 PCS V2 pair（与 `TaxToken(token).pair()` / `mainPool` 一致）。**迁移 DEX 后即可** stake LP，无需额外绑 pair。

**UISchema：** `stake` · `withdraw` · `claim`（stake 前 `approve(vault, amount)` LP）

#### 只读

| 方法 | 返回 | 备注 |
|------|------|------|
| `getStatus()` | `StakeStatus` | **首选** · 含 `pair` |
| `getUserInfo(address user)` | `UserInfo` | **首选** |
| `pendingReward(address user)` | `uint256` | 待领 BNB |
| `pair()` | `address` | 发币时预测写入（= `token.pair()`） |
| `totalStaked()` / `accRewardPerShare()` / `pendingBnb()` | … | 池子 |
| `totalRewardsIn()` / `totalClaimed()` / `participantCount()` | … | 统计 |
| `staked(user)` / `rewardDebt(user)` / `claimed(user)` | `uint256` | 个人 |

`StakeStatus`：`pair` · `vaultBnb` · `totalStaked` · `pendingBnb` · `accRewardPerShare` · `totalRewardsIn` · `totalClaimed` · `participantCount`  
`UserInfo`：同 token-staking

#### 操作

| 方法 | 调用方 | 前置 | 备注 |
|------|--------|------|------|
| `stake(uint256 amount)` | **用户** | DEX 已迁移 · `approve LP` | 质押 LP |
| `withdraw(uint256 amount)` | **用户** | — | 解押 LP |
| `claim()` | **用户** | — | → `uint256` 领 BNB |

事件：`Staked` · `Withdrawn` · `Claimed`

---

### `rank-burn-dividend` — 黑洞排行榜燃烧分红

80% 权重池（同 burn-dividend）+ 20% 在每次 `burn` 前按当前 Top10 算力记入 `rankCredit`。**claim 清零 credit，不清零 burned**。

**UISchema：** `burn` · `claim`（burn 前 `approve(vault, amount)` taxToken）

#### 只读

| 方法 | 返回 | 备注 |
|------|------|------|
| `getStatus()` | `RankStatus` | **首选** · 含 Top10 |
| `getUserInfo(address user)` | `UserInfo` | **首选** · `rank` 1–10 / 0=未上榜 |
| `pendingReward(address user)` | `uint256` | 权重待领 + rankCredit |
| `pendingWeight(user)` / `pendingRank(user)` | `uint256` | 分拆 |
| `topBurnersList()` | `(address[10], uint256[10])` | 完整榜 |
| `topBurners(i)` / `topBurnedAmounts(i)` | … | 单项 |
| `WEIGHT_POOL_BPS()` / `RANK_POOL_BPS()` / `BPS()` / `TOP_N()` | 常量 | 8000 / 2000 / 10000 / 10 |
| `minBurnAmount()` | `uint256` | 最小燃烧 |
| `totalBurned()` / `accRewardPerShare()` / `pendingWeightBnb()` / `rankPendingBnb()` | … | 池子 |
| `totalRewardsIn()` / `totalWeightClaimed()` / `totalRankDistributed()` / `totalRankClaimed()` / `participantCount()` | … | 统计 |
| `burned` / `rewardDebt` / `rankCredit` / `weightClaimed` / `rankClaimed` | mapping | 个人 |

`RankStatus`：`vaultBnb` · `totalBurned` · `pendingWeightBnb` · `rankPendingBnb` · `accRewardPerShare` · `totalRewardsIn` · `totalWeightClaimed` · `totalRankDistributed` · `totalRankClaimed` · `participantCount` · `minBurnAmount` · `topBurners[10]` · `topBurnedAmounts[10]`  
`UserInfo`：`burned` · `rewardDebt` · `weightPending` · `rankCredit` · `weightClaimed` · `rankClaimed` · `pendingTotal` · `shareBps` · `rank`

#### 操作

| 方法 | 调用方 | 前置 | 备注 |
|------|--------|------|------|
| `burn(uint256 amount)` | **用户** | `amount ≥ minBurnAmount` · approve | 先分 20% 给当前 Top10，再累加算力 |
| `claim()` | **用户** | — | → `uint256`；领权重+rankCredit；算力保留 |

事件：`Burned` · `RankPoolDistributed` · `Claimed(user, weightAmt, rankAmt)`

---

### vaultType → UISchema / 前端首选速查

| vaultType | UISchema methods | 只读首选 | 写操作 |
|-----------|------------------|----------|--------|
| `split` | `dispatch`, `claim` | `getStatus` · `getUserInfo` | `claim(user)` · `dispatch` |
| `scheduled-buyback` | `canTrigger`, `getStatus`, `countdownSeconds` | `getStatus` | keeper → `TriggerService.trigger` |
| `burn-dividend` | `burn`, `claim` | `getStatus` · `getUserInfo` | `burn` · `claim` |
| `token-staking-dividend` | `stake`, `withdraw`, `claim` | `getStatus` · `getUserInfo` | `stake` · `withdraw` · `claim` |
| `lp-staking-dividend` | `stake`, `withdraw`, `claim` | `getStatus` · `getUserInfo` | `stake` · `withdraw` · `claim` |
| `rank-burn-dividend` | `burn`, `claim` | `getStatus` · `getUserInfo` · `topBurnersList` | `burn` · `claim` |

---

## CosmDividend 方法

地址：`getToken(token).dividend`（仅 `dividendBps>0` 时非零）。

| 方法 | 返回 | 调用方 | 备注 |
|------|------|--------|------|
| `withdrawableDividends(user) view` | `uint256` | 前端 | 待领 |
| `withdrawableDividendOf(user) view` | `uint256` | 前端 | 同上 |
| `accumulativeDividendOf(user) view` | `uint256` | 前端 | 累计 |
| `withdrawDividends()` | `bool` | **用户** | 领给自己；WBNB 会 unwrap→BNB |
| `withdrawDividendsFor(address user)` | `bool` | 任何人 | 代领并 unwrap |
| `withdrawDividendsFor(address user, bool unwrapWETH)` | `bool` | 任何人 | 控制是否 unwrap |
| `userInfo(user)` | `(share, rewardDebt, pendingBalance)` | 前端 | — |
| `withdrawnDividends(user)` / `totalShares()` / `minimumShareBalance()` | … | 前端 | — |
| `dividendToken()` / `taxToken()` / `totalDividendsDistributed()` | … | 前端 | — |
| `excludedFromDividends(user)` | `bool` | 前端 | — |

非前端：`deposit`（TaxSplitter）· `setShare`（仅税币）· `distributeDividend`。

---

## CosmTriggerService 方法

地址（proxy）：`0x22EE465D493EfF231693fC337f92b69149Dd14a0`  
默认：`getFee()=0.0002 ether` · `getMaxCallbackGas()=2_000_000` · 以链上 `getFee()` / `getMaxCallbackGas()` 为准。

scheduled-buyback 金库实现 `ITriggerReceiver.trigger(uint256 requestId)`；keeper 执行 `CosmTriggerService.trigger(requestId)`。

| 方法 | 调用方 | 备注 |
|------|--------|------|
| `requestTrigger(uint64 executeAfter) payable → requestId` | 请求方合约 | `msg.value ≥ getFee()`；`executeAfter=0` 立即可执行 |
| `trigger(requestId)` / `triggerMultiple(ids)` | **keeper（`TRIGGER_ROLE`）** | 回调 requester |
| `retryTrigger(requestId)` | 任何人 | 仅 `FAILED` |
| `getFee` / `getMaxCallbackGas` / `getRequest` / `getRequestCount` / `isRequestReady` | 前端/服务端 | 查询 |
| `getRequests` / `getRequestsPaginated` / `getRequestsByRequesterPaginated` | 服务端 | 分页 |
| `setRequiredGasFee` / `setMaxCallbackGas` | admin | 运营 |

`TriggerRequest`：`requester` · `executeAfter` · `status`(PENDING/EXECUTED/FAILED) · `feePaid`。

### Keeper 服务端（scheduled-buyback）

```typescript
// 1. 订阅 CosmTriggerService CosmTriggerRequested
// 2. 过滤 requester 为 scheduled-buyback vault 地址
// 3. isRequestReady(requestId) && vault.getStatus().ready 时：

const ready = await triggerService.read.isRequestReady([requestId]);
const status = await vault.read.getStatus();
if (ready && status.ready) {
  await triggerService.write.trigger([requestId]);
}
// 批量：triggerMultiple([id1, id2, ...])

// 4. CosmTriggerExecuted(success=false) → retryTrigger(requestId)
```

**不要**直调 `vault.trigger()`（仅 `CosmTriggerService` 可回调）。前端只读 `vault.getStatus()` / `pendingRequestId()`。

### 运营（可选）

- 调 fee/gas：`script/ConfigureCosmTrigger.s.sol`（`TRIGGER_GAS_FEE` · `TRIGGER_MAX_CALLBACK_GAS`）
- 仅换 scheduled 工厂：`script/DeployScheduledBuybackFactory.s.sol`（本次全量部署已含 TriggerService 集成版工厂，通常不必单独跑）

---

## CosmToken / CosmTaxToken

发币返回的 clone。曲线买卖走 Portal；余额用标准 ERC20。

**CosmToken（V7）**：ERC20 + `permit` · `metaURI()` · `maxSupply()`

**CosmTaxToken（V6）**

| 方法 | 备注 |
|------|------|
| ERC20 + `permit` | DEX 阶段 transfer 可能扣税 |
| `buyTaxBps()` / `sellTaxBps()` | 税率 |
| `taxProcessor()` | TaxSplitter |
| `vault()` / `dexTaxExempt()` | 路径 B 金库（同值） |
| `pair()` / `poolState()` | `0` 曲线 · `1` 迁移中 · `2` anti-farmer · `3` 仅主池收税 · `4` 无税 |
| `taxRate()` | Flap：`max(buy,sell)` 单值 |
| `getPoolStateData()` | 状态·买卖税率·清算阈值·税到期·anti-farmer 到期 |
| `TransferCosmToken` | from/to **不** indexed |

卖出前：曲线 → `approve(Portal)`；已迁移 → `approve(PCS Router)`。  
DEX 积税清算：卖到主池且达到 `liquidationThreshold`（无 token 侧 `dispatchTax`）。

---

## 曲线买卖步骤

1. `getTokenV8Safe(token)` → `status` / `quoteTokenAddress` / `nativeToQuoteSwapEnabled`
2. `quoteExactInput` → 设 `minOutputAmount`（扣滑点）；曲线与 DEX 同一入口
3. 买入：
   - BNB quote：`inputToken=0` + `msg.value`
   - ERC20 quote：`approve(Portal)` + `inputToken=quote`；或（`nativeToQuoteSwapEnabled`）`inputToken=0` + `msg.value` 一键买
4. 卖出：`token.approve(Portal, amount)` → `swapExactInput`
5. `status=DEX` 时 Portal 转发 PCS（税币 V2 · 普通币 V3/Infinity）；**入口 ABI 与曲线相同**，见 [§5 迁移后交易](#5-迁移后交易status--dex仍走-portal)

---

## TypeScript 常量

```typescript
export const CHAIN_ID = 56;
export const PORTAL = "0xaC11A6ee36Ed4a7A6A0F2aEe7F54aF0c841B3234";
export const VAULT_PORTAL = "0x39BcdA6cfF9a4807B7a4571D94DD675b5E306e60";
export const TRIGGER_SERVICE = "0x22EE465D493EfF231693fC337f92b69149Dd14a0";
export const VANITY_SUFFIX_TAX = 0x0111;
export const VANITY_SUFFIX_STANDARD = 0x0222;

export const QUOTE = {
  BNB: "0x0000000000000000000000000000000000000000",
  USDT: "0x55d398326f99059fF775485246999027B3197955",
  USDC: "0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d",
  USD1: "0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d",
  USDX: "0x95ffc15Ccfbf883B9eE2105F9F7587D6D43829C6",
} as const;

export const VAULT_FACTORY = {
  split: "0x88bb9377096DfADcC1cb8A58A28139E048C3CC03",
  scheduledBuyback: "0xA75ad6018654B2ce0227c4Ab17c093d7550cb30A",
  burnDividend: "0xbc7600BBEb37147BE7b052B0bdAffFcf3A77615c",
  lpStakingDividend: "0x698f4d7a19dD9288Faf8de71A818a3faC7e8Ede4",
  tokenStakingDividend: "0xD1568fF4AEe3F557771b5cCD53988560f17CDc50",
  rankBurnDividend: "0xcdfc393E60432a631512cF7a38c32e9D44eB4AC0",
} as const;
```

Solidity 类型见 `contracts/CosmTypes.sol`。

## 获取 ABI JSON

```bash
# 接口 ABI（默认 profile）
forge build src/interfaces/
mkdir -p abi
jq '.abi' out/IPortal.sol/IPortal.json > abi/IPortal.json
jq '.abi' out/IVaultPortal.sol/IVaultPortal.json > abi/IVaultPortal.json

# 完整实现 ABI（含 vault 实例）
FOUNDRY_PROFILE=cosm forge build
# out-cosm/CosmPortal.sol/CosmPortal.json
# out-cosm/CosmSplitVault.sol/CosmSplitVault.json 等
```
