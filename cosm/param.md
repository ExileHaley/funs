# Cosm 项目结构体与枚举参考

> 来源：`contracts/`（不含 `vendor/`）。字段顺序与 Solidity 一致。  
> 注释格式：`字段 // 使用场景 · 约束/默认值 · [Flap 兼容] 说明`

---

## 创建买满 8 亿：`quoteAmt` 怎么估

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

**有没有链上直接估？** 没有 `quoteAmtForThreshold()` 这类专用 view。可用：

| 时机 | 做法 |
|------|------|
| **发币前（推荐）** | `simulateContract(newTokenV6/V7, { value: quoteAmt })`，看模拟结果里 `circulatingSupply` 是否到 `800_000_000 ether`；不够就加大 `quoteAmt` 再 simulate |
| **已有 token（曲线阶段）** | `quoteExactInput({ inputToken: quote, outputToken: token, inputAmount: X })` → 返回能买到的 token 数；`X` 试到输出 ≥ `dexSupplyThresh − circulatingSupply`（**非 view**，会走 Quoter） |
| **发币前（离线）** | 任意已发 Cosm 币调 `getTokenV8Safe` 取 `r/h/k/dexSupplyThresh`，本地用 `LibCurve.estimateReserve` + 上表公式（与链上同一套曲线） |

> 发币前 **不能** 对「还不存在的 token 地址」调 `quoteExactInput`（Portal 里还没有状态）。

---

# CosmTypes (`contracts/CosmTypes.sol`)

常量：`COSM_VERSION_TAX_V3 = 6`（税币 CosmTaxToken）· `COSM_VERSION_STANDARD_V3 = 7`（普通币 CosmToken）

## enum TokenStatus

> 场景：`getToken()` / `TokenState.status` / 列表页 `getTokenV8Safe.status`（uint8 同 ordinal）

```solidity
enum TokenStatus {
    Invalid,    // 0  未注册或未知地址；默认初始态
    Tradable,   // 1  曲线阶段；Portal 可 quote/swap；可 auto-migrate
    InDuel,     // 2  [Flap 占位] Cosm BSC 未用；索引器勿当作有效态
    Killed,     // 3  [Flap 占位] Cosm BSC 未用
    DEX,        // 4  已迁移；曲线 reserve 清零；DEX 买卖 / transfer tax 生效
    Staged      // 5  [Flap 占位] Cosm BSC 未用
}
```

## enum FeeProfile

> 场景：发币写入 `TokenState.feeProfile`；决定曲线协议费与迁移费档位

```solidity
enum FeeProfile {
    FEE_GLOBAL_DEFAULT,  // 0  默认；曲线协议费读 Portal.protocolFeeBps(100) / protocolSellFeeBps(100)；迁移费读 liquidityFeeBps / reserveFeeBps
    FEE_FLAPSALE_V0,     // 1  [Flap 兼容] 曲线买卖协议费固定 100 bps；迁移 LP/储备费=0
    FEE_ZERO             // 2  曲线协议费=0；迁移费=0；白名单/特殊活动
}
```

## enum MigratorType

> 场景：发币 `LaunchParams.migratorType`；税币 launch 代码强制 V2

```solidity
enum MigratorType {
    V3_MIGRATOR,              // 0  普通币可选；迁移到 PCS V3 + GoPlus locker
    V2_MIGRATOR,              // 1  税币强制；迁移到 PCS V2 pair
    V4_UNI_MIGRATOR,          // 2  [Flap 预留] BSC 未启用
    PCS_INFINITY_CL_MIGRATOR  // 3  普通币 V7 默认；Infinity CL + hook
}
```

## struct TaxAllocation

> 场景：税币发币四路拆分；`mkt+deflation+dividend+lp` 须 = 10000 bps；普通币全 0  
> 路径 A：`mktBps` → beneficiary 钱包；路径 B：`mktBps` → 金库地址

```solidity
struct TaxAllocation {
    uint16 mktBps;                  // 营销/金库份额(bps)；曲线/DEX 税费经 TaxSplitter 拆分后进 market；全 0 时 Portal 默认 10000
    uint16 deflationBps;            // 回购销毁(bps)；dispatch 时买本币打 0xdead
    uint16 dividendBps;             // 持币分红(bps)；>0 部署 CosmDividend；=0 不部署
    uint16 lpBps;                   // 加 LP(bps)；迁移后 TaxSplitter 用 quote 加流动性
    uint256 minimumShareBalance;    // 分红最小持币量(最小单位)；dividendBps>0 建议 ≥ 10_000 ether(1万整币)
    uint8 dividendMode;             // 分红发放资产：0=quote(WBNB 若 BNB) · 1=本代币 · 2=其他 ERC20
    address dividendToken;          // mode=2 必填具体 token；mode=0/1 填 address(0)
    uint256 antiFarmerDuration;     // DEX 后 anti-farmer 秒数；窗口内全 pools 收税；0=跳过窗口仅 mainPool；上限 365 天
    uint256 taxDuration;            // 税收总有效期(秒)；0=永不过期；>0 须 ≥ antiFarmerDuration；上限约 100 年
    uint16 mktBps2;                 // 从 mktBps 再切分(bps)；mkt2+mkt3+mkt4 ≤ mktBps；默认 0
    uint16 mktBps3;                 // 第三路营销(bps)；默认 0
    uint16 mktBps4;                 // 第四路营销(bps)；默认 0
    address market2;                // mktBps2>0 必填；dispatch 收款地址
    address market3;                // mktBps3>0 必填
    address market4;                // mktBps4>0 必填
    address converter;              // dividendMode=2 必填；Case3 唯一可触发 quote→dividendToken swap 的地址
}
```

## struct LaunchParams

> 场景：Portal 内部统一发币结构；由 `newTokenV6/V7` 或 VaultPortal 组装

```solidity
struct LaunchParams {
    string name;                    // ERC20.name()；链上代币名称
    string symbol;                  // ERC20.symbol()；链上符号
    string meta;                    // metaURI(IPFS/HTTPS)；图标/描述在 JSON 内
    bytes32 salt;                   // CREATE2 salt；预测地址低 16 bit：税币 0x0111 · 普通 0x0222
    address quoteToken;             // 曲线 quote；address(0)=BNB · 或 USDT/USDC/USD1/USDX
    uint256 quoteAmt;               // 发币同步买入 quote 量(最小单位)；0=只发不买；BNB 用 msg.value · ERC20 先 approve
    address beneficiary;            // 营销/金库收款(TaxSplitter.market)；路径 A=钱包 · B=金库 · 普通无税=0
    uint16 buyTaxBps;               // 曲线买入侧 tax(bps)；普通币=0；税币至少 buy/sell 一侧>0
    uint16 sellTaxBps;              // 曲线卖出侧 tax(bps)；普通币=0
    bool isTaxed;                   // true=税币(tokenVersion 6/newTokenV6) · false=普通(7/newTokenV7)
    TaxAllocation tax;              // 四路 tax；普通币全 0
    MigratorType migratorType;      // 迁移目标；税币强制 V2 · 普通默认 Infinity 可选 V3
}
```

## struct TokenState

> 场景：`CosmPortal.getToken(token)` 完整链上状态；含 taxSplitter/dividend/vault 等扩展字段

```solidity
struct TokenState {
    TokenStatus status;             // 生命周期；Invalid=0 · Tradable=1 · DEX=4
    uint8 tokenVersion;             // 6=CosmTaxToken · 7=CosmToken
    address quoteToken;             // 发币锁定 quote；交易/税/迁移均用此 token
    uint128 reserve;                // 曲线 quote 储备(最小单位)；迁移后=0
    uint128 circulatingSupply;      // 曲线流通量(最小单位)；≥ dexSupplyThresh 可迁移
    uint256 buyTaxBps;              // 买入 transfer tax(bps)；DEX 阶段 token 转账税
    uint256 sellTaxBps;             // 卖出 transfer tax(bps)
    address beneficiary;            // 营销/金库地址；LP 费 claim 也要求此身份
    uint256 progress;               // 迁移进度 Wad：circulatingSupply*1e18/dexSupplyThresh；迁移后=1e18
    TaxAllocation tax;              // 发币时锁定的四路配置(不可改)
    address taxSplitter;            // CosmTaxSplitter clone；普通无税可能为 TaxSplitterLite；无=0
    address dividend;               // CosmDividend clone；dividendBps=0 时为 0
    address pool;                   // 迁移后主池；税=V2 pair · 普通=V3/Infinity pool；迁移前=0
    address vault;                  // 路径 B 金库；路径 A/普通=0；也可 VaultPortal.tryGetVault
    FeeProfile feeProfile;          // 协议费档位；默认 FEE_GLOBAL_DEFAULT(0)
    MigratorType migratorType;      // 发币选择的迁移器
    address infinityHook;           // Infinity CL tax hook；非 Infinity 路径=0
    uint16 bondingCurveFeeBps;      // V7 普通币曲线协议费(bps)；默认 125(1.25%)；税币=0
    uint8 lpFeeProfile;             // [Flap V3LPFeeProfile ordinal] 写入 token；V3 常用 0(0.25% tier)
    uint128 dexSupplyThresh;        // 迁移阈值(token 最小单位)；0=默认 800_000_000 ether
    bytes32 extensionID;            // 扩展插件 ID；无扩展=bytes32(0)；注册 extension 时可非 0
    uint8 dexId;                    // 发币时 MDR preferredDexId(0/1/2)；≠ getTokenV8Safe.dexId(路由 hint)
}
```

## struct TokenStateV8Safe

> 场景：`getTokenV8Safe` 列表/行情页轻量查询；缺 taxSplitter/dividend/beneficiary 等 → 用 getToken

```solidity
struct TokenStateV8Safe {
    uint8 status;                   // TokenStatus ordinal；0=Invalid · 1=Tradable(曲线) · 4=DEX
    uint256 reserve;                // 曲线 quote 储备；DEX 后通常 0
    uint256 circulatingSupply;      // 曲线流通量
    uint256 price;                  // 边际价 quote/1e18 token(Wad)；DEX 阶段用 dexSupplyThresh 估算
    uint8 tokenVersion;             // 6=税 · 7=普通
    uint256 r;                      // 曲线虚拟 quote 储备 r；全局曲线参数
    uint256 h;                      // 曲线虚拟 token 储备 h
    uint256 k;                      // 曲线常数 k
    uint256 dexSupplyThresh;        // 迁移阈值；默认 800_000_000 ether
    address quoteTokenAddress;      // quote 地址；BNB=address(0)
    bool nativeToQuoteSwapEnabled;  // ERC20 quote 是否支持一键 BNB 买；USDT/USDC/USD1=true · BNB/USDX=false
    bytes32 extensionID;            // 扩展插件 ID；= TokenState.extensionID；无扩展=bytes32(0)
    uint256 buyTaxRate;             // = buyTaxBps
    uint256 sellTaxRate;            // = sellTaxBps
    address pool;                   // 迁移后 DEX 池；未迁移=0
    uint256 progress;               // Wad 进度；1e18=已达/已过迁移阈值
    uint8 lpFeeProfile;             // LP 费率档位 ordinal；V3 常用 0 · Infinity 实际 8534ppm
    uint8 dexId;                    // 路由 hint(_dexKind)：2=V2 · 3=V3 · 4=Infinity；税币恒 2
}
```

## struct TokenStateV9Safe

> 场景：`getTokenV9Safe`；V8Safe + bondingCurveFeeRate（Flap SDK 兼容）

```solidity
struct TokenStateV9Safe {
    uint8 status;                   // 同 TokenStatus ordinal；0=Invalid · 1=Tradable · 4=DEX
    uint256 reserve;                // 曲线 quote 储备(最小单位)；DEX 后=0；展示用
    uint256 circulatingSupply;      // 曲线流通量(最小单位)；迁移进度计算分子
    uint256 price;                  // 边际价 Wad(quote/1e18 token)；DEX 用阈值 supply 估算
    uint8 tokenVersion;             // 6=税币 CosmTaxToken · 7=普通 CosmToken
    uint256 r;                      // 曲线参数 r；与 getTokenV8 相同全局曲线
    uint256 h;                      // 曲线参数 h
    uint256 k;                      // 曲线参数 k
    uint256 dexSupplyThresh;        // 迁移阈值 token 量；默认 800_000_000 ether
    address quoteTokenAddress;      // quote 地址；BNB=address(0)
    bool nativeToQuoteSwapEnabled;  // 是否支持 msg.value 一键 BNB 买；USDT/USDC/USD1=true
    bytes32 extensionID;            // 扩展 ID；无插件=0
    uint256 buyTaxRate;             // 买入 tax bps；= TokenState.buyTaxBps
    uint256 sellTaxRate;            // 卖出 tax bps；= TokenState.sellTaxBps
    address pool;                   // 迁移后主 DEX 池；未迁移=0
    uint256 progress;               // 迁移进度 Wad；1e18=已达阈值
    uint8 lpFeeProfile;             // V3LPFeeProfile ordinal；V3 默认 0
    uint8 dexId;                    // 路由 hint：2=V2 · 3=V3 · 4=Infinity；≠ TokenState.dexId
    uint16 bondingCurveFeeRate;     // V7 曲线协议费 bps；= bondingCurveFeeBps；默认 125 · 税币 0
}
```

## struct QuoteParams

> 场景：`quoteExactInput` 询价；曲线或 DEX 均可

```solidity
struct QuoteParams {
    address inputToken;             // 付出的 token；买 meme=quote · 卖 meme=meme 地址
    address outputToken;            // 收到的 token；买 meme=meme · 卖 meme=quote
    uint256 inputAmount;            // 输入数量(最小单位)
}
```

## struct SwapParams

> 场景：`swapExactInput` 成交；Tradable 或 DEX

```solidity
struct SwapParams {
    address inputToken;             // 付出 token；方向同 QuoteParams
    address outputToken;            // 收到 token
    uint256 inputAmount;            // 输入(最小单位)；BNB 买时 msg.value≥此值
    uint256 minOutputAmount;        // 滑点保护最小输出；不足 revert Slippage
    bytes permitData;               // 可选 EIP-2612：abi.encode(deadline,v,r,s)；空=须 approve
}
```

## struct SaltLockEntry

> 场景：CREATE2 salt 预约；`lockSalt` / 发币成功后标记

```solidity
struct SaltLockEntry {
    address locker;                 // 预约者；0=未锁定
    uint8 tokenVersion;             // 预留模板：6 或 7
    bool isUsed;                    // 发币成功后 true；同 salt 不可再用
}
```

---

# ICosmPortalTypes (`contracts/interfaces/ICosmPortalTypes.sol`)

> Flap SDK 兼容 ABI；CosmPortal 对外 `newTokenV6/V7`、quote 配置、swap 参数

Magic：`COSM_MAGIC_DIVIDEND_SELF = 0xfEED…`（本代币分红）· `COSM_MAGIC_DIVIDEND_COMPUTED = 0xC0De…`（Vault 路径解析）

## enum CurveType

> 场景：`QuoteTokenConfiguration.defaultCurve/alternativeCurve`；决定 bonding curve (r,h,k)

```solidity
enum CurveType {
    CURVE_LEGACY_15,              // 0   [Flap 遗留] Cosm 未用
    CURVE_4,                      // 1   [Flap 遗留] Cosm 未用
    CURVE_0_974,                  // 2   [Flap 遗留] Cosm 未用
    CURVE_0_5,                    // 3   [Flap 遗留] Cosm 未用
    CURVE_1000,                   // 4   [Flap 遗留] Cosm 未用
    CURVE_20000,                  // 5   [Flap 遗留] Cosm 未用
    CURVE_2500,                   // 6   [Flap 遗留] Cosm 未用
    CURVE_500,                    // 7   [Flap 遗留] Cosm 未用
    CURVE_2,                      // 8   [Flap 遗留] Cosm 未用
    CURVE_6,                      // 9   [Flap 遗留] Cosm 未用
    CURVE_75,                     // 10  [Flap 遗留] Cosm 未用
    CURVE_4M,                     // 11  [Flap 遗留] Cosm 未用
    CURVE_28,                     // 12  [Flap 遗留] Cosm 未用
    CURVE_21_25,                  // 13  [Flap 遗留] Cosm 未用
    CURVE_RH_UNUSED,              // 14  [Flap 占位] Cosm 未用
    CURVE_RH_28D25_108002126,     // 15  [Flap RH] Cosm 未用
    CURVE_RH_14981_108002125,     // 16  [Flap RH] Cosm 未用
    CURVE_RH_TOSHI_MORPH_2ETH,    // 17  [Flap RH] Cosm 未用
    CURVE_RH_TOSHI,               // 18  [Flap RH] Cosm 未用
    CURVE_RH_BGB,                 // 19  [Flap RH] Cosm 未用
    CURVE_RH_BNB,                 // 20  BSC BNB quote 默认曲线(_quoteTokenConfiguration 默认)
    CURVE_RH_USD,                 // 21  稳定币 quote 常用曲线
    CURVE_RH_MONAD,               // 22  [Flap 其他链] Cosm 未用
    CURVE_RH_MONAD_V2,            // 23  [Flap 其他链] Cosm 未用
    CURVE_RH_KGST,                // 24  [Flap 其他链] Cosm 未用
    CURVE_RH_TOSHI_5ETH,          // 25  [Flap RH] Cosm 未用
    CURVE_RH_RESERVED,            // 26  [Flap 占位] Cosm 未用
    CURVE_RH_BNB_1X5_32BNB        // 27  [Flap RH 变体] Cosm 未用
}
```

## enum DexThreshType

> 场景：`NewTokenV6/V7Params.dexThresh` → 映射 `dexSupplyThresh`

```solidity
enum DexThreshType {
    TWO_THIRDS,     // 0 → 667_000_000 ether (66.7%)
    FOUR_FIFTHS,    // 1 → 800_000_000 ether (80%) · Cosm 默认
    HALF,           // 2 → 500_000_000 ether (50%)
    _95_PERCENT,    // 3 → 950_000_000 ether
    _81_PERCENT,    // 4 → 810_000_000 ether
    _1_PERCENT      // 5 → 10_000_000 ether (1%)
}
// Flap SDK ordinal 6 亦映射 800M（legacy 兼容）
```

## enum MigratorType

> 同 CosmTypes.MigratorType；ordinal 一致

```solidity
enum MigratorType {
    V3_MIGRATOR,              // 0  普通币可选；迁移 PCS V3
    V2_MIGRATOR,              // 1  税币强制；迁移 PCS V2 pair
    V4_UNI_MIGRATOR,          // 2  [Flap 预留] BSC 未启用
    PCS_INFINITY_CL_MIGRATOR  // 3  普通 V7 默认；Infinity CL
}
```

## enum DEXId

> 场景：发币 `dexId` → MultiDexRouter preferredDexId(0/1/2)

```solidity
enum DEXId {
    DEX0,   // 0 → MDR index 0 · BSC 默认
    DEX1,   // 1 → MDR index 2
    DEX2    // 2 → MDR index 2
}
```

## enum NativeToQuoteSwapType

> 场景：`QuoteTokenConfiguration`；BNB 一键买 ERC20-quote 代币时的换汇路径

```solidity
enum NativeToQuoteSwapType {
    SWAP_DISABLED,           // 0  不支持 native→quote；BNB quote 默认
    SWAP_VIA_V2_POOL,        // 1  经 V2 池换 quote
    SWAP_VIA_V3_2500_POOL,   // 2  经 V3 0.25% 池
    SWAP_VIA_V3_500_POOL,    // 3  经 V3 0.05% 池
    SWAP_VIA_V3_3000_POOL,   // 4  经 V3 0.3% 池
    SWAP_VIA_V3_10000_POOL,  // 5  经 V3 1% 池
    SWAP_VIA_MIXED_ROUTER    // 6  混合路由
}
```

## enum PoolType

> 场景：`PackedDexPool` / SwapRegistry；池子类型标识

```solidity
enum PoolType {
    V2,              // 0  Pancake V2 pair
    V3,              // 1  Pancake V3 CL pool
    V4,              // 2  Uniswap V4 方向 · Case3 swap
    PCS_INFINITY_CL  // 3  Pancake Infinity CL
}
```

## enum V3LPFeeProfile

> 场景：税币 V6 发币 `lpFeeProfile`；影响迁移后 V3 LP 费率 tier（税币走 V2 时仅写入 state）

```solidity
enum V3LPFeeProfile {
    LP_FEE_PROFILE_STANDARD,  // 0  0.25% (2500) · 默认
    LP_FEE_PROFILE_LOW,       // 1  0.05% (500)
    LP_FEE_PROFILE_HIGH       // 2  1% (10000)
}
```

## enum TokenVersion

> 场景：Flap SDK 发币版本 ordinal；Cosm 仅用后两项

```solidity
enum TokenVersion {
    TOKEN_LEGACY_MINT_NO_PERMIT,           // 0  [Flap 遗留] Cosm 未用
    TOKEN_LEGACY_MINT_NO_PERMIT_DUPLICATE, // 1  [Flap 遗留] Cosm 未用
    TOKEN_V2_PERMIT,                       // 2  [Flap 遗留] Cosm 未用
    TOKEN_GOPLUS,                          // 3  [Flap 遗留] Cosm 未用
    TOKEN_TAXED,                           // 4  [Flap 遗留] Cosm 未用
    TOKEN_TAXED_V2,                        // 5  [Flap 遗留] Cosm 未用
    TOKEN_TAXED_V3,                        // 6  Cosm 税币 · newTokenV6 必填
    TOKEN_V3_PERMIT                        // 7  Cosm 普通币 · newTokenV7
}
```

## enum FeeType

> 场景：V7 `FeeConfig[4]` 四槽位类型；每槽最多一种

```solidity
enum FeeType {
    NONE,                 // 0  空槽；Flap 默认未用槽=0
    MARKETING_OR_VAULT,   // 1  营销/金库 bps + marketingAddress
    DIVIDEND,             // 2  分红 bps + dividendToken + minimumShareBalance
    DEFLATION,            // 3  销毁 bps
    LP_BPS                // 4  加 LP bps（普通 V7 可与 mkt 组合，和须=10000）
}
```

## struct QuoteTokenConfiguration

> 场景：Portal owner `setQuoteTokenConfiguration`；未设置时用 BNB 默认

```solidity
struct QuoteTokenConfiguration {
    uint8 enabled;                              // 1=启用该 quote 发币/交易 · 0=禁用
    CurveType defaultCurve;                     // 默认 CURVE_RH_BNB(BNB) 或 CURVE_RH_USD(稳定币)
    CurveType alternativeCurve;                 // 备用曲线；默认与 default 相同
    NativeToQuoteSwapType nativeToQuoteSwapType; // ERC20 quote 的 BNB 换汇方式；BNB quote=SWAP_DISABLED(0)
    uint8 dexId;                                // 默认 MDR preferredDexId；BNB 默认 0(DEX0)
}
```

## struct PackedDexPool

> 场景：Portal 存储 `_tokenPackedDexPools`；[Flap 扩展 DEX 打包布局]

```solidity
struct PackedDexPool {
    address pool;           // 池地址
    uint24 fee;             // V3/CL fee tier (ppm 或 uint24 fee)
    PoolType poolType;      // V2/V3/V4/Infinity
    uint24 tickSpacing;     // CL tick spacing；V2 可 0
    uint40 unused;          // [Flap 对齐 padding] Cosm 未用 · 默认 0
}
```

## struct FeeConfig

> 场景：V7 发币 `feeConfigs[4]`；解析为 TaxAllocation + beneficiary

```solidity
struct FeeConfig {
    FeeType feeType;                // 槽位类型；NONE=空
    uint16 bps;                     // 该路 bps；四路有效项之和须=10000(V7 tax) 或 mkt+lp=10000(普通)
    address marketingAddress;       // feeType=MARKETING_OR_VAULT 时 beneficiary/mkt 地址
    address dividendToken;          // feeType=DIVIDEND；0/SELF magic 或具体 token
    uint256 minimumShareBalance;    // feeType=DIVIDEND 时最小持币；建议 ≥10_000 ether
}
```

## struct NewTokenV6Params

> 场景：`CosmPortal.newTokenV6` 路径 A 税币发币（Flap 布局兼容）

```solidity
struct NewTokenV6Params {
    string name;                    // 代币名称
    string symbol;                  // 代币符号
    string meta;                    // metaURI
    DexThreshType dexThresh;        // 迁移阈值枚举；默认 FOUR_FIFTHS(800M)
    bytes32 salt;                   // CREATE2；低 16 bit=0x0111
    MigratorType migratorType;      // 传入值忽略；Cosm 强制 V2_MIGRATOR
    address quoteToken;             // address(0)=BNB 或 whitelist ERC20
    uint256 quoteAmt;               // 发币同步买入；0=不买
    address beneficiary;            // 路径 A 营销钱包；必填非 0
    bytes permitData;               // quote ERC20 发币 approve 替代；BNB 填空
    bytes32 extensionID;            // 扩展插件 ID；无=bytes32(0)
    bytes extensionData;            // 扩展初始化 calldata；无扩展填空
    DEXId dexId;                    // MDR 序号；默认 DEX0
    V3LPFeeProfile lpFeeProfile;    // 写入 TokenState；税币走 V2 时仍记录 · 默认 STANDARD(0)
    uint16 buyTaxRate;              // 买入 tax(bps)；至少一侧>0
    uint16 sellTaxRate;             // 卖出 tax(bps)
    uint64 taxDuration;             // 税收总时长(秒)；0=永不过期
    uint64 antiFarmerDuration;      // anti-farmer(秒)；0=跳过
    uint16 mktBps;                  // 营销 bps；四路和=10000
    uint16 deflationBps;            // 销毁 bps
    uint16 dividendBps;             // 分红 bps
    uint16 lpBps;                   // 加 LP bps
    uint256 minimumShareBalance;    // 分红最小持币
    address dividendToken;          // Flap magic：0=quote · SELF=本币 · 其他=mode2
    address commissionReceiver;     // 佣金接收；0=无佣金
    TokenVersion tokenVersion;      // 须 TOKEN_TAXED_V3(6)
}
```

## struct NewTokenV7Params

> 场景：`CosmPortal.newTokenV7` 普通币或 V7 税币（feeConfigs）

```solidity
struct NewTokenV7Params {
    string name;                    // ERC20.name()；前端展示/钱包显示
    string symbol;                  // ERC20.symbol()；交易对符号
    string meta;                    // metaURI；代币图标/描述 JSON 链接
    DexThreshType dexThresh;        // 迁移阈值枚举；默认 FOUR_FIFTHS→800M
    bytes32 salt;                   // CREATE2；普通低16bit=0x0222 · 税=0x0111
    MigratorType migratorType;      // 普通默认 Infinity(3)；可显式 V3(0)
    address quoteToken;             // 曲线 quote；0=BNB · 或 USDT/USDC/USD1/USDX
    uint256 quoteAmt;               // 发币同步买入量；0=不买；BNB→msg.value
    bytes permitData;               // ERC20 quote 的 EIP-2612；空=须先 approve Portal
    bytes32 extensionID;            // 已注册扩展 ID；无=bytes32(0)
    bytes extensionData;            // 扩展 onLaunch calldata；无扩展=空 bytes
    DEXId dexId;                    // MultiDexRouter 序号；默认 DEX0
    uint16 buyTaxRate;              // 买入 tax bps；普通币=0 · V7 税币>0
    uint16 sellTaxRate;             // 卖出 tax bps；普通币=0
    uint64 taxDuration;             // 税收总时长(秒)；普通=0 · 0=永不过期
    uint64 antiFarmerDuration;      // DEX 后 anti-farmer(秒)；普通=0
    address commissionReceiver;     // 税币佣金地址；0=无 commission
    TokenVersion tokenVersion;      // TOKEN_V3_PERMIT(7) 普通 · TOKEN_TAXED_V3(6) 税
    FeeConfig[4] feeConfigs;        // 四槽 fee 配置；普通 mkt+lp=10000 · 税四路=10000
}
```

## struct ExactInputV3Params

> 场景：Flap `swapExactInputV3` 兼容入口；Cosm DEX V3 买卖

```solidity
struct ExactInputV3Params {
    address inputToken;             // 输入 token
    address outputToken;            // 输出 token
    uint256 inputAmount;            // 输入量
    uint256 minOutputAmount;        // 滑点下限
    bytes permitData;               // EIP-2612；空=approve
    bytes extensionData;            // [Flap 扩展回调] Cosm 扩展用；通常空
}
```

---

# CosmVaultPortal (`ICosmVaultPortalTypes.sol`)

> 路径 B：发币 + 金库 · BSC proxy `0x9e9e9a2392a379fA03c268098Cd9374d7885c55D`  
> 薄代理 delegatecall：Lens(读) / Launch(发) / Tweak(管理)

## enum RiskLevel

> 场景：工厂/金库风险标签；前端展示

```solidity
enum RiskLevel {
    UNVERIFIED,       // 0  无许可/未注册工厂默认；前端应警示
    LOW_RISK,         // 1  官方审计低危工厂
    LOW_MEDIUM_RISK,  // 2  中偏低；admin 手动标注
    MEDIUM_RISK,      // 3  中等风险
    HIGH_RISK         // 4  高危/未审计；前端强警示
}
```

## enum VaultCategory

> 场景：[Flap 遗留 ABI] `setVaultCategory` / `getVaultCategory`

```solidity
enum VaultCategory {
    NONE,                    // 0  未分类 · 默认
    TYPE_AI_ORACLE_POWERED   // 1  [Flap 占位] Cosm 非 NONE 映射到此
}
```

## enum CosmVaultCategory

> 场景：Cosm 扩展分类 `getCosmVaultCategory` / `getCosmFactoryCategory`

```solidity
enum CosmVaultCategory {
    NONE,      // 0  未分类 · 默认
    SPLIT,     // 1  CosmSplitVault 拆分
    BUYBACK,   // 2  定时回购类
    DIVIDEND,  // 3  质押/燃烧分红类
    AIRDROP,   // 4  历史占位 · 当前未用
    GAME       // 5  游戏类 · 预留
}
```

## enum FactoryPermissionPolicy

> 场景：注册工厂时策略；控制谁可调用该工厂发币

```solidity
enum FactoryPermissionPolicy {
    OPEN,            // 0  任何人可发 · 默认(未注册工厂)
    TIME_DEPENDENT,  // 1  限时独占：过期前仅 developer
    DISABLED         // 2  禁止发币
}
```

## enum FactoryValidationMode

> 场景：工厂注册 `validationMode`；发币前 hook 类型

```solidity
enum FactoryValidationMode {
    NONE,      // 0  不校验 factory hook · permissionless 默认
    LEGACY_V6, // 1  Flap onBeforeNewTokenV6WithVault 布局
    V22        // 2  扩展校验(含 dividendMode/converter 等)
}
```

## struct NewTokenV6WithVaultParams

> 场景：`newTokenV6WithVault` Flap 布局路径 B

```solidity
struct NewTokenV6WithVaultParams {
    string name;                                    // 代币名称
    string symbol;                                  // 符号
    string meta;                                    // metaURI
    ICosmPortalTypes.DexThreshType dexThresh;       // 迁移阈值；默认 FOUR_FIFTHS
    bytes32 salt;                                   // CREATE2；0x0111
    ICosmPortalTypes.MigratorType migratorType;     // 忽略；Cosm 强制 V2
    address quoteToken;                             // 须 vaultFactory.isQuoteTokenSupported；BNB-only 工厂须 address(0)
    uint256 quoteAmt;                               // 同步买入；BNB msg.value
    bytes permitData;                               // ERC20 quote permit；空=approve
    bytes32 extensionID;                          // 默认 bytes32(0)
    bytes extensionData;                            // 扩展数据；默认空
    ICosmPortalTypes.DEXId dexId;                   // 默认 DEX0
    ICosmPortalTypes.V3LPFeeProfile lpFeeProfile;   // 默认 STANDARD(0)
    uint16 buyTaxRate;                              // 买入 tax bps
    uint16 sellTaxRate;                             // 卖出 tax bps
    uint64 taxDuration;                             // 0=永不过期
    uint64 antiFarmerDuration;                      // 0=跳过 anti-farmer 窗口
    uint16 mktBps;                                  // 进金库份额(bps)；须>0；TaxSplitter.market=金库地址
    uint16 deflationBps;                            // 销毁 bps；四路之和=10000
    uint16 dividendBps;                             // 分红 bps；>0 部署 CosmDividend
    uint16 lpBps;                                   // 加 LP bps
    uint256 minimumShareBalance;                    // 分红最小持币；dividendBps>0 建议≥10_000 ether
    address dividendToken;                            // Flap magic：0=quote · SELF=本币 · 具体地址=其他 token
    address commissionReceiver;                     // 0=无佣金
    ICosmPortalTypes.TokenVersion tokenVersion;     // 须 TOKEN_TAXED_V3(6)
    address vaultFactory;                           // 金库工厂；必填
    bytes vaultData;                                // abi.encode 工厂参数；见 vaultDataSchema
}
```

## struct NewCosmTokenV6WithVaultParams

> 场景：`newCosmTokenV6WithVault` Cosm 推荐路径 B（显式 dividendMode）

```solidity
struct NewCosmTokenV6WithVaultParams {
    string name;                    // ERC20.name()
    string symbol;                  // ERC20.symbol()
    string meta;                    // metaURI
    bytes32 salt;                   // CREATE2；税币低16bit=0x0111
    address quoteToken;             // 须 factory.isQuoteTokenSupported(quote)
    uint256 quoteAmt;               // 发币同步买入；BNB 含在 msg.value
    uint16 buyTaxBps;               // 曲线买入 tax(bps)；至少 buy/sell 一侧>0
    uint16 sellTaxBps;              // 曲线卖出 tax(bps)
    uint16 mktBps;                  // 营销份额→金库；四路 bps 和=10000
    uint16 deflationBps;            // 销毁 bps
    uint16 dividendBps;             // 分红 bps；>0 部署 Dividend
    uint16 lpBps;                   // 加 LP bps
    uint256 minimumShareBalance;   // 分红门槛(最小单位)
    uint8 dividendMode;             // 0=quote · 1=本代币 · 2=其他 ERC20
    address dividendToken;          // mode=2 必填目标 token；0/1 填 0
    address converter;              // mode=2 必填；Case3 swap 触发者
    uint256 antiFarmerDuration;     // 秒；0=跳过 anti-farmer 窗口
    uint256 taxDuration;            // 秒；0=税收永不过期
    address vaultFactory;           // 金库工厂合约；必填
    bytes vaultData;                // abi.encode 工厂参数；见 factory.vaultDataSchema()
}
```

## struct NewTokenV7WithVaultParams

> 场景：`newTokenV7WithVault` 普通/税币 + 金库

```solidity
struct NewTokenV7WithVaultParams {
    string name;                                    // ERC20.name()
    string symbol;                                  // ERC20.symbol()
    string meta;                                    // metaURI
    ICosmPortalTypes.DexThreshType dexThresh;       // 迁移阈值；默认 FOUR_FIFTHS
    bytes32 salt;                                   // CREATE2 vanity 后缀
    ICosmPortalTypes.MigratorType migratorType;     // 普通默认 Infinity(3)
    address quoteToken;                             // factory 支持的 quote
    uint256 quoteAmt;                               // 同步买入量；0=不买
    bytes permitData;                               // ERC20 permit；空=approve
    bytes32 extensionID;                            // 扩展 ID；默认 0
    bytes extensionData;                            // 扩展 calldata；默认空
    ICosmPortalTypes.TokenVersion tokenVersion;     // 6=税 · 7=普通
    ICosmPortalTypes.DEXId dexId;                   // MDR 序号；默认 DEX0
    uint64 antiFarmerDuration;                      // 税币 anti-farmer 秒数；普通=0
    uint16 buyTaxRate;                              // 买入 tax bps
    uint16 sellTaxRate;                             // 卖出 tax bps
    uint64 taxDuration;                             // 税收时长(秒)；0=不过期
    address commissionReceiver;                     // 佣金地址；0=无
    ICosmPortalTypes.FeeConfig[4] feeConfigs;       // V7 四槽 fee；和=10000
    address vaultFactory;                           // 金库工厂
    bytes vaultData;                                // 工厂编码参数
}
```

## struct NewTaxTokenWithVaultParams（已废弃）

> `newTaxTokenWithVault` 直接 revert；ABI 保留

```solidity
struct NewTaxTokenWithVaultParams {
    string name;                                    // [废弃] 代币名称
    string symbol;                                  // [废弃] 符号
    string meta;                                    // [废弃] metaURI
    ICosmPortalTypes.DexThreshType dexThresh;       // [废弃] 迁移阈值
    bytes32 salt;                                   // [废弃] CREATE2 salt
    uint16 taxRate;                                 // [废弃] 对称买卖 tax；现用 buyTaxRate/sellTaxRate
    ICosmPortalTypes.MigratorType migratorType;     // [废弃] 迁移器
    address quoteToken;                             // [废弃] quote
    uint256 quoteAmt;                               // [废弃] 同步买入
    bytes permitData;                               // [废弃] permit
    bytes32 extensionID;                            // [废弃] 扩展
    bytes extensionData;                            // [废弃] 扩展数据
    ICosmPortalTypes.DEXId dexId;                   // [废弃] MDR id
    ICosmPortalTypes.V3LPFeeProfile lpFeeProfile;   // [废弃] LP 档位
    uint64 taxDuration;                             // [废弃] 税收时长
    uint64 antiFarmerDuration;                      // [废弃] anti-farmer
    uint16 mktBps;                                  // [废弃] 营销 bps
    uint16 deflationBps;                            // [废弃] 销毁 bps
    uint16 dividendBps;                             // [废弃] 分红 bps
    uint16 lpBps;                                   // [废弃] LP bps
    uint256 minimumShareBalance;                    // [废弃] 分红门槛
    address vaultFactory;                           // [废弃] 工厂
    bytes vaultData;                                // [废弃] 工厂数据
}
```

## struct VaultInfo

> 场景：`getVault` / `tryGetVault` 对外返回

```solidity
struct VaultInfo {
    address vault;                  // 金库合约地址
    address vaultFactory;           // 创建该金库的工厂
    string description;             // 金库可读描述(vault.description())
    bool isOfficial;                // 是否官方工厂创建
    RiskLevel riskLevel;            // 风险等级；未注册工厂=UNVERIFIED
    // 分类另调 getCosmVaultCategory(taxToken)
}
```

## struct VaultFactoryInfo

> 场景：`getVaultFactory(factory)`

```solidity
struct VaultFactoryInfo {
    bool registered;                        // 是否经 admin 注册
    bool enabled;                           // false 时禁止发币
    bool official;                          // 官方标记
    RiskLevel riskLevel;                    // 工厂风险
    CosmVaultCategory category;             // Cosm 分类；默认 NONE
    FactoryPermissionPolicy permissionPolicy; // 默认 OPEN(未注册)
    FactoryValidationMode validationMode;   // 默认 NONE
}
```

## struct LegacyVaultInfo

> 场景：VaultPortal 内部 `_vaults` 存储；Lens 映射为 VaultInfo

```solidity
struct LegacyVaultInfo {
    address vault;                  // 金库合约地址；Launch 时 factory.newVault 返回
    bool isOfficial;                // 创建时工厂是否 official；影响 VaultInfo.isOfficial
    RiskLevel riskLevel;            // 发币时工厂风险等级快照
    CosmVaultCategory category;     // Cosm 分类；默认 NONE；admin 可改
    address vaultFactory;           // 创建该金库的工厂地址
    string description;             // 金库 description() 缓存；Lens 返回给前端
}
```

## struct LegacyVaultFactoryInfo

> 场景：内部 `_vaultFactories`；Lens 映射为 VaultFactoryInfo

```solidity
struct LegacyVaultFactoryInfo {
    bool registered;                // admin 是否 registerFactory；false=permissionless
    bool enabled;                   // false 时 revert FactoryDisabled
    bool official;                  // 官方工厂标记；影响 isOfficial
    RiskLevel riskLevel;            // admin 设置的风险等级
    CosmVaultCategory category;     // admin 设置的 Cosm 分类
}
```

---

# CosmVaultSchemas (`contracts/vault/base/CosmVaultSchemas.sol`)

> 场景：工厂 `vaultDataSchema` / `vaultUISchema`；前端表单生成

## struct FieldDescriptor

```solidity
struct FieldDescriptor {
    string name;         // 字段名(vaultData key 或 method 参数名)
    string fieldType;    // 类型 hint：address/uint8/uint256/bool 等
    string description;  // UI 展示文案
    uint8 decimals;      // 数值小数位；非数值=0 · BNB=18
}
```

## struct VaultDataSchema

```solidity
struct VaultDataSchema {
    string description;           // vaultData 整体说明
    FieldDescriptor[] fields;     // 各字段描述
    bool isArray;                 // true=struct 数组编码(如 Split Recipient[])
}
```

## struct ApproveAction

```solidity
struct ApproveAction {
    string tokenType;        // 需 approve 的 token 类型 hint(taxToken/quote 等)
    string amountFieldName;  // inputs 中数量字段名
}
```

## struct VaultMethodSchema

```solidity
struct VaultMethodSchema {
    string name;                  // 合约方法名
    string description;           // 方法说明
    FieldDescriptor[] inputs;     // 入参
    FieldDescriptor[] outputs;    // 返回值
    ApproveAction[] approvals;    // 写操作前 approve 提示
    bool isInputArray;            // 入参是否数组
    bool isOutputArray;           // 返回值是否数组
    bool isWriteMethod;           // true=写 · false=view
}
```

## struct VaultUISchema

```solidity
struct VaultUISchema {
    string vaultType;             // 类型标识 split/scheduled-buyback 等
    string description;           // 金库整体说明
    VaultMethodSchema[] methods;  // view/write 方法列表
}
```

## struct FactoryPolicy

```solidity
struct FactoryPolicy {
    string target;       // 约束字段名(如 quoteToken)
    string operator;     // 比较符 eq/ne/gt/lt
    bytes value;         // abi.encode 约束值
    string description;  // UI 说明；链上 hook 仍为准
}
```

---

# ICosmVaultFactory

## struct LaunchValidationDataV1

> 场景：工厂 `onBeforeLaunch` staticcall 校验 payload(v2.2)

```solidity
struct LaunchValidationDataV1 {
    uint8 tokenVersion;           // 6=税 · 7=普通
    address quoteToken;           // 曲线 quote
    uint16 buyTaxRate;            // 买入 tax bps
    uint16 sellTaxRate;           // 卖出 tax bps
    uint16 vaultBps;              // 进金库的 mkt 份额(=mktBps)
    uint16 deflationBps;          // 销毁 bps
    uint16 dividendBps;           // 分红 bps
    uint16 lpBps;                 // LP bps；四路=10000
    address dividendToken;          // 分红 token 或 magic
    uint256 minimumShareBalance;  // 分红门槛
}
```

---

# ITokenV3 / CosmToken

## enum PoolState

> 场景：普通币 `poolState.state`；无 tax 四态

```solidity
enum PoolState {
    BondingCurve,         // 0  曲线阶段；禁止 pool 转账
    Migrating,            // 1  迁移中；tax-free
    EnforcedAntiFarmer,   // 2  DEX 初期；全 pools 可 gate
    Free                  // 3  anti-farmer 结束；自由转账
}
```

## struct PackedPoolState

```solidity
struct PackedPoolState {
    PoolState state;                    // 当前 PoolState
    uint48 antiFarmerExpirationTime;    // anti-farmer 截止 Unix 时间；0=未设
}
```

## struct InitParams

> 场景：Portal clone 后 `CosmToken.initialize`

```solidity
struct InitParams {
    address[] pools;                // 所有预测/已知 DEX 池；anti-farmer 注册；MDR.getAllTradingPools
    address mainPool;               // 主池地址；须在 pools[] 内；swap gate 识别用
    string name;                    // ERC20.name()；Portal initialize 传入
    string symbol;                  // ERC20.symbol()
    string meta;                    // metaURI
    uint256 maxSupply;              // 最大供应量；Cosm 固定 1_000_000_000 ether
    uint256 antiFarmerDuration;     // DEX 后 anti-farmer 秒数；普通币通常 0
    address dividendContract;       // CosmDividend；普通无税=address(0)
    address taxProcessor;           // TaxSplitterLite(普通 V7) 或 0
    address cosmSwapGateHook;       // V3=swapGateHub · Infinity=infinityHook
}
```

---

# ICosmTaxTokenV3 / CosmTaxToken

## enum PoolState

> 场景：税币五态生命周期

```solidity
enum PoolState {
    BondingCurve,             // 0  曲线；禁止 pool 转账
    Migrating,                // 1  迁移中
    TaxEnforcedAntiFarmer,    // 2  DEX；全 pools 收税
    TaxEnforced,              // 3  仅 mainPool 收税
    TaxFree                   // 4  taxDuration 到期；税率清零
}
```

## struct InitParams

> 场景：Portal `CosmTaxToken.initialize`

```solidity
struct InitParams {
    string name;                    // ERC20.name()
    string symbol;                  // ERC20.symbol()
    string meta;                    // metaURI
    uint16 buyTax;                  // 买入 transfer tax(bps)；max 1000(10%)
    uint16 sellTax;                 // 卖出 transfer tax(bps)
    address taxProcessor;           // CosmTaxSplitter clone 地址
    address dividendContract;       // CosmDividend；dividendBps=0 时=0
    address quoteToken;             // 曲线 quote；address(0)=BNB
    uint256 liqExpectedOutputAmount; // [Flap 兼容] 预期 LP 输出；Cosm 发币固定 0
    uint256 taxDuration;            // init 存相对秒数；finalizeMigration 转绝对 timestamp
    address[] pools;                // 预测 V2/V3 等池列表；anti-farmer/tax 判定
    address v2Router;               // Pancake V2 Router；liquidation 路径
    uint256 antiFarmerDuration;     // DEX 后全 pools 收税窗口(秒)；0=仅 mainPool
}
```

## struct PackedPoolState（CosmTaxToken 实现）

> 单 slot 打包；`poolState()` 公开读

```solidity
struct PackedPoolState {
    uint8 state;                        // PoolState ordinal
    uint16 buyTaxRate;                  // 买 tax bps(1/10000)
    uint16 sellTaxRate;                 // 卖 tax bps
    bool notLiquidating;                // processTaxTokens 重入锁
    uint96 liquidationThreshold;        // 合约持币≥此值触发卖税；动态±1%
    uint64 taxExpirationTime;           // init=相对秒 · finalize 后=绝对 timestamp
    uint48 antiFarmerExpirationTime;    // anti-farmer 截止；0=未设
}
```

---

# ICosmTaxProcessor / CosmTaxSplitter

## struct PackedFeeConfig / V2 / V3

> 场景：只读 view，供 Flap 索引器 / 前端读 TaxSplitter 当前配置；**不是发币入参**  
> 调用：`feeConfig()` · `feeConfigV2()` · `feeConfigV3()`（V3 含四路 marketing 地址）  
> Cosm 税币发币时：`feeRate=6000(60%)` · 迁移后 `4100(41%)` · `commissionBps=0`（除非设了 commissionReceiver）

```solidity
struct PackedFeeConfig {
    uint16 marketBps;       // 营销/金库份额(bps)；= InitParams.mktBps；dispatch 时从剩余 quote 按此比例进 marketQuoteBalance
    uint16 deflationBps;    // 回购销毁份额(bps)；= InitParams.deflationBps；dispatch 买本币打 0xdead
    uint16 lpBps;           // 加 LP 份额(bps)；= InitParams.lpBps；迁移后 addLiquidity 用
    uint16 dividendBps;     // 持币分红份额(bps)；= InitParams.dividendBps；swap 后打入 CosmDividend
    uint16 feeRate;         // TaxSplitter 内部协议抽成(bps)；从每笔进入的 quote/tax **先于四路拆分**扣给 feeReceiver；Cosm 税币发币=6000(60%) · 迁移后=4100(41%) · Standard V7 TaxSplitterLite 时为 10000-splitRatio
    bool isWeth;            // quote 是否为 BNB/WBNB；true=原生或 WBNB 地址 · 索引器展示用；计算：isNative(quote)||quote==router.WETH()
}

struct PackedFeeConfigV2 {
    uint16 marketBps;       // 同 PackedFeeConfig.marketBps
    uint16 deflationBps;    // 同 PackedFeeConfig.deflationBps
    uint16 lpBps;           // 同 PackedFeeConfig.lpBps
    uint16 dividendBps;     // 同 PackedFeeConfig.dividendBps
    uint16 feeRate;         // 同 PackedFeeConfig.feeRate；Cosm 税币曲线=6000 · DEX=4100
    bool isWeth;            // 同 PackedFeeConfig.isWeth
    uint16 commissionBps;   // 佣金 bps；从 quote 先于四路拆分扣给 commissionReceiver；公式 floor(60000/effectiveTax) · 无 receiver 时发币置 0
    address dividendToken;  // 实际分红 ERC20 地址；view 解析：mode0=WBNB(若BNB)/quote · mode1=taxToken · mode2=dividendRewardToken
}

struct PackedFeeConfigV3 {
    uint16 mktOrVaultBps1;  // 主 marketing 路 bps；= mktBps - mktBps2 - mktBps3 - mktBps4；dispatch 分给 market
    uint16 mktOrVaultBps2;  // 第二路 marketing bps；= InitParams.mktBps2；分给 market2
    uint16 mktOrVaultBps3;  // 第三路 bps；= mktBps3；分给 market3
    uint16 mktOrVaultBps4;  // 第四路 bps；= mktBps4；分给 market4
    uint16 deflationBps;    // 销毁 bps；同 V1
    uint16 lpBps;           // 加 LP bps；同 V1
    uint16 dividendBps;     // 分红 bps；同 V1
    uint16 feeRate;         // 内部协议抽成 bps；同 V1；Cosm 税币曲线=6000 · DEX=4100
    bool isWeth;            // 同 V1
    uint16 commissionBps;   // 同 V2
    address dividendToken;  // 同 V2；dividendToken() 解析结果
    address mktOrVaultAddr1; // 主 marketing 收款；= InitParams.market(=beneficiary/金库)
    address mktOrVaultAddr2; // 第二路收款；= market2；bps2>0 时非 0
    address mktOrVaultAddr3; // 第三路；= market3
    address mktOrVaultAddr4; // 第四路；= market4
}
```

> **资金流顺序（曲线/DEX tax 进 TaxSplitter）：**  
> `quoteIn` → 扣 `feeRate%`(→feeReceiver) → 扣 `commissionBps%`(→commissionReceiver) → 剩余按 mkt/deflation/lp/dividend 四路入账 → `dispatch()` 才真正转出


## struct InitParams（CosmTaxSplitter）

> 场景：发币 clone 后 initialize

```solidity
struct InitParams {
    address token;                  // 税币地址；clone 后 immutable 关联
    address quoteToken;             // 曲线 quote；与 Portal TokenState 一致
    address market;                 // 营销/金库收款=LaunchParams.beneficiary；dispatch 转出目标
    address dividend;               // CosmDividend clone；dividendBps=0 时为 0
    address router;                 // PancakeSwap V2 router；swap/buyback 用
    address portal;                 // CosmPortal；onlyPortal 校验
    address feeReceiver;            // TaxSplitter.feeRate 抽成 + 内部 fee 桶最终接收
    uint16 mktBps;                  // 营销 bps；与 TaxAllocation 一致；四路之一
    uint16 deflationBps;            // 销毁 bps
    uint16 dividendBps;             // 分红 bps
    uint16 lpBps;                   // 加 LP bps；四路和须=10000
    uint16 feeRate;                 // Splitter 内部抽成 bps；Cosm 税币发币=6000 · migrate 后=4100
    uint16 commissionBps;           // 佣金 bps；有 commissionReceiver 时按公式计算
    address commissionReceiver;     // 佣金接收；0 则 commissionBps=0
    uint8 dividendMode;             // 0=quote · 1=本币 · 2=其他
    address dividendRewardToken;    // 分红实际 ERC20；mode0 BNB=WBNB 地址
    uint256 liqExpectedOutputAmount; // [Flap 兼容] Cosm=0
    uint16 mktBps2;                 // 从 mktBps 切分；默认 0
    uint16 mktBps3;                 // 第三路 marketing bps
    uint16 mktBps4;                 // 第四路 marketing bps
    address market2;                // mktBps2 收款；bps2>0 必填
    address market3;                // mktBps3 收款
    address market4;                // mktBps4 收款
    address converter;              // dividendMode=2；唯一可触发 Case3 swap
    address swapRegistry;           // CosmSwapRegistry；Case3 路径查询；0=仅 V2 直连
}
```

## struct V4LPFeeSource

> 场景：Infinity 迁移后 TaxSplitter 登记 LP NFT 来源

```solidity
struct V4LPFeeSource {
    uint8 migratorType;             // MigratorType ordinal；Infinity=PCS_INFINITY_CL_MIGRATOR(3)
    address poolManager;            // Infinity CL PoolManager；collect 路径用
    address positionManager;        // Infinity PositionManager；NFT 持仓查询
    uint256 tokenId0;               // 下区间(quote-heavy) LP NFT tokenId
    uint256 tokenId1;               // 上区间(token-heavy) LP NFT tokenId
}
```

---

# CosmDividend

## struct UserInfo

> 场景：MasterChef 式分红账务

```solidity
struct UserInfo {
    uint256 share;              // 用户有效持币份额(排除 excluded/blackhole)；setShare 更新
    uint256 rewardDebt;         // magnified 债务；claim 时结算 pending
    uint256 pendingBalance;     // 尚未 claim 的累积分红(quote/ token/WBNB)
}
```

---

# CosmTaxConverter

## enum OperationType

```solidity
enum OperationType {
    BatchDispatch,              // 0  批量 dispatch TaxSplitter
    BatchDistributeDividend,    // 1  批量分红 settle
    TriggerSplit,               // 2  触发 token split/liquidation
    BatchDispatchPermissionless // 3  无权限 batch dispatch
}
```

## struct CachedMEVStatus

```solidity
struct CachedMEVStatus {
    bool isCached;                  // 是否已缓存该 taxProcessor 的 MEV 状态
    bool requiresMEVProtection;     // true 时 dispatch 走 MEV 保护路径；Converter 记录
}
```

## struct CallInfo

> 场景：`latestCallInfo(OperationType)` 记录每种批量操作最后一次调用

```solidity
struct CallInfo {
    address caller;     // 最近一次触发该 OperationType 的 msg.sender
    uint64 timestamp;   // 调用时 block.timestamp；审计/限频用
}
```

---

# ISwapRegistry / CosmSwapRegistry

## enum PoolType

> 同 ICosmPortalTypes.PoolType ordinal

## struct SwapInfo

> 场景：`getSwapInfo(from,to)` Case3 分红 swap 路径

```solidity
struct SwapInfo {
    address pool;           // 池/router 地址
    uint8 dexId;            // MDR dex index
    uint24 feeTier;         // V3/CL fee
    PoolType poolType;      // 池类型
    int24 tickSpacing;      // CL tick spacing
    address hooks;          // CL hooks；V2=0
    bool supported;         // 路径是否可用
}
```

## struct PathEntry

> 场景：CosmSwapRegistry 内部 `_paths` 存储；布局同 SwapInfo

```solidity
struct PathEntry {
    address pool;           // 池或 router 合约地址；Case3 quote→dividendToken swap
    uint8 dexId;            // MultiDexRouter dex index(0/1/2)
    uint24 feeTier;         // V3/CL fee tier(如 2500=0.25%)
    PoolType poolType;      // V2/V3/V4/Infinity；决定 swap 调用方式
    int24 tickSpacing;      // CL tick spacing；V2 填 0
    address hooks;          // Infinity/V4 hooks 地址；无 hooks=0
    bool supported;         // admin 配置路径是否启用；false 则 isSwapSupported=false
}
```

---

# IMultiDexRouter

## struct ExactInputSingleParams

> 场景：V3 exactInputSingle swap

```solidity
struct ExactInputSingleParams {
    address tokenIn;                // 输入 ERC20 地址
    address tokenOut;               // 输出 ERC20 地址
    uint24 fee;                     // V3 pool fee tier(500/2500/10000)
    address recipient;              // swap 输出接收地址
    uint256 amountIn;               // 输入数量(最小单位)
    uint256 amountOutMinimum;       // 最小输出；滑点保护
    uint160 sqrtPriceLimitX96;      // 价格限制；0=不限制
}
```

## struct V4SwapExactInputSingleParams

> 场景：MDR.exactInputSingleV4 / exactInputSinglePCSInfinityCL

```solidity
struct V4SwapExactInputSingleParams {
    address tokenIn;                // 输入 token
    address tokenOut;               // 输出 token
    uint24 fee;                     // CL pool fee
    int24 tickSpacing;              // CL tick spacing
    address hooks;                  // pool hooks 合约
    uint128 amountIn;               // 输入量(128 位足够大多数 swap)
    uint128 amountOutMinimum;       // 最小输出
    address recipient;              // 输出接收者
}
```

---

# ICosmTriggerService

> 场景：CosmScheduledBuybackVault 定时回调

## enum TriggerStatus

```solidity
enum TriggerStatus {
    PENDING,    // 0  已预约未执行
    EXECUTED,   // 1  已成功 callback
    FAILED      // 2  执行失败可 retry
}
```

## struct TriggerRequest

```solidity
struct TriggerRequest {
    address requester;      // 发起预约的合约(如 CosmScheduledBuybackVault)
    uint64 executeAfter;    // Unix 时间戳后可执行；0=注册后立即可 trigger
    TriggerStatus status;   // PENDING/EXECUTED/FAILED；retry 仅 FAILED
    uint128 feePaid;        // requestTrigger 时支付的 BNB 服务费(wei)
}
```

---

# Vault 模板

## CosmSplitVault

> vaultData = abi.encode(Recipient[]) · 1–10 人 · bps 和=10000

```solidity
struct Recipient {
    address recipient;  // 收款钱包地址；须在 vaultData Recipient[] 中唯一
    uint16 bps;         // 该地址拆分比例(bps)；所有 recipient bps 之和须=10000
}

struct UserBalance {
    uint128 accumulated;  // 该 recipient 累计应得 quote/BNB(最小单位)
    uint128 claimed;      // 已累计 claim() 领取的量
}

struct RecipientInfo {
    address recipient;      // 收款地址；来自 vaultData Recipient.recipient
    uint16 bps;             // 拆分 bps；与 vaultData 一致
    uint128 accumulated;    // 累计入账应得 quote
    uint128 claimed;        // 已领取 quote
    uint256 claimable;      // 当前可领 = accumulated - claimed；view 计算
}

struct SplitStatus {
    uint256 vaultBalance;       // 金库合约当前持有的 quote/BNB 余额
    uint256 totalDistributed;   // 历史从 TaxSplitter 分发进金库的 quote 总量
    uint256 totalClaimed;       // 所有 recipient 已 claim 合计
    uint256 totalClaimable;     // 全员当前可领合计(sum claimable)
    uint256 uncredited;         // TaxSplitter 已转但尚未 credit 到各 recipient 的 pending
    uint256 recipientCount;     // Recipient 列表人数；1–10
    RecipientInfo[] recipients; // 各 recipient 明细；前端表格数据源
}

struct UserInfo {
    uint16 bps;             // 若 msg.sender 是 recipient 则为其 bps；否则=0
    uint256 accumulated;    // 该用户累计应得(同 RecipientInfo)
    uint256 claimed;        // 该用户已领
    uint256 claimable;      // 该用户当前可领
    bool isRecipient;       // true=地址在 Recipient 列表；false=只能 view 不能 claim
}
```

## CosmScheduledBuybackVault

> vaultData = abi.encode(triggerMode,buybackMode,interval,minBnb,maxBnb[,firstExecutableAt])

```solidity
struct BuybackStatus {
    uint8 triggerMode;              // 0=定时 · 1=金额+最小间隔 · 2=时间与金额同时满足
    uint8 buybackMode;              // 0=Token 回购销毁 · 1=LP 回购(失败 fallback Token)
    uint256 intervalSeconds;        // 触发间隔(秒)
    uint256 minBnbAmount;           // 模式 1/2 最小 BNB 余额门槛
    uint256 maxBnbPerTrigger;       // 单次上限；0=不限
    uint256 lastTriggeredAt;        // 上次执行时间
    uint256 nextTriggerAt;          // 下次可触发时间
    uint256 countdownSeconds;       // 距下次触发秒数
    uint256 vaultBnb;               // 金库 BNB 余额
    uint256 nextSpendBnb;           // 下次预计花费
    uint256 totalTokensBurned;      // 累计销毁 token
    uint256 totalBnbSpent;          // 累计花费 BNB
    uint256 triggerCount;           // 触发次数
    uint256 pendingRequestId;       // TriggerService 请求 id；0=无
    bool ready;                     // 当前是否满足触发条件
    string buybackModeLabel;        // UI 可读标签；如 "Token Burn" / "LP Buyback"
    string triggerModeLabel;        // UI 可读标签；如 "Interval" / "Amount+Interval"
    string executionPathLabel;      // UI 可读标签；实际 swap 路径(V2/V3/Infinity)
}
```

## CosmBurnDividendVault

> 燃烧本币获 power · tax BNB 按 power 分红

```solidity
struct BurnStatus {
    uint256 vaultBnb;               // 金库 BNB
    uint256 totalBurned;            // 全网燃烧 power 合计
    uint256 pendingBnb;             // 无 power 时暂存 BNB
    uint256 accRewardPerShare;      // 累积每 power 奖励(1e18 精度)
    uint256 totalRewardsIn;         // 累计进入分红 BNB
    uint256 totalClaimed;           // 所有用户已 claim 分红 BNB 合计
    uint256 participantCount;       // 有过燃烧的用户数
}

struct UserInfo {
    uint256 burned;         // 用户 power(燃烧量)
    uint256 rewardDebt;     // 债务快照
    uint256 pending;        // 可领 BNB
    uint256 claimed;        // 已领
    uint256 shareBps;       // 占 totalBurned 的 bps
}
```

## CosmRankBurnDividendVault

> 80% power 分红 + 20% Top10 排行直分

```solidity
struct RankStatus {
    uint256 vaultBnb;               // 金库当前 BNB 余额
    uint256 totalBurned;            // 全网燃烧 power 合计(决定 80% 池权重)
    uint256 pendingWeightBnb;       // 80% 权重池在 totalBurned=0 时暂存 BNB
    uint256 rankPendingBnb;         // 20% Top10 排行池待分配 BNB
    uint256 accRewardPerShare;      // 80% 池累积每 power 奖励(1e18 精度)
    uint256 totalRewardsIn;         // 累计进入金库的 BNB 总量
    uint256 totalWeightClaimed;     // 80% 权重池已领取合计
    uint256 totalRankDistributed;   // 20% 排行池已分配给 Top10 合计
    uint256 totalRankClaimed;       // Top10 已 claim 排行奖励合计
    uint256 participantCount;       // 至少燃烧过一次的用户数
    uint256 minBurnAmount;          // vaultData 配置的单次最小燃烧量；低于 revert
    address[10] topBurners;         // 当前 Top10 燃烧者地址(按 burned 降序)
    uint256[10] topBurnedAmounts;   // 对应 Top10 各地址的 burned power
}

struct UserInfo {
    uint256 burned;         // 用户燃烧 power；决定 80% 池权重
    uint256 rewardDebt;     // 80% 池 magnified 债务快照
    uint256 weightPending;  // 80% 权重池当前可领 BNB
    uint256 rankCredit;     // 20% 排行池累计可领(仅 Top10 非 0)
    uint256 weightClaimed;  // 80% 池已领
    uint256 rankClaimed;    // 20% 排行池已领
    uint256 pendingTotal;   // weightPending + rankCredit 可领合计
    uint256 shareBps;       // burned/totalBurned*10000；占比展示
    uint256 rank;           // 排行榜名次；0=未上榜 · 1..10=Top10
}
```

## CosmTokenStakingDividendVault

> 质押 tax token · 分 BNB

```solidity
struct StakeStatus {
    uint256 vaultBnb;               // 金库当前 BNB 余额
    uint256 totalStaked;            // 全网质押 tax token 总量
    uint256 pendingBnb;             // 无质押者时暂存 BNB；有人 stake 后分发
    uint256 accRewardPerShare;      // 累积每 staked token 奖励(1e18 精度)
    uint256 totalRewardsIn;         // 累计进入金库的 BNB
    uint256 totalClaimed;           // 所有用户已 claim 合计
    uint256 participantCount;       // 至少 stake 过一次的用户数
}

struct UserInfo {
    uint256 staked;         // 用户当前质押 tax token 量
    uint256 rewardDebt;     // magnified 债务；stake/unstake 时更新
    uint256 pending;        // 当前可领 BNB
    uint256 claimed;        // 历史已领 BNB
    uint256 shareBps;       // staked/totalStaked*10000；占比展示
}
```

## CosmLPStakingDividendVault

> 质押 LP token · 分 BNB

```solidity
struct StakeStatus {
    address pair;                   // PCS V2 LP pair 地址；须质押此 LP token
    uint256 vaultBnb;               // 金库当前 BNB 余额
    uint256 totalStaked;            // 全网质押 LP token 总量
    uint256 pendingBnb;             // 无 LP 质押时暂存 BNB
    uint256 accRewardPerShare;      // 累积每 LP token 奖励(1e18 精度)
    uint256 totalRewardsIn;         // 累计进入金库 BNB
    uint256 totalClaimed;           // 全员已 claim 合计
    uint256 participantCount;       // 至少 stake LP 过一次的用户数
}

struct UserInfo {
    uint256 staked;         // 用户质押 LP token 数量
    uint256 rewardDebt;     // magnified 债务快照
    uint256 pending;        // 当前可领 BNB
    uint256 claimed;        // 已领 BNB
    uint256 shareBps;       // staked/totalStaked*10000
}
```

---

# Libraries / 内部

## LibCurve.Curve

> 曲线 (1e9+h-s)*(r+eth)=k · price=k/(h+1e9-s)²

```solidity
struct Curve {
    uint256 r;  // 虚拟 quote 储备；曲线 (r+eth)*(h+supply)=k 中的 r；全局 quote 配置决定
    uint256 h;  // 虚拟 token 储备；发币初始 h 决定曲线陡峭度
    uint256 k;  // 常数积参数 k=r*h；迁移阈值 supply 与 r,h 共同决定价格路径
}
```

## CosmPortalLaunchLib

```solidity
struct V7ParsedFees {
    uint16 deflationBps;            // 从 feeConfigs[DEFLATION] 解析的销毁 bps
    uint16 lpBps;                   // 从 feeConfigs[LP_BPS] 解析的加 LP bps
    uint16 dividendBps;             // 从 feeConfigs[DIVIDEND] 解析的分红 bps
    uint256 minimumShareBalance;    // DIVIDEND 槽 minimumShareBalance；建议 ≥10_000 ether
    address dividendToken;          // 分红 token 地址；magic 已解析为实际 ERC20
    address mktAddr1;               // 第一 marketing 地址=beneficiary/金库
    uint16 mktBps1;                 // 主 marketing bps；=总 mkt - mkt2 - mkt3 - mkt4
    address mktAddr2;                 // 第二路 marketing 收款；mktBps2>0 时非 0
    uint16 mktBps2;                 // 第二路 marketing bps
    address mktAddr3;                 // 第三路收款
    uint16 mktBps3;                 // 第三路 bps
    address mktAddr4;                 // 第四路收款
    uint16 mktBps4;                 // 第四路 bps
    uint16 totalMktBps;             // mktBps1+2+3+4；须 ≤10000 且与 deflation/dividend/lp 和=10000(税)
}

struct LaunchMeta {
    uint8 tokenVersion;             // 6=税币 · 7=普通；决定 clone 模板与 migrator 约束
    uint16 bondingCurveFeeBps;      // V7 曲线协议费 bps；普通默认 125(1.25%) · 税币=0
    uint8 lpFeeProfile;             // V6 来自 params.lpFeeProfile · V7 固定=0
    uint8 dexId;                    // MDR preferredDexId(0/1/2)；写入 TokenState
    uint16 commissionBps;            // 税币佣金 bps；有 commissionReceiver 时按公式；否则=0
    address commissionReceiver;     // 佣金接收地址；0=无佣金
    uint128 dexSupplyThresh;        // 来自 dexThresh 枚举映射；默认 800_000_000 ether
    bytes32 extensionID;            // 扩展插件 ID；无扩展=bytes32(0)
}
```

## CosmPortalStorage

```solidity
struct ExtensionInfo {
    address addr;       // 扩展合约地址
    uint8 version;      // 扩展版本；onTrade 路由用
}

struct MigrationFeeSplit {
    uint256 tokenLiquidity;   // 进 LP 的 token 量
    uint256 quoteLiquidity;   // 进 LP 的 quote 量
    uint256 reserveFee;       // 从 reserve 扣的协议费
    uint256 liquidityFee;     // 从 token 扣的 LP 协议费
    uint256 tokensBurned;     // 打 0xdead 的 excess token
}
```

## CosmPortalDex / Pancake

```solidity
struct PCSInfPoolKey {
    address currency0;          // pool token0；须 currency0<currency1 排序
    address currency1;          // pool token1
    address hooks;              // Infinity CL hooks 合约；无 hooks=address(0)
    address poolManager;        // Infinity CL PoolManager 地址
    uint24 fee;                 // CL pool fee tier(如 2500=0.25%)
    bytes32 parameters;         // CL pool parameters 打包(tickSpacing 等)；Infinity 内部解码
}

struct CLSwapExactInputSingleParams {
    PCSInfPoolKey poolKey;      // 目标 Infinity CL pool 标识
    bool zeroForOne;            // true=currency0→currency1 · false=反向
    uint128 amountIn;           // 输入 token 数量
    uint128 amountOutMinimum;   // 最小输出；滑点保护
    bytes hookData;             // 传给 hooks 的附加 calldata；无 hook=空 bytes
}
```

### IPCSInfCLQuoter.QuoteExactSingleParams

```solidity
struct QuoteExactSingleParams {
    PCSInfPoolKey poolKey;      // 询价的 Infinity CL pool
    bool zeroForOne;            // 交换方向；同 swap
    uint128 exactAmount;        // 精确输入量(quote)或输出量(取决于 quoter 模式)
    bytes hookData;             // hooks 附加数据
}
```

### IPancakeV3SwapRouter.ExactInputSingleParams

```solidity
struct ExactInputSingleParams {
    address tokenIn;                // 输入 ERC20
    address tokenOut;               // 输出 ERC20
    uint24 fee;                     // V3 pool fee tier
    address recipient;              // swap 输出接收地址
    uint256 deadline;               // Unix 截止时间；过期 revert
    uint256 amountIn;               // 输入数量(最小单位)
    uint256 amountOutMinimum;       // 最小输出；滑点保护
    uint160 sqrtPriceLimitX96;      // 价格限制；0=不限制
}
```

### IPancakeV3QuoterV2.QuoteExactInputSingleParams

```solidity
struct QuoteExactInputSingleParams {
    address tokenIn;                // 输入 token
    address tokenOut;               // 输出 token
    uint256 amountIn;               // 输入量(询价基准)
    uint24 fee;                     // V3 pool fee tier
    uint160 sqrtPriceLimitX96;      // 价格限制；0=不限制
}
```

### INonfungiblePositionManager.MintParams

> 场景：V3 迁移 addLiquidity mint

```solidity
struct MintParams {
    address token0;             // pair token0(地址较小者)
    address token1;             // pair token1
    uint24 fee;                 // V3 pool fee tier
    int24 tickLower;            // LP 区间下界 tick
    int24 tickUpper;            // LP 区间上界 tick
    uint256 amount0Desired;     // 期望投入 token0 量
    uint256 amount1Desired;     // 期望投入 token1 量
    uint256 amount0Min;         // token0 最小投入；滑点/比例保护
    uint256 amount1Min;         // token1 最小投入
    address recipient;          // LP NFT 接收者；通常 migrator 或 locker
    uint256 deadline;           // mint 截止 Unix 时间
}
```

## CosmInfinityLiquidityMath.SplitPositionParams

> 场景：Infinity 迁移双区间 LP 拆分

```solidity
struct SplitPositionParams {
    int24 tickLower0;           // 下区间 NFT tick 下界(quote-heavy 侧)
    int24 tickUpper0;           // 下区间 tick 上界；通常=当前 pool tick
    int24 tickLower1;           // 上区间 NFT tick 下界(=当前 tick)
    int24 tickUpper1;           // 上区间 tick 上界(token-heavy 侧)
    uint128 liquidity0;         // 下区间 mint 的 liquidity 数量
    uint128 liquidity1;         // 上区间 mint 的 liquidity 数量
    uint256 amount0ForPos0;     // 下区间 position 需要的 token0 量
    uint256 amount1ForPos0;     // 下区间 position 需要的 token1 量
    uint256 amount0ForPos1;     // 上区间 position 需要的 token0 量
    uint256 amount1ForPos1;     // 上区间 position 需要的 token1 量
}
```

---

# 常用默认值速查

| 项 | 默认值 | 场景 |
|----|--------|------|
| tokenVersion | 6 税 / 7 普通 | 发币 |
| dexSupplyThresh | 800_000_000 ether | FOUR_FIFTHS / TokenState=0 |
| bondingCurveFeeBps | 125(V7) / 0(税) | 曲线协议费 |
| protocolFeeBps | 100 (1%) | 全局曲线协议费(税币→feeReceiver) |
| feeProfile | FEE_GLOBAL_DEFAULT | TokenState |
| migratorType | V2(税) / Infinity(普通) | 强制/默认 |
| vanity suffix | 0x0111 税 / 0x0222 普通 | CREATE2 |
| lpFeeProfile | 0 STANDARD | V3 0.25% |
| dexId (TokenState) | 0 DEX0 | MDR |
| dexId (V8Safe) | 2/3/4 | _dexKind 路由 |
| extensionID | bytes32(0) | 无插件 |
| TaxSplitter.feeRate | 6000(曲线) / 4100(DEX) | 发币 init / migrateToDex |
| mkt 四路 | sum=10000 bps | 税币 |
| commissionBps | floor(60000/effectiveTax) | 有 commissionReceiver 时 |
