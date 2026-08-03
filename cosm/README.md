# Cosm 协议流程说明（BSC 主网）

> **全量参数/返回值逐项说明（作用 · 传参 · 场景）见文末 **[全协议参数与返回值参考](#全协议参数与返回值参考)**。**  
> **所有枚举 / 模式字段（uint8 档位）逐项说明见 **[协议枚举参数参考](#协议枚举参数参考)**。**  
> 分场景流程：无税 V7 · 有税路径 A/B · 买卖 · Schema · 金库 · 持币分红。  
> **Vanity salt 搜索（发币前必做）见 **[Vanity Salt 搜索指南（CREATE2）](#vanity-salt-搜索指南create2)**。**

---

# Vanity Salt 搜索指南（CREATE2）

> 适用：**所有发币路径**（无税 V7 · 有税路径 A · 有税路径 B）  
> Portal：`0x59E3f460c45Bdb910f27346cFAdF496E91C97AfD`  
> **纯链下计算 + 发币前一次链上复核**；发币 tx 本身不再「搜 salt」。

---

## 一、结论：salt 是什么、为什么要搜

| 项 | 说明 |
|----|------|
| **`salt`** | `bytes32` 随机数，参与 CREATE2 决定**新代币合约地址** |
| **为何要搜** | Cosm 要求代币地址 **低 16 bit（末尾 4 个十六进制字符）** 必须等于链上配置的 **vanity 后缀**，否则发币 revert |
| **在哪搜** | **本地 CPU 暴力枚举**（不调 Portal 循环）；搜到后再用 `predictTokenAddress` **复核一次** |
| **谁部署** | 克隆部署者永远是 **Portal 代理地址**（不是用户钱包、不是 impl） |
| **用几次** | 每个 `(salt, 模板类型)` **全局只能成功发币一次**；已用过的 salt 会 `SaltAlreadyUsed` |

**Vanity 后缀（BSC 主网默认，以链上读数为准）：**

| 代币类型 | 发币入口 | `isTaxed` / 模板 | 默认后缀（低 16 bit） | 示例地址末尾 |
|----------|----------|------------------|------------------------|--------------|
| 无税 V7 | `Portal.newTokenV7` | `standardTokenImpl` | **`0x0222`** | `…xxxx**0222**` |
| 有税 V6（路径 A/B） | `Portal.newTokenV6` / `VaultPortal.newTokenV6WithVault` | `taxTokenImpl` | **`0x0111`** | `…xxxx**0111**` |

有税与无税 **后缀不同、模板不同、salt 不通用**：同一个 salt 值，换模板会得到**完全不同**的地址，且只有一侧可能满足 vanity。

---

## 二、链上规则（发币时如何校验）

发币流程内部顺序（概念上）：

```
读取 params.salt
    ↓
用对应 impl（tax 或 standard）+ Portal 作 deployer，CREATE2 算出 predicted 地址
    ↓
取 predicted 地址的低 16 bit，与 vanitySuffixTax 或 vanitySuffixStandard 比较
    ↓
不等 → revert VanityMismatch(predicted, expected)
相等 → 继续发币（cloneDeterministic）
    ↓
若该 salt 已被成功发过 → revert SaltAlreadyUsed
```

**要点：**

- 校验的是 **uint16 数值**（低 16 bit），不是「地址字符串里是否包含某段子串」。
- 主网默认 **`0x0111` / `0x0222`** 指的都是这 16 bit；若 Portal 管理员日后改了 `vanitySuffixTax()` / `vanitySuffixStandard()`，**必须以链上读数为准**。
- 路径 B（VaultPortal）仍用 **有税模板 + `0x0111`**，与路径 A 相同；额外校验在 vaultData，与 salt 无关。

---

## 三、发币前要读的链上配置

在本地开始搜 salt **之前**，先对 Portal 做只读调用（eth_call，不耗 gas）：

| 调用 | 作用 | 无税 | 有税 |
|------|------|------|------|
| `standardTokenImpl()` | 无税克隆模板地址 | ✅ | — |
| `taxTokenImpl()` | 有税克隆模板地址 | — | ✅ |
| `vanitySuffixStandard()` | 无税要求的后缀 uint16 | ✅ | — |
| `vanitySuffixTax()` | 有税要求的后缀 uint16 | — | ✅ |
| `vanitySuffixFor(isTaxed)` | 上面两后者的便捷封装 | `false` | `true` |

**可选但推荐：**

| 调用 | 作用 |
|------|------|
| `predictTokenAddress(isTaxed, salt)` | 用 Portal 官方算法复核某个候选 salt |
| `Portal` 代理地址 | CREATE2 里的 **deployer**，必须与搜 salt 时使用的 Portal 地址一致（主网 proxy，不是 impl） |

仓库内可用 `tools/find_vanity.py` 自动从链上读取 impl + suffix 再搜索；流程等价于下节手工步骤。

---

## 四、CREATE2 预测地址（理解即可，不必手算）

Cosm 发币使用 **EIP-1167 最小代理克隆**：Portal 对 `taxTokenImpl` 或 `standardTokenImpl` 做 `cloneDeterministic(salt)`。

预测地址由四要素唯一决定：

| 要素 | Cosm 中的取值 |
|------|----------------|
| **Deployer** | CosmPortal **代理**地址 |
| **Salt** | 你枚举的 `bytes32` |
| **Init code hash** | 由**当前模板 impl 地址**导出的最小代理字节码哈希（impl 升级后哈希变，旧 salt 预测地址也会变） |
| **CREATE2 公式** | `keccak256(0xff ++ deployer ++ salt ++ initCodeHash)` 取后 20 字节为地址 |

因此：

- 换 **Portal 地址**、换 **impl**、换 **salt** 任一都会得到不同 token 地址。
- 搜 salt 时必须用 **与发币同一网络、同一 Portal proxy、同一时刻链上 impl**；fork 测试网若 impl 不同，主网搜到的 salt **不能照搬**。

---

## 五、本地搜索 salt 的详细过程

### 5.1 准备

1. 确定发币路径 → 选定 **有税或无税**（决定模板 + 后缀）。
2. 从链上读取 **impl 地址** 与 **vanitySuffix**（第三节表格）。
3. 确认 CREATE2 **deployer = Portal 代理**（非 VaultPortal；VaultPortal 发币仍走内部 Portal clone）。
4. 准备本地算力：普通 CPU 即可；可开多进程并行分段枚举。

### 5.2 枚举策略

1. 将 `salt` 视为 256 bit 空间；实践中从 **简单递增整数** 转成 `bytes32` 即可（如 0, 1, 2, …）。
2. 对每个候选 salt：
   - 用与链上相同的 CREATE2 规则算出 **predicted 地址**；
   - 取 `predicted & 0xFFFF`，与目标 `vanitySuffix` 比较；
   - 相等则记录该 salt，停止或继续搜「更漂亮」的地址（协议只要求低 16 bit，更高位无要求）。
3. **不要**在循环里对每个 salt 调 RPC `predictTokenAddress`（太慢）；应本地批量算，**只对最终候选**调一次链上 view 复核。

### 5.3 概率与耗时（经验值）

- 低 16 bit 完全随机时，单次命中概率 ≈ **1/65536**。
- 平均约 **3～4 万次** 尝试命中一次；坏运气可能十几次才中，仍属正常。
- 多 worker 线性加速；设上限（如百万次）未命中则扩大范围或检查 impl/portal/suffix 是否填错。

### 5.4 搜到之后

1. **链上复核：** `predictTokenAddress(isTaxed, salt)` 返回地址，确认低 16 bit = 目标后缀。
2. **写入发币参数：** Flap `NewTokenV7Params.salt` / `NewTokenV6Params.salt` / `NewTokenV6WithVaultParams.salt` / `LaunchParams.salt`。
3. **（可选）预占：** `lockSalt(salt, tokenVersion)` 付 `saltLockFee` 绑定发币权，防他人抢发（见第六节）。
4. **尽快发币：** 未 lock 时，他人若先用同一 salt 发币成功，你的 tx 会 `SaltAlreadyUsed`。

---

## 六、可选：`lockSalt` 预占

| 项 | 说明 |
|----|------|
| **作用** | 在正式发币前，把满足 vanity 的 salt **绑定到你的地址** |
| **费用** | `msg.value = saltLockFee`（读 Portal 配置，转给 `feeReceiver`） |
| **校验** | 与发币相同：vanity 必须已对；该 predicted 地址尚未部署 |
| **tokenVersion** | 有税 **`6`**（TOKEN_TAXED_V3）· 无税 **`7`**（TOKEN_STANDARD_V3） |
| **场景** | 前端先生成地址展示、团队协作、防止 MEV/抢 salt |
| **注意** | lock 后须由 locker 在有效期内发币；占用后 `isUsed=true`，他人不可用 |

路径 B 经 VaultPortal 发币时，salt lock 仍在 **Portal** 上操作（版本 6）。

---

## 七、三条发币路径下的 salt 对照

| 路径 | 入口 | `predictTokenAddress` 第一个参数 | 后缀 | salt 能否与无税共用 |
|------|------|----------------------------------|------|---------------------|
| 无税 V7 | `Portal.newTokenV7` | **`false`** | `vanitySuffixStandard`（默认 0x0222） | — |
| 有税 A | `Portal.newTokenV6` | **`true`** | `vanitySuffixTax`（默认 0x0111） | ❌ |
| 有税 B | `VaultPortal.newTokenV6WithVault` | **`true`** | 同左 | ❌ |

路径 B 其余步骤（vaultData、onBeforeLaunch）**不改变** salt 规则。

---

## 八、发币前检查清单（salt 部分）

```
□ 已确认有税 / 无税，并读取对应 impl + vanitySuffix
□ 本地搜索使用的 Portal 地址 = 发币将使用的 Portal proxy
□ 候选 salt 已通过 predictTokenAddress 复核
□ predicted 地址低 16 bit 与链上 suffix 一致
□ （若曾 lockSalt）当前 msg.sender 为 locker 且 salt 未 isUsed
□ 发币 params 里 salt 字段 = 复核通过的 bytes32（勿填错 endian/截断）
```

---

## 九、常见错误与排查

| 现象 | 原因 | 处理 |
|------|------|------|
| `VanityMismatch` | salt 不对；或用了有税 salt 发无税（反之亦然） | 重新搜；确认 `isTaxed` 与入口一致 |
| `SaltAlreadyUsed` | 该 salt 已被他人或测试先发币 | 换 salt 重搜 |
| `SaltAlreadyLocked` | lock 时 salt 已被别人 lock | 换 salt |
| 本地命中但链上 predict 不一致 | Portal/impl/RPC 网络不一致 | 对齐主网 proxy 与最新 impl |
| 搜很久不中 | suffix/impl/portal 填错 | 重新读链；检查是否误用 impl 当 deployer |
| 路径 B revert 与 salt 无关 | vaultData / quote / mktBps 等 | 见路径 B 校验章；salt 仍可能是 0x0111 问题 |

---

## 十、与发币整体流程的衔接

```
读 impl + vanitySuffix（链上）
    ↓
本地枚举 salt → 命中低 16 bit
    ↓
predictTokenAddress 复核
    ↓
（可选）lockSalt
    ↓
准备 quote / approve / vaultData（路径 B）
    ↓
newTokenV7 / newTokenV6 / newTokenV6WithVault（params.salt = 上述 salt）
    ↓
链上返回 token 地址 = 预测地址（vanity 已满足）
```

无税 / 有税 A / 有税 B 各章「步骤 2」仅保留路径差异；**详细搜 salt 过程以本章为准**。

---

# 无税代币发币流程（Cosm · BSC 主网）

> 适用：**普通无税代币（V7 Standard）**  
> 主网 Portal Proxy：`0x59E3f460c45Bdb910f27346cFAdF496E91C97AfD`  
> 版本：`cosm-v0.7.0` · Chain ID `56`

---

## 一、结论：调哪个合约、哪个方法

| 项 | 值 |
|----|-----|
| **合约** | `CosmPortal`（Transparent Proxy，不是 impl） |
| **方法** | `newTokenV7(NewTokenV7Params params) payable` |
| **返回** | `address token` — 新代币地址 |
| **不要用** | `VaultPortal`（那是**有税 + 金库**路径 B） |
| **不要用** | `newTokenV6`（那是**有税**路径 A） |

Cosm 也提供简化版 `newTokenV7(LaunchParams)`（见文末附录），Flap 对齐/SDK 对接请用 **`NewTokenV7Params`**。

---

## 二、整体流程（用户 / 前端视角）

```
准备钱包与 quote
    ↓
① 读链上配置（impl、vanity 后缀、quote 白名单）
    ↓
② 本地搜索 vanity salt（地址低 16 bit = 0x0222）
    ↓
③ （可选）predictTokenAddress 复核地址
    ↓
④ 若 quote 是 ERC20 → approve(Portal, quoteAmt)
    ↓
⑤ Portal.newTokenV7{value}(params)  — 一笔 tx 发币
    ↓
⑥ 读 getTokenV8Safe / getToken 确认 status=Tradable
    ↓
⑦ （可选）曲线买卖、达阈值后自动/手动 migrateToDex
```

**无税发币通常只需 1 笔链上交易**（步骤 ⑤）；②③④ 是发币前准备。

---

## 三、发币前：逐步说明

### 步骤 1 — 读 Portal 配置（view，不耗 gas 或单独 eth_call）

| 调用 | 作用 |
|------|------|
| `Portal.standardTokenImpl()` | CREATE2 模板地址，搜 salt 时用 |
| `Portal.vanitySuffixStandard()` | 无税后缀，默认 **`0x0222`** |
| `Portal.getSupportedQuoteTokens()` | 可用 quote 列表 |
| `Portal.protocolFeeBps()` | 曲线协议费（默认 100 = 1%） |

**Quote 白名单（发币时选定，终身锁定）：**

| 代币 | `quoteToken` | 支付方式 |
|------|--------------|----------|
| BNB | `0x0000000000000000000000000000000000000000` | `msg.value` |
| USDT | `0x55d398326f99059fF775485246999027B3197955` | 先 `approve(Portal)` |
| USDC | `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d` | 同上 |
| USD1 | `0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d` | 同上 |
| USDX | `0x95ffc15Ccfbf883B9eE2105F9F7587D6D43829C6` | 同上 |

### 步骤 2 — 本地搜索 vanity salt（链下）

> **完整过程见 **[Vanity Salt 搜索指南（CREATE2）](#vanity-salt-搜索指南create2)**（无税：`predictTokenAddress(false, …)`，后缀读 `vanitySuffixStandard()`，默认 `0x0222`）。**

要点：部署者 = Portal 代理 · 模板 = `standardTokenImpl()` · 每个 salt 全局仅用一次 · 搜到后必须链上复核再写入 `params.salt`。

### 步骤 3 — （可选）复核预测地址

与 Vanity 指南 §5.4 相同：`predictTokenAddress(false, salt)` 返回地址低 16 bit 须等于 `vanitySuffixStandard()`。

### 步骤 4 — 准备支付（首买）

| 场景 | 操作 |
|------|------|
| `quoteAmt == 0` | 只发币不买；BNB 时 `msg.value = 0` |
| BNB 首买 | `msg.value >= quoteAmt`（多付的 BNB 会退回） |
| ERC20 首买 | 先 `IERC20(quoteToken).approve(Portal, quoteAmt)`，`msg.value = 0` |
| ERC20 + permit | 可把 EIP-2612 签名塞进 `permitData`，省 approve tx |

---

## 四、核心交易：newTokenV7 参数

> **每个字段的作用、传法、适用场景见 [A1 NewTokenV7Params](#a1-newtokenv7params无税-v7) · [A2 FeeConfig](#a2-feeconfig无税-v7-可选)。**

### 4.1 `NewTokenV7Params` 字段说明（无税必填/推荐值）

```solidity
struct NewTokenV7Params {
    string   name;              // 代币名称，如 "My Token"
    string   symbol;            // 符号，如 "MTK"
    string   meta;              // 元数据 URI（IPFS/HTTPS），可 ""
    DexThreshType dexThresh;    // 迁移阈值类型，推荐 DEX_THRESH_DEFAULT(6) → 8亿枚
    bytes32  salt;              // 步骤 2 搜到的 CREATE2 salt
    MigratorType migratorType;  // 迁移目标：推荐 PCS_INFINITY_CL_MIGRATOR(3)
    address  quoteToken;        // 0 = BNB；或 USDT/USDC/USD1/USDX
    uint256  quoteAmt;          // 发币同时首买数量（wei）；0 = 不首买
    bytes    permitData;        // ERC20 permit，无则 0x
    bytes32  extensionID;       // 插件 ID，无插件填 0
    bytes    extensionData;     // 插件数据，无则 0x
    DEXId    dexId;             // 推荐 PCS_INFINITY(3)；V3 迁移填 PCS_V3(2)
    uint16   buyTaxRate;        // 无税：0
    uint16   sellTaxRate;       // 无税：0
    uint64   taxDuration;       // 无税：0
    uint64   antiFarmerDuration;// 无税：0
    address  commissionReceiver;// 无税：0（commission 仅税币有意义）
    TokenVersion tokenVersion;  // 0 或 TOKEN_V3_PERMIT(2)；无税均可
    FeeConfig[4] feeConfigs;    // 无税：全 NONE，或见下表
}
```

### 4.2 枚举取值

> **完整说明（每个枚举值的含义 · 如何传 · 适用场景）见 **[协议枚举参数参考](#协议枚举参数参考)** §一–§三。**

无税 V7 发币常用组合：`dexThresh = 6 (DEX_THRESH_DEFAULT)` · `migratorType = 3 (Infinity)` · `dexId = 3 (PCS_INFINITY)` · `tokenVersion = 0 或 7` · `feeConfigs` 全 `NONE`。

---

## 六、链上内部做了什么（一笔 newTokenV7 内）

用户只发一笔 tx，Portal 内部顺序大致为：

| 顺序 | 动作 | 说明 |
|------|------|------|
| 1 | `_preLaunchChecks` | 校验 name/symbol、quote 白名单、spam 名单 |
| 2 | `_collectQuote` | 若 `quoteAmt>0`：收 BNB 或 pull ERC20 |
| 3 | `_cloneVanityToken` | CREATE2 克隆 `CosmToken` 实现 |
| 4 | `_launchStandard` | 克隆 Lite TaxSplitter、初始化 TokenV3、写 TokenState |
| 5 | `_recordLaunch` | `status = Tradable`，emit `TokenCreated` |
| 6 | `_refundLaunchPayment` | 若 `quoteAmt>0`：曲线买入，代币打给发币人；退多余 BNB |

**无税币不会部署：** CosmDividend、完整 TaxSplitter 四路分配、Vault。

---

## 七、发币后验证


监听事件：`TokenCreated`、`TokenVersionSet`（version = 7）。

---

## 八、发币后常见操作（非发币必需）

> 曲线/DEX 买卖完整流程见文末 **[用户买卖流程（曲线 · DEX）](#用户买卖流程曲线--dex)**。

| 需求 | 合约 | 方法 |
|------|------|------|
| 曲线买入 | Portal | `swapExactInput` + BNB `msg.value` 或 ERC20 approve |
| 曲线卖出 | Portal | `token.approve(Portal)` → `swapExactInput` |
| 估价 | Portal | `quoteExactInput` |
| 手动迁移 | Portal | `migrateToDex(token)`（通常买到阈值自动迁） |
| 领 Infinity LP 费 | Portal | `claim(token)` / `delegateClaim(token)`（beneficiary） |

曲线阶段：`status = Tradable(1)`。  
`circulatingSupply >= dexSupplyThresh`（默认 8 亿）后迁移到 DEX，`status = DEX(4)`，买卖仍走 Portal。

---

## 九、常见错误

| Revert | 原因 |
|--------|------|
| `VanityMismatch` | salt 不对，地址低 16 bit ≠ `0x0222` |
| `InvalidParams` | name/symbol 空、税率非 0 却走无税路径、feeConfigs 合计 ≠ 10000 等 |
| approve 不足 | ERC20 首买未 approve Portal |
| `msg.value` 不对 | BNB quote 时 value < quoteAmt；ERC20 quote 时不应带 value |
| salt 已用 | 同一 salt 不能重复发币 |

---

## 附录 A — 简化入参 `LaunchParams`（Cosm 原生）

> **LaunchParams + TaxAllocation 每个字段见 [A5](#a5-taxallocation--launchparams)。**

若不走 Flap `NewTokenV7Params`，可直接：

```solidity
Portal.newTokenV7(LaunchParams p) payable
```

无税最小示例：

```solidity
LaunchParams({
    name: "My Token",
    symbol: "MTK",
    meta: "ipfs://...",
    salt: bytes32(...),
    quoteToken: address(0),       // BNB
    quoteAmt: 0.1 ether,
    beneficiary: address(0),      // 0 = 默认发币人为 LP 费领取人
    buyTaxBps: 0,
    sellTaxBps: 0,
    isTaxed: false,
    tax: TaxAllocation({          // 全 0
        mktBps: 0, deflationBps: 0, dividendBps: 0, lpBps: 0,
        minimumShareBalance: 0, dividendMode: 0, dividendToken: address(0),
        antiFarmerDuration: 0, taxDuration: 0,
        mktBps2: 0, mktBps3: 0, mktBps4: 0,
        market2: address(0), market3: address(0), market4: address(0),
        converter: address(0)
    }),
    migratorType: MigratorType.PCS_INFINITY_CL_MIGRATOR  // = 3
});
```

---

## 附录 B — 与有税发币的区别（勿混）

| | 无税 V7 | 有税 V6 |
|--|---------|---------|
| 方法 | `newTokenV7` | `newTokenV6` |
| vanity | `0x0222` | `0x0111` |
| impl | `standardTokenImpl` | `taxTokenImpl` |
| 税率 | 0 | >0 |
| 金库 | 无 | 可选 VaultPortal 路径 B |
| dispatch | 不需要 | 需要 keeper `dispatch()` |

---

*文档对应主网部署：`deployments/bsc-56.json` · 更完整 API 见 `api.md`*

---

# 有税代币发币流程（路径 A · 不带 Vault）

> 适用：**有税 · 营销税进钱包**（不是金库）  
> 主网 Portal Proxy：`0x59E3f460c45Bdb910f27346cFAdF496E91C97AfD`  
> 版本：`cosm-v0.7.0` · Chain ID `56`

---

## 一、结论：调哪个合约、哪个方法

| 项 | 值 |
|----|-----|
| **合约** | `CosmPortal`（Transparent Proxy） |
| **方法** | `newTokenV6(NewTokenV6Params params) payable` |
| **返回** | `address token` — 新税币地址 |
| **不要用** | `VaultPortal.newTokenV6WithVault`（那是**路径 B · 税进金库**） |
| **不要用** | `newTokenV7` 发纯无税币（有税请走 V6 或 V7+feeConfigs，路径 A 推荐 **V6**） |

路径 A 特征：`beneficiary` = **营销钱包地址**（EOA），`getToken(token).vault == 0`。

Cosm 也提供简化版 `newTokenV6(LaunchParams)`（见本文 **附录 A**）。

---

## 二、整体流程（用户 / 前端视角）

```
准备钱包、quote、营销钱包、税率与 tax 四路分配
    ↓
① 读链上配置（taxTokenImpl、vanity 0x0111、quote 白名单）
    ↓
② 本地搜索 vanity salt（地址低 16 bit = 0x0111）
    ↓
③ （可选）predictTokenAddress(true, salt)
    ↓
④ 若 dividendMode=2 → 可选 Portal.hasDividendLiquidity(dividendToken)
    ↓
⑤ 若 quote 是 ERC20 → approve(Portal, quoteAmt)
    ↓
⑥ Portal.newTokenV6{value}(params)  — 一笔 tx 发币
    ↓
⑦ 读 getToken 确认 status=Tradable、taxSplitter、dividend
    ↓
⑧ 曲线买卖（带买卖税）
    ↓
⑨ circulatingSupply 达阈值 → 自动/手动 migrateToDex（税币固定 PCS V2）
    ↓
⑩ keeper / 用户调 TaxSplitter.dispatch() 把税打到 mkt/分红/LP
    ↓
⑪ 持币者调 CosmDividend.withdrawDividends()（若开了分红）
```

**发币本身 1 笔 tx**；**dispatch 是发币后的独立操作**（税不会自动打到营销钱包）。

---

## 三、发币前：逐步说明

### 步骤 1 — 读 Portal 配置

| 调用 | 作用 |
|------|------|
| `Portal.taxTokenImpl()` | CREATE2 模板，搜 salt 时用 |
| `Portal.vanitySuffixTax()` | 有税后缀，默认 **`0x0111`** |
| `Portal.getSupportedQuoteTokens()` | quote 白名单 |
| `Portal.defaultTaxConverter()` | Case3 分红默认 converter（发币时可留空） |
| `Portal.protocolFeeBps()` | 曲线协议费（默认 1%） |

**Quote 白名单**（与无税相同，五档均可）：

| 代币 | `quoteToken` |
|------|--------------|
| BNB | `0x0000…0000` |
| USDT | `0x55d398326f99059fF775485246999027B3197955` |
| USDC | `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d` |
| USD1 | `0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d` |
| USDX | `0x95ffc15Ccfbf883B9eE2105F9F7587D6D43829C6` |

### 步骤 2 — 本地搜索 vanity salt

> **完整过程见 **[Vanity Salt 搜索指南（CREATE2）](#vanity-salt-搜索指南create2)**（有税：`predictTokenAddress(true, …)`，后缀读 `vanitySuffixTax()`，默认 `0x0111`）。**

要点：模板 = `taxTokenImpl()` · 与无税 salt **不通用** · 搜到后写入 `NewTokenV6Params.salt`。

### 步骤 3 — （可选）复核预测地址

与 Vanity 指南 §5.4 相同：`predictTokenAddress(true, salt)`。

### 步骤 4 — 规划 tax 四路分配（核心）

**规则：** `mktBps + deflationBps + dividendBps + lpBps = 10000`（万分比）

| 字段 | 含义 | 实际去向（dispatch / 累账时） |
|------|------|-------------------------------|
| `mktBps` | 营销税份额 | **`beneficiary` 钱包**（路径 A）或 **金库**（路径 B） |
| `deflationBps` | 通缩销毁 | **见下表**：本币税直烧 `0xdead`；quote 税先累账再回购 burn |
| `dividendBps` | 持币分红 | `CosmDividend` 合约（用户 `withdrawDividends`） |
| `lpBps` | 加 LP | 迁移后 TaxSplitter 加池；**LP 打到 `0xdead` 永久锁死**（无人可领） |

**deflation 两种路径（不是一律回购）：**

| 税以什么形式进来 | 典型场景 | 处理方式 |
|------------------|----------|----------|
| **本币**（`processTaxTokens`） | DEX/曲线 **卖出** 侧收税 | deflation 份额 **直接 `transfer(0xdead)`**，注释：*非先卖后买* |
| **quote**（`depositQuoteAndSplit`） | 曲线 **买入** 侧收税（BNB/USDX 等） | 记入 `preBondBurnFunds`；**dispatch 时**用 quote 买本币再 burn |

因此：卖出税可以直接烧本币；买入税手里只有 quote，必须先买再烧。Flap 对齐的 **累账 + dispatch** 模型把两路统一成四路 bucket，批量处理 gas/MEV，而不是 swap 当下即时拆分。

**lpBps 细节：**

- quote 份额先进 `lpQuoteBalance`；本币份额可和已有 quote **直接配对加池**（`_addLpWithBalances`），否则先卖成 quote 再累账。
- **迁移前**（`pair` 未就绪）：只累账，不加池。
- **迁移后 dispatch**：约一半 quote 买本币 → `addLiquidity` / `addLiquidityETH`，**LP 接收地址 = `0x000…dEaD`**，永久锁定（相当于 burn LP，深度留在池里但无人能撤）。
- 这与 **`lp-staking-dividend` 金库**不同：金库玩法是用户 stake **自己的** LP 领分红；tax 的 `lpBps` 是协议自动加池并锁死 LP。

四路全填 `0` 时，链上自动 **`mktBps = 10000`**（100% 营销）。

**分红模式**（由 `dividendToken` 推导）：

| `dividendToken` | mode | 分红发什么 |
|-----------------|------|------------|
| `0` + BNB quote | 0 | WBNB（用户 withdraw 得 BNB） |
| `0` + USDX quote | 0 | USDX |
| `0xfEEDFEED…`（SELF 魔法值） | 1 | 本税币 |
| 其他 ERC20 地址 | 2 | 该代币（Case3，需 converter） |

魔法常量：

```text
COSM_MAGIC_DIVIDEND_SELF     = 0xfEEDFEEDfeEDFEedFEEdFEEDFeEdfEEdFeEdFEEd  // 同 Flap
COSM_MAGIC_DIVIDEND_COMPUTED = 0xC0Dec0dec0DeC0Dec0dEc0DEC0DEC0DEC0DEC0dE  // 路径 B 金库
```

Case3 发币前建议：`Portal.hasDividendLiquidity(dividendToken)`；`converter` 可留空，Portal 写入 `defaultTaxConverter()`。

### 步骤 5 — 准备支付（首买）

与无税相同：BNB 用 `msg.value`；ERC20 先 `approve(Portal, quoteAmt)`。

### 步骤 6 — commissionReceiver（可选）

- 传 **`address(0)`**：无 keeper commission
- 传 **平台地址**：dispatch 时从税里抽 commission（bps 链上自动算）
- **不要**让用户随便填（会从税收入里分走）

---

## 四、核心交易：newTokenV6 参数

> **每个字段的作用、传法、适用场景见 [A3 NewTokenV6Params](#a3-newtokenv6params有税路径-a)。**

### 4.1 `NewTokenV6Params` 字段说明

```solidity
struct NewTokenV6Params {
    string   name;
    string   symbol;
    string   meta;                 // 元数据 URI
    DexThreshType dexThresh;       // 迁移阈值，推荐 6 → 8亿本币
    bytes32  salt;                 // vanity salt（0x0111）
    MigratorType migratorType;     // V6 税币会被强制为 V2，填什么都行
    address  quoteToken;           // 0=BNB 或 USDT/USDC/USD1/USDX
    uint256  quoteAmt;             // 首买 quote 量；0=不首买
    address  beneficiary;          // 【必填】营销税收款钱包，≠0
    bytes    permitData;           // ERC20 permit，无则 0x
    bytes32  extensionID;          // 插件，无则 0
    bytes    extensionData;
    DEXId    dexId;                // 推荐 PCS_V2(1)（税币迁移 V2）
    V3LPFeeProfile lpFeeProfile;   // 通常 DEFAULT(0)
    uint16   buyTaxRate;           // 买入税 bps，100=1%；至少 buy/sell 一侧 >0
    uint16   sellTaxRate;          // 卖出税 bps
    uint64   taxDuration;            // 收税总时长秒；0=永久
    uint64   antiFarmerDuration;     // anti-farmer 秒；0=跳过窗口，迁移后直接仅 mainPool 收税
    uint16   mktBps;               // 四路之一，合计 10000
    uint16   deflationBps;
    uint16   dividendBps;
    uint16   lpBps;
    uint256  minimumShareBalance;   // dividendBps>0 时建议 ≥ 10000e18
    address  dividendToken;          // 见 §步骤4 分红模式表
    address  commissionReceiver;     // 0 或平台 keeper 地址
    TokenVersion tokenVersion;       // 必须 TOKEN_TAXED_V3(6)
}
```

### 4.2 与无税的关键差异

| 项 | 路径 A 有税 V6 | 无税 V7 |
|----|----------------|---------|
| vanity | `0x0111` | `0x0222` |
| impl | `taxTokenImpl` | `standardTokenImpl` |
| `beneficiary` | **必填** 营销钱包 | 可 0 |
| 买卖税 | >0 | 0 |
| 迁移 | **强制 PCS V2** | Infinity / V3 可选 |
| 曲线 bonding 费 | 0（税币） | 1.25% 给 beneficiary |
| dispatch | **需要** | 不需要 |
| 金库 | `vault = 0` | — |

### 4.3 迁移阈值（与无税相同）

`dexThresh = DEX_THRESH_DEFAULT(6)` → **`circulatingSupply` 达到 8 亿枚本币**（不是 USDX/BNB 数量）。

---

## 六、链上内部做了什么（一笔 newTokenV6 内）

| 顺序 | 动作 | 说明 |
|------|------|------|
| 1 | `_preLaunchChecks` | quote 白名单、beneficiary≠0、税率校验 |
| 2 | `_collectQuote` | 收 BNB / pull USDX 等 |
| 3 | `_cloneVanityToken` | CREATE2 克隆 `CosmTaxToken` |
| 4 | `_setupTaxToken` | 克隆 **CosmTaxSplitter** +（可选）**CosmDividend** |
| 5 | `_initTaxToken` | 初始化税币；`_taxInitPools` 将 CREATE2 预测 V2 pair 置顶为 `pools[0]`（= `mainPool`） |
| 6 | `_recordLaunch` | `status=Tradable`，`migratorType=V2`，写 taxSplitter/dividend |
| 7 | `_refundLaunchPayment` | 首买曲线买入 |

**路径 A 不会：** 创建 Vault、写 `vaults[token]`（除非 beneficiary 误填成合约地址且像金库）。

---

## 七、发币后验证

读 `Portal.getToken` / `getTokenV8Safe` 确认 `status`、`taxSplitter`、`beneficiary`、`vault`（路径 B）等字段；监听 `TokenCreated`。
---

## 八、发币后必知：税如何到账（dispatch）

> **持币分红（`dividendBps > 0`）** 完整领取流程见文末 **[持币分红领取流程（CosmDividend）](#持币分红领取流程cosmdividend--有税币)**。

曲线/DEX 买卖时税 **只累账**，不自动打款：

```text
用户 swap → TaxSplitter 累账（BondingCurveTax / processTaxTokens）
    ↓
keeper 或任何人调 taxSplitter.dispatch()
    ↓
按 mkt/deflation/dividend/lp 四路打出
    ↓
营销税 → beneficiary 钱包（路径 A）
分红   → CosmDividend（用户 withdrawDividends 领取）
```

批量：`CosmTaxConverter.batchDispatch` / `triggerSplit`（见 `api.md` §Keeper）。

---

## 九、迁移（USDX 例子）

发币时选 `quoteToken = USDX`，迁移逻辑与 quote 无关，**看本币流通量**：

```text
1. 用户用 USDX 在 Portal 曲线买 TAXM
2. circulatingSupply 增加，reserve 里锁 USDX
3. circulatingSupply >= 8亿 TAXM → migrateToDex(TAXM)
4. Portal 把剩余 TAXM + reserve 里 USDX 交给 MigratorV2
5. 在 PancakeSwap V2 建池 TAXM/USDX
6. status → DEX；之后买卖仍走 Portal（内部走 V2）
7. flushLpReserve 等在迁移 finalize 时清曲线累账
```

**税币不能迁 Infinity/V3**（链上强制 V2）。

发币时 Portal 已将 CREATE2 预测的 V2 pair 设为 `CosmTaxToken.mainPool`（`pools[0]`）；迁移创建的池须与此地址一致，反农民结束后仅在该池收税。

---

## 十、发币后常见操作

> 曲线/DEX 买卖完整流程见 **[用户买卖流程（曲线 · DEX）](#用户买卖流程曲线--dex)**。

| 需求 | 合约 | 方法 |
|------|------|------|
| 曲线/DEX 买卖 | Portal | `quoteExactInput` / `swapExactInput` |
| 税分配 | `getToken.taxSplitter` | `dispatch()` |
| 领分红 | `getToken.dividend` | `withdrawDividends()` |
| 手动迁移 | Portal | `migrateToDex(token)` |
| 查进度 | Portal | `getTokenV8Safe` → `circulatingSupply` / `progress` |

---

## 十一、常见错误

| Revert | 原因 |
|--------|------|
| `VanityMismatch` | salt 不对，低 16 bit ≠ `0x0111` |
| `ZeroAddress` | `beneficiary = 0` |
| `InvalidParams` | 四路 bps ≠ 10000；买卖税都为 0；name/symbol 空 |
| `InvalidParams` | Case3 分红币无流动性；`minimumShareBalance` 过低等 |
| 税不到账 | 只 swap 从未调 `dispatch()` |
| 误走路径 B | 用了 `VaultPortal.newTokenV6WithVault` |

---

## 附录 A — 简化入参 `LaunchParams`

> **LaunchParams + TaxAllocation 每个字段见 [A5](#a5-taxallocation--launchparams)。**

```solidity
Portal.newTokenV6(LaunchParams p) payable
```

路径 A 最小示例：

```solidity
LaunchParams({
    name: "Tax Meme",
    symbol: "TAXM",
    meta: "ipfs://...",
    salt: bytes32(...),
    quoteToken: address(0),           // 或 USDX
    quoteAmt: 0.1 ether,
    beneficiary: marketingWallet,       // 必填
    buyTaxBps: 500,
    sellTaxBps: 500,
    isTaxed: true,
    tax: TaxAllocation({
        mktBps: 8000,
        deflationBps: 0,
        dividendBps: 2000,
        lpBps: 0,
        minimumShareBalance: 10000 ether,
        dividendMode: 0,              // 与 dividendToken 一致
        dividendToken: address(0),
        antiFarmerDuration: 0,
        taxDuration: 0,
        mktBps2: 0, mktBps3: 0, mktBps4: 0,
        market2: address(0), market3: address(0), market4: address(0),
        converter: address(0)         // Case3 可留空用 defaultTaxConverter
    }),
    migratorType: MigratorType.V2_MIGRATOR  // 税币；填了也会被强制 V2
});
```

---

## 附录 B — 路径 A vs 路径 B（带 Vault）

| | 路径 A（本文） | 路径 B（带金库） |
|--|----------------|------------------|
| 入口 | `Portal.newTokenV6` | `VaultPortal.newTokenV6WithVault` |
| `beneficiary` | 营销**钱包** | **金库合约**地址 |
| quote | 五档均可 | **仅 BNB** |
| `mktBps` | 打进钱包 | 打进金库（split/buyback 等） |
| Schema | 不需要 | 需要 `vaultDataSchema` |
| `getToken.vault` | `0` | 金库地址 |
| 税领取 | 钱包收款 + dispatch | 金库内 claim / keeper |

---

*更完整 API · Keeper · 金库玩法见 `api.md`*

---

# 有税代币发币流程（路径 B · 带 Vault）

> 适用：**有税 · 营销税进金库**（split / 定时回购 / 燃烧分红等玩法）  
> 主网 VaultPortal Proxy：`0x9e9e9a2392a379fA03c268098Cd9374d7885c55D`  
> 关联 Portal Proxy：`0x59E3f460c45Bdb910f27346cFAdF496E91C97AfD`  
> 版本：`cosm-v0.7.0` · Chain ID `56`

---

## 一、结论：调哪个合约、哪个方法

| 项 | 值 |
|----|-----|
| **合约** | `CosmVaultPortal`（Transparent Proxy + Lens/Launch/Tweak 模块，**不是** Portal） |
| **方法** | `newTokenV6WithVault`（Flap 布局）或 **`newCosmTokenV6WithVault`**（Cosm 简化，推荐） |
| **返回** | `address token` — 新税币地址 |
| **内部** | 工厂 `newVault` → `Portal.newTokenV6`（`beneficiary`=金库）→ 写入 `vaults[token]` |
| **不要用** | `Portal.newTokenV6` 直接发（那是**路径 A · 税进钱包**） |

路径 B 硬约束：

| 约束 | 说明 |
|------|------|
| `quoteToken` | **仅 BNB**（`address(0)`） |
| `mktBps` | **必须 > 0**（营销份额进金库） |
| `beneficiary` | 发币时由 VaultPortal **自动设为金库地址**，前端不传 |
| `vaultData` | 按所选工厂 `vaultDataSchema()` **abi.encode** |
| vanity | 低 16 bit = **`0x0111`**（与路径 A 相同） |

---

## 二、整体流程（含中间校验）

```
准备 BNB、税率、四路分配、选金库玩法
    ↓
① 读 VaultPortal / Portal / 工厂列表
    ↓
② 本地搜索 vanity salt（0x0111）
    ↓
③ 用户选工厂 → factory.vaultDataSchema() 渲染表单
    ↓
④ （可选）factory.tokenCreationPolicies() 展示「仅 BNB」等提示
    ↓
⑤ 用户填表 → abi.encode → vaultData + 本地业务校验
    ↓
⑥ 链上预检 factory.onBeforeLaunch(validationData)  【必做】
    ↓
⑦ VaultPortal.getVaultFactory / isQuoteTokenSupported  【建议】
    ↓
⑧ publicClient.simulateContract(newTokenV6WithVault)   【强烈建议】
    ↓
⑨ VaultPortal.newTokenV6WithVault{value}(params)  — 一笔 tx 发币
    ↓
⑩ tryGetVault(token) + getToken 确认 vault / taxSplitter
    ↓
⑪ 曲线买卖 → TaxSplitter.dispatch() 把税打进金库
    ↓
⑫ 金库页读 vaultUISchema()（见 Schema 指南 §四）→ getStatus / claim / burn 等
    ↓
⑬ （scheduled-buyback）keeper 调 CosmTriggerService.trigger
```

**关键点：** `vaultDataSchema()` 只做**表单发现**，**不校验**；`onBeforeLaunch` **不校验 vaultData**；`vaultData` 错误会在 `factory.newVault` decode 时整笔 revert。因此 **本地规则 + onBeforeLaunch + simulateContract** 三道防线缺一不可。

---

## 三、发币前：逐步说明

### 步骤 1 — 读链上配置

| 调用 | 合约 | 作用 |
|------|------|------|
| `Portal.taxTokenImpl()` | Portal | vanity salt 模板 |
| `Portal.vanitySuffixTax()` | Portal | 默认 **`0x0111`** |
| `VaultPortal.getVaultFactory(factory)` | VaultPortal | 工厂是否 registered / enabled |
| `factory.vaultDataSchema()` | 工厂 | 发币表单结构 |
| `factory.tokenCreationPolicies()` | 工厂 | UI 约束文案（advisory） |
| `factory.isQuoteTokenSupported(0)` | 工厂 | 路径 B 应为 `true` |
| `factory.factorySpecVersion()` | 工厂 | 当前 `"v2.2"` |

**六个官方工厂（主网）：** `vaultData` 编码规则见 Schema 指南 §3.3。

| vaultType | 工厂地址 |
|-----------|----------|
| `split` | `0xB915d88e2c336e0089e221F0A965aE662092c2f7` |
| `scheduled-buyback` | `0x173762B34fcC23E91F0fA49F44f08c7ef4dc3dc5` |
| `burn-dividend` | `0x1AD1F4F8e46acb907db94E5148Ac95F79c101bcE` |
| `lp-staking-dividend` | `0x54FB535ad9bf86B33079899c5bCfD704D3AcEAEb` |
| `token-staking-dividend` | `0x18BF375d070E2ED8d99405773947757b81D73C06` |
| `rank-burn-dividend` | `0xEEBBa479C48540A087D2c720E752AC978fb23787` |

### 步骤 2 — vanity salt（与路径 A 相同）

> **完整过程见 **[Vanity Salt 搜索指南（CREATE2）](#vanity-salt-搜索指南create2)** §七（有税 / `0x0111`）；与路径 A 完全一致。**

### 步骤 3 — Schema 表单与 `vaultData` 编码

> **完整说明见 **[金库 Schema 使用指南（完整版）](#金库-schema-使用指南完整版)** §三（流程）· §七–§九（字段/传参/场景）。**

| 子步骤 | 动作 |
|--------|------|
| 3.1 | `factory.vaultDataSchema()` → 按 `fields[]` + `isArray` 渲染发币表单 |
| 3.2 | （可选）`factory.tokenCreationPolicies()` → 展示「仅 BNB」等 advisory 提示 |
| 3.3 | 用户填表 → **本地校验**（Schema 指南 §3.3）→ `abi.encode` → `vaultData` |
| 3.4 | 写入 `NewTokenV6WithVaultParams.vaultFactory` / `vaultData` |

**要点：** `vaultDataSchema()` 只做字段发现；`onBeforeLaunch` **不校验** `vaultData`；编码/顺序错误会在 `factory.newVault` decode 时整笔 revert。六工厂字段、编码形状与本地校验规则均在 Schema 指南 §3.2–§3.3，此处不再重复。

### 步骤 4 — 提交前校验清单（必做顺序）

```
┌─────────────────────────────────────────────────────────┐
│ 1. VaultPortal.getVaultFactory(factory)                 │
│    registered && enabled（未注册工厂 permissionless 但无官方标）│
├─────────────────────────────────────────────────────────┤
│ 2. factory.isQuoteTokenSupported(quoteToken) → true      │
│    路径 B：quoteToken 必须 address(0)                    │
├─────────────────────────────────────────────────────────┤
│ 3. predictTokenAddress(true, salt) 低 16 bit = 0x0111   │
├─────────────────────────────────────────────────────────┤
│ 4. VaultPortal 层参数                                   │
│    · mktBps > 0                                         │
│    · mktBps + deflationBps + dividendBps + lpBps = 10000│
│    · buyTaxBps > 0 或 sellTaxBps > 0                    │
├─────────────────────────────────────────────────────────┤
│ 5. 本地 vaultData 规则（见 Schema 指南 §3.3，按 vaultType）│
├─────────────────────────────────────────────────────────┤
│ 6. factory.onBeforeLaunch(validationData) → (true, "")  │
├─────────────────────────────────────────────────────────┤
│ 7. simulateContract(newTokenV6WithVault)                │
└─────────────────────────────────────────────────────────┘
```

#### 4.1 VaultPortal 链上校验（发币 tx 内）

| 检查 | revert |
|------|--------|
| `vaultFactory == 0` | `ZeroAddress` |
| `mktBps == 0` | `InvalidMktBps` |
| 四路 bps ≠ 10000 | `InvalidTaxAllocation` |
| 工厂 registered 且 disabled | `FactoryDisabled` |
| TIME_DEPENDENT 策略且非 developer | `FactoryPolicyDenied` |
| quote 不被工厂支持 | `QuoteTokenNotSupported` |
| `onBeforeLaunch` 失败 | `FactoryValidationFailed(factory, reason)` |
| `newVault` decode/构造失败 | 整笔 tx revert（常见：vaultData 格式错） |
| 预测地址 ≠ 实际 token | `"token address mismatch"` |

#### 4.2 `onBeforeLaunch` — 链上预检（必调）

VaultPortal 发币前 staticcall 工厂，payload 为 `abi.encode(LaunchValidationDataV1)`：

```solidity
struct LaunchValidationDataV1 {
    uint8   tokenVersion;          // 6 = TOKEN_TAXED_V3
    address quoteToken;
    uint16  buyTaxRate;
    uint16  sellTaxRate;
    uint16  vaultBps;              // = mktBps
    uint16  deflationBps;
    uint16  dividendBps;
    uint16  lpBps;
    address dividendToken;
    uint256 minimumShareBalance;
}
```


> **`onBeforeLaunch` 不校验 `vaultData`**。split 收款人、bps 等须在步骤 3.3 本地校验 + 步骤 4 清单第 7 项 simulate。

#### 4.3 `simulateContract`（强烈建议）


模拟通过 ≈ 正式交易可通过（含 `vaultData` decode、`newVault`、`Portal.newTokenV6`）。

**链上实际顺序（供对照）：**

```
predictTokenAddress(true, salt)
    → onBeforeLaunch(validationData)
    → factory.newVault(predicted, quote, msg.sender, vaultData)
    → Portal.newTokenV6(beneficiary = vault)
    → vaults[token] = VaultInfo
    → emit CosmTaxVaultTokenCreated
```

### 步骤 5 — 分红相关（可选，与路径 A 相同）

若 `dividendBps > 0`，仍须配置 `dividendMode` / `dividendToken` / `minimumShareBalance`：

| `dividendToken` | mode | 说明 |
|-----------------|------|------|
| `0` | 0 | 分红发 WBNB |
| `COSM_MAGIC_DIVIDEND_SELF` | 1 | 分红发本税币 |
| 其他 ERC20 | 2 | Case3，须 `converter`（可留空用 defaultTaxConverter） |
| `COSM_MAGIC_DIVIDEND_COMPUTED` | — | 仅 v2.3+ 工厂；由工厂解析 |

路径 B 常见配置：**100% mkt 进金库**，`dividendBps = 0`。

### 步骤 6 — 支付并发币

- `quoteToken = address(0)` → `msg.value = quoteAmt`
- **不要** approve USDX 等（路径 B 不支持）

---

## 四、核心交易：路径 B 发币参数

> **Flap 兼容：** `newTokenV6WithVault(NewTokenV6WithVaultParams)` — 见 [A4](#a4-newtokenv6withvaultparams路径-b)  
> **Cosm 推荐：** `newCosmTokenV6WithVault(NewCosmTokenV6WithVaultParams)` — 含 `dividendMode` / `converter`，字段见 [A4.1](#a41-newcosmtokenv6withvaultparams-cosm-简化)

### 4.1 Flap 布局 `NewTokenV6WithVaultParams`

```solidity
struct NewTokenV6WithVaultParams {
    string   name;
    string   symbol;
    string   meta;
    DexThreshType dexThresh;
    bytes32  salt;                 // 0x0111 vanity
    MigratorType migratorType;     // 税币链上强制 V2
    address  quoteToken;           // 必须 address(0)
    uint256  quoteAmt;
    bytes    permitData;
    bytes32  extensionID;
    bytes    extensionData;
    DEXId    dexId;
    V3LPFeeProfile lpFeeProfile;
    uint16   buyTaxRate;
    uint16   sellTaxRate;
    uint64   taxDuration;
    uint64   antiFarmerDuration;
    uint16   mktBps;               // 必须 >0，进金库
    uint16   deflationBps;
    uint16   dividendBps;
    uint16   lpBps;                // 四路合计 10000
    uint256  minimumShareBalance;
    address  dividendToken;          // 无 dividendMode；用 magic 常量
    address  commissionReceiver;
    TokenVersion tokenVersion;       // 6
    address  vaultFactory;
    bytes    vaultData;
}
```

### 4.2 Cosm 简化 `NewCosmTokenV6WithVaultParams`

```solidity
struct NewCosmTokenV6WithVaultParams {
    string   name;
    string   symbol;
    string   meta;
    bytes32  salt;
    address  quoteToken;           // 必须 address(0)
    uint256  quoteAmt;
    uint16   buyTaxBps;
    uint16   sellTaxBps;
    uint16   mktBps;
    uint16   deflationBps;
    uint16   dividendBps;
    uint16   lpBps;
    uint256  minimumShareBalance;
    uint8    dividendMode;
    address  dividendToken;
    address  converter;
    uint256  antiFarmerDuration;
    uint256  taxDuration;
    address  vaultFactory;
    bytes    vaultData;
}
```

---

## 六、链上内部做了什么（一笔 newTokenV6WithVault 内）

| 顺序 | 动作 | 说明 |
|------|------|------|
| 1 | `_pendingSaltOwner = msg.sender` | salt lock / spam 用 |
| 2 | 校验 mktBps、四路 bps、工厂 enabled | VaultPortal |
| 3 | `isQuoteTokenSupported(quoteToken)` | 须 BNB |
| 4 | `predictTokenAddress(true, salt)` | 预测 token |
| 5 | `_resolveComputedDividendToken` | 若用 COMPUTED 魔法值 |
| 6 | `factory.onBeforeLaunch(validationData)` | staticcall 预检 |
| 7 | `factory.newVault(predicted, quote, creator, vaultData)` | 克隆金库实例 |
| 8 | 组装 `LaunchParams`，`beneficiary = vault` | |
| 9 | `Portal.newTokenV6{value}(p)` | 发税币 + 可选首买 |
| 10 | `vaults[token] = VaultInfo` | 写入映射 |
| 11 | `emit CosmTaxVaultTokenCreated` | |

金库创建在 **发币之前**（用 predicted token 地址初始化）。

---

## 七、发币后验证

读 `Portal.getToken` / `getTokenV8Safe` 确认 `status`、`taxSplitter`、`beneficiary`、`vault`（路径 B）等字段；监听 `TokenCreated`。
---

## 八、发币后：税进金库 + 金库操作

> 六个金库 **`split` / `scheduled-buyback` / `burn-dividend` / `token-staking` / `lp-staking` / `rank-burn`** 的逐步调用见文末 **[六个金库调用流程](#六个金库调用流程路径-b--发币后)**。

### 8.1 TaxSplitter.dispatch（所有有税币必做）

```text
用户 swap → 税累在 TaxSplitter
    ↓
taxSplitter.dispatch()（或 CosmTaxConverter.batchDispatch）
    ↓
mkt 份额 → 金库 receive() 收 BNB
    ↓
金库内玩法：split.claim / burn-dividend.burn / staking.stake 等
```


Case3 分红：`CosmTaxConverter.batchDispatchPermissionless([taxSplitter])`（见 `api.md`）。

### 8.2 金库详情页

发币后读 **金库实例**（非工厂）的 `vaultUISchema()`，按 Schema 动态渲染按钮与 approve 提示。

> 完整用法见 Schema 指南 **§四**（UISchema 流程）· **§十**（各方法传参/返回值/场景）；各玩法操作见 **[六个金库调用流程](#六个金库调用流程路径-b--发币后)**。

### 8.3 scheduled-buyback keeper 流程

```text
1. 税 dispatch 进金库 → vault 收 BNB
2. vault.receive() 自动 requestTrigger（扣 getFee()，默认 ~0.0002 BNB）
3. keeper 监听 CosmTriggerRequested
4. isRequestReady(id) && vault.getStatus().ready === true
5. TriggerService.trigger(requestId)（须 TRIGGER_ROLE）
6. 回调 vault.trigger(requestId) 执行 PCS V2 回购 burn
```

TriggerService：`0xAeE5Cc03275559Bcd4013E47351C72e00930A9D6`

---

## 九、迁移

与路径 A 相同：**本币 `circulatingSupply` 达 8 亿**触发；税币**强制 PCS V2**；quote 为 BNB 时建 **TOKEN/WBNB** 池。

税币 **`mainPool`**：发币时 Portal 用 CREATE2 预测 V2 pair 并置顶为 `pools[0]`；迁移后反农民窗口结束，**仅 mainPool 买卖收税**（窗口内全 `pools` 收税）。

`lp-staking-dividend`：pair 与 `CosmTaxToken.mainPool()` 一致，迁移后可直接 stake LP。

---

## 十、常见错误

| Revert / 现象 | 原因 |
|---------------|------|
| `InvalidMktBps` | `mktBps = 0`（路径 B 必须 >0） |
| `QuoteTokenNotSupported` | 用了 USDX/USDT 等 |
| `FactoryValidationFailed(..., "vault requires native BNB quote")` | quote 非 BNB |
| `FactoryDisabled` | 工厂被 admin 禁用 |
| 整笔 revert 无明确 reason | **`vaultData` encode 格式错**（最常见） |
| split `InvalidBpsSum` | 收款人 bps 合计 ≠ 10000 |
| 税进不了金库 | 只 swap 从未 `dispatch()` |
| buyback 不执行 | keeper 未调 `TriggerService.trigger` 或 vault BNB 不足 |
| 误走路径 A | 用了 `Portal.newTokenV6` 且 beneficiary 填钱包 |

---

## 附录 A — 三条发币路径总览

| | 无税 V7 | 有税路径 A | 有税路径 B（本文） |
|--|---------|------------|-------------------|
| 入口 | `Portal.newTokenV7` | `Portal.newTokenV6` | `VaultPortal.newTokenV6WithVault` |
| vanity | `0x0222` | `0x0111` | `0x0111` |
| quote | 五档 | 五档 | **仅 BNB** |
| 营销税去向 | bonding 1.25%（可选 beneficiary） | **钱包** | **金库** |
| Schema | 无 | 无 | `vaultDataSchema` + `vaultUISchema` |
| 提交前校验 | salt + quote | salt + tax bps | **+ vaultData + onBeforeLaunch + simulate** |
| `getToken.vault` | `0` | `0` | 金库地址 |

---

## 附录 B — Schema

见独立章节 **[金库 Schema 使用指南（完整版）](#金库-schema-使用指南完整版)**。

---

*更完整 ABI · Keeper 服务端 · 金库方法列表见 `api.md`*

---

# 用户买卖流程（曲线 · DEX）

> **统一入口：** 无论迁移前后，前端都调 **`CosmPortal`**（`0x59E3f460c45Bdb910f27346cFAdF496E91C97AfD`）  
> Portal 根据 `getTokenV8Safe(token).status` 自动走路径：**Tradable → 曲线** · **DEX → PancakeSwap**

---

## 一、结论：调哪个合约、哪个方法

| 项 | 值 |
|----|-----|
| **合约** | `CosmPortal` proxy |
| **估价** | `quoteExactInput(QuoteParams)` |
| **成交** | `swapExactInput(SwapParams) payable` |
| **插件币** | `extensionID ≠ 0` 时用 `swapExactInputV3`（带 `extensionData`） |
| **不要** | 迁移后换 PCS Router 直连（可选高级用法；站内统一走 Portal 即可） |

**判断走曲线还是 DEX：**

| `status` | 含义 | 内部路由 |
|----------|------|----------|
| `1` Tradable | 曲线阶段 | Portal 内置 bonding curve |
| `4` DEX | 已迁移 | 税币 → PCS **V2** · 无税 → **V3** 或 **Infinity**（看 `dexId`） |


---

## 二、整体流程（用户 / 前端视角）

```
读 getTokenV8Safe(token) → status / quote / buyTaxRate / sellTaxRate
    ↓
① quoteExactInput 估输出量（含曲线费/税）
    ↓
② 准备支付
   · 买入 quote=BNB → msg.value
   · 买入 quote=USDX 等 → approve(Portal) 或 permitData
   · 卖出 → token.approve(Portal, amount)
    ↓
③ swapExactInput（一笔 tx 成交）
    ↓
④ 用户收到 token 或 quote
    ↓
⑤ （有税币）税在 TaxSplitter 累账 → 须 dispatch 才到钱包/金库/分红
    ↓
⑥ （无税 V7）曲线 1.25% 经 TaxSplitterLite 即时分给 beneficiary
    ↓
⑦ 曲线最后一笔买到阈值 → 可能同 tx 内自动 migrateToDex
```

**用户买卖本身 1 笔 tx**；有税币的 **`dispatch()` 是独立后续操作**。

---

## 三、交易前：逐步说明

### 步骤 1 — 读代币状态

> **返回值字段见 [A8 TokenState / TokenStateV8Safe](#a8-读状态-gettoken--gettokenv8safe--gettokenv9safe)。**

| 调用 | 作用 |
|------|------|
| `getTokenV8Safe(token)` | `status` · `quoteTokenAddress` · 税率 · `progress` · `dexId` |
| `getToken(token)` | 完整：`taxSplitter` · `beneficiary` · `dividend` · `vault` |

### 步骤 2 — 构造 QuoteParams / SwapParams

> **逐项说明见 [A7 QuoteParams / SwapParams](#a7-买卖-quoteparams--swapparams)。**

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
    uint256 minOutputAmount;   // 滑点保护
    bytes    permitData;       // ERC20 permit；无则 0x
}
```

| 方向 | `inputToken` | `outputToken` |
|------|--------------|---------------|
| **买入** | `quoteToken`（BNB=`0`） | `token` |
| **卖出** | `token` | `quoteToken`（BNB=`0`） |

### 步骤 3 — 支付与 approve

| quote | 买入 | 卖出 |
|-------|------|------|
| **BNB** | `swapExactInput{ value: inputAmount }` | 收到 BNB 到 `msg.sender`；**卖出时 value 须为 0** |
| **USDT/USDC/USD1/USDX** | 先 `quote.approve(Portal, amount)` 或 `permitData` | 先 `token.approve(Portal, amount)` |

**BNB 一键买 ERC20 quote 币**（`nativeToQuoteSwapEnabled=true`）：


---

## 四、曲线阶段买卖（`status = Tradable`）

### 4.1 曲线买入 — 链上做了什么

```text
用户付 quote（BNB / USDX …）
    ↓
Portal._splitFees(quoteIn, protoBps, buyTaxBps)
    ├─ 无税 V7：protoBps = bondingCurveFeeBps(125) = 1.25%
    └─ 有税 V6：protoBps = 协议费档位 + buyTaxBps = 买入税
    ↓
_payFees → 税/协议费 quote 打进 TaxSplitter（depositQuoteAndSplit）
    ↓
净 quote 进曲线 reserve，按 AMM 公式增发 token
    ↓
token 转给买家；更新 circulatingSupply / progress
    ↓
若 circulatingSupply >= dexSupplyThresh → 尝试同 tx migrateToDex（失败不 revert 买入）
```

**有税币曲线买入税形态：** 从 **quote** 扣（不是本币），deflation 份额进 `preBondBurnFunds`，dispatch 时回购 burn。

### 4.2 曲线卖出 — 链上做了什么

```text
用户 approve + 卖出 token
    ↓
Portal 收 token，按曲线公式算应得 quoteGross
    ↓
_splitFees(quoteGross, protoBps, sellTaxBps) — 从 quote 扣卖出税/协议费
    ↓
_payFees → TaxSplitter 累账
    ↓
净 quote 转给卖家
```

**有税币曲线卖出税形态：** 同样从 **quote** 扣（曲线阶段 token 不走 pool 收税 hook）。

### 4.3 曲线费用对照

| 代币类型 | 曲线扣费 | 去向 | 是否需 dispatch |
|----------|----------|------|-----------------|
| **无税 V7** | 1.25%（`bondingCurveFeeBps`） | `TaxSplitterLite` → **即时**分给 `beneficiary` | ❌ 不需要 |
| **有税 V6/A/B** | `buyTaxBps` / `sellTaxBps` + 协议费 | `CosmTaxSplitter` **累账** | ✅ 需要 `dispatch()` |
| 路径 A mkt | — | dispatch → **营销钱包** | |
| 路径 B mkt | — | dispatch → **金库** | |

---

## 五、DEX 阶段买卖（`status = DEX`）

迁移后 **仍走 Portal 同一组 API**；内部按 **`getTokenV8Safe.dexId`**（路由 hint）转发：

> ⚠️ **`getToken(token).dexId`** 是发币 MDR 序号（0/1/2），**不要**用于路由；列表/买卖请读 **V8Safe/V9Safe 的 dexId**。

| `dexId` | 代币 | 底层 |
|---------|------|------|
| `2` | 有税 V6 | PancakeSwap **V2**（SupportingFeeOnTransfer） |
| `3` | 无税 V7（V3 迁移） | PCS **V3** SwapRouter |
| `4` | 无税 V7（Infinity 迁移） | PCS **Infinity CL** |

### 5.1 DEX 买入 — 链上做了什么

```text
用户付 quote
    ↓
Portal._dexBuy → PCS V2/V3/Infinity 换 token
    ↓
（有税币）token transfer 可能触发 buyTax hook → 税 token 留 token 合约
    ↓
达 liquidationThreshold 时 → processTaxTokens → TaxSplitter 累账
    ↓
token 到买家
```

### 5.2 DEX 卖出 — 链上做了什么

```text
用户 token.approve(Portal) + swapExactInput
    ↓
Portal._dexSell → 经 PCS 卖出
    ↓
（有税币）卖出 tax：transfer hook 扣本币 → 留 token 合约
    ↓
向 mainPool 转账时 _liquidateTax → processTaxTokens
    ↓
TaxSplitter._processFeeToken：
    · deflation 份额 → 直接 transfer(0xdead) 本币
    · 其余份额卖成 quote → 分入 mkt/dividend/lp buckets
    ↓
净 quote 给卖家
```

**DEX 与曲线的关键差异：**

| | 曲线 Tradable | DEX |
|--|---------------|-----|
| 有税 **买入** | 税从 **quote** 扣 | 税从 **本币** transfer hook 扣 |
| 有税 **卖出** | 税从 **quote** 扣 | 税从 **本币** 扣；deflation 可 **直烧 dead** |
| 路由 | Portal 曲线 | PCS V2/V3/Infinity |

---

## 六、有税币：买卖后的税分配（dispatch）

> **`dividendBps > 0` 持币分红领取**（`withdrawDividends`）见 **[持币分红领取流程](#持币分红领取流程cosmdividend--有税币)**。

曲线/DEX 买卖 **只累账，不自动打款**（Flap 对齐）：

```text
用户 swap
    ↓
TaxSplitter 累账
  · 曲线：depositQuoteAndSplit（quote 税）
  · DEX：processTaxTokens（本币税 → 卖/烧/分桶）
    ↓
taxSplitter.dispatch()  （或 CosmTaxConverter.batchDispatch）
    ↓
mktBps     → 钱包（路径 A）/ 金库（路径 B）
deflation  → 通缩（直烧或回购 burn）
dividend   → CosmDividend
lpBps      → 加 LP（LP 锁 0xdead）
    ↓
持币者：dividend.withdrawDividends()
路径 B 金库：vault.claim / burn / stake 等（见 vaultUISchema）
```


---

## 七、曲线 vs DEX 总览

| 项 | 曲线 `status=1` | DEX `status=4` |
|----|-----------------|----------------|
| **前端 API** | `quoteExactInput` / `swapExactInput` | **相同** |
| **进度** | `progress` → 8 亿本币可迁 | 已在池子交易 |
| **无税 V7 费** | 1.25% → beneficiary（即时） | Infinity/V3 LP 费 → `claim(token)` |
| **有税扣税载体** | **quote** | **本币**（transfer hook） |
| **dispatch** | 有税币需要 | 有税币需要 |
| **自动迁移** | 买到阈值可同 tx 触发 | — |

---

## 八、买卖常见错误

| Revert / 现象 | 原因 |
|---------------|------|
| `InvalidParams` | `inputToken`/`outputToken` 与 status 不匹配；卖出带了 `msg.value` |
| `InsufficientValue` | BNB 买入 `msg.value < inputAmount` |
| approve 不足 | 卖 token 或 ERC20 quote 未 approve Portal |
| `TokenWithExtensionNotSupported` | 插件币用了 `swapExactInput` 而非 V3 |
| `NotTradable` / 0 输出 | 代币未发行或已 Invalid |
| `Slippage` | `minOutputAmount` 过高 |
| 税不到账 | 有税币只 swap 从未 `dispatch()` |
| 买到一半没迁 | 自动迁移失败（非 fatal）；可手动 `migrateToDex(token)` |

---

## 附录 — SwapParams 速查

> **QuoteParams / SwapParams 每个字段 · 支付场景见 [A7](#a7-买卖-quoteparams--swapparams)。**

买入：`inputToken=quote`，`outputToken=token`。卖出相反。BNB quote 时 `inputToken=0` 且 tx 带 `msg.value`。

---

*发币流程见本文前三章（无税 V7 · 有税路径 A · 有税路径 B）· 完整 ABI 见 `api.md`*

---

# 金库 Schema 使用指南（完整版）

> **仅路径 B（有税 · 带 Vault）需要 Schema。**  
> 无税 V7、有税路径 A **无金库**，不读 Schema。  
> Schema 是链上 **UI 元数据**，不参与签名；**不做业务校验**（校验在 `onBeforeLaunch` / `newVault` / 各写方法内）。

**本章结构：**

| 节 | 内容 |
|----|------|
| §一 | 两套 Schema、两个时机 |
| §二 | 结构体定义（源码） |
| §三 | 发币前 DataSchema 流程与 encode |
| §四 | 发币后 UISchema 流程 |
| §五–§六 | 前后对照 · 常见错误 |
| **§七–§十** | Schema / 金库 API 每个字段与方法（完整版） |
| **[全协议参考 A1–A12](#全协议参数与返回值参考)** | 发币/买卖/读状态/分红 全部 struct |
| 附录 | 工厂地址 |

---

## 一、结论：两套 Schema、两个时机

| Schema | 挂在哪 | 何时读 | 用途 |
|--------|--------|--------|------|
| **`vaultDataSchema()`** | **金库工厂** | 发币前 · 用户选玩法后 | 渲染发币表单 → 编码为 `bytes vaultData` |
| **`vaultUISchema()`** | **金库实例** | 发币后 · 代币详情「金库 Tab」 | 渲染只读查询 + 写操作按钮（含 approve 提示） |

| 阶段 | 入口合约 | 关键调用 |
|------|----------|----------|
| 发币前 | 工厂 `0x9f50…` 等 | `factory.vaultDataSchema()` · `tokenCreationPolicies()` · `onBeforeLaunch` |
| 发币 | VaultPortal | `newTokenV6WithVault(params)`，`params.vaultData` = 编码结果 |
| 发币后 | 金库实例 | `VaultPortal.tryGetVault(token)` → `vault.vaultUISchema()` |

**Schema 不做的事：**

| 方法 | 能否代替校验 |
|------|--------------|
| `vaultDataSchema()` | ❌ 仅表单字段发现 |
| `tokenCreationPolicies()` | ❌ 仅 advisory 文案 |
| `vaultUISchema()` | ❌ 仅按钮/参数发现 |
| `onBeforeLaunch` | ✅ 税率/四路/quote（**不校验 vaultData**） |
| 本地 vaultData 规则 | ✅ 必做 |
| `simulateContract(newTokenV6WithVault)` | ✅ 强烈建议（含 decode） |

---

## 二、Schema 类型定义（链上结构）

与 `CosmVaultSchemas.sol` 一致，**字段顺序固定**：

```solidity
struct FieldDescriptor {
    string name;         // 表单字段名 / 方法参数名
    string fieldType;    // "address" | "uint8" | "uint16" | "uint256" | …
    string description;  // 展示文案
    uint8  decimals;     // 金额类展示小数位；地址类为 0
}

struct VaultDataSchema {
    string description;           // 玩法说明
    FieldDescriptor[] fields;       // 表单字段（顺序 = abi.encode 顺序）
    bool isArray;                   // 见 §三
}

struct ApproveAction {
    string tokenType;          // "taxToken" | "lpToken"
    string amountFieldName;    // 对应 inputs 里哪个字段要 approve
}

struct VaultMethodSchema {
    string name;
    string description;
    FieldDescriptor[] inputs;
    FieldDescriptor[] outputs;
    ApproveAction[] approvals;   // 写方法前需 approve 的提示
    bool isInputArray;
    bool isOutputArray;
    bool isWriteMethod;          // false=view · true=tx
}

struct VaultUISchema {
    string vaultType;              // 如 "split"
    string description;
    VaultMethodSchema[] methods;   // 建议按数组顺序渲染（view 在前）
}

struct FactoryPolicy {
    string target;       // 如 "quoteToken"
    string operator;     // 如 "eq"
    bytes  value;        // abi.encode 的比较值
    string description;  // UI 提示，不强制执行
}
```

> **字段逐项说明：** §二 为结构体源码；**§七–§十** 为每个成员、链上 API 传参/返回值及适用场景的完整说明。

---

## 三、发币前：`vaultDataSchema()` 怎么用

### 3.1 整体流程

```
展示六个工厂列表（硬编码或读 VaultFactoryRegistered 事件）
    ↓
用户选中 factoryAddress
    ↓
① factory.vaultDataSchema()        → 渲染发币表单
② factory.tokenCreationPolicies()  → （可选）展示「仅 BNB」等提示
③ 用户填写 → 本地校验 → abi.encode → vaultData
④ factory.onBeforeLaunch(data)     → 必做链上预检
⑤ simulateContract(newTokenV6WithVault) → 强烈建议
⑥ VaultPortal.newTokenV6WithVault({ vaultFactory, vaultData, … })
```

### 3.2 `isArray` 与编码规则

| `isArray` | `vaultData` 形状 | 适用 |
|-----------|------------------|------|
| **`false`** | `abi.encode(f1, f2, …)` 按 `fields[]` **顺序** | scheduled-buyback · rank-burn |
| **`true`** | `abi.encode(Row[])`，每行按 `fields[]` 顺序 | split 收款人列表 |
| fields 为空 | `0x` 空 bytes | burn / token-stake / lp-stake |

**规则：** `fieldType` 须与工厂 `newVault` 内 `abi.decode` 类型 **完全一致**；顺序错一位整笔发币 revert。

| fieldType（Schema） | abi.encode 类型 | 常见字段 |
|---------------------|-----------------|----------|
| `address` | `address` | recipient · user · dividendToken |
| `uint8` | `uint8` | triggerMode · buybackMode |
| `uint16` | `uint16` | bps · taxRate |
| `uint256` | `uint256` | amount · intervalSeconds · minBnbAmount |

### 3.3 六个工厂 Schema 与编码对照

#### `split` — `isArray = true`

```solidity
struct Recipient {
    address recipient;
    uint16 bps;
}
// vaultData = abi.encode(Recipient[] recipients)
```

| fields（每行） | fieldType | vaultData |
|----------------|-----------|-----------|
| `recipient` | address | `abi.encode(Recipient[])` |
| `bps` | uint16 | `Recipient { address recipient; uint16 bps; }` |

**本地校验（必做）：**

| 规则 | 链上 revert |
|------|-------------|
| 1 ≤ 人数 ≤ 10 | `NoRecipients` / `TooManyRecipients` |
| 每个 `recipient ≠ 0` | `ZeroRecipient` |
| 地址去重 | `DuplicateRecipient` |
| `Σ bps = 10000` | `InvalidBpsSum` |

#### `scheduled-buyback` — `isArray = false`

| fields（顺序） | fieldType | 取值 / 含义 |
|----------------|-----------|-------------|
| `triggerMode` | uint8 | `0` 仅时间 · `1` 累积 BNB+间隔 · `2` 两者 |
| `buybackMode` | uint8 | `0` 回购本币 burn · `1` 回购 LP burn |
| `intervalSeconds` | uint256 | > 0，触发间隔（秒） |
| `minBnbAmount` | uint256 | wei，mode 1/2 门槛 |
| `maxBnbPerTrigger` | uint256 | wei；`0` = 无上限 |
| `firstExecutableAt` | uint256 | 可选第 6 字段；`0` = 发币后过一个 interval |

编码：`abi.decode` 支持 **5 字段 tuple**，或 **6 字段**（`vaultData.length >= 192` 时读 `firstExecutableAt`）。

**本地校验（必做）：**

| 规则 | 链上 revert |
|------|-------------|
| `triggerMode ≤ 2` | `"bad triggerMode"` |
| `buybackMode ≤ 1` | `"bad buybackMode"` |
| `intervalSeconds > 0` | `"bad interval"` |
| `maxBnbPerTrigger = 0` | 合法，表示无上限 |

#### `burn-dividend` / `token-staking-dividend` / `lp-staking-dividend`

| fields | vaultData |
|--------|-----------|
| 空 | `0x` |

#### `rank-burn-dividend` — `isArray = false`

| fields | fieldType | vaultData |
|--------|-----------|-----------|
| `minBurnAmount` | uint256 | `abi.encode(minBurnAmount)` 或 `0x`（=0） |

### 3.4 `tokenCreationPolicies()`（可选）

工厂若继承 BNB-only 基类，返回例如：

| target | operator | 含义 |
|--------|----------|------|
| `quoteToken` | `eq` | 须 `address(0)`（路径 B 仅 BNB） |

**仅 UI 提示**；链上强制以 `isQuoteTokenSupported` + `onBeforeLaunch` 为准。

### 3.5 `vaultData` 写入发币参数

`NewTokenV6WithVaultParams.vaultFactory` = 工厂地址  
`NewTokenV6WithVaultParams.vaultData` = 上节编码 bytes  

发币 tx 内：`factory.newVault(predictedToken, quote, creator, vaultData)` 解码并构造金库实例。

---

## 四、发币后：`vaultUISchema()` 怎么用

### 4.1 整体流程

```
VaultPortal.tryGetVault(token) → found / info.vault
    ↓
vault.vaultUISchema()  → vaultType + methods[]
    ↓
对每个 method：
  · isWriteMethod=false → eth_call vault.method(inputs)
  · isWriteMethod=true  → 先处理 approvals → 发 tx
    ↓
（并行）taxSplitter.dispatch() 把 mkt 税打进金库
```

**读 Schema 的地址是金库实例**（`info.vault`），**不是工厂**。

### 4.2 渲染 `methods[]` 的规则

| 字段 | 用法 |
|------|------|
| `methods` 顺序 | 建议按链上顺序：通常 **view 在前、write 在后** |
| `isWriteMethod` | `false` → 只读按钮/自动刷新；`true` → 需签名 tx |
| `inputs[]` | 按 `FieldDescriptor` 渲染表单；`name` 即合约参数名 |
| `outputs[]` | 展示返回结构（多数聚合在 `getStatus`） |
| `isInputArray` / `isOutputArray` | 为 `true` 时该方法的入参/返回值是数组（当前六玩法均为 `false`） |
| `decimals`（FieldDescriptor） | 仅影响**展示**（如 BNB 金额用 18）；**不参与** encode 类型选择 |
| `approvals[]` | 写 tx **之前** 引导用户 approve |

### 4.3 `approvals` 与 tokenType

| tokenType | 含义 | 典型方法 |
|-----------|------|----------|
| `taxToken` | 本税币 `approve(vault, amount)` | `burn` · `stake` |
| `lpToken` | PCS V2 LP `approve(vault, amount)` | `stake`（lp-staking） |

`amountFieldName` 指向 `inputs` 中同名 uint256 字段（如 `amount`）。

### 4.4 六个工厂 UISchema 对照

> 各方法**传参、返回值、适用场景**的逐项说明见 **§10.4–§10.8**。

| vaultType | methods（链上顺序） | 写操作 | approve |
|-----------|---------------------|--------|---------|
| `split` | `getStatus` · `getUserInfo(user)` · **`claim(user)`** · `dispatch` | claim · dispatch | 无 |
| `scheduled-buyback` | `canTrigger` · `getStatus` · `countdownSeconds` | **无用户写** | 无 |
| `burn-dividend` | `getStatus` · `getUserInfo(user)` · **`burn(amount)`** · **`claim()`** | burn · claim | taxToken → burn |
| `token-staking-dividend` | `getStatus` · `getUserInfo(user)` · **`stake`** · **`withdraw`** · **`claim`** | 三者 | taxToken → stake |
| `lp-staking-dividend` | 同 staking | 同左 | **lpToken** → stake |
| `rank-burn-dividend` | 同 burn-dividend | burn · claim | taxToken → burn |

**聚合只读首选：**

| 方法 | 作用 |
|------|------|
| `getStatus()` | 池子总览（各 vaultType 返回不同 struct） |
| `getUserInfo(address user)` | 当前用户可领/质押/算力/排名 |

`scheduled-buyback` 的用户侧 **只有读**；回购由 keeper 调 `CosmTriggerService.trigger`（见 **[六个金库调用流程](#六个金库调用流程路径-b--发币后)** §五）。

### 4.5 发币后操作顺序（推荐）

```
1. taxSplitter.dispatch()           — 税进金库（mkt 路）
2. vault.getStatus()                — 刷新页面
3. vault.getUserInfo(用户地址)       — 个人数据
4. 若 UISchema 写方法需要 approve   — 先 approve 再写
5. 调 claim / burn / stake / …
6. （scheduled-buyback）keeper 独立监听 Trigger 事件
```

### 4.6 Schema 与直接调合约的关系

- Schema **不替代 ABI**：写 tx 仍调 `vault.claim(...)` 等真实方法名。
- Schema **可扩展**：工厂升级 UISchema 后前端自动出现新按钮（需适配新 `name`）。
- **TaxSplitter.dispatch** 不在 UISchema 内：所有有税币通用，须在金库 Tab **单独提供**「税分配 / dispatch」入口。

---

## 五、发币前 vs 发币后 对照

| | 发币前 DataSchema | 发币后 UISchema |
|--|-------------------|-----------------|
| 调用对象 | **工厂** | **金库实例** |
| 典型方法 | `vaultDataSchema()` | `vaultUISchema()` |
| 产出 | `bytes vaultData` | UI 按钮 + 参数表单 |
| 是否签名 | 随 `newTokenV6WithVault` 一次提交 | 各写方法单独 tx |
| 空 Schema | `fields=[]` → `vaultData=0x` | 仍有 methods（如 burn/claim） |

---

## 六、常见错误

| 现象 | 原因 |
|------|------|
| 发币 revert 无明确 reason | `vaultData` encode 类型/顺序与 Schema 不一致 |
| 对工厂调 `vaultUISchema` | UISchema 在**实例**上；工厂只有 DataSchema |
| 对实例调 `vaultDataSchema` | DataSchema 在**工厂**上；实例已创建完毕 |
| 表单通过但发币失败 | 未做 `onBeforeLaunch` / 未 simulate |
| 金库页无按钮 | 未 `tryGetVault` 或路径 A 无金库 |
| 有按钮但 tx 失败 | 未 dispatch / 未 approve / lp-staking 未迁 DEX |
| 把 Schema 当校验 | 须本地规则 + onBeforeLaunch + simulate |

---

## 七、Schema 元数据字段详解（§二 各成员）

### 7.1 `FieldDescriptor` — 表单/方法参数字段描述

| 字段 | 类型 | 作用 | 如何传/读 | 适用场景 |
|------|------|------|-----------|----------|
| `name` | string | 字段或合约参数名；**encode 时按此顺序对应 struct 成员** | 发币前：表单 `name` 属性；发币后：调 `vault.method(name, …)` 时的参数名 | DataSchema 每一行；UISchema 的 `inputs[]` / `outputs[]` |
| `fieldType` | string | ABI 类型字符串，决定 `abi.encode` / 合约入参类型 | 须与链上 decode **完全一致**（如 `"uint16"` 不能写成 `"uint256"`） | 所有需编码或签名的字段 |
| `description` | string | 链上提供的展示文案（英文为主） | 只读展示，不参与 tx | 发币表单 label、按钮 tooltip |
| `decimals` | uint8 | **仅 UI 展示**用小数位（如 BNB 金额显示 18 位） | 不参与 encode；展示时 `value / 10^decimals` | `minBnbAmount`、`amount` 等金额类；`address` 类固定为 `0` |

### 7.2 `VaultDataSchema` — `vaultDataSchema()` 返回值

| 字段 | 类型 | 作用 | 如何传/读 | 适用场景 |
|------|------|------|-----------|----------|
| `description` | string | 该玩法发币表单顶部的说明 | `factory.vaultDataSchema().description` | 用户选工厂后展示玩法简介 |
| `fields` | FieldDescriptor[] | 发币时要填的字段列表；**顺序 = abi.encode 顺序** | 遍历渲染表单；空数组表示 `vaultData = 0x` | 所有路径 B 发币前 |
| `isArray` | bool | 决定 `vaultData` 是「单行 tuple」还是「多行数组」 | `true` → `abi.encode(Row[])`；`false` → `abi.encode(f1,f2,…)` | `split=true`；其余多为 `false` |

### 7.3 `VaultMethodSchema` — UISchema 里每个按钮/查询

| 字段 | 类型 | 作用 | 如何传/读 | 适用场景 |
|------|------|------|-----------|----------|
| `name` | string | **真实合约方法名**（如 `claim`、`burn`） | 写：`vault.claim(...)`；读：`eth_call vault.getStatus()` | 金库详情页每个操作 |
| `description` | string | 按钮或只读区块说明 | 只读展示 | 金库 Tab |
| `inputs` | FieldDescriptor[] | 调用该方法需要的参数表单 | 按 `name` + `fieldType` 组装 calldata | 如 `claim(user)`、`burn(amount)` |
| `outputs` | FieldDescriptor[] | 返回值字段说明（多数为空，实际在 struct 里） | 解析 eth_call 返回；**当前预设多为空**，以 `getStatus` struct 为准 | 文档/未来扩展 |
| `approvals` | ApproveAction[] | 写 tx 前需 approve 的 token 提示 | 见 §7.4 | `burn` / `stake` |
| `isInputArray` | bool | 入参是否为数组 | 当前六玩法均为 `false` | 扩展用 |
| `isOutputArray` | bool | 返回值是否为数组 | 当前六玩法均为 `false` | 扩展用 |
| `isWriteMethod` | bool | `false`=view 只读；`true`=需用户签名 tx | 决定是否弹钱包 | 区分「刷新状态」与「领取/质押」 |

### 7.4 `ApproveAction` — 写操作前授权

| 字段 | 类型 | 作用 | 如何传/读 | 适用场景 |
|------|------|------|-----------|----------|
| `tokenType` | string | 要 approve 的 token 种类 | `"taxToken"` → 本税币地址 `vault.taxToken()`；`"lpToken"` → PCS V2 `pair` | token-staking / burn / rank-burn / lp-staking |
| `amountFieldName` | string | 对应 `inputs` 里哪个 uint256 字段作为 approve 额度 | 如 `"amount"` → `IERC20.approve(vault, amount)` | 与 `burn(amount)`、`stake(amount)` 联动 |

### 7.5 `VaultUISchema` — `vaultUISchema()` 返回值

| 字段 | 类型 | 作用 | 如何传/读 | 适用场景 |
|------|------|------|-----------|----------|
| `vaultType` | string | 玩法标识，与工厂 `vaultType()` 一致 | 如 `"split"`、`"burn-dividend"` | 路由 UI 皮肤、文案 |
| `description` | string | 金库页总说明 | 只读 | 代币详情「金库 Tab」头部 |
| `methods` | VaultMethodSchema[] | 建议按顺序渲染的操作列表 | 通常 view 在前、write 在后 | 动态按钮区 |

### 7.6 `FactoryPolicy` — `tokenCreationPolicies()` 返回值（advisory）

| 字段 | 类型 | 作用 | 如何传/读 | 适用场景 |
|------|------|------|-----------|----------|
| `target` | string | 被约束的表单字段名 | 如 `"quoteToken"` | 提示「须 BNB」 |
| `operator` | string | 比较运算符字符串 | 当前仅 `"eq"` | UI 校验提示 |
| `value` | bytes | `abi.encode` 后的比较值 | BNB-only：`abi.encode(address(0))` | 与发币表单 quote 联动展示 |
| `description` | string | 给人看的约束说明 | 只读 | **不强制执行**；链上以 `isQuoteTokenSupported` 为准 |

---

## 八、发币前链上 API：传参、返回值与场景

### 8.1 `factory.vaultDataSchema()` → `VaultDataSchema`

| 项 | 说明 |
|----|------|
| **调用方** | 金库工厂（六个官方地址之一） |
| **传参** | 无 |
| **返回值** | `VaultDataSchema`（§7.2） |
| **场景** | ① 用户选玩法后渲染发币表单 ② 离线文档/校验 code gen ③ **不**用于链上校验 |

### 8.2 `factory.tokenCreationPolicies()` → `FactoryPolicy[]`

| 项 | 说明 |
|----|------|
| **传参** | 无 |
| **返回值** | `FactoryPolicy[]`（§7.6）；六官方工厂通常 1 条 BNB-only |
| **场景** | 发币页展示「路径 B 仅支持 BNB quote」；可忽略但不建议 |

### 8.3 `factory.onBeforeLaunch(bytes validationData)` → `(bool success, string reason)`

**传参：** `validationData = abi.encode(LaunchValidationDataV1)`：

| 字段 | 类型 | 作用 | 如何填 | 场景 |
|------|------|------|--------|------|
| `tokenVersion` | uint8 | 税币版本 | 路径 B 固定 **`6`**（`TOKEN_TAXED_V3`） | 所有路径 B 预检 |
| `quoteToken` | address | 发币 quote | 路径 B **必须 `address(0)`**（BNB） | BNB-only 工厂 |
| `buyTaxRate` | uint16 | 买税 bps | = `NewTokenV6WithVaultParams.buyTaxBps` | 与发币参数一致 |
| `sellTaxRate` | uint16 | 卖税 bps | = `sellTaxBps` | 同上 |
| `vaultBps` | uint16 | 营销税份额 | = **`mktBps`**（进金库） | 路径 B 须 `>0` |
| `deflationBps` | uint16 | 通缩路 | 四路之一 | 四路合计 10000 |
| `dividendBps` | uint16 | 分红路 | 四路之一 | 可与金库并行（走 CosmDividend） |
| `lpBps` | uint16 | LP 路 | 四路之一 | 迁移后加 LP 等 |
| `dividendToken` | address | 分红代币 | `0`/魔法值/ERC20；见路径 B §步骤 5 | `dividendBps>0` 时 |
| `minimumShareBalance` | uint256 | 最低持币分红门槛 | = 发币参数同名字段 | 持币分红 |

**返回值：**

| 字段 | 含义 | 场景 |
|------|------|------|
| `success` | 是否通过预检 | `false` 则不要发币 |
| `reason` | 失败原因字符串 | 展示给用户；如 `"vault requires native BNB quote"` |

**注意：** **不校验 `vaultData`**；split 收款人等须在本地 + simulate。

### 8.4 `VaultPortal.getVaultFactory(address factory)` → `VaultFactoryInfo`

| 字段 | 类型 | 作用 | 场景 |
|------|------|------|------|
| `registered` | bool | 是否在 VaultPortal 登记 | 官方工厂为 true |
| `enabled` | bool | admin 是否启用 | false 则发币 revert |
| `official` | bool | 是否官方认证 | UI 展示「官方」标 |
| `riskLevel` | enum | 风险档 0–4 | 展示用 |
| `category` | enum | 玩法大类 SPLIT/BUYBACK/DIVIDEND/GAME | 列表分组 |

### 8.5 `NewTokenV6WithVaultParams` — 与金库相关的字段

| 字段 | 类型 | 作用 | 如何传 | 场景 |
|------|------|------|--------|------|
| `vaultFactory` | address | 金库工厂 | 六官方之一 | **所有路径 B** |
| `vaultData` | bytes | 玩法初始化数据 | 按 Schema §3.3 encode | 随工厂变化 |
| `mktBps` | uint16 | 买卖税中进金库比例 | **必须 >0**；通常 10000 或与其他路组合 | 路径 B 核心 |
| `quoteToken` | address | quote | **必须 `0`** | BNB-only 金库 |
| `buyTaxBps` / `sellTaxBps` | uint16 | 买卖税率 | 至少一个 >0 | 有税发币 |
| `deflationBps` / `dividendBps` / `lpBps` | uint16 | 另外三路 | 与 `mktBps` 合计 **10000** | 四路分配 |
| 其余字段 | — | 名称、salt、首买、分红模式等 | 同路径 A | 见路径 B 章 §四 |

**返回值：** `address token` — 新税币地址。

### 8.6 `VaultPortal.newTokenV6WithVault(params)` 支付

| 项 | 说明 |
|----|------|
| **msg.value** | `quoteAmt`（BNB wei）；`quoteAmt=0` 则不首买 |
| **场景** | 一笔 tx：创建金库 + 发税币 + 可选曲线首买 |

---

## 九、vaultData 各字段：作用、传参与场景（六工厂）

### 9.1 `split` — `abi.encode(Recipient[])`

| 字段 | 类型 | 作用 | 如何传 | 典型场景 |
|------|------|------|--------|----------|
| `recipient` | address | 分账收款人 | 每行一个 EOA/合约；**非零、去重** | 2–10 人按比例分 mkt 税 |
| `bps` | uint16 | 该收款人占比 | 全体 **合计 10000** | 例：运营 6000 + 社区 4000 |

**整体传参：** `isArray=true` → 整个 payload 是 `Recipient[]`，不是多 tuple。

| 场景 | vaultData 示例 |
|------|----------------|
| 单人全收 | 1 人 bps=10000 |
| 双人 60/40 | 2 行 recipient + 6000/4000 |
| 无效 | 0 人、>10 人、bps≠10000、重复地址 |

### 9.2 `scheduled-buyback` — tuple encode

| 字段 | 类型 | 作用 | 如何传 | 典型场景 |
|------|------|------|--------|----------|
| `triggerMode` | uint8 | 触发条件 | `0` 仅时间 · `1` 金额+间隔 · `2` 两者都要 | 低频大额 vs 定时小额 |
| `buybackMode` | uint8 | 回购标的 | `0` 本币 burn · `1` LP burn（失败回退 token） | 通缩叙事 vs 减 LP |
| `intervalSeconds` | uint256 | 最小触发间隔（秒） | **>0** | 每小时=3600 |
| `minBnbAmount` | uint256 | 累积 BNB 门槛（wei） | mode 1/2 生效 | 攒够 0.1 BNB 才回购 |
| `maxBnbPerTrigger` | uint256 | 单次最多花 BNB | `0` = 无上限（链上存 max uint） | 防止一次花光 |
| `firstExecutableAt` | uint256 | 首次可执行 unix 时间 | **可选第 6 字段**；`0`=部署后满一个 interval | 延迟上线回购 |

| 场景 | triggerMode | 说明 |
|------|-------------|------|
| 纯定时 | 0 | 每隔 interval 只要有 BNB 就回购 |
| 攒够再买 | 1 | 须 `balance >= minBnbAmount` 且过 interval |
| 保守 | 2 | 时间到 **且** 金额够 |

### 9.3 空 `vaultData`（`0x`）

| vaultType | 场景 |
|-----------|------|
| `burn-dividend` | 用户 burn 本币获算力分 BNB |
| `token-staking-dividend` | 质押本币分 BNB |
| `lp-staking-dividend` | 质押 PCS LP 分 BNB（须已迁 DEX） |

### 9.4 `rank-burn-dividend` — `abi.encode(uint256)` 或 `0x`

| 字段 | 类型 | 作用 | 如何传 | 场景 |
|------|------|------|--------|------|
| `minBurnAmount` | uint256 | 单次 burn 下限（wei 单位本币） | `0x` 等价 0；否则 encode 一个 uint256 | 防 dust burn；防刷榜 |

---

## 十、发币后链上 API：传参、返回值与场景

### 10.1 `VaultPortal.tryGetVault(address taxToken)` → `(bool found, VaultInfo info)`

| 字段 | 类型 | 作用 | 场景 |
|------|------|------|------|
| `found` | bool | 是否识别为路径 B 金库币 | false → 无金库 Tab |
| `info.vault` | address | 金库实例地址 | 后续所有 `vault.*` 调用 |
| `info.vaultFactory` | address | 创建工厂 | 审计/展示 |
| `info.isOfficial` | bool | 是否官方工厂创建 | UI 标 |
| `info.riskLevel` | enum | 风险档 | 展示 |
| `info.category` | enum | 玩法大类 | 列表 |
| `info.description` | string | 金库动态描述 | 含余额等摘要 |

**传参：** `taxToken` = 税币合约地址（非金库地址）。

### 10.2 `vault.vaultUISchema()` → `VaultUISchema`

见 §7.5；**必须在金库实例上调用**，不是工厂。

### 10.3 通用：`taxSplitter.dispatch()`

| 项 | 说明 |
|----|------|
| **传参** | 无 |
| **作用** | 把累积税按四路分配；**mkt 路 BNB 转入金库 `receive()`** |
| **场景** | 每次买卖后 tax 未自动 dispatch 时；金库页「同步税」按钮 |
| **返回值** | 无（tx） |

---

### 10.4 `split` 金库

#### 写方法

| 方法 | 传参 | 返回值 | 谁调 | 场景 |
|------|------|--------|------|------|
| `claim(address user)` | `user`：领款地址（通常 `msg.sender`） | 无；BNB 转给 `user` | 收款人 | 领取 `claimable` |
| `dispatch()` | 无 | 无；可选代所有人 push | keeper / 任何人 | 不想逐个 claim 时 |
| `accrue()` | 无 | 无 | 内部/高级 | 把未记账 BNB 入账本 |

#### `getStatus()` → `SplitStatus`

| 字段 | 作用 |
|------|------|
| `vaultBalance` | 金库当前 BNB 余额 |
| `totalDistributed` | 已入账分配总量 |
| `totalClaimed` | 已领取总量 |
| `totalClaimable` | 全体未领合计 |
| `uncredited` | 尚未入账本的 BNB |
| `recipientCount` | 收款人数 |
| `recipients[]` | 每人 bps / accumulated / claimed / claimable |

#### `getUserInfo(address user)` → `UserInfo`

| 字段 | 作用 |
|------|------|
| `bps` | 该用户分账比例 |
| `accumulated` / `claimed` / `claimable` | 账本与可领 |
| `isRecipient` | 是否在发币时 recipient 列表中 |

---

### 10.5 `scheduled-buyback` 金库

#### 只读方法（UISchema 中无用户写方法）

| 方法 | 传参 | 返回值 | 场景 |
|------|------|--------|------|
| `canTrigger()` | 无 | `bool` | keeper 判断是否可执行 |
| `countdownSeconds()` | 无 | `uint256` 秒 | 页面倒计时 |
| `getStatus()` | 无 | `BuybackStatus` | 一页展示全部 |

#### `getStatus()` → `BuybackStatus` 主要字段

| 字段 | 作用 |
|------|------|
| `triggerMode` / `buybackMode` | 发币时 vaultData 固化 |
| `intervalSeconds` / `minBnbAmount` / `maxBnbPerTrigger` | 触发参数 |
| `lastTriggeredAt` / `nextTriggerAt` / `countdownSeconds` | 时间窗 |
| `vaultBnb` / `nextSpendBnb` | 余额与下回花费 |
| `totalTokensBurned` / `totalBnbSpent` / `triggerCount` | 历史统计 |
| `pendingRequestId` | TriggerService 预约 id |
| `ready` | `canTrigger()` 快照 |
| `*Label` | 人类可读模式名 |

#### Keeper：`TriggerService.trigger(uint256 requestId)`

| 项 | 说明 |
|----|------|
| **传参** | `requestId` = `getStatus().pendingRequestId` 或事件中的 id |
| **权限** | `TRIGGER_ROLE` |
| **场景** | `ready=true` 且 BNB 足够时执行回购 |

---

### 10.6 `burn-dividend` 金库

| 方法 | 传参 | 返回值 | approve | 场景 |
|------|------|--------|---------|------|
| `burn(uint256 amount)` | 销毁本币数量（wei） | 无 | **taxToken → vault** | 增加算力、分后续 BNB |
| `claim()` | 无 | `uint256 amount` 领取 BNB | 无 | 领累计分红；**算力不清零** |
| `getStatus()` | 无 | `BurnStatus` | — | 池子总览 |
| `getUserInfo(user)` | 用户地址 | `UserInfo` | — | 个人算力/待领 |

#### `BurnStatus` / `UserInfo` 主要字段

| 字段 | 作用 |
|------|------|
| `totalBurned` / `burned` | 全网/个人算力（销毁量） |
| `pendingBnb` | 尚无算力时暂存的税 |
| `accRewardPerShare` | 分红累加器 |
| `pending` / `shareBps` | 待领 BNB / 占比 |

---

### 10.7 `token-staking-dividend` / `lp-staking-dividend`

| 方法 | 传参 | 返回值 | approve | 场景 |
|------|------|--------|---------|------|
| `stake(uint256 amount)` | 质押数量 | 无 | token-stake：**taxToken**；lp-stake：**lpToken（pair）** | 参与分 BNB |
| `withdraw(uint256 amount)` | 赎回数量 | 无 | 无 | 退出质押 |
| `claim()` | 无 | `uint256` | 无 | 只领奖励不撤质押 |
| `getStatus()` | 无 | `StakeStatus` | — | lp 版多 `pair` 字段 |
| `getUserInfo(user)` | 用户地址 | `UserInfo` | — | staked / pending / shareBps |

| 场景 | 说明 |
|------|------|
| 发币后未迁 DEX | lp-staking **无法** stake（无 pair 流动性） |
| 只领不撤 | 调 `claim()` |
| 加减仓 | `stake` / `withdraw`（会先 harvest 待领） |

---

### 10.8 `rank-burn-dividend` 金库

与 `burn-dividend` 相同表面方法，额外机制：

| 项 | 说明 |
|----|------|
| `burn(amount)` | 须 `amount >= minBurnAmount`（发币 vaultData） |
| 税分配 | 80% 按算力 acc/debt；20% 按 Top10 在 burn 时记入 `rankCredit` |
| `claim()` | 一次领 weight 分红 + rankCredit |
| `getStatus()` | 含 `topBurners[10]`、`rankPendingBnb` 等 |
| `getUserInfo(user)` | 含 `rank`（0=未上榜，1–10=名次）、`weightPending`、`rankCredit`、`pendingTotal` |

---

### 10.9 场景速查：该调哪个读/写 API

| 用户意图 | 先决条件 | 调用 |
|----------|----------|------|
| 发币前填金库表单 | 选工厂 | `vaultDataSchema()` |
| 发币前校验税率/quote | 表单填完 | `onBeforeLaunch(encode(...))` |
| 发币 | simulate 通过 | `newTokenV6WithVault` |
| 打开金库页 | 路径 B 币 | `tryGetVault` → `vaultUISchema` |
| 税没进金库 | 有买卖 | `taxSplitter.dispatch()` |
| 看池子 | dispatch 后 | `getStatus()` |
| 看自己 | 连接钱包 | `getUserInfo(我的地址)` |
| 领 split 分账 | 是 recipient | `claim(我的地址)` |
| 燃烧/质押参与 | approve 够 | `burn` / `stake` |
| 领分红 | 有 pending | `claim()` |
| 定时回购执行 | keeper | `TriggerService.trigger(requestId)` |

---

## 附录 — 工厂地址与 Schema 入口

| vaultType | 工厂 | 发币读 | 发币后读 |
|-----------|------|--------|----------|
| split | `0xB915d88e2c336e0089e221F0A965aE662092c2f7` | `factory.vaultDataSchema()` | `vault.vaultUISchema()` |
| scheduled-buyback | `0x173762B34fcC23E91F0fA49F44f08c7ef4dc3dc5` | 同上 | 同上 |
| burn-dividend | `0x1AD1F4F8e46acb907db94E5148Ac95F79c101bcE` | 同上 | 同上 |
| lp-staking-dividend | `0x54FB535ad9bf86B33079899c5bCfD704D3AcEAEb` | 同上 | 同上 |
| token-staking-dividend | `0x18BF375d070E2ED8d99405773947757b81D73C06` | 同上 | 同上 |
| rank-burn-dividend | `0xEEBBa479C48540A087D2c720E752AC978fb23787` | 同上 | 同上 |

发币后金库地址：`VaultPortal.tryGetVault(token).info.vault` 或 `Portal.getToken(token).beneficiary`（路径 B 时 = 金库）。

---

*各玩法具体操作见 **[六个金库调用流程](#六个金库调用流程路径-b--发币后)** · 路径 B 发币见 **[有税代币发币流程（路径 B）](#有税代币发币流程路径-b--带-vault)***

---

# 六个金库调用流程（路径 B · 发币后）

> 适用：**有税 · 带 Vault** 发币成功后，金库详情页 / keeper 对接  
> VaultPortal：`0x9e9e9a2392a379fA03c268098Cd9374d7885c55D`  
> CosmTriggerService：`0xAeE5Cc03275559Bcd4013E47351C72e00930A9D6`（仅 scheduled-buyback）

---

## 一、结论：金库相关调哪些合约

| 步骤 | 合约 | 方法 | 谁调 |
|------|------|------|------|
| 0. 税进金库 | `getToken(token).taxSplitter` | `dispatch()` | keeper / 任何人 |
| 1. 判断有无金库 | VaultPortal | `tryGetVault(token)` | 前端 |
| 2. 读 UI 说明 | **金库实例** | `vaultUISchema()` | 前端 |
| 3. 读状态 | 金库实例 | `getStatus()` / `getUserInfo(user)` | 前端 |
| 4. 用户操作 | 金库实例 | `claim` / `burn` / `stake` … | 用户 |
| 5. 定时回购 | CosmTriggerService | `trigger(requestId)` | **keeper**（`TRIGGER_ROLE`） |

**金库只处理 `mktBps` 打进来的 BNB**；deflation / dividend / lp 仍由 TaxSplitter 四路分配，不经过金库玩法。

---

## 二、通用前置流程（六个金库相同）

```
用户 Portal 买卖税币
    ↓
① taxSplitter.dispatch()  — mkt 份额 BNB → 金库 receive()
    ↓
② VaultPortal.tryGetVault(token) → info.vault
    ↓
③ vault.vaultUISchema()  — 动态渲染按钮
    ↓
④ vault.getStatus() + vault.getUserInfo(connectedAccount)
    ↓
⑤ 按 vaultType 调写方法（claim / burn / stake …）
    ↓
⑥ （scheduled-buyback）keeper 另走 TriggerService 链路
```


**六个官方工厂（主网）：**

| vaultType | 工厂地址 |
|-----------|----------|
| `split` | `0xB915d88e2c336e0089e221F0A965aE662092c2f7` |
| `scheduled-buyback` | `0x173762B34fcC23E91F0fA49F44f08c7ef4dc3dc5` |
| `burn-dividend` | `0x1AD1F4F8e46acb907db94E5148Ac95F79c101bcE` |
| `lp-staking-dividend` | `0x54FB535ad9bf86B33079899c5bCfD704D3AcEAEb` |
| `token-staking-dividend` | `0x18BF375d070E2ED8d99405773947757b81D73C06` |
| `rank-burn-dividend` | `0xEEBBa479C48540A087D2c720E752AC978fb23787` |

---

## 三、金库详情页：Schema 驱动渲染

> **完整 Schema 用法见 **[金库 Schema 使用指南（完整版）](#金库-schema-使用指南完整版)** §四（发币后 UISchema）。**

| `approvals[].tokenType` | 含义 |
|-------------------------|------|
| `taxToken` | 本税币 `approve(vault, amount)` |
| `lpToken` | PCS V2 LP `approve(vault, amount)` |

---

## 四、`split` — 多人分账领取

### 4.1 结论

| 项 | 值 |
|----|-----|
| **金库** | `info.vault` |
| **税怎么进** | dispatch → BNB 按发币时 `Recipient[]` bps **记账** |
| **用户领取** | `claim(user)` |
| **可选** | `dispatch()` 代全部收款人推送（非必须） |

### 4.2 整体流程

```
dispatch → BNB 进金库
    ↓
accrue（claim 内部也会调）— 按 bps 记入各 recipient 账本
    ↓
收款人调 claim(自己的地址) → BNB 到账
    ↓
（可选）任何人调 vault.dispatch() — 一次性推送全部可领
```

### 4.3 逐步说明

| 步骤 | 调用 | 说明 |
|------|------|------|
| 1 | `getStatus()` | `vaultBalance` · `totalClaimable` · `recipients[]` |
| 2 | `getUserInfo(user)` | `claimable` · `isRecipient` |
| 3 | `claim(user)` | 领 `accumulated − claimed`；可代领 |

### 4.5 常见现象

| 现象 | 原因 |
|------|------|
| `claimable = 0` | 未 dispatch / 不是 Recipient / 已领完 |
| 余额有但未记账 | 调 `accrue()` 或等下次 `claim` 内部 accrue |

---

## 五、`scheduled-buyback` — 定时回购销毁

### 5.1 结论

| 项 | 值 |
|----|-----|
| **用户** | **只读** `getStatus` / 倒计时；可转 BNB 充值金库 |
| **keeper** | `CosmTriggerService.trigger(requestId)`（`TRIGGER_ROLE`） |
| **不要** | 用户/EOA 直调 `vault.trigger()`（仅 TriggerService 回调） |

### 5.2 整体流程

```
dispatch / 用户转 BNB → vault.receive()
    ↓
金库 _tryScheduleTrigger → TriggerService.requestTrigger（扣 getFee() BNB）
    ↓
emit CosmTriggerRequested
    ↓
keeper：isRequestReady(id) && vault.getStatus().ready === true
    ↓
TriggerService.trigger(requestId)  — keeper 付 gas
    ↓
回调 vault.trigger(requestId) → PCS V2 回购本币或 LP 并 burn
    ↓
（失败）retryTrigger(requestId)
```

### 5.3 逐步说明

| 角色 | 步骤 | 调用 |
|------|------|------|
| 前端 | 1 | `getStatus()` → `ready` · `countdownSeconds` · `pendingRequestId` |
| 前端 | 2 | `canTrigger()` · `countdownSeconds()` |
| keeper | 3 | 订阅 `CosmTriggerRequested`，过滤 `requester == vault` |
| keeper | 4 | `triggerService.isRequestReady(id)` |
| keeper | 5 | `triggerService.trigger([id])`（须 `TRIGGER_ROLE`） |
| 任何人 | 6 | 失败时 `triggerService.retryTrigger(id)` |

**费用：** 每次预约从金库扣 `TriggerService.getFee()`（默认约 **0.0002 BNB**）；keeper 只付 tx gas。

### 5.4 keeper 伪代码


### 5.5 常见现象

| 现象 | 原因 |
|------|------|
| 有钱不回购 | keeper 未 trigger / `ready=false` / 未到时间窗 |
| 金库 BNB 减少 | 正常：预约费 + 回购花费 |

---

## 六、`burn-dividend` — 燃烧分红

### 6.1 结论

| 项 | 值 |
|----|-----|
| **算力** | 用户 `burn(amount)` 本币进 dead，累加算力 |
| **分红** | 税 BNB 按 `accRewardPerShare` 分给 burner |
| **claim 后** | **算力不清零**，可继续领后续税 |

### 6.2 整体流程

```
dispatch → BNB 进金库（分红池）
    ↓
用户 approve(taxToken, vault) + burn(amount)
    ↓
本币 → 0xdead，个人 burned↑，accRewardPerShare 更新
    ↓
用户 claim() → 按算力比例领 BNB
```

### 6.3 逐步说明

| 步骤 | 调用 | 前置 |
|------|------|------|
| 1 | `getStatus()` | — |
| 2 | `getUserInfo(user)` | `pending` · `shareBps` |
| 3 | `taxToken.approve(vault, amount)` | 必须 |
| 4 | `burn(amount)` | — |
| 5 | `claim()` | 无 approve |

---

## 七、`token-staking-dividend` — 本币质押分红

### 7.1 结论

| 项 | 值 |
|----|-----|
| **质押物** | 本税币 |
| **分红** | 税 BNB 按质押量 `accRewardPerShare` 分配 |
| **解押** | `withdraw(amount)` 随时可取回质押 |

### 7.2 整体流程

```
dispatch → BNB 进金库
    ↓
approve(taxToken) + stake(amount)
    ↓
（可选）withdraw(amount) 解押
    ↓
claim() 领 BNB
```

### 7.3 逐步说明

| 步骤 | 方法 | 前置 |
|------|------|------|
| 1 | `getStatus()` / `getUserInfo(user)` | — |
| 2 | `stake(amount)` | `approve(vault, amount)` taxToken |
| 3 | `withdraw(amount)` | 无 |
| 4 | `claim()` | 无 |

---

## 八、`lp-staking-dividend` — LP 质押分红

### 8.1 结论

| 项 | 值 |
|----|-----|
| **质押物** | PCS V2 **LP**（TOKEN/WBNB pair） |
| **前置** | 代币须 **已迁移 DEX**（`status=4`）才有 LP |
| **pair** | 发币时 CREATE2 预测，**= `CosmTaxToken.mainPool()`**，**无需** `setPair` |

### 8.2 整体流程

```
migrateToDex 完成 → 用户去 PCS 加池或买入获得 LP
    ↓
dispatch → BNB 进金库
    ↓
approve(LP, vault) + stake(lpAmount)
    ↓
claim() / withdraw(lpAmount)
```

### 8.3 逐步说明

| 步骤 | 方法 | 前置 |
|------|------|------|
| 1 | `getStatus()` | 含 `pair` 地址 |
| 2 | 取 LP | `vault.pair()` 或 `CosmTaxToken.mainPool()` |
| 3 | `stake(amount)` | `LP.approve(vault, amount)` |
| 4 | `withdraw` / `claim` | 同 token-staking |

### 8.5 常见现象

| 现象 | 原因 |
|------|------|
| stake revert | 未迁移 DEX / 无 LP / approve 不足 |

---

## 九、`rank-burn-dividend` — 排行榜燃烧分红

### 9.1 结论

| 项 | 值 |
|----|-----|
| **分红池** | 80% 权重池（同 burn-dividend）+ 20% 每次 burn 分给 **当前 Top10** |
| **发币参数** | `vaultData = abi.encode(minBurnAmount)` 或空 |
| **claim** | 领权重 pending + `rankCredit`；**不清零 burned 算力** |

### 9.2 整体流程

```
dispatch → BNB 进金库（80% 权重池 + 20% rank 池）
    ↓
用户 burn(amount) — amount ≥ minBurnAmount
    ├─ 20% rank 池按当前 Top10 记入 rankCredit
    └─ 累加个人 burned（争榜）
    ↓
Top10 更新（getStatus().topBurners）
    ↓
claim() → 权重分红 + rankCredit 一并领取
```

### 9.3 逐步说明

| 步骤 | 调用 | 说明 |
|------|------|------|
| 1 | `getStatus()` | `topBurners[10]` · `minBurnAmount` |
| 2 | `getUserInfo(user)` | `rank`（1–10 或 0）· `pendingTotal` |
| 3 | `topBurnersList()` | 完整榜 |
| 4 | `burn(amount)` | approve taxToken · `amount ≥ minBurnAmount` |
| 5 | `claim()` | 领权重 + rank |

---

## 十、六个金库对照总表

| vaultType | 税进库后用户做什么 | 写方法 | approve | keeper 额外 |
|-----------|-------------------|--------|---------|-------------|
| `split` | 收款人 **claim** | `claim(user)` | 无 | 可选 `dispatch` |
| `scheduled-buyback` | **只读**状态 | —（用户可转 BNB） | 无 | **`TriggerService.trigger`** |
| `burn-dividend` | **burn** 本币 → **claim** | `burn` · `claim` | taxToken | — |
| `token-staking-dividend` | **stake** 本币 → **claim** | `stake` · `withdraw` · `claim` | taxToken | — |
| `lp-staking-dividend` | **stake LP** → **claim** | 同上 | **lpToken** | — |
| `rank-burn-dividend` | **burn** 争榜 → **claim** | `burn` · `claim` | taxToken | — |

**与 CosmDividend 区别：** 发币时 `dividendBps > 0` 会另部署 `getToken.dividend`，用户调 `withdrawDividends()` —— 那是 **TaxSplitter 四路里的 dividend 路**，与金库玩法 **并行独立**。

---

## 十一、常见错误（金库层）

| 现象 | 原因 |
|------|------|
| 金库 BNB 始终为 0 | 未调 `taxSplitter.dispatch()` |
| split 无法 claim | 地址不在 Recipient 列表 |
| burn/stake revert | 未 approve 或 amount < minBurnAmount |
| lp stake revert | 未迁 DEX / 无 LP |
| buyback 不执行 | 非 scheduled-buyback keeper 问题；见 §五 |
| 用户调 vault.trigger 失败 | 仅 TriggerService 可回调 |

---

## 附录 — 路径 B 完整时间线（买卖 + 金库）

```text
1. VaultPortal.newTokenV6WithVault  — 发币 + 创建金库
2. Portal.swapExactInput            — 用户买卖
3. taxSplitter.dispatch()           — mkt → 金库
4. vault 玩法操作                   — 见 §四–§九
5. （scheduled-buyback）keeper trigger — 回购 burn
6. （若 dividendBps>0）CosmDividend.withdrawDividends — 与金库无关
```

---

*路径 B 发币见 **[有税代币发币流程（路径 B · 带 Vault）](#有税代币发币流程路径-b--带-vault)** · ABI 详见 `api.md`*

---

# 持币分红领取流程（CosmDividend · 有税币）

> 适用：发币时 **`dividendBps > 0`** 的税收代币（路径 A / 路径 B 均可）  
> 与 **金库玩法**（burn-dividend / staking 等）**独立** — 这是 TaxSplitter **四路分配里的 dividend 路**  
> Portal：`0x59E3f460c45Bdb910f27346cFAdF496E91C97AfD`

---

## 一、结论：调哪个合约、哪个方法

| 项 | 值 |
|----|-----|
| **前提** | 发币时 `dividendBps > 0`；`getToken(token).dividend ≠ 0` |
| **税进分红池** | `taxSplitter.dispatch()` — 内部 `CosmDividend.deposit()` |
| **查待领** | `CosmDividend.withdrawableDividends(user)` |
| **领取** | `CosmDividend.withdrawDividends()` |
| **份额** | 税币 **transfer 时自动** `setShare`（持币量 ≥ `minimumShareBalance`） |

**用户领取分红本身 1 笔 tx**；之前须有人（keeper/任何人）先 **`dispatch()`** 把税打进分红合约。

---

## 二、整体流程（用户 / 前端视角）

```
发币 dividendBps > 0 → 部署 CosmDividend 克隆
    ↓
用户 Portal 买卖（税累在 TaxSplitter）
    ↓
① taxSplitter.dispatch()  — dividend 份额 → CosmDividend.deposit
    ↓
② 税币每次转账自动 setShare（持币者份额更新）
    ↓
③ dividend.withdrawableDividends(user)  — 查待领
    ↓
④ dividend.withdrawDividends()  — 一笔 tx 领到钱包
```

```text
买卖税 ──累账──> TaxSplitter
                  │
           dispatch（必做）
                  │
                  ▼
            CosmDividend.deposit（按持币份额记账）
                  │
         用户 withdrawDividends
                  │
                  ▼
            钱包收到 BNB / USDX / 本币 / Case3 代币
```

---

## 三、发币时：分红相关参数

> **TaxAllocation 全字段见 [A5 TaxAllocation](#a5-taxallocation--launchparams) · CosmDividend API 见 [A11](#a11-cosmdividend持币分红)。**

### 3.1 四路中的 dividend

| 字段 | 含义 |
|------|------|
| `dividendBps` | 买卖税里分给 **持币分红** 的份额（万分比）；**=0 不部署** CosmDividend |
| `minimumShareBalance` | 最低持币门槛；低于此余额 **份额=0、不参与分红**（建议 ≥ `10000e18`） |
| `dividendMode` / `dividendToken` | 决定分红发什么（见下表） |

四路仍须合计 10000：`mktBps + deflationBps + dividendBps + lpBps`。

### 3.2 分红模式（发币时选定，终身锁定）

| `dividendToken` | mode | 用户领到 | dispatch 时 deposit |
|-----------------|------|----------|---------------------|
| `0` + BNB quote | 0 | **BNB**（内部 WBNB，领取 unwrap） | quote → WBNB → deposit |
| `0` + USDX 等 quote | 0 | **同 quote**（如 USDX） | quote → deposit |
| `COSM_MAGIC_DIVIDEND_SELF` | 1 | **本税币** | 本币 tax 直留 / 或 quote 换本币 → deposit |
| 其他 ERC20 | 2（Case3） | **该 ERC20** | quote 经 converter 兑换 → deposit |

魔法值：

```text
COSM_MAGIC_DIVIDEND_SELF = 0xfEEDFEEDfeEDFEedFEEdFEEDFeEdfEEdFeEdFEEd
```

Case3 发币：`converter` 可留空 → 用 `Portal.defaultTaxConverter()`；**dispatch 须走 CosmTaxConverter**（MEV 保护），EOA 直调可能 revert。

### 3.3 路径 A vs 路径 B

| | 路径 A | 路径 B |
|--|--------|--------|
| `dividendBps` | 可 >0 | 可 >0（常与 mkt 组合，如 80/20） |
| 与金库关系 | 独立 | **并行**：mkt→金库，dividend→CosmDividend |
| quote | 五档均可 | 路径 B 仅 BNB；dividend mode 0 → WBNB/BNB |

---

## 四、领取前：逐步说明

### 步骤 1 — 确认代币开了分红


### 步骤 2 — dispatch（税进分红池）

**分红池有钱的前提：** 已有买卖产生税，且已 `dispatch()`。


dispatch 内部对 dividend 路：

```text
quote / 本币 tax 份额
    → （必要时 WBNB wrap / Case3 swap）
    → CosmDividend.deposit(amount)
    → magnifiedDividendPerShare 按 totalShares 增加
```

### 步骤 3 — 份额如何计算（无需用户操作）

税币 **每次 `_transfer`** 后自动：

```text
CosmTaxToken → CosmDividend.setShare(holder, balanceOf(holder))
    · balance < minimumShareBalance → share = 0
    · 换仓时旧份额 pending 累进 pendingBalance（不会丢）
```

前端只需展示：**当前持币量是否 ≥ minimumShareBalance**。

### 步骤 4 — 查待领

**调用：** `dividend.withdrawableDividends(user) → uint256`

| 场景 | 返回值 |
|------|--------|
| 未 dispatch | `0` |
| 已 dispatch，持币 ≥ minimumShareBalance | >0（按份额） |
| 持币 < minimumShareBalance | `0`（无份额） |

详见 [A11 CosmDividend](#a11-cosmdividend持币分红)。

### 步骤 5 — 领取

**调用：** `dividend.withdrawDividends() → bool`

| 项 | 说明 |
|----|------|
| **传参** | 无；领到 `msg.sender` |
| **返回值** | `true` 成功 |
| **场景** | 用户一键领 BNB/USDX/本币/Case3 代币 |
| **代领** | `withdrawDividendsFor(user)` / 带 `unwrapWETH` |

**前置：** 步骤 2 已 `dispatch()`；用户当前持币满足门槛（步骤 3）。

---

## 五、完整示例（路径 A · 20% 分红 · BNB quote）

发币参数示意：`mktBps=8000` · `dividendBps=2000` · `dividendToken=0` · BNB quote。


**时序要点（与 fork 测试一致）：**

| 顺序 | `withdrawableDividends` |
|------|-------------------------|
| 买完税币，**未** dispatch | `0` |
| dispatch 之后 | `> 0`（按当时份额） |
| 再买入 + 再 dispatch | 继续增加 |

---

## 六、与金库分红的区别（勿混）

| | **CosmDividend**（本章） | **金库 burn-dividend / staking** |
|--|--------------------------|----------------------------------|
| 来源 | TaxSplitter **`dividendBps`** | TaxSplitter **`mktBps` → 金库** |
| 参与方式 | **持币** 即可（≥ 门槛） | 须 **burn / stake / stake LP** |
| 合约 | `getToken.dividend` | `VaultPortal.tryGetVault().vault` |
| 领取 | `withdrawDividends()` | `vault.claim()` |
| 发币参数 | `dividendBps > 0` | 路径 B + 选 burn/stake 工厂 |

同一代币 **可以同时**：`dividendBps=2000` + 路径 B `mktBps=8000` 进金库 — 两路独立。

---

## 七、批量代领（keeper / 平台可选）


内部对每个用户调 `CosmDividend.distributeDividend(users)`。

---

## 八、常见错误

| 现象 | 原因 |
|------|------|
| `dividend == 0` | 发币时 `dividendBps = 0` |
| 待领始终为 0 | 未 dispatch / 持币 < `minimumShareBalance` |
| dispatch 后仍为 0 | 尚无足够买卖税 / 份额为 0 |
| Case3 dispatch revert | EOA 直调 splitter；改走 **CosmTaxConverter** |
| 卖光币后领不到 | 卖前份额已 `setShare(0)`；应在卖前 claim 或保留 ≥ 门槛 |
| 领到 0 BNB | `withdrawDividends` 返回 false（无可领） |
| 与金库 claim 混淆 | 金库用 `vault.claim()`；本章用 `dividend.withdrawDividends()` |

---

## 九、只读字段速查

| 方法 | 用途 |
|------|------|
| `dividendToken()` | 分红代币地址（BNB 场景为 WBNB） |
| `taxToken()` | 关联税币 |
| `minimumShareBalance()` | 参与门槛 |
| `totalShares()` | 当前有效持币总份额 |
| `withdrawableDividends(user)` | **前端首选** · 待领 |
| `accumulativeDividendOf(user)` | 待领 + 历史已领 |
| `withdrawnDividends(user)` | 累计已领 |
| `excludedFromDividends(user)` | 是否被 owner 排除 |
| `totalDividendsDistributed()` | 累计注入分红池 |

---

## 附录 — 持币分红时间线（路径 A 示例）

```text
1. newTokenV6（dividendBps=2000, mktBps=8000, beneficiary=营销钱包）
2. 用户 swapExactInput 买卖 → 税累 TaxSplitter
3. taxSplitter.dispatch()
      ├─ 80% mkt → 营销钱包
      └─ 20% dividend → CosmDividend.deposit
4. 持币者 withdrawDividends() → 收 BNB/USDX/本币
5. （并行）营销钱包已收 mkt，与步骤 4 无关
```

---

*买卖流程见 **[用户买卖流程（曲线 · DEX）](#用户买卖流程曲线--dex)** · dispatch 见同章 §六 · 金库见 **[六个金库调用流程](#六个金库调用流程路径-b--发币后)***

---

# 全协议参数与返回值参考

> 本文档所有链上 API 的**权威逐项说明**。各流程章节约表 + 流程；此处统一说明 **作用 · 类型 · 如何传 · 返回值 · 适用场景**。

**快速索引：**

| 节 | 内容 |
|----|------|
| [A1](#a1-newtokenv7params无税-v7) | 无税发币 `NewTokenV7Params` |
| [A2](#a2-feeconfig无税-v7-可选) | `FeeConfig` |
| [A3](#a3-newtokenv6params有税路径-a) | 有税路径 A `NewTokenV6Params` |
| [A4](#a4-newtokenv6withvaultparams路径-b--flap-布局) | 路径 B Flap 布局 |
| [A4.1](#a41-newcosmtokenv6withvaultparams-cosm-简化--推荐) | 路径 B Cosm 简化（推荐） |
| [A5](#a5-taxallocation--launchparams) | `TaxAllocation` · `LaunchParams` |
| [A6](#a6-onbeforelaunch--launchvalidationdatav1) | 金库预检载荷 |
| [A7](#a7-买卖-quoteparams--swapparams) | 买卖估价/成交 |
| [A8](#a8-读状态-gettoken--gettokenv8safe--gettokenv9safe) | 读代币状态 |
| [A9](#a9-vaultportal-读状态) | `tryGetVault` · `VaultFactoryInfo` |
| [A10](#a10-其他-portal-方法) | `migrateToDex` · `claim` · 插件 swap |
| [A11](#a11-cosmdividend持币分红) | 持币分红 |
| [A12](#a12-金库-schema-与实例方法) | 金库 Schema §七–§十 + 六玩法流程 |
| [枚举参考](#协议枚举参数参考) | 全部 enum / 模式字段 |

**Schema 章内**另含 `FieldDescriptor`、`VaultMethodSchema`、各金库 `getStatus`/`getUserInfo` 返回字段的逐项表（§七–§十）。

---

## A1. `NewTokenV7Params`（无税 V7）

**调用：** `Portal.newTokenV7(NewTokenV7Params params) payable`  
**返回值：** `address token` — CREATE2 新币地址（vanity 低 16 bit = `0x0222`）

| 字段 | 类型 | 作用 | 如何传 | 典型场景 |
|------|------|------|--------|----------|
| `name` | string | ERC20 名称 | 非空字符串 | 所有发币 |
| `symbol` | string | ERC20 符号 | 非空 | 所有发币 |
| `meta` | string | 元数据 URI（logo/描述 JSON） | IPFS/HTTPS；可 `""` | 有详情页 |
| `dexThresh` | enum | 迁移阈值类型（按 circulatingSupply） | 推荐 **`6` DEX_THRESH_DEFAULT** = 8 亿枚 | 绝大多数 |
| `salt` | bytes32 | CREATE2 salt | 链下 vanity 搜索；每 salt 一次 | 必做；**详见 [Vanity Salt 搜索指南](#vanity-salt-搜索指南create2)** |
| `migratorType` | enum | 迁移执行器 | **`3` Infinity** 或 **`0` V3** | Infinity 主流；V3 兼容 |
| `quoteToken` | address | 曲线 quote | `0`=BNB；或白名单 USDT/USDC/USD1/USDX | BNB 最简单 |
| `quoteAmt` | uint256 | 发币同时首买 quote 量（wei） | `0`=不首买；BNB→`msg.value`；ERC20→approve | 发币+首买一笔 tx |
| `permitData` | bytes | EIP-2612 permit | 无则 **`0x`**；有则 `abi.encode(deadline,v,r,s)` | ERC20 首买免 approve |
| `extensionID` | bytes32 | 插件扩展 ID | 无插件 **`0`** | 普通币 |
| `extensionData` | bytes | 插件数据 | 无插件 **`0x`** | 普通币 |
| `dexId` | enum | 迁移后 DEX 类型提示 | **`3` Infinity** 或 **`2` V3** | 与 migratorType 一致 |
| `buyTaxRate` | uint16 | 买税 bps | **无税固定 `0`** | V7 无税 |
| `sellTaxRate` | uint16 | 卖税 bps | **无税固定 `0`** | V7 无税 |
| `taxDuration` | uint64 | 收税总时长 | **无税 `0`** | V7 无税 |
| `antiFarmerDuration` | uint64 | anti-farmer | **无税 `0`** | V7 无税 |
| `commissionReceiver` | address | keeper 抽成地址 | **无税 `0`** | 仅税币有意义 |
| `tokenVersion` | enum | 模板版本 | **`2` V3_PERMIT** 或 **`0`** | 均可 |
| `feeConfigs` | FeeConfig[4] | V7 费分配（非四路税） | 见 A2；全 NONE 最简单 | LP 费领取人 / 迁移加 LP |

**msg.value：** `quoteToken=0` 时 `msg.value >= quoteAmt`；ERC20 quote 时 `msg.value=0`。

---

## A2. `FeeConfig`（无税 V7 可选）

| 字段 | 类型 | 作用 | 如何传 | 场景 |
|------|------|------|--------|------|
| `feeType` | enum | 费类型 | `0` NONE · `1` MARKETING_OR_VAULT · `4` LP_BPS 等 | 见下表 |
| `bps` | uint16 | 该条占比 | 四条 **合计 10000**（有填 NONE 的槽位 bps=0） | 与 feeType 成对 |
| `marketingAddress` | address | 营销/LP 费收款 | MARKETING_OR_VAULT 时必填 | Infinity LP 费 |
| `dividendToken` | address | 分红代币 | 无税 **不填 DIVIDEND** | — |
| `minimumShareBalance` | uint256 | 分红门槛 | 无税不用 | — |

**feeType 场景：**

| feeType | 场景 |
|---------|------|
| 全 `NONE` | 默认；beneficiary=发币人领 LP 费 |
| `MARKETING_OR_VAULT` bps=10000 | 指定地址收迁移后 LP 费 |
| `MARKETING_OR_VAULT` + `LP_BPS` 合计 10000 | 例 9000 营销 + 1000 迁移加 LP |

---

## A3. `NewTokenV6Params`（有税路径 A）

**调用：** `Portal.newTokenV6(NewTokenV6Params params) payable`  
**返回值：** `address token`（vanity `0x0111`）

| 字段 | 类型 | 作用 | 如何传 | 典型场景 |
|------|------|------|--------|----------|
| `name` / `symbol` / `meta` | string | 代币信息 | 同 A1 | — |
| `dexThresh` | enum | 迁移阈值 | 推荐 `6` | 8 亿本币迁 DEX |
| `salt` | bytes32 | vanity salt | 低 16 bit = **`0x0111`** | 税币 |
| `migratorType` | enum | 迁移器 | 填什么都行；**链上强制 V2** | 税币仅 PCS V2 |
| `quoteToken` | address | quote | BNB 或五档 ERC20 | USDX 曲线 |
| `quoteAmt` | uint256 | 首买 | 同 A1 | — |
| `beneficiary` | address | **营销税收款** | **必填非零** 钱包 | 路径 A 核心 |
| `permitData` | bytes | permit | `0x` 或 encode | ERC20 首买 |
| `extensionID` / `extensionData` | bytes32/bytes | 插件 | 无则 0 / 0x | 普通税币 |
| `dexId` | enum | DEX 提示 | 推荐 **`1` PCS_V2** | 税币 |
| `lpFeeProfile` | enum | V3 LP 档 | 通常 **`0` DEFAULT** | 税币迁 V2 后影响小 |
| `buyTaxRate` / `sellTaxRate` | uint16 | 买卖税 bps | 至少一侧 **>0**；500=5% | 5%/5% 常见 |
| `taxDuration` | uint64 | 税过期秒数 | `0`=永久 | 限时税 |
| `antiFarmerDuration` | uint64 | 迁移后 anti-farmer | `0`=跳过窗口，直接仅 mainPool 收税 | 防狙击 |
| `mktBps` | uint16 | 营销路 | 四路之一 | 80% 进钱包 |
| `deflationBps` | uint16 | 通缩路 | 回购 burn 或 DEX 直烧 | 2% 通缩 |
| `dividendBps` | uint16 | 持币分红路 | >0 部署 CosmDividend | 20% 分红 |
| `lpBps` | uint16 | 加 LP 路 | 迁移后加池；LP 锁 dead | 加池叙事 |
| `minimumShareBalance` | uint256 | 分红最低持仓 | dividend>0 建议 **≥10000e18** | 防 dust 分红 |
| `dividendToken` | address | 分红资产 | `0`/SELF/ERC20；见 A5 | BNB/WBNB/本币/Case3 |
| `commissionReceiver` | address | keeper 佣金 | **`0`** 或平台地址 | 慎填 |
| `tokenVersion` | enum | 版本 | **必须 `6` TOKEN_TAXED_V3** | 税 V3 |

**四路约束：** `mkt+deflation+dividend+lp = 10000`。

---

## A4. `NewTokenV6WithVaultParams`（路径 B · Flap 布局）

**调用：** `VaultPortal.newTokenV6WithVault(params) payable`  
**返回值：** `address token`

| 字段 | 类型 | 作用 | 如何传 | 典型场景 |
|------|------|------|--------|----------|
| `name` / `symbol` / `meta` | string | 代币信息 | 同 A3 | — |
| `dexThresh` | enum | 迁移阈值 | 默认 `FOUR_FIFTHS`(6)→8e8 | 同无税 |
| `salt` | bytes32 | vanity | `0x0111` | 税币 |
| `migratorType` | enum | 迁移器 | 任意；**链上强制 V2** | — |
| `quoteToken` | address | quote | **必须 `address(0)` BNB** | 金库 BNB-only |
| `quoteAmt` | uint256 | 首买 BNB | `msg.value` | 0=不首买 |
| `permitData` / `extensionID` / `extensionData` | | Flap 兼容 | 通常空/0 | — |
| `dexId` / `lpFeeProfile` | enum | MDR/LP hint | 税币通常 DEX0 + DEFAULT | — |
| `buyTaxRate` / `sellTaxRate` | uint16 | 税率 | 同 A3 | — |
| `taxDuration` / `antiFarmerDuration` | uint64 | 税策略 | 同 A3 | — |
| `mktBps` | uint16 | 进**金库**份额 | **必须 >0** | 100% mkt 进 split |
| `deflationBps` / `dividendBps` / `lpBps` | uint16 | 其他三路 | 合计 10000 | 可与 mkt 组合 |
| `minimumShareBalance` | uint256 | 分红门槛 | dividend>0 时 | — |
| `dividendToken` | address | 分红币 | `0`/SELF/COMPUTED/ERC20 | 无 `dividendMode` 字段 |
| `commissionReceiver` | address | 佣金 | 通常 `0` | — |
| `tokenVersion` | enum | 版本 | **必须 `6`** | — |
| `vaultFactory` / `vaultData` | | 金库 | Schema encode | split 收款人等 |

**与 A3 差异：** 无 `beneficiary`（链上设成金库）；**仅 BNB quote**。

---

## A4.1 `NewCosmTokenV6WithVaultParams`（Cosm 简化 · 推荐）

**调用：** `VaultPortal.newCosmTokenV6WithVault(params) payable`

| 字段 | 类型 | 作用 | 如何传 |
|------|------|------|--------|
| `name` / `symbol` / `meta` / `salt` | | 同 A4 | |
| `quoteToken` / `quoteAmt` | | **仅 BNB** | |
| `buyTaxBps` / `sellTaxBps` | uint16 | 税率 | |
| `mktBps` … `lpBps` | uint16 | 四路；`mktBps>0` | |
| `minimumShareBalance` | uint256 | 分红门槛 | |
| `dividendMode` / `dividendToken` / `converter` | | 同 `TaxAllocation` / A5 | |
| `antiFarmerDuration` / `taxDuration` | uint256 | 秒 | |
| `vaultFactory` / `vaultData` | | 工厂 + 编码 | |

---

## A5. `TaxAllocation` · `LaunchParams`

**调用：** `Portal.newTokenV7(LaunchParams)` / `Portal.newTokenV6(LaunchParams)`（Cosm 原生，非 Flap struct）

### `LaunchParams` 主要字段

| 字段 | 类型 | 作用 | 如何传 | 场景 |
|------|------|------|--------|------|
| `name` / `symbol` / `meta` | string | 代币信息 | 非空 | — |
| `salt` | bytes32 | CREATE2 | 无税 0x0222 · 有税 0x0111 | — |
| `quoteToken` / `quoteAmt` | address/uint256 | quote 与首买 | 同 A1/A3 | — |
| `beneficiary` | address | mkt 收款 | 无税可 `0` · 有税 A 钱包 · B 由 VaultPortal 设金库 | 路径相关 |
| `buyTaxBps` / `sellTaxBps` | uint16 | 税率 | 无税 0 | — |
| `isTaxed` | bool | 是否税币 | V7 **`false`** · V6 **`true`** | 与入口方法一致 |
| `tax` | TaxAllocation | 四路配置 | 见下表 | 有税核心 |
| `migratorType` | enum | 迁移器 | 无税 Infinity/V3 · 有税 V2 | — |

### `TaxAllocation` 每个字段

| 字段 | 类型 | 作用 | 如何传 | 场景 |
|------|------|------|--------|------|
| `mktBps` | uint16 | 营销份额 | 路径 A→钱包；路径 B→金库 | 8000 |
| `deflationBps` | uint16 | 通缩 | 曲线 quote 回购 burn；DEX 本币直烧 dead | 1000 |
| `dividendBps` | uint16 | 持币分红 | >0 部署 Dividend | 2000 |
| `lpBps` | uint16 | 加 LP | 迁移后 TaxSplitter 加池 | 1000 |
| `minimumShareBalance` | uint256 | 分红门槛 | ≥10000e18 | dividend>0 |
| `dividendMode` | uint8 | 0=quote · 1=本币 · 2=其他 | 与 dividendToken 一致 | — |
| `dividendToken` | address | 分红资产地址 | `0`/SELF/ERC20/COMPUTED | 见分红章 |
| `antiFarmerDuration` | uint256 | anti-farmer 秒 | `0`=跳过窗口；>0 窗口内全 pools 收税 | 迁移后税范围 |
| `taxDuration` | uint256 | 税总时长 | 0=永久 | 限时项目 |
| `mktBps2/3/4` | uint16 | 从 mkt 再切分 | 合计 ≤ mktBps | 多营销钱包 |
| `market2/3/4` | address | 子营销地址 | bps>0 时必填 | 团队+社区 |
| `converter` | address | Case3 兑换 | mode=2；可 0 | 自定义分红币 |

**默认：** 四路全 0 时 Portal 内部 **mktBps=10000**。

---

## A6. `onBeforeLaunch` · `LaunchValidationDataV1`

**调用：** `factory.onBeforeLaunch(bytes validationData) view`  
**传参：** `validationData = abi.encode(LaunchValidationDataV1)` — 字段见 Schema 指南 §8.3  
**返回：** `(bool success, string reason)`

| 场景 | success | 典型 reason |
|------|---------|-------------|
| quote 非 BNB | false | vault requires native BNB quote |
| mkt=0 路径 B | false | 工厂/Portal 预检失败 |
| 四路≠10000 | false | 各类 bps 错误 |
| 通过 | true | `""` |

---

## A7. 买卖 `QuoteParams` · `SwapParams`

**估价：** `Portal.quoteExactInput(QuoteParams) → uint256 outputAmount`  
**成交：** `Portal.swapExactInput(SwapParams) payable → uint256 outputAmount`

### `QuoteParams`

| 字段 | 类型 | 作用 | 如何传 | 场景 |
|------|------|------|--------|------|
| `inputToken` | address | 支付 token | 买：quote（BNB=0）；卖：meme | 方向核心 |
| `outputToken` | address | 得到 token | 买：meme；卖：quote | 方向核心 |
| `inputAmount` | uint256 | 输入量 wei | 用户输入 | 估价 |

### `SwapParams`（在 Quote 基础上）

| 字段 | 类型 | 作用 | 如何传 | 场景 |
|------|------|------|--------|------|
| `minOutputAmount` | uint256 | 滑点保护 | `< quote 输出`；过小易 Slippage revert | 必设 |
| `permitData` | bytes | ERC20 permit | `0x` 或 approve 替代 | 卖 token / ERC20 买 |

**支付场景：**

| quote | 买入 | 卖出 |
|-------|------|------|
| BNB | `msg.value = inputAmount` | `msg.value = 0` |
| ERC20 | approve Portal 或 permit | `token.approve(Portal)` |

**插件币：** `extensionID ≠ 0` 时用 `swapExactInputV3(ExactInputV3Params)`，多 **`extensionData`** 字段（通常 `0x`）。

---

## A8. 读状态 `getToken` · `getTokenV8Safe` · `getTokenV9Safe`

### `getToken(token) → TokenState`（完整）

| 字段 | 类型 | 作用 | 场景 |
|------|------|------|------|
| `status` | enum | 0 Invalid · 1 Tradable · 4 DEX | 路由买卖 |
| `tokenVersion` | uint8 | 6 税 · 7 无税 | 展示 |
| `quoteToken` | address | 锁定 quote | 构造 SwapParams |
| `reserve` | uint128 | 曲线 quote 储备 | Tradable |
| `circulatingSupply` | uint128 | 流通量 | 迁移进度 |
| `buyTaxBps` / `sellTaxBps` | uint256 | 税率 | 展示/估价 |
| `beneficiary` | address | mkt 收款（钱包或金库） | 路径 A/B |
| `progress` | uint256 | 迁移进度 Wad（1e18=满） | 进度条 |
| `tax` | TaxAllocation | 发币锁定的四路 | 详情页 |
| `taxSplitter` | address | 税分配合约 | dispatch |
| `dividend` | address | CosmDividend；0=未开 | 分红 |
| `pool` | address | 迁移后 DEX 池 | DEX 阶段 |
| `vault` | address | 路径 B 金库；0=无 | 金库 Tab |
| `feeProfile` | enum | 协议费档 | 高级 |
| `migratorType` | enum | 迁移器类型 | V2/V3/Infinity |
| `infinityHook` | address | Infinity hook | Infinity 币 |
| `bondingCurveFeeBps` | uint16 | 无税曲线 1.25%=125 | V7 |
| `lpFeeProfile` / `dexSupplyThresh` / `extensionID` / `dexId` | — | LP 档/阈值/插件/**MDR 序号** | 详情；`dexId`≠V8Safe |

### `getTokenV8Safe(token) → TokenStateV8Safe`（列表页轻量）

| 字段 | 作用 | 与 getToken 关系 |
|------|------|------------------|
| `status` | 生命周期 | 同 |
| `reserve` / `circulatingSupply` / `price` | 行情 | V8 用 uint256 |
| `tokenVersion` | 6/7 | 同 |
| `r` / `h` / `k` | 曲线参数 | 估价用 |
| `dexSupplyThresh` | 迁移阈值 | 通常 8e8 |
| `quoteTokenAddress` | quote | 同 quoteToken |
| `nativeToQuoteSwapEnabled` | 是否 BNB 一键买 ERC20 quote | Flap 兼容 |
| `extensionID` | 插件 | Cosm 普通币为 0 |
| `buyTaxRate` / `sellTaxRate` | 税率 | 同 buy/sellTaxBps |
| `pool` | DEX 池 | 同 |
| `progress` | 进度 Wad | 同 |
| `lpFeeProfile` | LP 档 | 同 |
| `dexId` | **路由 hint**（`_dexKind`）：2=V2 · 3=V3 · 4=Infinity；税币恒 2 | **决定 Portal DEX 路由**；≠ `getToken.dexId` |

**缺：** taxSplitter、dividend、vault、beneficiary → 需要时改调 `getToken`。

### `getTokenV9Safe(token) → TokenStateV9Safe`

在 V8 基础上多 **`bondingCurveFeeRate`**（uint16）：无税曲线费 bps，税币为 0。

---

## A9. VaultPortal 读状态

### `tryGetVault(taxToken) → (bool found, VaultInfo info)`

| 返回字段 | 作用 | 场景 |
|----------|------|------|
| `found` | 是否路径 B 金库币 | 是否展示金库 Tab |
| `info.vault` | 金库实例 | 调 vaultUISchema / claim |
| `info.vaultFactory` | 工厂 | 审计 |
| `info.description` | 动态摘要 | 头部文案 |
| `info.isOfficial` | 官方工厂 | UI 标 |
| `info.riskLevel` | 风险等级 | 列表 |

分类另调 **`getCosmVaultCategory(token)`**（SPLIT/BUYBACK/…）或 Flap 遗留 **`getVaultCategory`**（仅 0/1）。

### `getVaultFactory(factory) → VaultFactoryInfo`

| 字段 | 作用 | 场景 |
|------|------|------|
| `registered` / `enabled` | 能否官方路径 B 发币 | 发币前 |
| `official` | 官方认证 | 展示 |
| `riskLevel` | 风险 | 工厂列表 |
| `category` | **`CosmVaultCategory`** | 用 `getCosmFactoryCategory` |
| `permissionPolicy` | OPEN / TIME_DEPENDENT / DISABLED | 发币权限 |
| `validationMode` | NONE / LEGACY_V6 / V22 | factory hook 布局 |

---

## A10. 其他 Portal 方法

| 方法 | 传参 | 返回值 | 作用 | 场景 |
|------|------|--------|------|------|
| `migrateToDex(token)` | 税币/无税地址 | `address pool` | 手动触发迁移 | 自动迁移失败时 |
| `claim(token)` | 无税 Infinity/V3 币 | `(uint256 amount0, amount1)` | beneficiary 领 LP 费 | 无税迁 DEX 后 |
| `delegateClaim(token)` | 同上 | 同上 | 委托领取 | 多签/平台代领 |
| `predictTokenAddress(isTaxed, salt)` | bool + salt | `address` | 发币前复核 vanity | 发币前 |
| `swapExactInputV3(params)` | ExactInputV3Params | `uint256` | 插件币买卖 | extensionID≠0 |

---

## A11. CosmDividend（持币分红）

**前提：** `getToken(token).dividend ≠ 0`（发币 `dividendBps > 0`）

| 方法 | 传参 | 返回值 | 作用 | 场景 |
|------|------|--------|------|------|
| `withdrawableDividends(user)` | 用户地址 | `uint256` 待领 | 查可领 | 详情页 |
| `withdrawDividends()` | 无 | `bool` | 领至 `msg.sender` | 用户一键领 |
| `withdrawDividendsFor(user)` | 地址 | `bool` | 代领 | keeper/平台 |
| `withdrawDividendsFor(user, unwrapWETH)` | 地址+bool | `bool` | 代领并 unwrap WBNB | BNB 分红 |
| `deposit(amount)` | uint256 | `bool` | 注入分红池 | **仅 TaxSplitter 调** |
| `totalShares()` | 无 | uint256 | 有效份额总和 | 展示 |
| `minimumShareBalance()` | 无 | uint256 | 门槛 | 与用户余额比 |
| `withdrawnDividends(user)` | 地址 | uint256 | 历史已领 | 统计 |

**前置：** `taxSplitter.dispatch()` 把 dividend 路打进 CosmDividend。

**份额：** 税币 transfer 自动 `setShare`；余额 < minimum → 份额 0。

---

## A12. 金库 Schema 与实例方法

金库 **`vaultDataSchema` / `vaultUISchema` / vaultData 字段 / claim·burn·stake·getStatus`** 的逐项说明见独立章 **[金库 Schema 使用指南](#金库-schema-使用指南完整版) §七–§十**。

六个玩法操作步骤见 **[六个金库调用流程](#六个金库调用流程路径-b--发币后)**。

---

## 场景总表：该用哪组参数

| 目标 | 入口方法 | 主要参数 struct | 必读节 |
|------|----------|-----------------|--------|
| 无税发币 | `newTokenV7` | NewTokenV7Params + FeeConfig | A1 A2 |
| 有税·钱包收 mkt | `newTokenV6` | NewTokenV6Params | A3 A5 |
| 有税·金库玩法 | `newCosmTokenV6WithVault`（推荐）或 `newTokenV6WithVault` | NewCosmTokenV6WithVaultParams / Flap 布局 + vaultData | A4 A4.1 A6 |
| Cosm 简化发币 | `newTokenV7/V6(LaunchParams)` | LaunchParams + TaxAllocation | A5 |
| 曲线/DEX 买卖 | `swapExactInput` | SwapParams | A7 A8 |
| 税到账 | `taxSplitter.dispatch()` | 无 | 有税必做 |
| 持币分红 | `withdrawDividends` | 无 | A11 |
| 金库领取/质押 | `vault.*` | 见 UISchema | A12 |
| 读列表 | `getTokenV8Safe` / `getTokenV9Safe` | 无 | A8 |
| 读 tax/dividend/vault | `getToken` | 无 | A8 |

---

*与 `contracts/CosmTypes.sol` · `ICosmPortalTypes.sol` · `CosmVaultPortal.sol` 源码对齐 · 更完整 ABI 见 `api.md`*

---

# 协议枚举参数参考

> **Solidity `enum` 与协议约定的 `uint8` 模式字段**的权威说明。  
> 表格列：**值** = 链上整数 · **名称** = 源码枚举成员 · **作用** · **如何传** · **典型场景**。

**快速索引：**

| 节 | 枚举 / 模式 | 主要出现位置 |
|----|-------------|--------------|
| [§一](#一flap-发币与-dex-icosmportaltypes) | DexThresh · Migrator · DEXId · TokenVersion · FeeType · V3LPFeeProfile | `newTokenV6/V7` |
| [§二](#二cosmtypes-核心) | TokenStatus · FeeProfile · MigratorType · dividendMode | `getToken` · 买卖 · 四路税 |
| [§三](#三金库-vaultportal) | RiskLevel · VaultCategory · CosmVaultCategory · FactoryPermissionPolicy | VaultPortal |
| [§四](#四keeper--基础设施) | OperationType · TriggerStatus · PoolType | TaxConverter · Trigger · Case3 |
| [§五](#五税币池状态-poolstate) | PoolState | 税币 transfer hook |
| [§六](#六金库-vaultdata-模式字段非-solidity-enum) | triggerMode · buybackMode | scheduled-buyback `vaultData` |

---

## 一、Flap 发币与 DEX（`ICosmPortalTypes`）

### 1.1 `DexThreshType` — 迁移阈值（按 circulatingSupply）

**字段：** `NewTokenV6Params.dexThresh` · `NewTokenV7Params.dexThresh`  
**作用：** 曲线阶段本币流通量达到该值（18 位小数）时可迁移 DEX；写入 `TokenState.dexSupplyThresh`。

| 值 | 名称 | 阈值（枚，18 decimals） | 如何传 | 场景 |
|----|------|-------------------------|--------|------|
| `0` | `TWO_THIRDS` | 667,000,000 | 偏快迁移 | 测试 / 短曲线 |
| `1` | `FOUR_FIFTHS` | 800,000,000 | 与 DEFAULT 相同阈值 | 兼容 Flap 命名 |
| `2` | `HALF` | 500,000,000 | 半数即迁 | 极速迁移叙事 |
| `3` | `_95_PERCENT` | 950,000,000 | 高阈值 | 长曲线 |
| `4` | `_81_PERCENT` | 810,000,000 | 中高阈值 | 定制 |
| `5` | `_1_PERCENT` | 10,000,000 | 极低阈值 | 实验（易误触迁移） |
| `6` | `DEX_THRESH_DEFAULT` | **800,000,000** | **推荐默认** | 主网绝大多数发币 |

**注意：** 阈值单位是 **本币个数**，不是 BNB/USDX 数量；与 quote 无关。

---

### 1.2 `MigratorType` — 迁移执行器（发币参数）

**字段：** `NewTokenV6/V7Params.migratorType`（Flap 枚举；Cosm 内部映射到 `CosmTypes.MigratorType`）

| 值 | 名称 | 作用 | 如何传 | 场景 |
|----|------|------|--------|------|
| `0` | `V3_MIGRATOR` | 迁移到 PancakeSwap **V3** 池 | 无税 V7 可选 | 不要用于有税 |
| `1` | `V2_MIGRATOR` | 迁移到 PancakeSwap **V2** 池 | 有税 V6 链上**强制**此路径 | 税币唯一实际迁移器 |
| `2` | `V4_UNI_MIGRATOR` | Uniswap V4（BSC **未启用**） | 勿用 | 保留占位 |
| `3` | `PCS_INFINITY_CL_MIGRATOR` | 迁移到 Pancake **Infinity CL** | **无税 V7 推荐** | 主流无税路径 |

**Cosm 实际行为：**

| 发币类型 | 参数填什么 | 链上最终 |
|----------|------------|----------|
| 有税 V6 / 路径 B | 任意（常被忽略） | **强制 V2** |
| 无税 V7 | `3` Infinity 或 `0` V3 | 按参数 + `dexId` 解析 |

---

### 1.3 `DEXId` — 发币 MDR 序号 vs 路由 hint（两套 dexId）

**发币字段：** `NewTokenV6/V7Params.dexId`（Flap `DEXId` 枚举）  
**写入：** `getToken(token).dexId` — MDR 序号（`DEX0=0`；legacy `DEX1/2` 映射为 MDR index `2`）

**路由 hint：** `getTokenV8Safe` / `getTokenV9Safe` 的 **`dexId`** = `_dexKind(token)`，供 Portal swap 路由：

| V8Safe `dexId` | 含义 | 底层 DEX | 场景 |
|----------------|------|----------|------|
| **`2`** | V2 | Pancake V2 | **有税币**（migrator 强制 V2） |
| **`3`** | V3 | Pancake V3 | 无税 V3 迁移 |
| **`4`** | Infinity | Infinity CL | 无税 Infinity 迁移 |

Flap 发币 `DEXId` 枚举（PCS_V2=1 等）经 Portal 映射后写入 `TokenState.dexId`；**前端路由请读 V8Safe，不要读 `getToken.dexId`**。

---

### 1.4 `V3LPFeeProfile` — V3 LP 费率档

| 值 | 名称 | 作用 | 场景 |
|----|------|------|------|
| `0` | `LP_FEE_PROFILE_DEFAULT` | V3 池默认 LP 费档（0.25% 档） | 有税 V6 通常填 0 |

无税 Infinity 迁移主要走 Infinity hook，此字段影响有限。

---

### 1.5 `TokenVersion` — 发币模板版本（Flap 字段）

**字段：** `NewTokenV6Params.tokenVersion` · `NewTokenV7Params.tokenVersion` · `lockSalt(tokenVersion)`

**Cosm 链上实际只接受：**

| uint8 值 | Cosm 常量 | 含义 | 如何传 | 场景 |
|----------|-----------|------|--------|------|
| **`6`** | `COSM_VERSION_TAX_V3` | 有税 V3 模板（CosmTaxToken） | 路径 A `newTokenV6` **必为 6** | 有税发币 |
| **`7`** | `COSM_VERSION_STANDARD_V3` | 无税 V3 模板（CosmToken） | 无税 `newTokenV7` 推荐 **7** | 无税发币 |
| **`0`** | — | 无税（兼容） | 无税 V7 也可 **0** | 与 7 等价入口 |
| **`2`** | — | V3 permit 模板 | 无税 V7 可选 **2** | Flap 兼容 |

**`getToken(token).tokenVersion` 读到的也是 6 / 7**（不是 Flap 全量枚举其它值）。

**Flap SDK 枚举名**与 uint8 对齐以 Flap 文档为准；**发币 revert 时以 uint8 是否等于 6/7 为准**。

其它 enum 成员（LEGACY、TAXED_V2 等）在 Cosm BSC **发币入口会 InvalidParams**。

---

### 1.6 `FeeType` — 无税 V7 `FeeConfig[4]` 费类型

**字段：** `NewTokenV7Params.feeConfigs[i].feeType`  
**规则：** 最多 4 槽；有填写的槽位 **bps 合计须 = 10000**；无税 **不要** 用 DIVIDEND/DEFLATION。

| 值 | 名称 | 作用 | 如何传 | 场景 |
|----|------|------|--------|------|
| `0` | `NONE` | 该槽不用 | `bps=0` | 最简发币（四槽全 NONE） |
| `1` | `MARKETING_OR_VAULT` | 营销 / LP 费领取或迁移受益 | 配 `marketingAddress` + bps | 指定 Infinity LP 费收款人 |
| `2` | `DIVIDEND` | 持币分红（V7 fee 路径） | 配 dividend 字段 | **无税不要用** |
| `3` | `DEFLATION` | 通缩 | — | **无税不要用** |
| `4` | `LP_BPS` | 迁移时加 LP 比例 | 与 MARKETING 组合 bps | 例 9000 mkt + 1000 lp |

---

## 二、`CosmTypes` 核心

### 2.1 `TokenStatus` — 代币生命周期

**读取：** `getToken.status` · `getTokenV8Safe.status`（uint8）

| 值 | 名称 | 含义 | 前端行为 |
|----|------|------|----------|
| `0` | `Invalid` | 未发行 / 无效 | 不可买卖 |
| `1` | `Tradable` | **曲线阶段** | `swapExactInput` 走 bonding curve |
| `2` | `InDuel` | Flap 占位，Cosm 不用 | — |
| `3` | `Killed` | Flap 占位，Cosm 不用 | — |
| `4` | `DEX` | **已迁移 DEX** | 同一 swap API，内部走 PCS |
| `5` | `Staged` | Cosm 不用 | — |

---

### 2.2 `FeeProfile` — 协议费档位

**读取：** `getToken.feeProfile`（发币后固定，默认 GLOBAL）

| 值 | 名称 | 买/卖协议费 | 迁移 liquidity/reserve 费 | 场景 |
|----|------|-------------|---------------------------|------|
| `0` | `FEE_GLOBAL_DEFAULT` | 读 `protocolFeeBps` / `protocolSellFeeBps`（默认 100=1%） | 读 `liquidityFeeBps` 等 | **默认** |
| `1` | `FEE_FLAPSALE_V0` | 固定 **100 bps（1%）** | **0** | Flap 活动档 |
| `2` | `FEE_ZERO` | **0** | **0** | 免协议费（需链上配置） |

**与无税 1.25% 曲线费关系：** 无税 V7 曲线另读 `bondingCurveFeeBps`（默认 125），与 FeeProfile 叠加逻辑见买卖章。

---

### 2.3 `CosmTypes.MigratorType` — 状态里记录的迁移器

与 Flap `MigratorType` **数值对齐**（见 §1.2）。发币后 `getToken.migratorType` 决定 `migrateToDex` 走 V2/V3/Infinity 哪条实现。

---

### 2.4 `dividendMode`（`uint8`，非 enum，四路税约定）

**字段：** `TaxAllocation.dividendMode` · `NewTokenV6WithVaultParams.dividendMode`  
**前提：** `dividendBps > 0` 才生效；与 `dividendToken` 成对。

| 值 | 名称 | 分红发放资产 | 如何传 `dividendToken` | 场景 |
|----|------|--------------|------------------------|------|
| `0` | Quote 模式 | 发 **quote**（BNB→WBNB unwrap） | `address(0)` | BNB/USDX 分红 |
| `1` | 本币模式 | 发 **本税币** | `COSM_MAGIC_DIVIDEND_SELF` | 持币分红本币 |
| `2` | Case3 | 发 **指定 ERC20** | 任意 ERC20 + `converter` | 自定义分红币 |

**路径 B：** 可用 `COSM_MAGIC_DIVIDEND_COMPUTED`（仅 v2.3+ 工厂解析，VaultPortal 处理）。

---

## 三、金库（`VaultPortal`）

### 3.1 `RiskLevel` — 工厂/金库风险档

**读取：** `getVaultFactory` · `tryGetVault` → `VaultInfo.riskLevel`

| 值 | 名称 | 含义 | 场景 |
|----|------|------|------|
| `0` | `UNVERIFIED` | 未验证 / permissionless 工厂 | 用户自担风险 |
| `1` | `LOW_RISK` | 低风险 | 官方标注 |
| `2` | `LOW_MEDIUM_RISK` | 中低 | 官方标注 |
| `3` | `MEDIUM_RISK` | 中 | 官方标注 |
| `4` | `HIGH_RISK` | 高 | 官方标注 |

仅 **展示与筛选**；不阻止链上调用。

---

### 3.2 `VaultCategory`（Flap 遗留 ABI）与 `CosmVaultCategory`

**Flap 遗留 `VaultCategory`**（`getVaultCategory` / `getFactoryCategory` 返回值）：

| 值 | 名称 | 说明 |
|----|------|------|
| `0` | `NONE` | 未分类 |
| `1` | `TYPE_AI_ORACLE_POWERED` | Flap 占位；Cosm 非 NONE 映射到此 |

**Cosm 产品分类 `CosmVaultCategory`**（`getCosmVaultCategory` / `getCosmFactoryCategory`）：

| 值 | 名称 | 对应工厂玩法 |
|----|------|--------------|
| `0` | `NONE` | 未分类 |
| `1` | `SPLIT` | split 分账 |
| `2` | `BUYBACK` | scheduled-buyback |
| `3` | `DIVIDEND` | burn/stake/rank 等分红类 |
| `4` | `AIRDROP` | 遗留占位 |
| `5` | `GAME` | 游戏类扩展 |

---

### 3.3 `FactoryPermissionPolicy` — 工厂发币权限

**读取：** `VaultPortal.getFactoryPermissionPolicy(factory)`

| 值 | 名称 | 作用 | 场景 |
|----|------|------|------|
| `0` | `OPEN` | 任何人可路径 B 发币 | 六官方工厂默认 |
| `1` | `TIME_DEPENDENT` | 仅 developer 或过期前窗口可发 | 限时官方玩法 |
| `2` | `DISABLED` | 禁止新发币 | admin 维护 |

违反策略 → `FactoryPolicyDenied`。

---

## 四、Keeper / 基础设施

### 4.1 `OperationType` — `CosmTaxConverter` 批量操作

**读取：** `latestCallInfo(operationType)` · 事件 `CosmBatchOperationCalled`

| 值 | 名称 | 作用 | 谁调 | 场景 |
|----|------|------|------|------|
| `0` | `BatchDispatch` | 批量 `taxSplitter.dispatch()` | DISPATCHER_ROLE | 主 keeper 路径 |
| `1` | `BatchDistributeDividend` | 批量分红分发 | DISPATCHER_ROLE | 分红维护 |
| `2` | `TriggerSplit` | Case3 trigger split | DISPATCHER_ROLE | 自定义分红币 |
| `3` | `BatchDispatchPermissionless` | 无角色 batch dispatch | **任何人** | 全普通税币 |

Case3 代币 dispatch 须走 Converter（MEV 保护），见 README keeper 说明。

---

### 4.2 `TriggerStatus` — 定时回调请求状态

**读取：** `TriggerService.getRequest(id).status`

| 值 | 名称 | 含义 | 场景 |
|----|------|------|------|
| `0` | `PENDING` | 已预约，未到点或未执行 | scheduled-buyback 等待 |
| `1` | `EXECUTED` | 已成功回调 | 完成 |
| `2` | `FAILED` | 执行失败 | 可调 `retryTrigger` |

Keeper：`isRequestReady(id) && vault.getStatus().ready` 时 `trigger(id)`。

---

### 4.3 `PoolType` — SwapRegistry 池类型（Case3）

**读取：** `swapRegistry.getSwapInfo(from, to).poolType`

| 值 | 名称 | DEX | 场景 |
|----|------|-----|------|
| `0` | `V2` | Pancake V2 | 税币 quote 兑换 |
| `1` | `V3` | Pancake V3 | 无税/Case3 |
| `2` | `V4` | Uniswap V4 占位 | BSC 少用 |
| `3` | `PCS_INFINITY_CL` | Infinity CL | 新池类型 |

Case3 分红：`converter` 经 SwapRegistry 查路径兑换 dividendToken。

---

## 五、税币池状态 `PoolState`

**合约：** 税币 `CosmTaxToken` / `ICosmTaxTokenV3`  
**作用：** 控制 transfer 是否收税、anti-farmer 窗口。

| 值 | 名称 | 含义 | 场景 |
|----|------|------|------|
| `0` | `BondingCurve` | 曲线阶段 | 与 Portal Tradable 对应 |
| `1` | `Migrating` | 迁移中 | 短暂过渡 |
| `2` | `TaxEnforcedAntiFarmer` | 迁移后 anti-farmer：**所有 `pools` 映射内池**买卖收税 | `antiFarmerDuration > 0` |
| `3` | `TaxEnforced` | 仅 **`mainPool`** 买卖收税 | 窗口结束 |
| `4` | `TaxFree` | 税过期，永久免税 | `taxDuration` 到期 |

前端一般读 Portal `status`；调试 tax hook 可读 token `poolState()`。

---

## 六、金库 `vaultData` 模式字段（非 Solidity enum）

scheduled-buyback 工厂 `vaultDataSchema` 中的 **`uint8` 字段**，语义为固定档位：

### 6.1 `triggerMode`

| 值 | 含义 | 触发条件 |
|----|------|----------|
| `0` | 仅时间 | 间隔到 + 有余额 |
| `1` | 金额 + 间隔 | 余额 ≥ `minBnbAmount` 且过间隔 |
| `2` | 时间 + 金额 | 两者同时满足 |

### 6.2 `buybackMode`

| 值 | 含义 | 链上行为 |
|----|------|----------|
| `0` | Token 回购 | PCS 买本币 → burn |
| `1` | LP 回购 | 买 LP → burn（失败回退 token） |

---

## 七、枚举传参场景速查

| 我要… | 关键枚举 / 值 |
|-------|----------------|
| 无税发币 + Infinity | `dexThresh=6` · `migratorType=3` · `dexId=3` · `tokenVersion=7` |
| 有税发币 + 钱包 mkt | `tokenVersion=6` · migrator 强制 V2 · 发币 `dexId=1`(PCS_V2) · **路由读 V8Safe.dexId=2** |
| 路径 B 金库 | `newCosmTokenV6WithVault`（推荐）或 Flap 布局 · `quoteToken=0` · `mktBps>0` |
| 判断曲线还是 DEX | `TokenStatus` 1 vs 4 |
| 持币分红发 BNB | `dividendMode=0` · `dividendToken=0` |
| 持币分红发本币 | `dividendMode=1` · SELF 魔法地址 |
| keeper dispatch | `OperationType.BatchDispatch` 或 Permissionless |
| 定时回购 | `triggerMode`/`buybackMode` in vaultData · `TriggerStatus.PENDING` |

---

*源码：`ICosmPortalTypes.sol` · `CosmTypes.sol` · `CosmVaultPortal.sol` · `CosmTaxConverter.sol` · `ICosmTriggerService.sol` · `ISwapRegistry.sol` · `ITokenV3.sol`*

