# Cosm 前端对接 API

> BSC 主网 · Chain ID `56` · 版本 `cosm-v0.8.0`  
> Vanity 后缀（代币地址**低 16 bit**须匹配）：有税 `0x0111` · 无税 `0x0222`（读 `vanitySuffixTax` / `vanitySuffixStandard`）  
> **结构体 / 枚举字段详解** → [`理解.md`](./理解.md)（与 `contracts/CosmTypes.sol` 同步）

### 前端对接索引

| 场景 | 合约 · 方法 | 章节 |
|------|-------------|------|
| 固定入口地址 | Portal · VaultPortal · Trigger · Converter · 6 工厂 | [§合约地址](#合约地址前端调用) |
| 无税发币 | `Portal.newTokenV7` | [§1 无税](#1-无税代币--创建入口) |
| 有税路径 A | `Portal.newTokenV6` | [§2 路径 A](#2-有税--不带-vault路径-a创建入口) |
| 有税路径 B + 金库 | `VaultPortal.newCosmTokenV6WithVault` | [§3 路径 B](#3-有税--带-vault路径-b创建入口) · [Schema 指南](#金库-schema-使用指南) |
| 首买 quote 估算 | simulate / 离线公式 | [§0 quoteAmt](#0-首买-quoteamt-怎么估) |
| 列表 / K 线 | `getTokenV8Safe` · `TokenProgressChanged` | [§TokenStateV8Safe](#tokenstatev8safe--portalgettokenv8safe列表--行情) |
| 详情 / 分红 / 税拆 | `getToken` | [§TokenState](#tokenstate--portalgettoken详情--索引) |
| 曲线 / DEX 买卖 | `quoteExactInput` · `swapExactInput` | [§4 曲线交易](#4-曲线阶段交易status--tradable) · [§5 DEX](#5-迁移后交易status--dex仍走-portal) |
| 插件币 | `swapExactInputV3` | [§ExactInputV3Params](#exactinputv3params--portalswapexactinputv3) |
| 税打出 / 分红领 | `TaxSplitter.dispatch` · `CosmDividend.withdrawDividends` | [§TaxSplitter 前端](#taxsplitter-前端只读--dispatch) · [§CosmDividend](#cosmdividend-方法) |
| 金库页 | `vaultUISchema` · `getStatus` · `getUserInfo` | [§六金库前端读写](#六金库前端读写方法用途与场景) · [§金库实例](#金库实例方法完整列表) |
| TypeScript 常量 | `PORTAL` / `VAULT_FACTORY` 等 | [§TypeScript 常量](#typescript-常量) |
| 常见 revert | simulate 错误文案 | [§常见 Revert](#常见-revert-错误前端) |

## 合约地址（前端调用）

> **2026-08-09 主网全量重部署** · `cosm-v0.8.0` · **Quote 白名单含 COSM**（`0x0D6aE45c…`，与 USDT/USDC/USD1 同档）· 税侧 Flap 对齐。  
> 旧批次（`0xF2846c…` Portal / `0x3F7730…` VaultPortal / `0x8F7dBa…` Trigger / `0x19bfc9…` Converter 等）已废弃。  
> 地址以 `deployments/bsc-56.json` / README 部署清单为准；整仓测试绿灯。  
> 下表为前端需**直接 call** 的固定地址；impl / facet / migrator / dispatch 等由协议内部使用，**勿写进前端配置**。  
> **金库工厂** = Transparent proxy；**金库实例** = BeaconProxy（前端业务方法名不变）。

### 协议入口

| 合约 | 地址 | 用途 |
|------|------|------|
| CosmPortal | `0xb4B057dEFda3822786F998FC54Aa93440caEDb6c` | 发币、曲线/DEX 买卖、状态查询、迁移 |
| CosmVaultPortal | `0xE3BDE2e728F5a9a5FD5bdda87B067a55bf593183` | 有税路径 B（带金库发币） |
| CosmTriggerService | `0x0B8dD41a583f456DD733b2a35CA28D61F6204e08` | 链上调度器；scheduled-buyback 金库通过它预约/回调回购 |
| CosmTaxConverter | `0x3725B42BfDa1Ef33a7eEb8c0465675Ee72aa0001` | Case3 / keeper：batch dispatch、trigger split（Flap `TaxDistributionHelper` proxy） |

### 发币后按 token 读取（勿写死）

| 来源 | 字段 | 前端用途 |
|------|------|----------|
| `Portal.getToken(token)` | `taxSplitter` | `dispatch()` 分配税 |
| `Portal.getToken(token)` | `dividend` | 持币分红 claim |
| `VaultPortal.getVault(token)` | `vault` | 金库页读写（`vaultUISchema` / claim 等） |
| 代币合约 `token` | — | `approve(Portal)` 卖出；金库写操作 approve |

Vanity salt 搜址：`Portal.standardTokenImpl()` / `Portal.taxTokenImpl()`（view，非固定地址）。

### 金库工厂（VaultPortal 已注册 official）

> 下表地址均为 **TransparentUpgradeableProxy**（非裸工厂）。运维升工厂逻辑走 `ProxyAdmin`；升该类型全部金库实例走 `factory.upgradeVaultImplementation(newImpl)`（`DEFAULT_ADMIN_ROLE`）。前端只 call 业务 view / schema。

| 工厂 | 地址 | vaultType |
|------|------|-----------|
| Split | `0x98B345fde625DAc83E1bB478996f6a3FB2deC93e` | `split` |
| Scheduled Buyback | `0x2F9BB21010e28983895aD50fff7bd80a9D7637CE` | `scheduled-buyback` |
| Burn Dividend | `0xFfa993aCaFE3F6B13E68FF8DC388aC0BBc5383E5` | `burn-dividend` |
| LP Staking Dividend | `0xAec0EcFd308a24039aC299A7Fb8Da165EC405074` | `lp-staking-dividend` |
| Token Staking Dividend | `0xf14a4Aa3D702af1416dF91e8372E7f9101F7c3a1` | `token-staking-dividend` |
| Rank Burn Dividend | `0x9abC0D6516d4a023280EADaB82397b99521AB98f` | `rank-burn-dividend` |

### 链上 impl / 模块（运维参考，前端勿配置）

完整清单：`deployments/bsc-56.json`。Portal 发币已拆为 **Launcher 簇**（`delegatecall`）；前端仍只调 **Portal proxy**。

| 键 | 地址 | 说明 |
|----|------|------|
| `portalImpl` | `0x33D034AAa42CB587B41b8DAfE958A2D601D1392f` | CosmPortal 薄代理 impl |
| `portalTrade` | `0xdC807D33b40da6845f9027f033DCacc8CC6F9990` | 买卖模块 |
| `portalTradeDex` | `0x56766E11D7E8e939433F0Bb5eba8f747038437f3` | DEX 路由辅助 |
| `portalLaunch` | `0x68009d2599Bd09b789A83dfd87F9Bed4A7503c11` | Launcher 总入口（`newTokenV6/V7` 路由） |
| `portalLaunchTwoStep` | `0x2638Ee07785f97BA8e7376aA034994a68868a2BC` | Two-step：`stageNewTokenV5` / `commitNewTokenV5`（**仅 saleForge**） |
| `portalLauncherV5` … `V7Tax` | 见 json / README | V5/V6/V7 具体实现（内部） |
| `portalMigrate` | `0xebF4eBccFc840B66106c4932D513411921077879` | 迁移模块 |
| `vaultImpl` | `0x17Edc23B129a3570c214B2FaAbB2488C24C86A24` | CosmVaultPortal 薄代理 impl |
| `proxyAdmin` | `0x2A4C566C9Aeb733D3FcC10ad8c2A6BE1cd3A7746` | Transparent 代理管理员（Portal/Vault/Trigger/Converter/六工厂） |
| `triggerImpl` | `0x5bec8FaDE3005aa28E841f66c6d0cA7fDbce5522` | CosmTriggerService impl |
| `vaultLens` / `vaultLaunch` / `vaultTweak` | 见 json | VaultPortal 三模块 |
| `defaultTaxConverter` | `0x3725B42BfDa1Ef33a7eEb8c0465675Ee72aa0001` | 与 `converterProxy` 相同 |
| `taxSplitterImpl` / `taxSplitterDispatchImpl` | 见 json | 税清算模板（含 2026-08-09 Flap 对齐） |

### Quote 白名单

发币时选定的 `quoteToken` 锁定全生命周期（曲线 / 协议费 / 税 / 迁移 LP）。

| 代币 | quoteToken | decimals | 支付 |
|------|------------|----------|------|
| BNB | `0x0000000000000000000000000000000000000000` | 18 | `msg.value` |
| USDT | `0x55d398326f99059fF775485246999027B3197955` | 18 | 先 `approve(Portal, amount)` |
| USDC | `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d` | 18 | 同上 |
| USD1 | `0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d` | 18 | 同上 |
| COSM | `0x0D6aE45c96eC4df860300087462266e19140F6dc` | 18 | 同上 |

| 发币路径 | 可用 quote |
|----------|-----------|
| 无税 `Portal.newTokenV7` | 上表全部 |
| 有税路径 A `Portal.newTokenV6`（无金库） | 上表全部 |
| 有税路径 B `VaultPortal.*`（带金库） | **仅 BNB** |

**协议默认（前端展示参考，非链上常量）：**

| 项 | 值 | 说明 |
|----|-----|------|
| 迁移阈值 | `800_000_000 ether`（8 亿枚） | 单 token 以 `getTokenV8Safe.dexSupplyThresh` 为准 |
| V7 曲线协议费 | `125` bps（1.25%） | 打给 `beneficiary`（发币人）；见 `bondingCurveFeeBps` |
| 曲线全局协议费 | `100` bps | `protocolFeeBps` / `protocolSellFeeBps` 默认 |
| TaxSplitter 内部抽成 | 曲线 / DEX 均为 **`1000` bps** | 发币与 migrate 均为 10% of tax → `feeReceiver`；读 `feeConfigV3().feeRate` |
| dividend magic | `SELF`=`0xfEED…` · `COMPUTED`=`0xC0De…` | Flap `newTokenV6WithVault` 用；Cosm 推荐 `dividendMode` |

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
| 有税 · TaxSplitter 四路打出 | **Flap BSC V2 语义**：swap **只累账**；keeper **主动**调 `TaxSplitter.dispatch()`（无 swap 后链上信号） |

### 步骤 A — 发币页（`vaultDataSchema`）

路径 B 入口：**推荐** `VaultPortal.newCosmTokenV6WithVault`（也可用 Flap 布局 `newTokenV6WithVault`）· quote **仅 BNB** · `mktBps > 0`。

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

### 0. 首买 quoteAmt 怎么估

**是的，由 `quoteAmt` 决定**（花多少 quote，不是填 token 数量）。买到默认阈值 `800_000_000 ether`（8 亿枚）链上自动停。

**BNB 池、默认曲线，估 `quoteAmt`：**

```
quoteAmt ≈ 16 ether × 10000 / (10000 − 协议费bps − buyTaxBps)
```

| 类型 | 协议费 | 代入示例 |
|------|--------|----------|
| 无税 `newTokenV7` | 125 bps | `16 × 10000 / 9875` ≈ **16.2 ether** → 填 **17 ether** |
| 有税 `newTokenV6` | 100 bps + `buyTaxBps` | buyTax=500 → `16 × 10000 / 8900` ≈ **18 ether** |

- BNB：`msg.value = quoteAmt`（略大一点可以，多的会退）
- USDT 等：同样数量的 `quoteAmt`，先 `approve(Portal)`，`msg.value = 0`
- `quoteAmt = 0` 只创建不买；估小了买不到 8 亿

**有没有链上直接估？** 没有 `quoteAmtForThreshold()` 这类专用 view。

| 时机 | 做法 |
|------|------|
| **发币前（推荐）** | `simulateContract(newTokenV6/V7, { value: quoteAmt })`，看模拟结果里 `circulatingSupply` 是否到 `800_000_000 ether`；不够就加大 `quoteAmt` 再 simulate |
| **已有 token（曲线阶段）** | `quoteExactInput({ inputToken: quote, outputToken: token, inputAmount: X })` → 返回能买到的 token 数；**非 view** |
| **发币前（离线）** | 任意已发 Cosm 币调 `getTokenV8Safe` 取 `r/h/k/dexSupplyThresh`，本地用 `LibCurve.estimateReserve` + 上表公式 |

> 发币前 **不能** 对「还不存在的 token 地址」调 `quoteExactInput`（Portal 里还没有状态）。  
> 字段级说明见 [`理解.md` §创建买满 8 亿](./理解.md)。

---

### 1. 无税代币 — 创建入口

| 项 | 值 |
|----|-----|
| **合约** | `CosmPortal` proxy |
| **方法** | `newTokenV7(LaunchParams p) payable` |
| **关键字段** | `isTaxed = false` · `buyTaxBps/sellTaxBps = 0` · `beneficiary = 0`（V7 会自动设为发币人） |
| **salt** | 对 `Portal.standardTokenImpl()` 搜 vanity **`0x0222`** |
| **quote** | BNB / USDT / USDC / USD1 / COSM 均可 |
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
| **迁移** | 强制 **PCS V2**；`CosmTaxToken.mainPool()` = 发币时 CREATE2 预测的 V2 pair（`pools[0]`） |

```typescript
await portal.write.newTokenV6([{
  ...meta, salt, quoteToken, quoteAmt,
  beneficiary: marketerWallet,
  buyTaxBps: 500, sellTaxBps: 500,
  isTaxed: true,
  tax: { mktBps: 8000, deflationBps: 0, dividendBps: 2000, lpBps: 0, ... },
  migratorType: 1, // 可填任意值；税币链上强制 V2_MIGRATOR(1)
}], { value });
```

---

### 3. 有税 · 带 Vault（路径 B）— 创建入口

| 项 | 值 |
|----|-----|
| **合约** | `CosmVaultPortal` proxy（**不是** Portal） |
| **方法** | `newTokenV6WithVault`（Flap 布局）或 `newCosmTokenV6WithVault`（Cosm 简化布局） |
| **quote** | **仅 BNB**（`quoteToken = 0`） |
| **mkt** | `mktBps` **必须 >0**（营销份额进金库，不进钱包） |
| **内部流程** | 工厂 `newVault` → `Portal.newTokenV6` → 写入 `vaults[token]` |

```typescript
// Cosm 简化入口（推荐前端）：含 dividendMode / converter
await vaultPortal.write.newCosmTokenV6WithVault([{
  name, symbol, meta, salt,
  quoteToken: zeroAddress,
  quoteAmt, buyTaxBps, sellTaxBps,
  mktBps: 10_000, deflationBps: 0, dividendBps: 0, lpBps: 0,
  minimumShareBalance, dividendMode, dividendToken, converter,
  antiFarmerDuration, taxDuration,
  vaultFactory: VAULT_FACTORY.split,
  vaultData: encodedFormData,
}], { value: quoteAmt });

// Flap v1.12.1 兼容入口：字段见 §NewTokenV6WithVaultParams（无 dividendMode，用 dividendToken magic）
// await vaultPortal.write.newTokenV6WithVault([{ ... }], { value: quoteAmt });
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
| **dexId**（`getTokenV8Safe` / `getTokenV9Safe`） | 路由 hint：`2`=V2（**税币恒 2**）· `3`=V3 · `4`=Infinity。与 `getToken.dexId`（发币 MDR 序号 0/1/2）不同 |

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
| 税打出 | `TaxSplitter(taxSplitter).dispatch()` · Case3 用 `TaxConverter.triggerSplit([token])` |
| 支持的 quote 列表 | `Portal.getSupportedQuoteTokens()` · `getQuoteTokenConfiguration(quote)` |
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
| 税打出 | `TaxSplitter.dispatch` / `TaxConverter.triggerSplit` | 地址=`getToken.taxSplitter` |
| 金库玩法 | 各 Vault 实例方法 | 地址=`VaultPortal.getVault(token).vault` |
| 金库表单自描述 | `factory.vaultDataSchema` / `vault.vaultUISchema` | 动态表单 |
| 延期回调 | `TriggerService.requestTrigger` / `trigger` | Keeper 持 `TRIGGER_ROLE` |

### 状态机

```
Invalid(0) ──发币成功──► Tradable(1) ──达阈值/手动迁移──► DEX(4)
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
| `2` | `InDuel` | Flap 占位（Cosm BSC 不用） |
| `3` | `Killed` | Flap 占位（Cosm BSC 不用） |
| `4` | `DEX` | 已迁移，买卖仍走 Portal（内部转发 PCS V2 / V3 / Infinity） |
| `5` | `Staged` | SaleForge `stageNewTokenV5` 预占；`commit` 后变 `Tradable` |

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
| `antiFarmerDuration` | `uint256` | 迁移后 anti-farmer 秒数。窗口内全 `pools` 收税，结束后仅 `mainPool`。`0`=跳过窗口；上限 365 天 |
| `taxDuration` | `uint256` | 收税总时长（秒，含 anti-farmer）。`0`=永不过期；若 >0 须 ≥ `antiFarmerDuration`；上限约 100 年 |
| `mktBps2/3/4` | `uint16` | 从 `mktBps` 划出；合计 ≤ `mktBps`；默认 0 |
| `market2/3/4` | `address` | 对应收款地址；bps>0 时必填 |
| `converter` | `address` | 仅 `dividendMode=2` 必填；Case3 MEV 兑换员。**须为** `CosmTaxConverter` proxy（或等效 helper），由它调 `TaxSplitter.dispatch` 兑 quote→分红币；发币留空则用 `Portal.defaultTaxConverter()`。mode 0/1 填 `0` |

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

**Flap SDK 兼容 overload（可选）：** Portal 同时暴露 `newTokenV6(NewTokenV6Params)` / `newTokenV7(NewTokenV7Params)`（见 `ICosmPortalTypes.sol`），字段布局与 Flap v1.12.1 一致。Cosm 自建前端推荐 **`LaunchParams`** 或 VaultPortal 的 **`NewCosmTokenV6WithVaultParams`**；若接 Flap SDK，完整字段见 [`理解.md` §ICosmPortalTypes](./理解.md)。

**V7 Flap 布局 `FeeConfig[4]`：** `feeType`（`0`=NONE · `1`=MKT · `2`=DIVIDEND · `3`=DEFLATION · `4`=LP_BPS）+ `bps` + `marketingAddress` / `dividendToken` / `minimumShareBalance`。Cosm `LaunchParams.tax` 四路与之等价，无需手填 `feeConfigs`。

### `StageNewTokenV5Params` / `CommitNewTokenV5Params` — Two-step（**仅 saleForge**）

> 对应 `stageNewTokenV5`（nonpayable → `address`）/ `commitNewTokenV5`（payable → 无返回值）。普通 dApp 勿调。

```solidity
struct StageNewTokenV5Params {
    DexThreshType dexThresh;
    bytes32 salt;
    bool isTaxToken;
    MigratorType migratorType;
    address quoteToken;
    DEXId dexId;
}

struct CommitNewTokenV5Params {
    bytes32 salt;
    uint16 taxRate;
    string name;
    string symbol;
    string meta;
    uint256 quoteAmt;
    address beneficiary;
    bytes permitData;
    uint64 taxDuration;
    uint64 antiFarmerDuration;
    uint16 mktBps;
    uint16 deflationBps;
    uint16 dividendBps;
    uint16 lpBps;
    uint256 minimumShareBalance;
}
```

---

### `ExactInputV3Params` — `Portal.swapExactInputV3`

`getTokenV8Safe.extensionID ≠ 0` 时**必须**用本方法（旧 `swapExactInput` 会 `TokenWithExtensionNotSupported`）。

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `inputToken` | `address` | 同 `SwapParams` |
| `outputToken` | `address` | 同 `SwapParams` |
| `inputAmount` | `uint256` | 实际支付量 |
| `minOutputAmount` | `uint256` | 滑点下限 |
| `permitData` | `bytes` | 可选 EIP-2612 |
| `extensionData` | `bytes` | 插件 `onTrade` 参数；由 `extensionID` 决定编码 |

支付规则与 `swapExactInput` 相同（BNB / ERC20 / nativeToQuote）。

```solidity
struct ExactInputV3Params {
    address inputToken;
    address outputToken;
    uint256 inputAmount;
    uint256 minOutputAmount;
    bytes permitData;
    bytes extensionData;
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
| `status` | `TokenStatus` | `0` Invalid · `1` Tradable · `4` DEX · `5` Staged（SaleForge）；`2`/`3` 占位不用 |
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
| `pool` | `address` | 迁移后池；Tax=V2 pair · 普通 V3/Infinity；未迁=`0`。税币 `mainPool` 发币时 CREATE2 预测，须与 V2 pair 一致 |
| `vault` | `address` | 路径 B 金库；路径 A / 普通=`0`。亦可 `VaultPortal.tryGetVault` |
| `feeProfile` | `FeeProfile` | 协议费档位，见上表 |
| `migratorType` | `MigratorType` | 实际迁移器；税币恒 `1`（V2） |
| `infinityHook` | `address` | Infinity 路径 Hook；非 Infinity=`0` |
| `bondingCurveFeeBps` | `uint16` | V7 曲线协议费 bps；税币=`0` |
| `lpFeeProfile` | `uint8` | Flap V3LPFeeProfile 占位 |
| `dexSupplyThresh` | `uint128` | 迁移阈值 token 数；默认 `800_000_000e18` |
| `extensionID` | `bytes32` | 扩展 ID；Cosm 默认 `0` |
| `dexId` | `uint8` | 发币 MDR 序号（`DEX0=0`；legacy `DEX1/2→2`）；**不是** V8Safe 的路由 hint |

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
    MigratorType migratorType;
    address infinityHook;
    uint16 bondingCurveFeeBps;
    uint8 lpFeeProfile;
    uint128 dexSupplyThresh;
    bytes32 extensionID;
    uint8 dexId;                 // MDR 发币序号；≠ getTokenV8Safe.dexId
}
```

---

### `TokenStateV8Safe` — `Portal.getTokenV8Safe`（列表 / 行情）

**不含** `taxSplitter` / `dividend` / `vault` / `feeProfile` / 完整 `tax`；需要时再调 `getToken`。

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `status` | `uint8` | 同 TokenStatus：`0`/`1`/`4`/`5`（`2`/`3` 占位） |
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
| `dexId` | `uint8` | **路由 hint**（`_dexKind`）：`2`=V2 · `3`=V3 · `4`=Infinity；税币恒 `2` |

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
    bool nativeToQuoteSwapEnabled;
    bytes32 extensionID;
    uint256 buyTaxRate;
    uint256 sellTaxRate;
    address pool;
    uint256 progress;
    uint8 lpFeeProfile;
    uint8 dexId;                   // 路由 hint；≠ getToken.dexId
}
```

---

### `TokenStateV9Safe` — `Portal.getTokenV9Safe`（V8 + 曲线费）

在 V8Safe 基础上多 `bondingCurveFeeRate`（= `getToken.bondingCurveFeeBps`）。列表页一般用 V8 即可。

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| （同 V8Safe 各字段） | | |
| `bondingCurveFeeRate` | `uint16` | V7 曲线协议费 bps；税币=`0` |

```solidity
struct TokenStateV9Safe {
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
    bool nativeToQuoteSwapEnabled;
    bytes32 extensionID;
    uint256 buyTaxRate;
    uint256 sellTaxRate;
    address pool;
    uint256 progress;
    uint8 lpFeeProfile;
    uint8 dexId;
    uint16 bondingCurveFeeRate; // = getToken.bondingCurveFeeBps
}
```

---

### VaultPortal 结构体（路径 B / 金库，非 CosmTypes 文件但前端常用）

#### `NewTokenV6WithVaultParams` — `VaultPortal.newTokenV6WithVault`（Flap v1.12.1 布局）

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `name` / `symbol` / `meta` | `string` | 同 LaunchParams |
| `dexThresh` | `DexThreshType` | 迁移阈值档位；默认 `FOUR_FIFTHS`→8e8 |
| `salt` | `bytes32` | 须匹配 tax vanity `0x0111` |
| `migratorType` | `MigratorType` | 可填任意；税币链上强制 `1`（V2） |
| `quoteToken` | `address` | **仅** BNB=`0` |
| `quoteAmt` | `uint256` | 首买；`msg.value` |
| `permitData` / `extensionID` / `extensionData` | | Flap 兼容字段；Cosm 通常空/0 |
| `dexId` / `lpFeeProfile` | | MDR / LP 档位 hint |
| `buyTaxRate` / `sellTaxRate` | `uint16` | 至少一侧 >0 |
| `taxDuration` / `antiFarmerDuration` | `uint64` | 秒 |
| `mktBps` | `uint16` | **必须 >0** |
| `deflationBps` / `dividendBps` / `lpBps` | `uint16` | 四路合计 10000 |
| `minimumShareBalance` | `uint256` | |
| `dividendToken` | `address` | 无 `dividendMode`；用 magic（`SELF` / `COMPUTED`） |
| `commissionReceiver` | `address` | 佣金；通常 `0` |
| `tokenVersion` | `TokenVersion` | `6`（TOKEN_TAXED_V3） |
| `vaultFactory` / `vaultData` | | 工厂 + 编码参数 |

#### `NewCosmTokenV6WithVaultParams` — `VaultPortal.newCosmTokenV6WithVault`（推荐）

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `name` / `symbol` / `meta` / `salt` | | 同左 |
| `quoteToken` | `address` | **仅** BNB |
| `quoteAmt` | `uint256` | |
| `buyTaxBps` / `sellTaxBps` | `uint16` | |
| `mktBps` … `lpBps` | `uint16` | 四路合计 10000；`mktBps>0` |
| `minimumShareBalance` | `uint256` | |
| `dividendMode` / `dividendToken` / `converter` | | 同 `TaxAllocation` |
| `antiFarmerDuration` / `taxDuration` | `uint256` | |
| `vaultFactory` / `vaultData` | | |

#### `VaultInfo` — `VaultPortal.getVault` / `tryGetVault`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `vault` | `address` | 金库实例 |
| `vaultFactory` | `address` | 创建工厂；`tryGetVault` 回退读可能为 `0` |
| `description` | `string` | 金库文案 |
| `isOfficial` | `bool` | 是否官方工厂 |
| `riskLevel` | `RiskLevel` | `0` UNVERIFIED · `1` LOW_RISK · `2` LOW_MEDIUM_RISK · `3` MEDIUM_RISK · `4` HIGH_RISK |

分类另调 `getCosmVaultCategory(token)`（`0` NONE · `1` SPLIT · `2` BUYBACK · `3` DIVIDEND · `4` AIRDROP · `5` GAME）或 Flap 遗留 `getVaultCategory`（仅 `0/1`）。

#### `VaultFactoryInfo` — `getVaultFactory`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `registered` | `bool` | |
| `enabled` | `bool` | |
| `official` | `bool` | |
| `riskLevel` | `RiskLevel` | 同上 |
| `category` | `CosmVaultCategory` | 用 `getCosmFactoryCategory` |
| `permissionPolicy` | `FactoryPermissionPolicy` | `0` OPEN · `1` TIME_DEPENDENT · `2` DISABLED |
| `validationMode` | `FactoryValidationMode` | `0` NONE · `1` LEGACY_V6 · `2` V22 |

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
python3 tools/find_vanity.py --predict --portal 0xb4B057dEFda3822786F998FC54Aa93440caEDb6c --tax
python3 tools/find_vanity.py --predict --portal 0xb4B057dEFda3822786F998FC54Aa93440caEDb6c
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

地址（proxy）：`0xb4B057dEFda3822786F998FC54Aa93440caEDb6c`

### 发币（用户）

| 方法 | 返回 | 备注 |
|------|------|------|
| `newTokenV7(LaunchParams p) payable` | `address token` | 普通币；`isTaxed` 必须 false；内部走 Launcher→V7 |
| `newTokenV6(LaunchParams p) payable` | `address token` | 路径 A；`beneficiary`≠0；税率至少一侧 >0；内部走 Launcher→V6 |
| `stageNewTokenV5(StageNewTokenV5Params)` | `address token` | Two-step 预占；**仅 `saleForge`**（`OnlySaleForge`）；**非 payable** |
| `commitNewTokenV5(CommitNewTokenV5Params) payable` | —（无返回值） | Two-step 提交；**仅 `saleForge`**；首买用 `quoteAmt`/`msg.value` |

> 前端日常发币用 `newTokenV6` / `newTokenV7`（Portal proxy）。`stage`/`commit` 给 SaleForge / vanity 市场，**普通 dApp 勿调**。  
> `portalLauncherV5/V6/V7*` 为内部 `delegatecall` 目标，勿写进 dApp 配置。  
> 另有 Flap 布局重载：`newTokenV6(ICosmPortalTypes.NewTokenV6Params)` / `newTokenV7(ICosmPortalTypes.NewTokenV7Params)`，Cosm 推荐用 `LaunchParams`。

### 曲线交易（用户）

| 方法 | 返回 | 备注 |
|------|------|------|
| `quoteExactInput(QuoteParams)` | `uint256 outputAmount` | 曲线/DEX 估价（非 view；DEX 调 Quoter） |
| `swapExactInputV3(ExactInputV3Params) payable` | `uint256 outputAmount` | 同 swapExactInput + **`extensionData`** → 插件 `onTrade` |
| `swapExactInput(SwapParams) payable` | `uint256 outputAmount` | extension 代币不可用（须 V3） |
| `migrateToDex(address token) payable` | `address pool` | 流通量须已达阈值；通常买入达阈自动迁 |
| `claim(address token)` | `(tokenAmount, ethAmount)` | **仅 beneficiary**；Infinity GoPlus 收 LP 费打给调用方 |
| `delegateClaim(address token)` | `(tokenAmount, quoteAmount)` | **owner / guardian / roller** 代领至 beneficiary |

### 查询（前端高频）

| 方法 | 返回 | 备注 |
|------|------|------|
| `getTokenV8Safe(address) view` | `TokenStateV8Safe` | 列表默认 |
| `getTokenV9Safe(address) view` | `TokenStateV9Safe` | V8 + `bondingCurveFeeRate` |
| `getToken(address) view` | `TokenState` | 详情 / 分红 / 金库 |
| `getVault(address) view` | `address` | 路径 B 金库；无则 `0` |
| `predictTokenAddress(bool isTaxed, bytes32 salt) view` | `address` | 复核 salt；勿暴力搜 |
| `isQuoteTokenSupported(address) view` | `bool` | 发币表单预检 |
| `getSupportedQuoteTokens() view` | `address[]` | 含 BNB=`0` |
| `getQuoteTokenConfiguration(address quoteToken) view` | `QuoteTokenConfiguration` | ERC20 quote 的 `nativeToQuoteSwapEnabled` 等；BNB 通常不必调 |
| `hasDividendLiquidity(address otherToken) view` | `bool` | mode2 分红币校验 |
| `enableTaxOnBondingCurve() pure` | `bool` | 恒 `true` |
| `version() pure` | `string` | Portal=`cosm-v0.8.0`；VaultPortal 为 Flap 兼容 `1.12.1` |
| `vanitySuffixFor(bool isTaxed) view` | `uint16` | 推荐取后缀 |
| `getFeeRate() view` | `(buy, sell)` | 全局默认买卖协议费 |
| `getMigrationFeeRate() view` | `(liquidity, reserve)` | 迁移费；默认 `0,0` |
| `feeExempt(address) view` | `bool` | 交易者是否豁免协议费 |
| `bitFlags() view` | `uint256` | 熔断位图：`1`=全局 · `2`=V7 · 默认 `3` |
| `isSpammerBlocked(address) view` | `bool` | 发币黑名单 |
| `getSaltLock(bytes32) view` | `SaltLockEntry` | vanity salt 预占；`locker=0`=未锁 |
| `getLockedSaltsByUserAndVersion(user, ver, offset, limit) view` | `(salts, entries, total)` | 分页查用户已锁 salt |
| `saltLockFee() view` | `uint256` | 预占费用（BNB）；默认 `0.01 ether` |
| `getLocks(address) view` | `uint256[]` | GoPlus lockId（Infinity 双 NFT 返回 2 个；V3 返回 1 个） |
| `infinityLockId(address) view` | `uint256` | Infinity GoPlus lockId0（quote-heavy lower） |
| `infinityLockId1(address) view` | `uint256` | Infinity GoPlus lockId1（token-heavy upper） |
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
| `defaultTaxConverter()` | `address` | Case3 默认 converter = **CosmTaxConverter proxy** `0x3725B42BfDa1Ef33a7eEb8c0465675Ee72aa0001` |
| `swapRegistry()` | `address` | Case3 SwapRegistry（V2/V3/Infinity CL/V4 · MultiDexRouter） |
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

**索引器最小订阅（列表 + 详情 + K 线）：**

| 事件 | 字段要点 |
|------|----------|
| `TokenCreated` | `token` · `creator` · `name` · `symbol` · `meta` · `tokenVersion`（6/7） |
| `TokenBought` / `TokenSold` | `token` · `trader` · `amount` · `quoteAmount` · `price` |
| `TokenProgressChanged` | `token` · `progress`（Wad） |
| `TokenMigrated` | `token` · `pool` · `migratorKind`（2=V2 · 3=V3 · 4=Infinity） |
| `CosmTaxVaultTokenCreated` | VaultPortal 路径 B：`token` · `vault` · `factory` |
| `FlapTaxProcessorBondingCurveTax` / `ProcessTaxTokens` | 税累账（可选展示 pending） |
| `FlapTaxProcessorDispatchExecuted` | 税已打出（刷新 mkt/分红余额） |

新币发现：`TokenCreated` + 校验 `predictTokenAddress` vanity。Quote 配置：`getQuoteTokenConfiguration(quote).nativeToQuoteSwapType` ≠ `SWAP_DISABLED` 时允许 BNB 一键买 ERC20 quote。

---

## CosmVaultPortal 方法

地址（proxy）：`0xE3BDE2e728F5a9a5FD5bdda87B067a55bf593183`

> **架构：** 薄代理 + Lens / Launch / Tweak 三模块（`delegatecall`，与 CosmPortal Trade/Launch/Migrate 同模式）。前端**只调 proxy**；模块地址见 `deployments/bsc-56.json`（`vaultLens` / `vaultLaunch` / `vaultTweak`）。  
> **权限：** `AccessControl` — `VAULT_ADMIN_ROLE` 注册工厂与分类；`AUDITOR_ROLE` 工厂 policy / `refreshTokenVault`；`DEFAULT_ADMIN_ROLE` 可授予上述角色。`renounceOwnership()` 会先撤销调用者全部角色再 renounce。

| 方法 | 返回 | 调用方 | 备注 |
|------|------|--------|------|
| `newTokenV6WithVault(NewTokenV6WithVaultParams) payable` | `token` | 用户 | Flap 布局；`MAGIC_DIVIDEND_COMPUTED` |
| `newCosmTokenV6WithVault(NewCosmTokenV6WithVaultParams) payable` | `token` | 用户 | **推荐**；含 `dividendMode` / `converter` |
| `newTokenV7WithVault(NewTokenV7WithVaultParams) payable` | `token` | 用户 | Flap V7 + feeConfigs + vault |
| `getVault(address taxToken) view` | `VaultInfo` | 前端 | 已知路径 B |
| `tryGetVault(address taxToken) view` | `(found, VaultInfo)` | 前端 | 不确定时用 |
| `predictTaxTokenV1Address(bytes32 salt) view` | `address` | 前端 | Flap 兼容 salt 复核（tax vanity `0x0111`） |
| `getVaultFactory(address factory) view` | `VaultFactoryInfo` | 前端 | 选工厂标签 |
| `getFactoryPolicy(factory) view` | `(policy, policyData, description)` | 前端 | `0=OPEN · 1=TIME_DEPENDENT · 2=DISABLED` |
| `getCosmVaultCategory` / `getCosmFactoryCategory` | `CosmVaultCategory` | 前端 | 玩法分类（SPLIT/BUYBACK/…） |
| `getVaultCategory` / `getFactoryCategory` | `VaultCategory` | 前端 | Flap 遗留（仅 `0/1`） |
| `portal() view` | `CosmPortal` | 前端 | 关联 Portal |
| `registerVaultFactory(...)` | — | **VAULT_ADMIN_ROLE** | 部署注册；默认 policy=OPEN |
| `disableFactory` / `setTimeDependentPolicy` / `resetFactoryPolicy` | — | **AUDITOR_ROLE** | 工厂权限 |
| `setCosmVaultCategory` / `setCosmFactoryCategory` | — | **VAULT_ADMIN_ROLE** | Cosm 分类 |
| `setVaultCategory` / `setFactoryCategory` | — | **VAULT_ADMIN_ROLE** | Flap 遗留分类 |
| `refreshTokenVault(token)` | — | **AUDITOR_ROLE** | 同步缓存 vault ← TaxSplitter.market |

事件：`CosmTaxVaultTokenCreated` · `VaultFactoryRegistered` · `FactoryPermissionPolicySet` · `VaultCategoryUpdated` · `FactoryCategoryUpdated` · `TokenVaultRefreshed`

---

## 金库工厂（Vault Factory）

> Schema 发币/详情步骤见 **[§金库 Schema 使用指南](#金库-schema-使用指南)**。

前端用工厂做**表单发现**（`vaultDataSchema`）；`newVault` **仅** VaultPortal 内部调用。

| 方法 | 返回 | 前端用途 |
|------|------|----------|
| `vaultType() pure` | `string` | 玩法名 |
| `vaultDataSchema() pure` | `VaultDataSchema` | 动态生成发币表单并 `abi.encode` |
| `isQuoteTokenSupported(address) pure` | `bool` | 全部工厂仅 BNB=`address(0)` |
| `factorySpecVersion() pure` | `string` | 当前 `"v2.3"`（须严格 `vMAJOR.MINOR`） |
| `tokenCreationPolicies() pure` | `FactoryPolicy[]` | UI 约束提示（如 quote=BNB）；**不强制执行** |
| `onBeforeLaunch(bytes) view` | `(bool, string)` | 发币前链上预检（税率/分配/quote）；前端**必调**，见 [§路径 B 提交前校验](#路径-b-提交前校验newtokenv6withvault) |
| `newVault(...)` | `address` | **勿直接调** |

### 工厂地址与 vaultData

| vaultType | 工厂地址 | vaultData 编码 |
|-----------|----------|----------------|
| `split` | `0x98B345fde625DAc83E1bB478996f6a3FB2deC93e` | `abi.encode(Recipient[])` · `Recipient{address recipient; uint16 bps}` · 1–10 人去重 · bps 合计 10000 |
| `scheduled-buyback` | `0x2F9BB21010e28983895aD50fff7bd80a9D7637CE` | `abi.encode(triggerMode, buybackMode, intervalSeconds, minBnbAmount, maxBnbPerTrigger[, firstExecutableAt])` · trigger:`0=time 1=amount+interval 2=both` · buyback:`0=token 1=LP` · `firstExecutableAt` 可选 unix 秒 |
| `burn-dividend` | `0xFfa993aCaFE3F6B13E68FF8DC388aC0BBc5383E5` | 空 `0x` |
| `lp-staking-dividend` | `0xAec0EcFd308a24039aC299A7Fb8Da165EC405074` | 空 `0x`（pair 发币时 CREATE2 预测，同 `CosmTaxToken.mainPool`） |
| `token-staking-dividend` | `0xf14a4Aa3D702af1416dF91e8372E7f9101F7c3a1` | 空 `0x` |
| `rank-burn-dividend` | `0x9abC0D6516d4a023280EADaB82397b99521AB98f` | `abi.encode(uint256 minBurnAmount)`；也可空=`0` |

---

---

## 六金库前端读写方法（用途与场景）

> **入口：** `const vault = (await vaultPortal.read.tryGetVault([token])).vault`（或 `getVault`）。  
> **前提：** 路径 B 发币（`newCosmTokenV6WithVault`）· quote **仅 BNB**。  
> **通用读（六个都有）：** `vaultUISchema()` · `vaultType()` · `getStatus()` · `taxToken()` · `factory()` · `creator()` · `quoteToken()` · `description()`  
> **税入账：** 被动 `receive()`，前端不用调。

### 速查：每个玩法前端要调什么

| vaultType | 场景 | 主要只读 | 主要写入 | 谁调写 |
|-----------|------|----------|----------|--------|
| `split` | 多人分税 BNB | `getStatus` · `getUserInfo(user)` · `claimable` | `claim(user)` · 可选 `dispatch` | 用户领；任何人可代 `dispatch` |
| `scheduled-buyback` | 定时回购销毁 | `getStatus` · `canTrigger` · `countdownSeconds` | 转 BNB 充值；**回购由 keeper** | 用户只充值；keeper `TriggerService.trigger` |
| `burn-dividend` | 烧币攒权领 BNB | `getStatus` · `getUserInfo` · `pendingReward` | `burn(amount)` · `claim()` | 用户（先 `taxToken.approve(vault)`） |
| `token-staking-dividend` | 质押税币领 BNB | `getStatus` · `getUserInfo` · `pendingReward` | `stake` · `withdraw` · `claim` | 用户（stake 前 approve 税币） |
| `lp-staking-dividend` | 质押 LP 领 BNB | 同上 + `pair()` | `stake` · `withdraw` · `claim` | 用户（stake 前 approve **LP=`pair()`**） |
| `rank-burn-dividend` | 烧币上榜领双池 | `getStatus` · `getUserInfo` · `topBurnersList` · `pendingRank` | `burn` · `claim` | 用户（approve 税币） |

### 1) `split` — 按 bps 分税

**用途：** 税 BNB 进金库后按收款人 bps **记账**，用户主动领取。

| 类型 | 方法 | 用途 / 场景 |
|------|------|-------------|
| 读 | `getStatus()` | 金库页总览：余额、已分配、可领合计、收款人列表 |
| 读 | `getUserInfo(user)` | 「我的」：是否收款人、bps、可领金额 |
| 读 | `claimable(user)` | 按钮旁显示可领（含未记账近似） |
| 读 | `getRecipientsInfo()` | 管理/详情：全员账本表 |
| 写 | `claim(user)` | **主操作**：用户点「领取」；可代领（款仍打给 `user`） |
| 写 | `dispatch()` | 可选：一键给全部收款人推送（运营/keeper） |
| 写 | `accrue()` | 可选：把误转入的未记账余额记入账本 |

### 2) `scheduled-buyback` — 定时回购销毁

**用途：** 税/充值 BNB 按时间或金额门槛回购本币或 LP 并销毁。前端**不执行回购**。

| 类型 | 方法 | 用途 / 场景 |
|------|------|-------------|
| 读 | `getStatus()` | **首选**：倒计时、`ready`、下次花费、累计销毁、`pendingRequestId` |
| 读 | `canTrigger()` / `countdownSeconds()` | 状态条 / 倒计时组件 |
| 读 | `nextTriggerAt()` · `nextSpendBnb()` · `pendingRequestId()` | 细节展示 |
| 写 | 向 vault 转 BNB（`receive`） | 用户「手动充值」；触发重新预约 |
| 写 | ~~`vault.trigger`~~ | **禁止前端直调**；仅 `TriggerService` 回调 |
| Keeper | `TriggerService.trigger(requestId)` | 见 `keeper.md`：`isRequestReady` 且 `getStatus().ready` |

### 3) `burn-dividend` — 燃烧分红

**用途：** 用户烧税币到黑洞攒 power，按权分税 BNB；claim **不清除** power。

| 类型 | 方法 | 用途 / 场景 |
|------|------|-------------|
| 读 | `getStatus()` | 池子：总燃烧、待分 BNB、总 power |
| 读 | `getUserInfo(user)` / `pendingReward(user)` | 「我的燃烧 / 可领」 |
| 写 | `burn(amount)` | 主操作：烧币；前置 `taxToken.approve(vault, amount)` |
| 写 | `claim()` | 领取累计 BNB 分红 |

### 4) `token-staking-dividend` — 本币质押分红

**用途：** 质押税币，按份额分税 BNB。

| 类型 | 方法 | 用途 / 场景 |
|------|------|-------------|
| 读 | `getStatus()` / `getUserInfo(user)` / `pendingReward(user)` | 总质押、个人质押、可领 |
| 写 | `stake(amount)` | 质押；前置 `taxToken.approve(vault, amount)` |
| 写 | `withdraw(amount)` | 解押取回税币 |
| 写 | `claim()` | 只领 BNB，不解押 |

### 5) `lp-staking-dividend` — LP 质押分红

**用途：** 质押 PCS V2 LP（`taxToken/WBNB`），按份额分税 BNB。

| 类型 | 方法 | 用途 / 场景 |
|------|------|-------------|
| 读 | `getStatus()` · `pair()` | 总览 + **LP 合约地址**（approve 对象） |
| 读 | `getUserInfo(user)` / `pendingReward(user)` | 个人质押 / 可领 |
| 写 | `stake(amount)` | 质押 LP；前置 **`IERC20(pair).approve(vault, amount)`** |
| 写 | `withdraw(amount)` | 取回 LP |
| 写 | `claim()` | 只领 BNB |

> `pair` 在发币时 CREATE2 预测，与迁移后 `CosmTaxToken.pair()` / `mainPool` 一致；迁移前也可读 `vault.pair()` 做 UI。

### 6) `rank-burn-dividend` — 排行榜燃烧分红

**用途：** 烧币进入权重池(80%)+排行榜池(20%)；Top10 分排行榜池。

| 类型 | 方法 | 用途 / 场景 |
|------|------|-------------|
| 读 | `getStatus()` | 双池余额、总燃烧、门槛 |
| 读 | `getUserInfo(user)` · `pendingReward` · `pendingWeight` · `pendingRank` | 个人可领拆分 |
| 读 | `topBurnersList()` | 排行榜 UI：`(address[10], amount[10])` |
| 写 | `burn(amount)` | 烧币上榜；`amount >= minBurnAmount`；先 approve 税币 |
| 写 | `claim()` | 领取权重分红 + 排行榜分红 |

### 前端接入步骤（所有 vaultType）

```typescript
const { found, vault: info } = await vaultPortal.read.tryGetVault([token]);
if (!found) return; // 非路径 B
const vault = getContract({ address: info.vault, abi: vaultAbi }); // 或按 vaultType 选 ABI

const type = await vault.read.vaultType();
const schema = await vault.read.vaultUISchema(); // 动态按钮文案 / 顺序
const status = await vault.read.getStatus();
const me = await vault.read.getUserInfo([userAddress]); // scheduled-buyback 无 getUserInfo，用 getStatus

// 写操作：按 type 分支；需要 token/LP 的先 approve(vault)
```

完整字段与事件见下节 [§金库实例方法完整列表](#金库实例方法完整列表)。

---

## 金库实例方法（完整列表）

地址：`VaultPortal.getVault(token).vault`（先 `tryGetVault` 确认 `found`）。  
收税均为被动入账（`receive`），无需主动调用。  
前端聚合首选：`getStatus()`（池子）+ `getUserInfo(user)`（个人）；写操作见各表「操作」小节。  
**场景说明优先读** [§六金库前端读写](#六金库前端读写方法用途与场景)。

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
| `pendingRequestId()` | `uint256` | 当前 TriggerService 预约 ID；**0=无 pending**（Trigger `requestId` **从 1 起号**，不会与哨兵冲突） |
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

发币时 Portal 已将 CREATE2 预测的 PCS V2 pair 置顶为 `pools[0]`（= `CosmTaxToken.mainPool()`）。**迁移 DEX 后即可** stake LP，无需额外绑 pair。

**UISchema：** `stake` · `withdraw` · `claim`（stake 前 `approve(vault, amount)` LP）

#### 只读

| 方法 | 返回 | 备注 |
|------|------|------|
| `getStatus()` | `StakeStatus` | **首选** · 含 `pair` |
| `getUserInfo(address user)` | `UserInfo` | **首选** |
| `pendingReward(address user)` | `uint256` | 待领 BNB |
| `pair()` | `address` | 发币时 CREATE2 预测；同 `CosmTaxToken.mainPool()` |
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

## TaxSplitter 前端只读 + dispatch

地址：`Portal.getToken(token).taxSplitter`（有税币才有；普通 V7 为 `0`）。

| 方法 | 返回 | 前端用途 |
|------|------|----------|
| `dispatch()` | — | **任何人**；把累账 tax 打出到 mkt / 分红 / LP / 销毁；路径 A/B 详情页都建议提供按钮 |
| `feeConfigV3() view` | `PackedFeeConfigV3` | 四路 bps · **`feeRate`**（曲线 / DEX 均为 **1000**）· `dividendToken` · mkt1–4 地址 |
| `marketAddress() view` | `address` | 营销收款（路径 A=钱包 · 路径 B=金库） |
| `dividendAddress() view` | `address` | CosmDividend；`dividendBps=0` 时可能为 `0` |
| `taxToken() view` | `address` | 关联税币 |
| `requiresMEVProtection() view` | `bool` | `dividendMode=2`（Case3）时为 true；前端应提示「须走 Converter 批量 dispatch」 |
| `dispatchThreshold() view` | `uint256` | Infinity LP fee 信号阈值；BSC V2 税币 UI 可忽略 |

**Case3（`dividendMode=2`）：** 用户按钮应调 `CosmTaxConverter.triggerSplit([token])` 或提示 keeper 调 `batchDispatch`；**不要** EOA 直调 `splitter.dispatch()`（MEV 保护会 revert）。

事件（索引 / 刷新 UI）：`FlapTaxProcessorBondingCurveTax` · `FlapTaxProcessorProcessTaxTokens` · `FlapTaxProcessorDispatchExecuted` · `FlapDispatchReady`（Infinity V7 + lpBps>0 时）。

---

## TaxSplitter dispatch · Keeper（Flap BSC V2 对齐）

BSC **V6 税币**走 PCS V2 + `CosmTaxSplitter`（= Flap `TaxProcessorUniV2`）。

### 链上行为（与 Flap 相同）

| 阶段 | swap 时 | dispatch |
|------|---------|----------|
| 曲线 | `depositQuoteAndSplit` / `BondingCurveTax` → **累账** | ❌ 不自动 |
| DEX | `processTaxTokens` → **累账**（可能卖税成 quote） | ❌ 不自动 |
| 打出 | — | ✅ 须单独 tx：`TaxSplitter.dispatch()`（**任何人**） |
| 迁移上 DEX | Portal 调 `flushLpReserve()` | ✅ 一次性清曲线累账 |

`CosmInfinityCLHook.afterSwap` 会 try 调 `checkAndNotifyDispatch()`。**标准 V7 Infinity + 完整 CosmTaxSplitter**（`lpBps>0`）在 pending LP fee ≥ `dispatchThreshold` 时会 **emit `CosmDispatchReady`**；BSC V2 税币 / 无 V4 源 / Lite processor 仍为 no-op。

`CosmDispatchReady` / `dispatchThreshold` 语义对齐 Flap `FlapDispatchReady`（TaxProcessorUniV4 Infinity LP fee）。Cosm 默认 `dispatchThreshold`：BNB ≈ `0.05 ether`。

### 服务端怎么做（Flap 模式，勿自创 bucket 监听逻辑）

1. **索引** `Portal.getToken(token).taxSplitter`（发币 `TokenCreated`）。
2. **keeper 钱包** 按策略 **主动** 调 `splitter.dispatch()`（permissionless；可用 `dispatchThreshold` 等 **view** 作链下门槛，**不是** Flap 链上信号）。
3. **Case3**（`dividendMode=2` / `requiresMEVProtection()==true`）：**不要** EOA 直调 splitter；须由 **`CosmTaxConverter` proxy**（`Portal.defaultTaxConverter()` 或发币时写入的 `converter`）调：
   - 单 token：`converter.batchDispatch([taxSplitter])`（需 `DISPATCHER_ROLE`，MEV RPC）
   - 或 permissionless 批量：`converter.batchDispatchPermissionless([taxSplitter, ...])`（自动跳过 MEV-protected processor）
   - 批量清 split：`converter.triggerSplit([taxToken, ...])` → 内部调 `taxProcessor.dispatch()`
4. **分红领取**：用户自己 `CosmDividend.withdrawDividends()`，无需 keeper。

**不要**把 `BondingCurveTax` / `ProcessTaxTokens` 当成 Flap 官方的 dispatch 触发信号——Flap BSC V2 也没有这类链上提醒。

### scheduled-buyback（另一套 keeper）

见下方 [§CosmTriggerService](#cosmtriggerservice-方法)：监听 **`CosmTriggerRequested`** → `TriggerService.trigger`（`TRIGGER_ROLE`）。与 TaxSplitter dispatch **无关**。

---

## CosmTaxConverter 方法（Flap TaxDistributionHelper）

地址（proxy）：`0x3725B42BfDa1Ef33a7eEb8c0465675Ee72aa0001`  
impl 可升级；前端 / keeper **只调 proxy**。

| 方法 | 调用方 | 备注 |
|------|--------|------|
| `batchDispatch(address[] taxProcessors) → successCount` | **`DISPATCHER_ROLE`** | 全量 dispatch（含 MEV-protected）；须 MEV RPC |
| `batchDispatchPermissionless(address[] taxProcessors) → successCount` | 任何人 | 跳过 `requiresMEVProtection()==true` 的 processor |
| `triggerSplit(address[] taxTokens)` | 任何人 | 对每个 taxToken 调内部 `triggerSingleSplit`；MEV processor 须调用方有 `DISPATCHER_ROLE` |
| `triggerSingleSplit(address taxToken)` | **仅** converter 自身（`triggerSplit` 内部）或 **`DISPATCHER_ROLE`** | 前端/EOA **勿直调**；单币 `dispatch` |
| `batchDistributeDividend(dividends, usersList) → successCounts` | 任何人 | 批量 `CosmDividend.distributeDividend` |
| `checkAndCacheMEVProtection(processor) → bool` | 任何人 | 缓存 Case3 MEV 标志 |
| `setDispatcher(addr, enabled)` | admin | 授予/撤销 `DISPATCHER_ROLE` |

Case3 发币：`converter` 字段留空 → Portal 写入 `defaultTaxConverter()`（即本 proxy）。  
TaxSplitter 内部重逻辑经 **delegatecall** 至链上 `CosmTaxSplitterDispatchImpl`（协议内部，前端勿依赖）。

事件：`CosmDispatchCalled` · `CosmSplitTriggered` · `CosmSplitFailed` · `CosmBatchOperationCalled` · `CosmMEVProtectionCached`

---

## CosmTriggerService 方法

地址（proxy）：`0x0B8dD41a583f456DD733b2a35CA28D61F6204e08`  
默认：`getFee()=0.0002 ether` · `getMaxCallbackGas()=2_000_000` · 以链上 `getFee()` / `getMaxCallbackGas()` 为准。  
**requestId 从 1 起号**（`++_requestCount`）；`0` 永不发放，供金库 `pendingRequestId==0` 表示「无 pending」。

scheduled-buyback 金库实现 `ITriggerReceiver.trigger(uint256 requestId)`；keeper 执行 `CosmTriggerService.trigger(requestId)`。

| 方法 | 调用方 | 备注 |
|------|--------|------|
| `requestTrigger(uint64 executeAfter) payable → requestId` | 请求方合约 | `msg.value ≥ getFee()`；`executeAfter=0` 立即可执行；**返回值 ≥ 1** |
| `trigger(requestId)` / `triggerMultiple(ids)` | **keeper（`TRIGGER_ROLE`）** | 回调 requester；`requestId=0` 无效 |
| `retryTrigger(requestId)` | 任何人 | 仅 `FAILED` |
| `getFee` / `getMaxCallbackGas` / `getRequest` / `getRequestCount` / `isRequestReady` | 前端/服务端 | 查询；合法 id 为 `1..=getRequestCount()` |
| `getRequests` / `getRequestsPaginated` / `getRequestsByRequesterPaginated` | 服务端 | 分页（1-based 索引） |
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

**CosmToken（V7）**

| 方法 | 备注 |
|------|------|
| ERC20 + `permit` | 卖出前 `approve(Portal)` |
| `metaURI()` | 发币 `meta` URI |
| `maxSupply()` | `1e9 ether` |

**CosmTaxToken（V6）**

| 方法 | 备注 |
|------|------|
| ERC20 + `permit` | DEX 阶段 transfer 可能扣税 |
| `buyTaxRate()` / `sellTaxRate()` | 当前有效税率（万分比） |
| `taxRate()` | Flap 兼容：`max(buy,sell)` |
| `taxProcessor()` | TaxSplitter |
| `dividendContract()` | CosmDividend；未开分红为 `0` |
| `mainPool()` / `pair()` | PCS V2 pair（发币 CREATE2 预测；`pools[0]`） |
| `vault()` / `dexTaxExempt()` | 路径 B 金库地址（同值） |
| `quoteToken()` | 发币锁定的 quote |
| `state()` | `0` BondingCurve · `1` Migrating · `2` TaxEnforcedAntiFarmer · `3` TaxEnforced · `4` TaxFree |
| `getPoolStateData()` | 一次读：状态 · 买卖税 · 清算阈值 · 税到期 · anti-farmer 到期 |
| `liquidationThreshold()` | DEX 大额卖触发清算阈值 |
| `TransferFlapToken` / `TransferCosmToken` | from/to **不** indexed（索引器用） |
| `PoolStateChanged` | DEX 税阶段切换 |

卖出：曲线 → `approve(Portal)`；已迁移 → 站内仍可走 Portal，或 `approve(PCS V2 Router)`。  
DEX 积税清算：卖到主池且达到 `liquidationThreshold`（无 token 侧 `dispatchTax`）。

---

## 常见 Revert 错误（前端）

`simulateContract` / 钱包报错时可对照（Portal 为主）：

| 错误 | 常见原因 |
|------|----------|
| `VanityMismatch(token, expected)` | salt 低 16 bit 不对；重新搜 vanity |
| `SaltAlreadyUsed` / `CosmSaltLockedByAnotherUser` | salt 已发币或被他人锁定 |
| `InvalidParams` | 税率/分配/路径参数不合法（如 V7 设了 tax、路径 B `mktBps=0`） |
| `UnsupportedQuoteToken` | quote 不在白名单 |
| `UnknownToken` | 地址未在 Portal 注册 |
| `NotTradable` / `NotDex` | 操作与 `status` 不符 |
| `Slippage` | `minOutputAmount` 过高 |
| `InsufficientValue` | BNB `msg.value` 不足 |
| `CircuitBroken(requiredMask)` | 全局/V7 熔断（读 `bitFlags()`） |
| `SpammerBlocked` | 发币地址在黑名单 |
| `TokenWithExtensionNotSupported` | 须改 `swapExactInputV3` |
| `NativeToQuoteSwapNotSupported` | ERC20 quote 未开 native 一键买 |
| `NotReadyForMigration` | 流通量未达阈值 |
| `NotBeneficiary` / `NotDelegatedClaimer` | `claim` / `delegateClaim` 权限 |

VaultPortal：`InvalidTaxAllocation` · `InvalidMktBps` · `InvalidFeeConfig` · 工厂 `onBeforeLaunch` 返回 `(false, reason)`。

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
export const PORTAL = "0xb4B057dEFda3822786F998FC54Aa93440caEDb6c";
export const VAULT_PORTAL = "0xE3BDE2e728F5a9a5FD5bdda87B067a55bf593183";
export const TRIGGER_SERVICE = "0x0B8dD41a583f456DD733b2a35CA28D61F6204e08";
export const TAX_CONVERTER = "0x3725B42BfDa1Ef33a7eEb8c0465675Ee72aa0001";
export const VANITY_SUFFIX_TAX = 0x0111;
export const VANITY_SUFFIX_STANDARD = 0x0222;
export const DEX_SUPPLY_THRESH = 800_000_000n * 10n ** 18n;
export const TOTAL_SUPPLY = 1_000_000_000n * 10n ** 18n;

/** Flap newTokenV6WithVault dividendToken magic（Cosm 推荐用 dividendMode） */
export const MAGIC_DIVIDEND_SELF = "0xfEEDFEEDfeEDFEedFEEdFEEDFeEdfEEdFeEdFEEd";
export const MAGIC_DIVIDEND_COMPUTED = "0xC0Dec0dec0DeC0Dec0dEc0DEC0DEC0DEC0DEC0dE";

/** TaxSplitter feeRate：曲线 / DEX 均为 1000（10% of tax → feeReceiver，对齐 Flap BSC 主网） */
export const TAX_SPLITTER_FEE_CURVE = 1000;
export const TAX_SPLITTER_FEE_DEX = 1000;
export const V7_BONDING_CURVE_FEE_BPS = 125;

export const QUOTE = {
  BNB: "0x0000000000000000000000000000000000000000",
  USDT: "0x55d398326f99059fF775485246999027B3197955",
  USDC: "0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d",
  USD1: "0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d",
  COSM: "0x0D6aE45c96eC4df860300087462266e19140F6dc",
} as const;

export const VAULT_FACTORY = {
  split: "0x98B345fde625DAc83E1bB478996f6a3FB2deC93e",
  scheduledBuyback: "0x2F9BB21010e28983895aD50fff7bd80a9D7637CE",
  burnDividend: "0xFfa993aCaFE3F6B13E68FF8DC388aC0BBc5383E5",
  lpStakingDividend: "0xAec0EcFd308a24039aC299A7Fb8Da165EC405074",
  tokenStakingDividend: "0xf14a4Aa3D702af1416dF91e8372E7f9101F7c3a1",
  rankBurnDividend: "0x9abC0D6516d4a023280EADaB82397b99521AB98f",
} as const;
```
