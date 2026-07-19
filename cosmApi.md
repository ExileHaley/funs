# Cosm 前端对接 API

> BSC 主网 · Chain ID `56` · 版本 `cosm-v0.6.0-flap-align`  
> Vanity 后缀（代币地址**低 16 bit**须匹配）：有税 `0x0111` · 无税 `0x0222`（读 `vanitySuffixTax` / `vanitySuffixStandard`）  
> 代理：Portal / VaultPortal / TriggerService 为 **TransparentUpgradeableProxy + ProxyAdmin**；前端只用 **proxy**

## 合约地址

> 2026-07-20 部署。旧 UUPS（`0xf954…` / `0xF7D1…`）已废弃。

| 合约 | 地址 | 用途 |
|------|------|------|
| ProxyAdmin | `0xA1f85d13e55B9CeB6Dc295157A06944ea5fdCFE9` | 升级管理（勿当业务入口） |
| CosmPortal **proxy** | `0x8ADab64A1A62bEa555CAe71A8061E10cE9d639F1` | 发币、曲线买卖、状态查询、迁移 |
| CosmPortal impl | `0xFEa31e0Cb6b718b30952c225fb1F274148587B09` | 实现（勿直接调用） |
| CosmVaultPortal **proxy** | `0x6CB77C7Cdf15153D9F31b89305875E90D6BB895C` | 税收币路径 B / 未上架 clone |
| CosmVaultPortal impl | `0x777Eb0Ff8f11D53F4b6eaCFF25276445E5216e71` | 实现（勿直接调用） |
| CosmTriggerService **proxy** | `0x7277712198e66A3B4A7c0e84062961f44CEe07f7` | 服务端延期回调（gasFee=`0.001` BNB） |
| CosmTriggerService impl | `0xa05967AB812e18684fCf4843ee172D6C48f9E5DF` | 实现（勿直接调用） |
| CosmToken impl | `0x2642fAa441d144EB59691924209662B09bf33Bfb` | 普通币 CREATE2 模板 |
| CosmTaxToken impl | `0x680036C3068938c6BFD1268Ca6e63F2E7eD555d4` | 税收币 CREATE2 模板 |
| CosmMigratorV2 | `0x4a942aD0E3A09b19258E6A614bfcD9a250a41688` | Tax → PCS V2 |
| CosmMigratorV3 | `0xE94c15A927c056146fa946145ADE332d90c0aCC2` | Standard → PCS V3 |
| CosmTaxSplitter impl | `0x39bc6ACd1B05eA530FF502560cA898099d09AA88` | 四路拆分模板 |
| CosmDividend impl | `0xa971E416863f1423c0c35a3883c1b23672D681d5` | 持币分红模板 |

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

## 前端调用路由（先看这里）

| 场景 | 合约 · 方法 | 说明 |
|------|-------------|------|
| 发普通币 | `Portal.newTokenV7` | `isTaxed=false` |
| 发税收币 · 税进钱包 | `Portal.newTokenV6` | 路径 A，`beneficiary`=钱包 |
| 发税收币 · mkt 进金库 | `VaultPortal.newTokenV6WithVault` | 路径 B；内部再调 Portal |
| 未上架按模板 clone | `VaultPortal.previewCloneFromToken` → `newTokenV6CloneFrom` | 源币须已挂金库 |
| 列表 / 行情 | `Portal.getTokenV8Safe` | 高频轻量 |
| 详情扩展 | `Portal.getToken` | `taxSplitter` / `dividend` / `vault` / `feeProfile` |
| 曲线报价 / 买卖 | `Portal.quoteExactInput` / `swapExactInput` | 仅 `status=Tradable` |
| 手动迁移 | `Portal.migrateToDex` | 通常自动迁；漏迁时点 |
| 迁移后交易 | PancakeSwap | `status=DEX` 后不走 Portal |
| 持币分红领取 | `CosmDividend.withdrawDividends` | 地址=`getToken.dividend` |
| 金库玩法 | 各 Vault 实例方法 | 地址=`VaultPortal.getVault(token).vault` |
| 金库表单自描述 | `factory.vaultDataSchema` / `vault.vaultUISchema` | 动态表单 |
| 延期回调 | `TriggerService.requestTrigger` / `trigger` | Keeper 持 `TRIGGER_ROLE` |

### 状态机

```
Invalid(0) ──发币成功──► Tradable(1) ──达阈值/手动迁移──► DEX(2)
                            │                              │
                       Portal 曲线买卖                  PancakeSwap
```

`circulatingSupply >= 800_000_000 ether` 可迁移；曲线阶段最后一笔买入可自动触发。

---

## CosmTypes（前端对接必读）

源码：`contracts/CosmTypes.sol`。ABI 编码顺序与下表字段顺序一致。  
常量：`COSM_VERSION_TAX_V3 = 6` · `COSM_VERSION_STANDARD_V3 = 7`。

### `TokenStatus` — 代币生命周期

| 值 | 名称 | 前端含义 |
|----|------|----------|
| `0` | `Invalid` | 未发行 / 未知地址 |
| `1` | `Tradable` | 曲线阶段，买卖走 Portal |
| `2` | `DEX` | 已迁移，买卖走 PancakeSwap |

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
| `antiFarmerDuration` | `uint256` | 迁移后 anti-farmer 秒数。`0`=跳过，直接仅主池收税；上限 365 天 |
| `taxDuration` | `uint256` | 收税总时长（秒，含 anti-farmer）。`0`=永不过期；若 >0 须 ≥ `antiFarmerDuration`；上限约 100 年 |

```solidity
struct TaxAllocation {
    uint16 mktBps;               // 营销
    uint16 deflationBps;         // 回购销毁
    uint16 dividendBps;          // 持币分红
    uint16 lpBps;                // 加 LP
    uint256 minimumShareBalance; // 分红门槛（wei）
    uint8 dividendMode;          // 0=quote 1=本币 2=其他
    address dividendToken;       // mode=2 时填
    uint256 antiFarmerDuration;  // 秒；0=跳过
    uint256 taxDuration;         // 秒；0=永久
}
```

---

### `LaunchParams` — `Portal.newTokenV6` / `newTokenV7` 入参

| 字段 | 类型 | 前端备注 | V7 普通币 | 路径 A（无金库） | 路径 B（VaultPortal 内部组装，勿手填调 Portal） |
|------|------|----------|-----------|------------------|--------------------------------------------------|
| `name` | `string` | ERC20 `name()` | 必填 | 必填 | 必填 |
| `symbol` | `string` | ERC20 `symbol()` | 必填 | 必填 | 必填 |
| `meta` | `string` | 元数据 URI（IPFS/HTTPS）→ `metaURI()`；图标/描述写 JSON，不单独上链 | 可空 | 可空 | 可空 |
| `salt` | `bytes32` | CREATE2 salt；预测地址**低 16 bit**须匹配 vanity，否则 `VanityMismatch` | 用 `standardTokenImpl` 搜 `0x0222` | 用 `taxTokenImpl` 搜 `0x0111` | 同左（tax） |
| `quoteToken` | `address` | 曲线支付币；`address(0)`=BNB。发币后锁定全生命周期 | 白名单五选一 | 白名单五选一 | **仅 BNB** |
| `quoteAmt` | `uint256` | 发币同时首买量（最小单位）。`0`=只发不买 | 按需 | 按需 | 按需 |
| `beneficiary` | `address` | 税收 mkt 收款方 | 必须 `address(0)` | 用户钱包 | Vault 地址（Portal 内部写入） |
| `buyTaxBps` | `uint16` | 买入税（万分比，100=1%） | `0` | 与 sell 至少一侧 >0 | 同左 |
| `sellTaxBps` | `uint16` | 卖出税（万分比） | `0` | 同上 | 同上 |
| `isTaxed` | `bool` | 须与调用方法一致 | `false` | `true` | `true` |
| `tax` | `TaxAllocation` | 见上表 | 全 0 | 四路合计 10000（或全 0） | 同左 |

**支付：**

- BNB：`msg.value >= quoteAmt`（多余会退）
- ERC20：先 `quote.approve(Portal, quoteAmt)`，`msg.value` 须为 `0`

```solidity
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
}
```

---

### `QuoteParams` — `Portal.quoteExactInput`（只读估价）

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

**支付：**

- 买入 BNB：`msg.value >= inputAmount`
- 买入 ERC20：先 `approve(Portal, inputAmount)`
- 卖出：先 `meme.approve(Portal, inputAmount)`，`msg.value=0`
- 仅 `status == Tradable(1)` 可调

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
}
```

---

### `TokenState` — `Portal.getToken`（详情 / 索引）

列表高频请用 `getTokenV8Safe`；需要分红入口、拆分器、金库、费档位时用本结构。

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `status` | `TokenStatus` | `0/1/2`，决定走 Portal 还是 PCS |
| `tokenVersion` | `uint8` | `6`=CosmTaxToken · `7`=CosmToken |
| `quoteToken` | `address` | 发币锁定的 quote；BNB=`0` |
| `reserve` | `uint128` | 曲线 quote 储备；迁移后清零 |
| `circulatingSupply` | `uint128` | 曲线流通供给；≥ `800_000_000e18` 可迁移 |
| `buyTaxBps` | `uint256` | 买入税率（万分比） |
| `sellTaxBps` | `uint256` | 卖出税率（万分比） |
| `beneficiary` | `address` | mkt 收款方（钱包或金库） |
| `progress` | `uint256` | `circulatingSupply * 10000 / DEX_SUPPLY_THRESH`；迁完=`10000`。进度条用 |
| `tax` | `TaxAllocation` | 发币锁定的四路配置（只读展示） |
| `taxSplitter` | `address` | CosmTaxSplitter；普通币为 `0` |
| `dividend` | `address` | CosmDividend；未开分红为 `0`。领取调其 `withdrawDividends` |
| `pool` | `address` | 迁移后池；Tax=PCS V2 pair · 普通=PCS V3 pool；未迁=`0` |
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
    uint256 progress;            // 0..10000
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
| `status` | `uint8` | 同 TokenStatus：`0/1/2` |
| `reserve` | `uint256` | 曲线 quote 储备 |
| `circulatingSupply` | `uint256` | 流通供给 |
| `price` | `uint256` | 当前曲线单价（quote wei / 1 token）；DEX 阶段用阈值供给估算 |
| `tokenVersion` | `uint8` | `6` / `7` |
| `r` / `h` / `k` | `uint256` | 曲线虚拟参数；一般 UI 不展示，高级图表可用 |
| `dexSupplyThresh` | `uint256` | 恒 `800_000_000e18`，进度条分母 |
| `quoteTokenAddress` | `address` | quote；BNB=`0` |
| `nativeToQuoteSwapEnabled` | `bool` | `true`=BNB quote → 买入必须带 `msg.value` |
| `extensionID` | `bytes32` | Flap 占位；Cosm 恒 `0x0`，可忽略 |
| `buyTaxRate` | `uint256` | 同 `buyTaxBps` |
| `sellTaxRate` | `uint256` | 同 `sellTaxBps` |
| `pool` | `address` | 迁移后池；未迁=`0` |
| `progress` | `uint256` | 万分比；`10000`=可迁/已迁 |
| `lpFeeProfile` | `uint8` | Flap 占位；Cosm 恒 `0`，可忽略 |
| `dexId` | `uint8` | 迁移后交易入口提示：`2`=PCS V2（税币）· `3`=PCS V3（普通币） |

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
    bool nativeToQuoteSwapEnabled; // true → 用 msg.value 买
    bytes32 extensionID;           // 恒 0
    uint256 buyTaxRate;
    uint256 sellTaxRate;
    address pool;
    uint256 progress;
    uint8 lpFeeProfile;            // 恒 0
    uint8 dexId;                   // 2=V2 3=V3
}
```

---

### VaultPortal 结构体（路径 B / 金库，非 CosmTypes 文件但前端常用）

#### `NewTokenV6WithVaultParams` — `VaultPortal.newTokenV6WithVault`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `name` / `symbol` / `meta` | `string` | 同 LaunchParams |
| `salt` | `bytes32` | 必须用 **taxTokenImpl** 搜到的 `0x0111` salt |
| `quoteToken` | `address` | **仅** `address(0)`（BNB） |
| `quoteAmt` | `uint256` | 首买；BNB 时 `msg.value`；ERC20 仍 approve **Portal** |
| `buyTaxBps` / `sellTaxBps` | `uint16` | 至少一侧 >0 |
| `mktBps` | `uint16` | **必须 >0**（mkt 进金库） |
| `deflationBps` / `dividendBps` / `lpBps` | `uint16` | 与 `mktBps` 合计 = 10000 |
| `minimumShareBalance` | `uint256` | 同 TaxAllocation |
| `dividendMode` | `uint8` | 同 TaxAllocation |
| `dividendToken` | `address` | 同 TaxAllocation |
| `antiFarmerDuration` | `uint256` | 同 TaxAllocation |
| `taxDuration` | `uint256` | 同 TaxAllocation |
| `vaultFactory` | `address` | 已注册且 `enabled` 的工厂（见工厂地址表） |
| `vaultData` | `bytes` | 按该工厂 `vaultDataSchema()` 编码 |

#### `NewTokenV6CloneFromParams` — `VaultPortal.newTokenV6CloneFrom`

字段与上表相同，**无** `vaultFactory`（从 `sourceToken` 解析）。发币前先 `previewCloneFromToken(source)`。

#### `VaultInfo` — `getVault` / `tryGetVault`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `vault` | `address` | 金库实例；未绑定=`0` |
| `isOfficial` | `bool` | 发币时快照：是否官方工厂 |
| `riskLevel` | `RiskLevel` | `0` UNVERIFIED · `1` LOW · `2` MEDIUM · `3` HIGH |
| `category` | `VaultCategory` | `0` NONE · `1` SPLIT · `2` BUYBACK · `3` DIVIDEND · `4` AIRDROP · `5` GAME |
| `vaultFactory` | `address` | 创建该金库的工厂 |

#### `VaultFactoryInfo` — `getVaultFactory`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `registered` | `bool` | 未注册不可选 |
| `enabled` | `bool` | 禁用不可发币 |
| `official` | `bool` | UI「官方」标签 |
| `riskLevel` | `RiskLevel` | 风险展示 |
| `category` | `VaultCategory` | 分类筛选 |

#### `TaxVaultTemplatePreview` — `previewCloneFromToken`

| 字段 | 类型 | 前端备注 |
|------|------|----------|
| `ok` | `bool` | `false` 则不可 clone |
| `vault` / `vaultFactory` | `address` | 源模板 |
| `vaultType` | `string` | 如 `"split"` |
| `dataSchema` | `VaultDataSchema` | 克隆时填 `vaultData` 用 |
| `uiSchema` | `VaultUISchema` | 操作区预览 |

### 金库 UI Schema 类型（`vaultDataSchema` / `vaultUISchema`）

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

- V7 → `standardTokenImpl` + `0x0222`
- V6 / WithVault → `taxTokenImpl` + `0x0111`
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
python3 tools/find_vanity.py --predict --portal 0x8ADab64A1A62bEa555CAe71A8061E10cE9d639F1 --tax
python3 tools/find_vanity.py --predict --portal 0x8ADab64A1A62bEa555CAe71A8061E10cE9d639F1
```

### 2. 支付再发币

| quote | 方式 |
|-------|------|
| BNB | `msg.value >= quoteAmt` |
| ERC20 | `approve(Portal, quoteAmt)`，`msg.value=0` |

路径 B 同样 approve **Portal**（不是 VaultPortal）。

路径 B 内部：`predictTokenAddress` → `onBeforeLaunch` → 工厂 `newVault` → `Portal.newTokenV6` → 写入 `vaults[token]`。

---

## CosmPortal 方法

地址（proxy）：`0x8ADab64A1A62bEa555CAe71A8061E10cE9d639F1`

### 发币（用户）

| 方法 | 返回 | 备注 |
|------|------|------|
| `newTokenV7(LaunchParams p) payable` | `address token` | 普通币；`isTaxed` 必须 false |
| `newTokenV6(LaunchParams p) payable` | `address token` | 路径 A；`beneficiary`≠0；税率至少一侧 >0 |

### 曲线交易（用户）

| 方法 | 返回 | 备注 |
|------|------|------|
| `quoteExactInput(QuoteParams) view` | `uint256 outputAmount` | 已扣协议费/税后的估算 |
| `swapExactInput(SwapParams) payable` | `uint256 outputAmount` | 仅 Tradable |
| `migrateToDex(address token)` | `address pool` | 流通量须已达阈值 |

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
| `version() pure` | `string` | `cosm-v0.6.0-flap-align` |
| `vanitySuffixFor(bool isTaxed) view` | `uint16` | 推荐取后缀 |
| `getFeeRate() view` | `(buy, sell)` | 全局默认买卖协议费 |
| `getMigrationFeeRate() view` | `(liquidity, reserve)` | 迁移费；默认 `0,0` |
| `feeExempt(address) view` | `bool` | 交易者是否豁免协议费 |

### 只读配置

| 方法 | 含义 |
|------|------|
| `TOTAL_SUPPLY()` | `1e9 ether` |
| `DEX_SUPPLY_THRESH()` | `8e8 ether` |
| `BPS()` | `10000` |
| `vanitySuffixTax()` / `vanitySuffixStandard()` | `0x0111` / `0x0222` |
| `protocolFeeBps()` / `protocolSellFeeBps()` | 默认买/卖协议费 |
| `liquidityFeeBps()` / `reserveFeeBps()` | 迁移费 |
| `standardTokenImpl()` / `taxTokenImpl()` | CREATE2 模板 |
| `feeReceiver()` / `pcsRouter()` / `nonce()` | 配置 / 统计 |
| `migratorV2()` / `migratorV3()` | Tax→V2 · 普通→V3 |
| `taxSplitterImpl()` / `dividendImpl()` | 模板 |
| `owner()` | 业务 owner（非 ProxyAdmin） |

### Owner 运营（前端一般不调）

`setFeeReceiver` · `setProtocolFeeRates` · `setProtocolFeeBps` · `setMigrationFeeRates` · `setFeeProfile` · `setFeeExemption` · `transferOwnership`

### 事件

| 事件 | 用途 |
|------|------|
| `TokenCreated(ts, creator, nonce, token, name, symbol, meta, tokenVersion)` | 刷新列表 |
| `TokenBought` / `TokenSold` | 成交 / 价格 |
| `TokenMigrated(ts, token, pool, tokenLiquidity, quoteLiquidity, migratorKind)` | 切 PCS；`migratorKind`：2=V2 · 3=V3 |
| `MigrationFeesPaid(token, reserveFee, liquidityFee, tokensBurned)` | 迁移抽成（费率为 0 时可能不发） |

---

## CosmVaultPortal 方法

地址（proxy）：`0x6CB77C7Cdf15153D9F31b89305875E90D6BB895C`

| 方法 | 返回 | 调用方 | 备注 |
|------|------|--------|------|
| `newTokenV6WithVault(NewTokenV6WithVaultParams) payable` | `token` | 用户 | 官方路径 B |
| `newTokenV6CloneFrom(address source, NewTokenV6CloneFromParams) payable` | `token` | 用户 | 未上架 clone |
| `previewCloneFromToken(address source) view` | `TaxVaultTemplatePreview` | 前端 | `ok=false` 不可 clone |
| `getVault(address taxToken) view` | `VaultInfo` | 前端 | 已知路径 B |
| `tryGetVault(address taxToken) view` | `(found, VaultInfo)` | 前端 | 不确定时用 |
| `getVaultFactory(address factory) view` | `VaultFactoryInfo` | 前端 | 选工厂标签 |
| `portal() view` | `CosmPortal` | 前端 | 关联 Portal |
| `registerVaultFactory(...)` | — | **owner** | 部署注册 |

事件：`FlapTaxVaultTokenCreated(token, vault, vaultFactory)` · `VaultFactoryRegistered(...)` · `TokenClonedFromTemplate(source, newToken, vaultFactory)`

---

## 金库工厂（Vault Factory）

前端用工厂做**表单发现**；`newVault` **仅** VaultPortal 内部调用。

| 方法 | 返回 | 前端用途 |
|------|------|----------|
| `vaultType() pure` | `string` | 玩法名 |
| `vaultDataSchema() pure` | `VaultDataSchema` | 动态生成发币表单并 `abi.encode` |
| `isQuoteTokenSupported(address) view` | `bool` | 全部工厂仅 BNB=`address(0)` |
| `factorySpecVersion() pure` | `string` | 当前 `"v2.2"` |
| `tokenCreationPolicies() pure` | `FactoryPolicy[]` | UI 约束提示（如 quote=BNB） |
| `onBeforeLaunch(bytes) view` | `(bool, string)` | VaultPortal 发币前校验；前端可预检 |
| `newVault(...)` | `address` | **勿直接调** |

### 工厂地址与 vaultData

| vaultType | 工厂地址 | vaultData 编码 |
|-----------|----------|----------------|
| `split` | `0xEe6da751a5E2265d92405Bc22297eaAc07956B05` | `abi.encode(Recipient[])` · `Recipient{address recipient; uint16 bps}` · 1–10 人去重 · bps 合计 10000 |
| `scheduled-buyback` | `0x7cCE9d8bb10136aB66408EA7233A25cE26e0620D` | `abi.encode(uint8 triggerMode, uint8 buybackMode, uint256 intervalSeconds, uint256 minBnbAmount, uint256 maxBnbPerTrigger)` · trigger:`0=time 1=amount 2=both` · buyback:`0=token 1=LP` |
| `omnidrop` | `0x380c0323f42029B7e16B1E1ce211399a73a719a7` | `abi.encode(uint8 dropMode, address[] recipients, uint256 minBnbToDrop)` · drop:`0=token 1=LP 2=BNB` |
| `silent` | `0x4532a7292fC7792cD15362C5F72bA5033C0b543a` | `abi.encode(uint256 maxSellPerTx)` |
| `burn-dividend` | `0xDBC5C7655178933FE6938deD89A646451cd3eDf2` | 空 `0x` |
| `lp-staking-dividend` | `0x2d3038DAB629Ff27EE0c1A5063Fcd4E88F178c8D` | 空 `0x`（迁移后 `setPair`） |
| `token-staking-dividend` | `0x48B83e78E40924Cb61A3Bf2fC90a7C1992e26992` | 空 `0x` |
| `rank-burn-dividend` | `0x893Ab0517e93D392728f397464A22c0A3F74D213` | `abi.encode(uint256 minBurnAmount)`；也可空=`0` |

---

## 金库实例方法（完整列表）

地址：`VaultPortal.getVault(token).vault`（先 `tryGetVault` 确认 `found`）。  
收税均为被动入账（`receive` / ERC20 转入），无需主动调用。

### 通用（全部 vaultType）

| 方法 | 返回 | 调用方 | 备注 |
|------|------|--------|------|
| `factory() view` | `address` | 前端 | 创建工厂 |
| `taxToken() view` | `address` | 前端 | 关联税币 |
| `creator() view` | `address` | 前端 | 发币人 |
| `quoteToken() view` | `address` | 前端 | 恒 BNB=`0` |
| `vaultType() view` | `string` | 前端 | 如 `"split"` |
| `description() view` | `string` | 前端 | 动态文案（可含余额） |
| `vaultUISchema() view` | `VaultUISchema` | 前端 | 渲染操作区 |

### `split` — 分账金库

**UISchema 方法：** `dispatch` · `claim`

| 方法 | 签名要点 | 调用方 | 备注 |
|------|----------|--------|------|
| `recipients(i)` | → `(address recipient, uint16 bps)` | 前端 | 收款人列表项 |
| `userBalances(user)` | → `(uint128 accumulated, uint128 claimed)` | 前端 | 个人账本 |
| `TOTAL_BPS()` | → `10000` | 前端 | 常量 |
| `recipientCount()` | → `uint256` | 前端 | 人数 |
| `getRecipientsInfo()` | → `RecipientInfo[]` | 前端 | `{recipient,bps,accumulated,claimed}` |
| `accrue()` | — | 任何人 / keeper | 把未记账余额记入账本 |
| `dispatch()` | — | **keeper 推荐** / 任何人 | 推送给全部收款人 |
| `claim(address user)` | — | 用户或代领 | 单人领取 |

### `scheduled-buyback` — 定时回购

**UISchema 方法：** `canTrigger`（用户只读；执行靠 keeper）

| 方法 | 签名要点 | 调用方 | 备注 |
|------|----------|--------|------|
| `OPERATOR_ROLE()` | → `bytes32` | 前端/keeper | 角色 id |
| `router()` / `triggerMode()` / `buybackMode()` | 配置 | 前端 | — |
| `intervalSeconds()` / `minBnbAmount()` / `maxBnbPerTrigger()` | 配置 | 前端 | — |
| `lastTriggeredAt()` / `totalBurned()` | 状态 | 前端 | — |
| `canTrigger()` | → `bool` | 前端 / keeper | 是否可触发 |
| `trigger()` | → `uint256 burnedAmount` | **keeper（`OPERATOR_ROLE`）** | 回购并销毁 |
| `hasRole` / `grantRole` / `revokeRole` / … | AccessControl | admin | 运营 |

### `omnidrop` — 固定名单空投

**UISchema 方法：** `airdrop`

| 方法 | 签名要点 | 调用方 | 备注 |
|------|----------|--------|------|
| `router()` / `dropMode()` / `minBnbToDrop()` | 配置 | 前端 | — |
| `fixedRecipientCount()` | → `uint256` | 前端 | — |
| `fixedRecipients()` | → `address[]` | 前端 | 名单 |
| `airdrop()` | → `uint256 count` | **用户 / 任何人** | 余额不足 `min` 时返回 0 |

### `silent` — 税收 BNB 接盘销毁

**UISchema 方法：** `quoteSilentSell` · `silentSell`（需 `approve(taxToken, vault, amount)`）

| 方法 | 签名要点 | 调用方 | 备注 |
|------|----------|--------|------|
| `router()` / `maxSellPerTx()` / `totalBurned()` | 配置/统计 | 前端 | — |
| `quoteSilentSell(uint256 tokenAmount)` | → `uint256 bnbOut` | 前端 | 预估 |
| `silentSell(uint256 tokenAmount)` | → `uint256 bnbOut` | **用户** | 卖给金库并烧币 |

### `burn-dividend` — 销毁换分红

**UISchema 方法：** `burn` · `claim`（burn 需 approve taxToken）

| 方法 | 签名要点 | 调用方 | 备注 |
|------|----------|--------|------|
| `totalBurned()` / `accRewardPerShare()` | 池子状态 | 前端 | — |
| `burned(user)` / `rewardDebt(user)` | 个人状态 | 前端 | — |
| `pendingReward(address user)` | → `uint256` | 前端 | 待领 BNB |
| `burn(uint256 amount)` | — | **用户** | 销毁税币并结算 |
| `claim()` | → `uint256 amount` | **用户** | 领取 BNB |

### `token-staking-dividend` — 质押本币分红

**UISchema 方法：** `stake` · `withdraw` · `claim`（stake 需 approve taxToken）

| 方法 | 签名要点 | 调用方 | 备注 |
|------|----------|--------|------|
| `totalStaked()` / `accRewardPerShare()` | 池子 | 前端 | — |
| `staked(user)` / `rewardDebt(user)` | 个人 | 前端 | — |
| `pendingReward(address user)` | → `uint256` | 前端 | — |
| `stake(uint256 amount)` | — | **用户** | 质押本币 |
| `withdraw(uint256 amount)` | — | **用户** | 解押 |
| `claim()` | → `uint256 amount` | **用户** | 领 BNB |

### `lp-staking-dividend` — 质押 LP 分红

**UISchema 方法：** `stake` · `withdraw` · `claim`（stake 需 approve LP）

| 方法 | 签名要点 | 调用方 | 备注 |
|------|----------|--------|------|
| `pair()` | → `address` | 前端 | 迁移后须先设置 |
| `totalStaked()` / `accRewardPerShare()` | 池子 | 前端 | — |
| `staked(user)` / `rewardDebt(user)` | 个人 | 前端 | — |
| `pendingReward(address user)` | → `uint256` | 前端 | — |
| `setPair(address pair_)` | — | **creator（一次）** | DEX 迁移后绑定 V2 pair |
| `stake(uint256 amount)` | — | **用户** | 质押 LP |
| `withdraw(uint256 amount)` | — | **用户** | 解押 LP |
| `claim()` | → `uint256 amount` | **用户** | 领 BNB |

### `rank-burn-dividend` — 燃烧榜分红

**UISchema 方法：** `burn` · `claim`（80% 权重池 + 20% Top10）

| 方法 | 签名要点 | 调用方 | 备注 |
|------|----------|--------|------|
| `WEIGHT_POOL_BPS()` / `RANK_POOL_BPS()` / `BPS()` / `TOP_N()` | 常量 | 前端 | 8000 / 2000 / 10000 / 10 |
| `minBurnAmount()` / `totalBurned()` / `accRewardPerShare()` | 配置/状态 | 前端 | — |
| `rankPoolBnb()` / `weightPoolDebtBnb()` | 池子 | 前端 | — |
| `burned(user)` / `rewardDebt(user)` / `rankClaimed(user)` | 个人 | 前端 | — |
| `topBurners(i)` / `topBurnedAmounts(i)` | 榜单项 | 前端 | — |
| `topBurnersList()` | → `(address[10], uint256[10])` | 前端 | 完整榜 |
| `pendingReward(address user)` | → `uint256` | 前端 | 权重+榜单待领 |
| `burn(uint256 amount)` | — | **用户** | ≥ `minBurnAmount` |
| `claim()` | → `uint256 amount` | **用户** | 领权重+榜单 BNB |

### vaultType → UISchema 方法速查

| vaultType | `vaultUISchema().methods[].name` |
|-----------|----------------------------------|
| `split` | `dispatch`, `claim` |
| `scheduled-buyback` | `canTrigger` |
| `omnidrop` | `airdrop` |
| `silent` | `quoteSilentSell`, `silentSell` |
| `burn-dividend` | `burn`, `claim` |
| `token-staking-dividend` | `stake`, `withdraw`, `claim` |
| `lp-staking-dividend` | `stake`, `withdraw`, `claim` |
| `rank-burn-dividend` | `burn`, `claim` |

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

地址（proxy）：`0x7277712198e66A3B4A7c0e84062961f44CEe07f7`  
当前：`getFee()=0.001 ether` · `getMaxCallbackGas()=500000` · `feeReceiver`=部署 owner。

请求方合约须实现 `ITriggerReceiver.trigger(uint256 requestId)`。

| 方法 | 调用方 | 备注 |
|------|--------|------|
| `requestTrigger(uint64 executeAfter) payable → requestId` | 请求方合约 | `msg.value ≥ getFee()`；`executeAfter=0` 立即可执行 |
| `trigger(requestId)` / `triggerMultiple(ids)` | **keeper（`TRIGGER_ROLE`）** | 回调 requester |
| `retryTrigger(requestId)` | 任何人 | 仅 `FAILED` |
| `getFee` / `getMaxCallbackGas` / `getRequest` / `getRequestCount` / `isRequestReady` | 前端/服务端 | 查询 |
| `getRequests` / `getRequestsPaginated` / `getRequestsByRequesterPaginated` | 服务端 | 分页 |
| `setRequiredGasFee` / `setMaxCallbackGas` | admin | 运营 |

`TriggerRequest`：`requester` · `executeAfter` · `status`(PENDING/EXECUTED/FAILED) · `feePaid`。

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
| `dispatchTax()` | keeper：清算合约累积税币 |

卖出前：曲线 → `approve(Portal)`；已迁移 → `approve(PCS Router)`。

---

## 曲线买卖步骤

1. `getTokenV8Safe(token)` → `status` / `quoteTokenAddress` / `nativeToQuoteSwapEnabled`
2. `status !== 1` → 去 PancakeSwap
3. `quoteExactInput` → 设 `minOutputAmount`（扣滑点）
4. 买入：BNB 带 `msg.value`；ERC20 先 approve → `swapExactInput`
5. 卖出：`token.approve(Portal, amount)` → `swapExactInput`

---

## TypeScript 常量

```typescript
export const CHAIN_ID = 56;
export const PROXY_ADMIN = "0xA1f85d13e55B9CeB6Dc295157A06944ea5fdCFE9";
export const PORTAL = "0x8ADab64A1A62bEa555CAe71A8061E10cE9d639F1";
export const VAULT_PORTAL = "0x6CB77C7Cdf15153D9F31b89305875E90D6BB895C";
export const TRIGGER_SERVICE = "0x7277712198e66A3B4A7c0e84062961f44CEe07f7";
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
  split: "0xEe6da751a5E2265d92405Bc22297eaAc07956B05",
  scheduledBuyback: "0x7cCE9d8bb10136aB66408EA7233A25cE26e0620D",
  omnidrop: "0x380c0323f42029B7e16B1E1ce211399a73a719a7",
  silent: "0x4532a7292fC7792cD15362C5F72bA5033C0b543a",
  burnDividend: "0xDBC5C7655178933FE6938deD89A646451cd3eDf2",
  lpStakingDividend: "0x2d3038DAB629Ff27EE0c1A5063Fcd4E88F178c8D",
  tokenStakingDividend: "0x48B83e78E40924Cb61A3Bf2fC90a7C1992e26992",
  rankBurnDividend: "0x893Ab0517e93D392728f397464A22c0A3F74D213",
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
