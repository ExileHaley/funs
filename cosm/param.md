# CosmTypes

## enum TokenStatus

```solidity
enum TokenStatus {
    Invalid,    // 0  无效 / 未注册
    Tradable,   // 1  曲线阶段，Portal 可买卖
    InDuel,     // 2  Flap 占位，Cosm 未用
    Killed,     // 3  Flap 占位，Cosm 未用
    DEX,        // 4  已迁移到 Pancake（V2 / V3 / Infinity）
    Staged      // 5  Flap 占位，Cosm 未用
}
```

## enum FeeProfile

```solidity
enum FeeProfile {
    FEE_GLOBAL_DEFAULT,  // 0  读全局 protocolFeeBps / protocolSellFeeBps（默认各 100 = 1%）
    FEE_FLAPSALE_V0,     // 1  买卖协议费固定 100 bps；迁移 LP/储备费为 0
    FEE_ZERO             // 2  买卖协议费 0；迁移费 0
}
```

## enum MigratorType

```solidity
enum MigratorType {
    V3_MIGRATOR,              // 0  PancakeSwap V3，普通币可选
    V2_MIGRATOR,              // 1  PancakeSwap V2，税收币强制
    V4_UNI_MIGRATOR,          // 2  Uniswap V4 方向，BSC 保留未启用
    PCS_INFINITY_CL_MIGRATOR  // 3  PancakeSwap Infinity CL，普通币 V7 默认
}
```

## struct TaxAllocation

```solidity
struct TaxAllocation {
    uint16 mktBps;                  // 营销份额 → 进 beneficiary（路径 A 钱包 / 路径 B 金库）
    uint16 deflationBps;            // 回购销毁份额 → TaxSplitter 回购本币并销毁
    uint16 dividendBps;             // 持币分红份额 → CosmDividend；为 0 不部署 Dividend
    uint16 lpBps;                   // 加 LP 份额 → 迁移后 TaxSplitter 添加流动性
    uint256 minimumShareBalance;    // 分红最小持币量（token 最小单位）；dividendBps>0 建议 ≥ 10_000 ether
    uint8 dividendMode;             // 分红发放代币：0=quote · 1=本代币 · 2=其他代币
    address dividendToken;          // dividendMode=2 必填；0/1 填 address(0)
    uint256 antiFarmerDuration;     // anti-farmer 秒数；迁移后仅 mainPool 收税；0=迁移后全池收税；上限 365 天
    uint256 taxDuration;            // 税收总有效期秒数；0=永不过期；>0 须 ≥ antiFarmerDuration；上限约 100 年
    uint16 mktBps2;                 // 从 mktBps 再切分的第二路营销份额
    uint16 mktBps3;                 // 第三路营销份额
    uint16 mktBps4;                 // 第四路营销份额
    address market2;                // mktBps2>0 时的收款地址
    address market3;                // mktBps3>0 时的收款地址
    address market4;                // mktBps4>0 时的收款地址
    address converter;              // dividendMode=2 时必填；仅该地址可触发 quote→dividendToken swap（Case3）
}
```

## struct LaunchParams

```solidity
struct LaunchParams {
    string name;                    // 代币名称 → ERC20.name()
    string symbol;                  // 代币符号 → ERC20.symbol()
    string meta;                    // 元数据 URI（IPFS/HTTPS）→ token.metaURI()
    bytes32 salt;                   // CREATE2 salt；预测地址低 16 bit 须匹配 vanity 后缀（税 0x0111 / 普通 0x0222）
    address quoteToken;             // 曲线 quote：address(0)=BNB，或 USDT/USDC/USD1/USDX
    uint256 quoteAmt;               // 发币同步买入 quote 数量；0=只发不买；BNB 用 msg.value，ERC20 先 approve
    address beneficiary;            // 营销收款：路径 A=钱包 · 路径 B=金库 · 普通无税币=0
    uint16 buyTaxBps;               // 买入 transfer tax（bps）；普通币=0；税币买卖至少一侧>0
    uint16 sellTaxBps;              // 卖出 transfer tax（bps）；普通币=0
    bool isTaxed;                   // true=税收币 V6 · false=普通币 V7；须与 newTokenV6/V7 调用一致
    TaxAllocation tax;              // 四路拆分；普通币全 0
    MigratorType migratorType;      // 迁移器；税币强制 V2；普通币默认 Infinity，可选 V3
}
```

## struct TokenState

```solidity
struct TokenState {
    TokenStatus status;             // 生命周期：Invalid=0 · Tradable=1 · DEX=4
    uint8 tokenVersion;             // 6=CosmTaxToken · 7=CosmToken
    address quoteToken;             // 发币锁定的 quote，交易/税收/迁移均用它
    uint128 reserve;                // 曲线 quote 储备；迁移后清零
    uint128 circulatingSupply;      // 曲线流通量；达到 dexSupplyThresh 可迁移
    uint256 buyTaxBps;              // 买入 transfer tax（bps）
    uint256 sellTaxBps;             // 卖出 transfer tax（bps）
    address beneficiary;            // 营销收款（钱包或金库）
    uint256 progress;               // 迁移进度 Wad：circulatingSupply*1e18/dexSupplyThresh；迁移后=1e18
    TaxAllocation tax;              // 发币时锁定的四路 tax 配置
    address taxSplitter;            // CosmTaxSplitter 地址；普通无税币=0
    address dividend;               // CosmDividend 地址；dividendBps=0 时为 0
    address pool;                   // 迁移后主池：税币=V2 pair · 普通=V3 pool；迁移前=0
    address vault;                  // 路径 B 金库；路径 A/普通币=0
    FeeProfile feeProfile;          // 协议费档位，默认 FEE_GLOBAL_DEFAULT
    MigratorType migratorType;      // 发币选的迁移器
    address infinityHook;           // Infinity CL Hook；非 Infinity=0
    uint16 bondingCurveFeeBps;      // V7 普通币曲线协议费（默认 125=1.25%）；税币=0
    uint8 lpFeeProfile;             // Flap V3LPFeeProfile 枚举值
    uint128 dexSupplyThresh;        // 迁移阈值 token 数量；0=默认 800_000_000 ether
    bytes32 extensionID;            // 扩展插件 ID；无扩展=0
}
```

## struct TokenStateV8Safe

```solidity
struct TokenStateV8Safe {
    uint8 status;                   // 同 TokenStatus 数值
    uint256 reserve;                // 曲线 quote 储备
    uint256 circulatingSupply;      // 曲线流通量
    uint256 price;                  // 单价 quote wei / token；DEX 阶段用阈值供应量估算
    uint8 tokenVersion;             // 6=税币 · 7=普通币
    uint256 r;                      // 曲线参数 r（虚拟 quote 储备）
    uint256 h;                      // 曲线参数 h（虚拟 token 储备）
    uint256 k;                      // 曲线参数 k（恒定乘积）
    uint256 dexSupplyThresh;        // 迁移阈值（默认 8 亿 token）
    address quoteTokenAddress;      // quote 地址；BNB=address(0)
    bool nativeToQuoteSwapEnabled;  // ERC20 quote 是否支持一键 BNB 买（WBNB→quote→token）；BNB quote=false
    bytes32 extensionID;            // Cosm 恒为 0
    uint256 buyTaxRate;             // 同 buyTaxBps
    uint256 sellTaxRate;            // 同 sellTaxBps
    address pool;                   // 迁移后 DEX 池地址
    uint256 progress;               // 迁移进度 Wad（1e18=可迁移/已迁移）
    uint8 lpFeeProfile;             // LP 费率档位；V3 常用 0（0.25%）；Infinity 固定 8534ppm
    uint8 dexId;                    // 2=PCS V2 · 3=PCS V3 · 4=PCS Infinity CL
}
```

## struct TokenStateV9Safe

```solidity
struct TokenStateV9Safe {
    uint8 status;                   // 同 TokenStatus
    uint256 reserve;                // 曲线 quote 储备
    uint256 circulatingSupply;      // 曲线流通量
    uint256 price;                  // 单价
    uint8 tokenVersion;             // 6=税币 · 7=普通币
    uint256 r;                      // 曲线 r
    uint256 h;                      // 曲线 h
    uint256 k;                      // 曲线 k
    uint256 dexSupplyThresh;        // 迁移阈值
    address quoteTokenAddress;      // quote 地址
    bool nativeToQuoteSwapEnabled;  // 是否支持一键 BNB 买
    bytes32 extensionID;            // Cosm 恒为 0
    uint256 buyTaxRate;             // 买入 tax bps
    uint256 sellTaxRate;            // 卖出 tax bps
    address pool;                   // 迁移后池地址
    uint256 progress;               // 迁移进度 Wad
    uint8 lpFeeProfile;             // LP 费率档位
    uint8 dexId;                    // DEX 类型 hint
    uint16 bondingCurveFeeRate;     // V7 曲线协议费 bps；对应 TokenState.bondingCurveFeeBps
}
```

## struct QuoteParams

```solidity
struct QuoteParams {
    address inputToken;             // 付出的 token；买 meme=quote · 卖 meme=meme
    address outputToken;            // 收到的 token；买 meme=meme · 卖 meme=quote
    uint256 inputAmount;            // 输入数量（最小单位）
}
```

## struct SwapParams

```solidity
struct SwapParams {
    address inputToken;             // 付出的 token；方向同 QuoteParams
    address outputToken;            // 收到的 token
    uint256 inputAmount;            // 输入数量（最小单位）
    uint256 minOutputAmount;        // 最小输出（滑点保护）；不足 revert Slippage
    bytes permitData;               // 可选 EIP-2612：abi.encode(deadline,v,r,s)；空=须 approve
}
```

## struct SaltLockEntry

```solidity
struct SaltLockEntry {
    address locker;                 // 锁定者；address(0)=未锁定
    uint8 tokenVersion;             // 预留模板：6 或 7
    bool isUsed;                    // 发币成功后 true；同 salt 不可再用
}
```

---

# CosmVaultPortal

## enum RiskLevel

```solidity
enum RiskLevel {
    UNVERIFIED,       // 0  未验证（permissionless 工厂默认）
    LOW_RISK,         // 1  低风险
    LOW_MEDIUM_RISK,  // 2  中低风险
    MEDIUM_RISK,      // 3  中风险
    HIGH_RISK         // 4  高风险
}
```

## enum VaultCategory

```solidity
enum VaultCategory {
    NONE,      // 0  未分类
    SPLIT,     // 1  拆分金库（CosmSplitVault）
    BUYBACK,   // 2  回购类（定时回购销毁等）
    DIVIDEND,  // 3  分红类
    AIRDROP,   // 4  历史占位（omnidrop 已移除）
    GAME       // 5  游戏类
}
```

## enum FactoryPermissionPolicy

```solidity
enum FactoryPermissionPolicy {
    OPEN,            // 0  任何人可用该工厂发币
    TIME_DEPENDENT,  // 1  限时独占：过期前仅 developer 可发；过期后开放
    DISABLED         // 2  禁止发币
}
```

## struct NewTokenV6WithVaultParams

```solidity
struct NewTokenV6WithVaultParams {
    string name;                    // 代币名称
    string symbol;                  // 代币符号
    string meta;                    // 元数据 URI
    bytes32 salt;                   // CREATE2 salt；须匹配 tax vanity 后缀
    address quoteToken;             // 曲线 quote；须被 vaultFactory 支持
    uint256 quoteAmt;               // 发币同步买入 quote 数量；0=只发不买
    uint16 buyTaxBps;               // 买入 transfer tax（bps）
    uint16 sellTaxBps;              // 卖出 transfer tax（bps）
    uint16 mktBps;                  // 营销份额 → 进金库；须 > 0
    uint16 deflationBps;            // 回购销毁份额
    uint16 dividendBps;             // 持币分红份额
    uint16 lpBps;                   // 加 LP 份额；四路合计须 = 10000
    uint256 minimumShareBalance;    // 分红最小持币量
    uint8 dividendMode;             // 分红发放：0=quote · 1=本代币 · 2=其他代币
    address dividendToken;          // dividendMode=2 或 magic COMPUTED 时解析
    address converter;              // dividendMode=2 必填（Case3 swap 触发者）
    uint256 antiFarmerDuration;     // anti-farmer 秒数
    uint256 taxDuration;            // 税收总有效期秒数
    address vaultFactory;           // 金库工厂地址
    bytes vaultData;                // 传给 factory.newVault 的编码参数
}
```

## struct NewTokenV7WithVaultParams

```solidity
struct NewTokenV7WithVaultParams {
    string name;                              // 代币名称
    string symbol;                            // 代币符号
    string meta;                              // 元数据 URI
    bytes32 salt;                             // CREATE2 salt
    address quoteToken;                       // 曲线 quote
    uint256 quoteAmt;                         // 发币同步买入数量
    IFlapPortalTypes.DexThreshType dexThresh; // 迁移阈值档位（默认 8e8 token 等）
    IFlapPortalTypes.MigratorType migratorType; // 迁移器；税币仍强制 V2
    bytes permitData;                         // 可选 ERC20 permit
    bytes32 extensionID;                      // 扩展插件 ID；默认 0
    bytes extensionData;                      // 扩展插件回调数据
    IFlapPortalTypes.TokenVersion tokenVersion; // 6=税币 · 7=普通币
    IFlapPortalTypes.DEXId dexId;             // DEX hint：2=V2 · 3=V3 · 4=Infinity
    uint64 antiFarmerDuration;                // anti-farmer 秒数（仅税币）
    uint16 buyTaxRate;                        // 买入 tax bps；0=普通无税币
    uint16 sellTaxRate;                       // 卖出 tax bps
    uint64 taxDuration;                       // 税收有效期（仅税币）
    address commissionReceiver;               // 佣金接收；0=无佣金
    IFlapPortalTypes.FeeConfig[4] feeConfigs; // 四路 fee 配置；mkt 路最终指向 vault
    address vaultFactory;                     // 金库工厂地址
    bytes vaultData;                          // factory.newVault 编码参数
}
```

## struct VaultFactoryInfo

```solidity
struct VaultFactoryInfo {
    bool registered;        // 是否经 owner registerVaultFactory 注册
    bool enabled;           // 是否允许发币
    bool official;          // 是否官方认证工厂
    RiskLevel riskLevel;    // 风险等级
    VaultCategory category; // 工厂默认金库类型
}
```

## struct VaultInfo

```solidity
struct VaultInfo {
    address vault;          // 金库合约地址（路径 B 的 marketing 收款方）
    bool isOfficial;        // 是否官方工厂创建
    RiskLevel riskLevel;    // 发币时工厂风险等级
    VaultCategory category; // 金库类型
    address vaultFactory;   // 创建该金库的工厂
    string description;     // 金库 description()
}
```

---

# CosmVaultSchemas

## struct FieldDescriptor

```solidity
struct FieldDescriptor {
    string name;         // 字段名（vaultData 编码 key 或 method 参数名）
    string fieldType;    // 类型 hint：address · uint8 · uint16 · uint256 · bool 等
    string description;  // UI 展示说明
    uint8 decimals;      // 数值字段小数位；非数值填 0；BNB 类填 18
}
```

## struct VaultDataSchema

```solidity
struct VaultDataSchema {
    string description;           // vaultData 整体编码说明
    FieldDescriptor[] fields;     // 各字段描述
    bool isArray;                 // true=struct 数组；false=单 struct
}
```

## struct ApproveAction

```solidity
struct ApproveAction {
    string tokenType;        // 要 approve 的 token 类型 hint
    string amountFieldName;  // inputs 里数量字段名
}
```

## struct VaultMethodSchema

```solidity
struct VaultMethodSchema {
    string name;                  // 方法名
    string description;           // 方法说明
    FieldDescriptor[] inputs;     // 入参描述
    FieldDescriptor[] outputs;    // 返回值描述
    ApproveAction[] approvals;    // 写方法前需要的 approve
    bool isInputArray;            // 入参是否为数组
    bool isOutputArray;           // 返回值是否为数组
    bool isWriteMethod;           // true=写方法 · false=view
}
```

## struct VaultUISchema

```solidity
struct VaultUISchema {
    string vaultType;             // 金库类型标识（如 split · scheduled-buyback）
    string description;           // 金库整体说明
    VaultMethodSchema[] methods;  // view / write 方法列表
}
```

## struct FactoryPolicy

```solidity
struct FactoryPolicy {
    string target;       // 约束目标字段名（如 quoteToken）
    string operator;     // 比较符：eq · ne · gt · lt 等
    bytes value;         // 约束值（abi.encode）
    string description;  // 约束说明
}
```

---

# ICosmVaultFactory

## struct LaunchValidationDataV1

```solidity
struct LaunchValidationDataV1 {
    uint8 tokenVersion;           // 6=CosmTaxToken · 7=CosmToken
    address quoteToken;           // 曲线 quote
    uint16 buyTaxRate;            // 买入 transfer tax（bps）
    uint16 sellTaxRate;           // 卖出 transfer tax（bps）
    uint16 vaultBps;              // 进金库的 mkt 份额
    uint16 deflationBps;          // 回购销毁份额
    uint16 dividendBps;           // 持币分红份额
    uint16 lpBps;                 // 加 LP 份额；四路合计须 = 10000
    address dividendToken;        // 分红代币地址
    uint256 minimumShareBalance;  // 分红最小持币量
}
```
