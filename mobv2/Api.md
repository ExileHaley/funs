# MÖB V2 前端与运维 API

本文依据当前 `out/` 编译产物ABI和实际源码整理。当前版本尚未部署，部署后必须把下列占位符替换为本次部署日志中的代理地址；前端、operator和keeper不得使用implementation地址。

## 1. 单位和通用规则

- MÖB、NCN数量均为代币原始最小单位，当前两者为18位精度。
- USDT金额均为部署所用USDT合约的原始最小单位。前端必须读取 `USDT.decimals()`，不能假定18位。
- 时间均为Unix秒。
- 比例统一使用基点：`10000 = 100%`、`15000 = 1.5倍`、`30000 = 3倍`、`60000 = 6倍`。
- “历史累计”不会因领取、卡牌出局或时间变化而减少；“当前待领/当前可用/剩余额度”会随交易变化；“预览”只读且不锁定价格。
- 所有UUPS业务合约都存在 `owner()`、`transferOwnership(address)`、`upgradeToAndCall(address,bytes)` 等标准管理ABI，普通前端不应调用。

## 2. 部署地址

下列地址是加入 ReferralRegistry UUPS 代理之前的历史部署记录，不能用于当前源码。重新全量部署后，所有业务调用地址必须以新日志为准，并额外记录 `REFERRAL_REGISTRY_IMPLEMENTATION`；正式交付前应整体替换本节，不能只替换邀请合约地址。

```text
  USDX 0x8e723f48EC1c32eE9EE56D6A77bC2C9f9542B046
  MOB_V2_GATEWAY_PROXY 0x6753Bf680823ABB0ACbD3E6c70F05D1283397FF2
  INTERACTION_POOL 0xd6dCac29DA4c56bC1bd819d3ce0617043Afc9B95
  MOB 0xaADb78F6d8f7CA31f751fa90773dF6dFF4b0CAb9
  NCN 0x7C354848d2594Ea0dDF0937035A61e042A711c6F
  PRICE_ORACLE 0x86869725c3eB770f1F55FCE0898B4C2D5425E19d
  REFERRAL_REGISTRY 0x5FFBaFc5DFa6c6013093d7305151680230CEd5A7
  CARD_MANAGER 0xb3A58237a878C6d06EE549FFE4b33a780FD3EC21
  MINING_EARNINGS_SETTLEMENT 0x11BE4eCA649e5EAd1C611ba20576144AC21F3f58
  MINING_EARNINGS_LEDGER 0xDb50e6aBD1C7827C19727093c1135882cc2B19E9
  MINING_SYSTEM 0xd4BAD5B6Eb047ef839BC6957BCF724df65724e5C
  CARD_COMPOSER 0xCBf8a929A3731782b46013bad988431492e669Ee
  TRADING_ROUTER 0x64659372E88355cc22931c49bBA9fd22cC9c9010
  USER_DASHBOARD 0x230EdFd1cC8C42EcB9a3a3e3B1C55Ff192914235
  REWARD_DISTRIBUTOR 0x3D697FED9E2B5864fd97a3E8660b1903B644aC40
  TEAM_PERFORMANCE 0xa641C44479Ad90F5032BB2c9479801B9410E583D
  TEAM_ELIGIBILITY 0x68dEF13Bcc3044E065e97AEc640165f652e0eEe8
  ORGANIZATION_BENEFIT 0xb06C2a8Fc41139B83C025cc6E64c0A800733bD65
  NODE_REWARD_POOL 0xa537c53ad3E6Af07F3b3f1973C3e36CC45fDCFA6
  MINING_CARD 0xd775D0DDCfBbf47850936B4D9dE269F79EE0bD58
  ORGANIZATION_CARD 0x8b730bd9f2345C55aD8875f19b6941EC59B9D716
  NODE_CARD 0x7A9CA6e77b7141D603119f691458869be64Ea5dB
  MOB_V2_GATEWAY_IMPLEMENTATION 0x44879482E9070D4dE21D5B21bC64FA226BC52AFf
  INTERACTION_POOL_IMPLEMENTATION 0x50d6121eBBcf162CA83408Ac47fF214004f16d33
  CARD_MANAGER_IMPLEMENTATION 0xd1A05F4A1D130405e6448aa60d0261f83C399BA4
  ORGANIZATION_BENEFIT_IMPLEMENTATION 0x3cDeebF14694809BeA5f5C9f4168D9E53b088EF9
  NODE_REWARD_POOL_IMPLEMENTATION 0x824B99142f98B8134466DEf4a285394f687f8A64
  REWARD_DISTRIBUTOR_IMPLEMENTATION 0x34d94cE5f47D20FE002DAfDFAF1C4619C9e72D1e
  TEAM_PERFORMANCE_IMPLEMENTATION 0x50fF879035e1e0eA46a70315D69368e26153792F
  TEAM_ELIGIBILITY_IMPLEMENTATION 0xE894db7b40D9D7D1DDb818920A3eCc71288FEb46
  MINING_EARNINGS_SETTLEMENT_IMPLEMENTATION 0x1837D7D05D2aA65f06b75B646b518d4c6c5bCec5
  MINING_EARNINGS_LEDGER_IMPLEMENTATION 0x49C1AD00ef9cf01cdC79d321e1Bc9eC1Ae8dcB98
  MINING_SYSTEM_IMPLEMENTATION 0xa2BEf3572F0e6A58Ca67b2FC81D4256b1ABe0937
  CARD_COMPOSER_IMPLEMENTATION 0xdfcBC4EBC8341159075DEe210F4D3F355D6b669e
  TRADING_ROUTER_IMPLEMENTATION 0x9a7ffD818560848F807D7c13bBCB4E40Cadb86A3
  USER_DASHBOARD_IMPLEMENTATION 0x5615F6FB6b4B95C46D1Ec17DA20C5e07Df086c0f
  REFERRAL_REGISTRY_IMPLEMENTATION 0x1520457492eBB492D54010A3B7B6b05261C4BA84
```

## 3. 枚举

```solidity
enum MiningCardType {
    U100,   // 0：100U矿机卡
    U300,   // 1：300U矿机卡
    U1000,  // 2：1,000U矿机卡
    U3000,  // 3：3,000U矿机卡
    U10000  // 4：10,000U矿机卡
}

enum MiningPaymentMethod {
    NcnAndUsdt, // 0：NCN + USDT，仅U100支持
    MobAndUsdt  // 1：MÖB + USDT；U100为100% MÖB
}

enum OrganizationType {
    RingOrganization, // 0：环组织卡
    MobiusRing         // 1：莫比乌斯环卡
}

enum NodeType {
    Foundation, // 0：基金会节点，不开放用户购买
    Genesis,    // 1：创世节点
    CoFounder,  // 2：联创节点
    Community   // 3：社区节点
}

enum RewardKind {
    Direct,   // 0：直推MÖB
    Level,    // 1：层级MÖB
    TeamMob,  // 2：团队MÖB
    TeamUsdt, // 3：铸币费团队USDT
    GradeSale // 4：V3/V4卖出手续费USDT
}

enum SwitchReason {
    None,                // 0：当前没有Pair模式触发原因
    DailyOutflow,        // 1：24小时流出触发
    SevenDayNetOutflow,  // 2：七日净流出触发
    PoolRatio            // 3：交互池/Pair余额比触发
}

// MiningSystem.Position.accelerationTier
// 0：当前没有质押加速本金；1：低档；2：高档。
```

## 4. 授权对象

| 操作 | 资产 | spender / NFT operator |
| --- | --- | --- |
| 购买矿机卡、节点卡 | MÖB、NCN、USDT | CardManager代理 |
| 激活矿机卡、续费 | MiningCard、USDT | MiningSystem代理 |
| 质押/升级加速 | MÖB | MiningSystem代理 |
| 合成钱包补差 | MÖB | CardComposer代理 |
| 质押组织卡 | OrganizationCard | OrganizationBenefit代理 |
| 质押节点卡 | NodeCard | NodeRewardPool代理 |
| 买入NCN | USDT | TradingRouter代理 |
| 卖出NCN | NCN | TradingRouter代理 |
| 卖出MÖB | MÖB | TradingRouter代理 |

Gateway只转发调用，不是资产授权对象。

---

### MobV2Gateway

前端统一写入口。返回值和下游业务代理一致，全部业务校验仍由下游合约执行。

```solidity
// 永久绑定邀请人。普通邀请人必须已登记且当前有有效工作卡；首码不要求工作卡。
function register(address inviter) external;

// 购买矿机卡。maxMob/maxNcn是用户接受的最大支付量；USDT没有单独上限。
// 返回新矿机NFT tokenId。
function buyMiningCard(
    MiningCardType cardType,
    MiningPaymentMethod method,
    uint256 maxMob,
    uint256 maxNcn
) external returns (uint256 tokenId);

// 购买创世、联创或社区节点卡；基金会节点不可购买。
// maxMob是用户接受的最大MÖB支付量，返回节点NFT tokenId。
function buyNodeCard(NodeType nodeType, uint256 maxMob) external returns (uint256 tokenId);

// 永久质押矿机NFT、支付首期铸币费并创建仓位。
// 返回positionId；后续静态收益、加速、续费、提现和合成都使用该编号。
function activateMiningCard(uint256 tokenId) external returns (uint256 positionId);

// 为指定仓位续缴一个30天铸币费周期。
function payMintFee(uint256 positionId) external;

// 首次为指定工作仓位质押MÖB加速本金，按当前Pair价判断低档或高档。
function stakeAcceleration(uint256 positionId, uint256 mobAmount) external;

// 将当前低档加速一次升级到高档；按补仓时价格检查总本金价值。
function upgradeAcceleration(uint256 positionId, uint256 additionalMob) external;

// 一次赎回指定仓位当前全部加速本金。
// 未出局仓位赎回后可以重新质押；出局仓位不能重新质押。
function redeemAcceleration(uint256 positionId) external;

// 正常提现指定单卡当前已成熟、尚未消费的基础和加速静态收益。
// 不检查NCN资格，不收20%费用；返回实际支付前毛MÖB。
function withdrawNormal(uint256 positionId) external returns (uint256 grossMob);

// 立即提现指定单卡当前未成熟、尚未消费的基础和加速静态收益。
// 返回毛MÖB和发送到dead的20% feeMob；用户实际到账为grossMob-feeMob。
function withdrawImmediate(uint256 positionId)
    external
    returns (uint256 grossMob, uint256 feeMob);

// 使用指定单卡已产生且未消费的基础/加速收益合成目标矿机卡。
// 不足部分从钱包MÖB补齐且不得超过maxWalletMob；返回新NFT tokenId。
function composeMiningCard(
    uint256 positionId,
    MiningCardType cardType,
    uint256 maxWalletMob
) external returns (uint256 tokenId);

// 永久质押组织卡。
function stakeOrganizationCard(uint256 tokenId) external;

// 永久质押节点卡。
function stakeNodeCard(uint256 tokenId) external;

// 一次领取全部动态奖励；返回实际到账MÖB和USDT。
// 实际到账MÖB会增加用户MÖB卖出额度。
function claimDynamicReward() external returns (uint256 mobAmount, uint256 usdtAmount);

// 一次领取全部组织权益USDT。
function claimOrganizationReward() external returns (uint256 usdtAmount);

// 一次领取四类节点的全部MÖB和USDT。
// 实际到账MÖB会增加用户MÖB卖出额度。
function claimNodeReward() external returns (uint256 mobAmount, uint256 usdtAmount);

// 使用USDT买入NCN，返回实际收到的NCN。
function buyNcn(uint256 usdtAmount) external returns (uint256 ncnReceived);

// 卖出NCN，返回实际收到的USDT；该交易量长期累计到MÖB卖出资格。
function sellNcn(uint256 ncnAmount) external returns (uint256 usdtReceived);

// 卖出MÖB，返回扣除手续费后实际收到的USDT。
// 每次都检查NCN累计资格、工作卡单次限额和MÖB剩余卖出额度。
function sellMob(uint256 mobAmount) external returns (uint256 receivedUsdt);

// 主动按当前业绩和持币条件刷新调用者团队等级；不是keeper方法。
function refreshMyTeamLevel() external returns (uint8 oldLevel, uint8 newLevel);

// 无权限补充检查交易模式。没有自动检查交易发生时可由任意地址调用。
function checkAndSwitchMode() external;

// 无权限每日加LP；合约内检查24小时间隔和模式条件。
// 返回使用的USDT、MÖB和获得的LP数量。
function executeDailyLiquidity()
    external
    returns (uint256 usdtAmount, uint256 mobAmount, uint256 liquidity);

// 无权限每日回购销毁；合约内检查24小时间隔和Pair模式。
function executeDailyBuyback() external;

// 无权限同步最多100条已到期V3/V4资格，返回本次处理数量。
function syncExpiredEligibility() external returns (uint256 processed);

// 通用受控转发。target必须在allowedTarget中；data为目标方法ABI编码。
// 正常前端优先使用上面的类型化入口。
function execute(address target, bytes calldata data)
    external
    payable
    returns (bytes memory result);

// 只读返回当前11个业务目标地址。
function targets()
    external
    view
    returns (
        address referralRegistry,
        address cardManager,
        address miningSystem,
        address cardComposer,
        address organizationBenefit,
        address nodeRewardPool,
        address rewardDistributor,
        address teamPerformance,
        address teamEligibilityManager,
        address interactionPool,
        address tradingRouter
    );

// owner一次性配置Gateway目标。部署组装方法，普通前端不得调用。
function configureTargets(Targets calldata targets_) external;

// 查询地址是否是Gateway允许转发的目标。
function allowedTarget(address target) external view returns (bool);
```

---

### UserDashboard

推荐前端主要读取入口。所有方法只读，不需要授权。

```solidity
struct UserOverview {
    address inviter;                       // 永久邀请人；未绑定时为0地址
    uint256 registrationAt;                // 首张真实卡激活或首张虚拟卡导入时间，Unix秒；也是NCN考核起点
    uint256 directReferralCount;           // 已永久绑定的直属地址总数，不要求直属已激活
    uint256 effectiveDirectReferralCount;  // 历史有效直属数；直属首次激活/导入后永久计数一次
    uint256 subtreeMintFeePaidUsdt;         // 伞下最多15层历史完整铸币费支付总额；不含本人，不是待领奖励
    uint256 workingCardCount;               // 当前未出局工作卡数量
    uint256 totalCardCount;                 // 历史仓位总数，包含工作、出局、真实和虚拟卡
    uint256 exitedCardCount;                // 历史已出局仓位数量
    uint8 teamLevel;                        // 当前V等级，0表示无等级，1-7表示V1-V7
    address maximumBranch;                  // 当前业绩最大的直属分区地址
    uint256 maximumBranchPerformance;       // 当前最大区业绩，USDT最小单位
    uint256 totalPerformance;               // 当前团队总业绩，USDT最小单位
    uint256 smallAreaPerformance;           // 当前小区业绩=总业绩-最大区，USDT最小单位
    uint256 walletMob;                      // 当前钱包MÖB余额
    uint256 walletUsdt;                     // 当前钱包USDT余额
    uint256 miningGrossMob;                 // 历史已结算矿机静态毛MÖB，含立即提现被销毁部分
    uint256 miningPaidMob;                  // 历史矿机静态实际支付到用户的MÖB
    uint256 miningImmediateFeeMob;          // 历史立即提现发送到dead的MÖB
    uint256 dynamicPendingMob;              // 当前待领取动态MÖB
    uint256 dynamicPendingUsdt;             // 当前待领取动态USDT，含团队和V3/V4账户
    uint256 dynamicClaimedMob;              // 历史实际领取动态MÖB
    uint256 dynamicClaimedUsdt;             // 历史实际领取动态USDT
    uint256 organizationRecordedUsdt;       // 历史记入组织权益的USDT总额
    uint256 organizationPendingUsdt;        // 当前待领取组织USDT
    uint256 organizationClaimedUsdt;        // 历史实际领取组织USDT
    uint256 nodePendingMob;                 // 当前四类节点合计待领取MÖB
    uint256 nodePendingUsdt;                // 当前四类节点合计待领取USDT
    uint256 nodeClaimedMob;                 // 历史实际领取节点MÖB
    uint256 nodeClaimedUsdt;                // 历史实际领取节点USDT
    uint256 cumulativeMobReceived;          // 历史经批准奖励合约实际支付的MÖB，是卖出额度来源
    uint256 cumulativeMobSold;              // 历史成功通过TradingRouter卖出的MÖB
    uint256 remainingMobSellAllowance;      // 当前剩余可卖MÖB=cumulativeMobReceived-cumulativeMobSold
    uint256 workingMobSellLimitUsdt;        // 当前所有工作卡累计的一笔MÖB卖出毛USDT上限
}

struct DynamicRewardTotals {
    uint256 directMob;    // 历史记账直推MÖB，不是当前待领
    uint256 levelMob;     // 历史记账层级MÖB，不是当前待领
    uint256 teamMob;      // 历史记账团队MÖB，不是当前待领
    uint256 teamUsdt;     // 历史记账铸币费团队USDT，不是当前待领
    uint256 gradeSaleUsdt;// 历史记账V3/V4卖出手续费USDT，不是当前待领
}

// 简单例子：A邀请B、B邀请C；B累计支付90U铸币费，C累计支付100U。
// A的subtreeMintFeePaidUsdt为190U，B的为100U；两者都不包含本人支付。
// teamUsdt只表示按团队规则记给用户的历史奖励，dynamicPendingUsdt才是当前可领取动态USDT；
// organizationPendingUsdt是组织卡当前可领取权益，三者都不等于subtreeMintFeePaidUsdt。

struct Position {
    address account;              // 仓位所属用户
    uint256 nftTokenId;           // 真实卡NFT编号；虚拟卡为0
    uint8 cardType;               // MiningCardType：0-4
    uint8 exitMultiple;           // 出局倍数：3或5
    uint8 accelerationTier;       // 当前加速档：0无、1低档、2高档
    uint64 activatedAt;           // 激活/导入时间
    uint64 activeUntil;           // 当前铸币费有效截止时间
    uint64 lowAccelerationAt;     // 当前周期进入低档的时间；未进入为0
    uint64 highAccelerationAt;    // 当前周期进入高档的时间；未进入为0
    uint64 exitedAt;              // 出局时间；未出局为0
    uint256 accelerationMob;      // 当前未赎回加速本金MÖB；已赎回为0
    uint256 exitCapUsdt;          // 该仓位总出局上限，USDT计价值
    uint256 exitUsedUsdt;         // 历史已占用出局额度，USDT计价值
    uint256 previous;             // 工作卡FIFO上一仓位编号；0表示没有
    uint256 next;                 // 工作卡FIFO下一仓位编号；0表示没有
    bool virtualCard;             // true表示owner导入的旧虚拟卡
    bool accelerationUpgraded;    // 当前加速周期是否已从低档升级过
    bool exited;                  // 是否已经出局
}

struct CardView {
    uint256 positionId;                  // 矿机仓位编号
    Position position;                   // 仓位完整状态
    uint256 consumedBaseUsdt;            // 历史已被提现/合成消费的基础收益USDT计价值
    uint256 consumedAccelerationUsdt;    // 历史已被提现/合成消费的加速收益USDT计价值
}

struct EarningsStatus {
    uint256 producedBaseUsdt;                    // 截至当前历史产生的基础收益USDT计价值
    uint256 producedAccelerationUsdt;            // 截至当前历史产生的加速收益USDT计价值
    uint256 maturedBaseUsdt;                     // 按当前成熟期截至当前已成熟的历史基础收益总值
    uint256 maturedAccelerationUsdt;             // 按当前成熟期截至当前已成熟的历史加速收益总值
    uint256 consumedBaseUsdt;                    // 历史已被提现/合成消费的基础收益总值
    uint256 consumedAccelerationUsdt;            // 历史已被提现/合成消费的加速收益总值
    uint256 availableMaturedBaseUsdt;            // 当前正常提现可用基础收益
    uint256 availableMaturedAccelerationUsdt;    // 当前正常提现可用加速收益
    uint256 availableUnmaturedBaseUsdt;          // 当前立即提现可用基础收益
    uint256 availableUnmaturedAccelerationUsdt;  // 当前立即提现可用加速收益
}

struct RewardRecord {
    address source;    // 触发本条奖励的用户地址
    uint256 amount;    // 记账数量；币种由token字段确定
    address token;     // MÖB或USDT合约地址
    uint8 kind;        // RewardKind：0-4
    uint64 recordedAt; // 记账时间
}

struct TradingModeDiagnosis {
    TradingModeStatus status;                    // TradingRouter原始模式状态；字段定义见TradingRouter章节
    uint256 switchedAt;                          // 最近一次模式切换时间，Unix秒；从未切换为0
    uint256 dailyOutflowBps;                     // 单日毛流出切出阈值，基点；operator通过setFlowSwitchThresholds设置
    uint256 sevenDayNetOutflowBps;               // 七日净流出切出阈值，基点；operator通过setFlowSwitchThresholds设置
    uint256 poolToPairExitRatioBps;              // 池/Pair余额切出倍数，基点；operator通过setPoolToPairExitRatioBps设置
    uint256 poolBalanceRecoveryRatioBps;         // 池余额回切倍数，基点；operator通过setPoolBalanceRecoveryRatioBps设置
    uint256 pairOutflowRecoveryMultipleBps;      // Pair流出回切倍数，基点；operator通过setPairOutflowRecoveryMultipleBps设置
    uint256 priceRecoveryBps;                    // 价格恢复回切比例，基点；operator通过setPriceRecoveryThresholds设置
    uint256 priceDropRecoveryBps;                // 价格下跌回切比例，基点；operator通过setPriceRecoveryThresholds设置
    uint256 poolRecoveryMinimumUsdt;             // 满足严格大于规则所需的交互池最低余额，USDT最小单位
    uint256 missingPoolRecoveryUsdt;             // 当前距余额回切最低值尚缺多少USDT；0表示已满足
    uint256 missingPairOutflowToTriggerUsdt;     // Pair流出距原始切出触发额尚缺多少USDT；0表示已达到
    uint256 pairOutflowMultipleTargetUsdt;       // Pair流出倍数回切所需累计目标，向上取整，USDT最小单位
    uint256 missingPairOutflowToMultipleUsdt;    // 当前距Pair流出倍数目标尚缺多少USDT；0表示已满足
    uint256 recoveryPriceMinimumUsdt;            // “达到原触发额且价格恢复”路径所需最低MÖB价格，向上取整
    uint256 missingRecoveryPriceUsdt;            // 当前MÖB价格距恢复目标尚缺多少USDT计价值；0表示已满足价格部分
    uint256 priceDropMaximumUsdt;                // 价格下跌回切允许的最高MÖB价格；当前价小于等于它即满足
    uint256 excessPriceAboveDropTargetUsdt;      // 当前价仍高于下跌目标多少；0表示已满足下跌条件
    bool usesPoolBalanceRecoveryRatio;           // 当前切入原因是否使用余额恢复倍数；Pair模式下始终为true
    bool usesPairOutflowRecoveryMultiple;        // 当前切入原因是否使用Pair流出倍数；仅流量原因Pair模式为true
    bool usesPriceRecoveryRatio;                 // 当前切入原因是否使用价格恢复比例；仅流量原因Pair模式为true
    bool usesPriceDropRecoveryRatio;             // 当前切入原因是否使用价格下跌比例；仅流量原因Pair模式为true
}

// 返回用户聚合总览。字段中历史累计、当前待领和当前余额必须按注释分别展示。
function getUserOverview(address account)
    external view returns (UserOverview memory overview);

// 返回MÖB卖出使用的累计NCN资格；字段定义见TradingRouter章节。
function getNcnSellQualification(address account)
    external view returns (NcnSellQualificationStatus memory status);

// 一次返回当前模式为何切换、切换基数、七项operator配置、所有回切目标和当前差额。
// activeReason=PoolRatio时只看usesPoolBalanceRecoveryRatio对应的余额恢复路径。
// activeReason=DailyOutflow或SevenDayNetOutflow时，四个uses字段对应的路径任一满足即可回切。
// 非Pair模式时四个uses字段均为false；此接口只读，不会主动执行模式切换。
function getTradingModeDiagnosis()
    external view returns (TradingModeDiagnosis memory diagnosis);

// 返回首张真实卡激活或首张虚拟卡导入时间；0表示尚无考核起点。
function firstActivationAt(address account) external view returns (uint256 activatedAt);

// 返回该地址当前能否作为激活奖励/动态奖励意义上的有效工作卡用户。
// 条件是至少一张未出局工作卡且最大铸币费有效期尚未到期。
function isRewardEligible(address account) external view returns (bool eligible);

// 返回单卡当前加速周期和全DApp当前加速本金。
// canRestake仅表示卡未出局、本金为0且全局加速未永久关闭。
function getPositionAccelerationStatus(uint256 positionId)
    external view returns (
        uint256 principalMob,
        uint8 currentTier,
        uint64 cycleStartedAt,
        uint64 cycleEndedAt,
        uint64 earningUntil,
        uint256 dailyExtraUsdt,
        bool earningActive,
        bool canRestake,
        uint256 totalAccelerationMobStaked
    );

// 返回指定单卡按秒静态收益和按当前Pair价格计算的MÖB预览。
// maturedWithdrawalMob：当前正常提现毛MÖB；
// immediateGross/Fee/NetMob：当前立即提现毛额、20%销毁费、预计实收；
// compositionMob：当前全部未消费收益用于合成时的MÖB价值。
// 所有MÖB结果是实时预览，交易前价格变化会导致实际值变化。
function getPositionEarnings(uint256 positionId)
    external view returns (
        EarningsStatus memory status,
        uint256 maturedWithdrawalMob,
        uint256 immediateGrossMob,
        uint256 immediateFeeMob,
        uint256 immediateNetMob,
        uint256 compositionMob
    );

// 返回五类动态奖励的历史记账总额，不是当前待领。
function getDynamicRewardTotals(address account)
    external view returns (DynamicRewardTotals memory totals);

// 分页读取当前工作卡FIFO。cursorPositionId=0从头开始；nextCursor=0表示结束。
// limit必须为1-100。
function getWorkingCards(address account, uint256 cursorPositionId, uint256 limit)
    external view returns (CardView[] memory cards, uint256 nextCursor);

// 按历史创建顺序分页读取全部仓位。limit必须为1-100。
function getCardHistory(address account, uint256 offset, uint256 limit)
    external view returns (CardView[] memory cards, uint256 nextOffset);

// 从历史仓位中扫描并返回已出局仓位。scanLimit是扫描条数而非保证返回条数，必须为1-100。
function getExitedCards(address account, uint256 offset, uint256 scanLimit)
    external view returns (CardView[] memory cards, uint256 nextOffset);

// 分页读取永久绑定的直属地址。limit必须为1-100。
function getDirectReferrals(address account, uint256 offset, uint256 limit)
    external view returns (address[] memory referrals, uint256 nextOffset);

// 分页读取动态奖励记账明细。limit必须为1-100。
function getDynamicRewardRecords(address account, uint256 offset, uint256 limit)
    external view returns (RewardRecord[] memory records, uint256 nextOffset);
```
---

### CardManager

用户购买和operator铸造入口。付款预览与实际购买复用同一价格和拆分逻辑。

```solidity
// 预览矿机卡实际付款拆分。返回值均为代币原始最小单位。
// mobAmount：需要支付并发送到dead的MÖB；
// ncnAmount：需要支付并发送到dead的NCN；
// usdtAmount：需要转入交互池的USDT。
function previewBuyMiningCard(MiningCardType cardType, MiningPaymentMethod method)
    external view
    returns (uint256 mobAmount, uint256 ncnAmount, uint256 usdtAmount);

// 购买矿机卡。maxMob/maxNcn是滑点保护上限，返回新NFT tokenId。
// 完成USDT入池后自动检查交易模式。
function buyMiningCard(
    MiningCardType cardType,
    MiningPaymentMethod method,
    uint256 maxMob,
    uint256 maxNcn
) external returns (uint256 tokenId);

// 预览节点卡付款拆分。
// mobAmount：发送到dead的MÖB；poolUsdt：转入交互池；recipientUsdt：转到节点售卡分账地址。
function previewBuyNodeCard(NodeType nodeType)
    external view
    returns (uint256 mobAmount, uint256 poolUsdt, uint256 recipientUsdt);

// 购买创世、联创或社区节点卡；基金会节点回滚。
// maxMob是MÖB支付上限，返回新NFT tokenId。
function buyNodeCard(NodeType nodeType, uint256 maxMob)
    external returns (uint256 tokenId);

// 返回矿机面值，单位为USDT最小单位。
function miningFaceValue(MiningCardType cardType) external view returns (uint256);

// 返回铸币费拆分。poolAmount进入交互池，foundationAmount进入基金会；
// 伞下统计累计两者之和。
function miningMintFee(MiningCardType cardType)
    external view returns (uint256 poolAmount, uint256 foundationAmount);

// 返回基础、低档总收益、高档总收益的日USDT计价值。
function miningDailyRates(MiningCardType cardType)
    external view
    returns (uint256 baseRate, uint256 lowTotalRate, uint256 highTotalRate);

// 返回该类型每张工作卡贡献的一笔MÖB卖出毛USDT上限。
function miningSellLimit(MiningCardType cardType) external view returns (uint256);

// 返回节点卡USDT面值。
function nodeFaceValue(NodeType nodeType) external view returns (uint256);

// 返回节点类型固定供应上限。
function nodeSupplyCap(NodeType nodeType) external pure returns (uint256);

// 返回节点类型当前已铸造、上限和剩余数量；三个字段都是NFT张数。
function nodeSupplyStatus(NodeType nodeType)
    external view
    returns (uint256 minted, uint256 cap, uint256 remaining);

// 以下六个方法必须由CardManager.operator直接调用CardManager代理，不经过Gateway。
// 调用后立即铸造NFT到非零recipient，不收取MÖB/NCN/USDT；owner不能直接代替operator调用。
// 单张免费铸造任意矿机、组织或节点卡到非零地址。
function operatorMintMining(address recipient, MiningCardType cardType)
    external returns (uint256 tokenId);
function operatorMintOrganization(address recipient, OrganizationType cardType)
    external returns (uint256 tokenId);
function operatorMintNode(address recipient, NodeType nodeType)
    external returns (uint256 tokenId);

// 批量免费铸造；quantity必须为1-100，返回按铸造顺序排列的tokenId。
// 节点整批先检查供应上限，任一超限则整笔回滚。
// 例如quantity=3会给同一recipient连续铸造3张同类型NFT。
function operatorMintMiningBatch(
    address recipient,
    MiningCardType cardType,
    uint256 quantity
) external returns (uint256[] memory tokenIds);
function operatorMintOrganizationBatch(
    address recipient,
    OrganizationType cardType,
    uint256 quantity
) external returns (uint256[] memory tokenIds);
function operatorMintNodeBatch(
    address recipient,
    NodeType nodeType,
    uint256 quantity
) external returns (uint256[] memory tokenIds);

// 仅owner直接调用CardManager代理，更换非零operator；不转移资产并立即生效。
function setOperator(address operator_) external;

// owner设置300U及以上矿机卡的MÖB支付比例，基点0-10000，修改后立即用于预览和购买。
function setMiningMobBps(uint256 newBps) external;

// owner设置节点售卡USDT分账地址。
function setNodePaymentRecipient(address recipient) external;

// 读取当前operator、矿机MÖB比例和节点售卡分账地址。
function operator() external view returns (address);
function miningMobBps() external view returns (uint256);
function nodePaymentRecipient() external view returns (address);
```

---

### CardComposer

```solidity
// 使用指定单卡已经产生且尚未消费的基础和加速收益合成。
// 最多消费目标面值对应的USDT价值；不足部分由钱包MÖB补齐。
// maxWalletMob是钱包补差上限，返回新矿机NFT tokenId。
function composeCard(
    uint256 positionId,
    MiningCardType targetType,
    uint256 maxWalletMob
) external returns (uint256 tokenId);
```

---

### TradingRouter

```solidity
struct NcnSellQualificationStatus {
    uint256 qualificationStartedAt; // 考核起点时间戳：首张真实卡激活或首张虚拟卡导入时间；0表示尚无起点
    uint256 elapsedDays;              // 从考核起点到当前，包含本周期的累计考核天数
    uint256 dailyRequirementUsdt;     // operator当前设置的每日X，USDT最小单位；0表示关闭NCN资格检查
    uint256 requiredTotalUsdt;        // 当前累计必须完成的NCN交易总额=elapsedDays*每日X
    uint256 cumulativeBuyUsdt;        // 历史经DApp买入NCN的实际USDT价值，不会在卖MÖB时扣减
    uint256 cumulativeSellUsdt;       // 历史经DApp卖出NCN的实际USDT价值，不会按日清零
    uint256 missingTotalUsdt;         // 距总交易额要求仍缺少的USDT；已满足、豁免或X=0时为0
    uint256 missingBuyUsdt;           // 距“买入严格大于要求50%”仍缺少的USDT；含最小单位+1规则
    bool qualified;                   // 当前是否满足MÖB卖出NCN资格
    bool exempt;                      // 是否因提现豁免地址而跳过NCN资格
}

struct MobSellPreview {
    bool pairMode;                  // true使用Uniswap Pair，false由交互池报价/支付
    uint256 grossUsdt;              // 扣手续费前预计毛USDT
    uint256 feeUsdt;                // 按当前所有条件预计扣除的USDT手续费
    uint256 receivedUsdt;           // 预计用户实收USDT=grossUsdt-feeUsdt
    uint256 workingSellLimitUsdt;   // 当前工作卡合计的一笔毛USDT卖出上限
    uint256 cumulativeMobReceived;  // 历史实际领取并计入卖出额度的MÖB总量，不含转入或购买所得
    uint256 cumulativeMobSold;      // 历史通过TradingRouter成功卖出的MÖB总量
    uint256 remainingMobAllowance;  // 当前奖励来源剩余可卖MÖB数量
    bool passesWorkingSellLimit;    // 当前预览毛USDT是否不超过工作卡限额
    bool passesMobAllowance;        // 本次mobAmount是否不超过剩余MÖB额度
    bool ncnQualified;              // 当前NCN累计资格是否满足
}

struct TradingModeStatus {
    bool pairMode;                         // 当前是否为Pair模式
    uint8 activeReason;                    // 当前Pair模式触发原因SwitchReason
    uint256 poolUsdtBalance;               // 当前交互池USDT余额
    uint256 pairUsdtBalance;               // 当前MÖB/USDT Pair的USDT余额
    uint256 poolToPairRatioBps;             // 当前池余额/Pair余额比；10000=1倍
    uint64 dailyWindowStartedAt;            // 当前24小时流量窗口起点
    uint256 dailyInitialPoolBalance;        // 24小时窗口起点交互池余额
    uint256 dailyOutflow;                   // 本窗口累计卖出USDT流出
    uint256 dailyInflow;                    // 本窗口累计业务USDT流入
    uint256 dailyNetOutflow;                // max(dailyOutflow-dailyInflow,0)，仅展示；单日触发仍按毛流出
    uint256 dailyTriggerUsdt;               // 按operator比例计算的24小时触发额
    bool dailyConditionMet;                 // 是否达到24小时流出触发条件
    uint64 sevenDayWindowStartedAt;         // 当前七日流量窗口起点
    uint256 sevenDayInitialPoolBalance;     // 七日窗口起点交互池余额
    uint256 sevenDayOutflow;                // 七日累计卖出USDT流出
    uint256 sevenDayInflow;                 // 七日累计业务USDT流入
    uint256 sevenDayNetOutflow;             // max(outflow-inflow,0)
    uint256 sevenDayTriggerUsdt;             // 按operator比例计算的七日触发额
    bool sevenDayConditionMet;               // 是否达到七日净流出触发条件
    uint256 switchPriceUsdt;                 // 进入Pair模式时记录的MÖB价格
    uint256 currentMobPriceUsdt;             // 当前MÖB价格
    uint256 switchTriggerUsdtAmount;         // 进入Pair模式时记录的流量触发额；余额比触发时可为0
    uint256 pairOutflowUsdt;                 // 进入Pair模式后当前统计的Pair流出
    uint256 pairOutflowRecoveryTargetUsdt;   // operator流出倍数对应的回切目标
    bool flowAndPriceRecoveryMet;            // 流出达到触发额且价格达到恢复比例
    bool pairOutflowMultipleMet;             // Pair流出达到operator倍数
    bool priceDropRecoveryMet;               // 价格下跌达到operator比例
    bool poolRatioExitMet;                   // 当前满足进入Pair模式的余额比
    bool poolRatioRecoveryMet;               // 当前满足返回交互池的余额恢复比
    bool canSwitch;                          // 现在调用checkAndSwitchMode是否会切换
    uint8 prospectiveReason;                 // 非Pair模式下若切换将采用的SwitchReason
}

// 买入NCN，返回实际收到的NCN；累计买入USDT增加。
function buyNcn(uint256 usdtAmount) external returns (uint256 ncnReceived);

// 卖出NCN，返回实际收到的USDT；累计卖出USDT增加。
function sellNcn(uint256 ncnAmount) external returns (uint256 usdtReceived);

// 卖出MÖB，返回实际收到USDT。
// 所有资格、手续费、销毁、奖励和模式检查在同一交易内原子执行。
function sellMob(uint256 mobAmount) external returns (uint256 receivedUsdt);

// 完整预览MÖB卖出。覆盖20%-70%手续费、保护额度分段和三项前置资格。
// 预览不锁价，实际交易可能因Pair储备或状态变化略有差异。
function previewSellMob(address seller, uint256 mobAmount)
    external view returns (MobSellPreview memory preview);

// 返回当前NCN累计资格；UserDashboard提供相同聚合入口。
function ncnSellQualificationStatus(address account)
    external view returns (NcnSellQualificationStatus memory status);

// 返回历史奖励实收MÖB、历史成功卖出MÖB和当前剩余额度。
function mobSellAllowance(address account)
    external view
    returns (uint256 receivedMob, uint256 soldMob, uint256 remainingMob);

// 仅返回当前剩余可卖MÖB。
function remainingMobSellAllowance(address account)
    external view returns (uint256);

// 返回模式切换所需的全部当前数据和判断结果。
function getTradingModeStatus()
    external view returns (TradingModeStatus memory status);

// 返回当前是否Pair模式。
function isPairMode() external view returns (bool);

// 无权限按当前状态检查并切换模式。
function checkAndSwitchMode() external;

// 以下三个配置方法必须直接调用TradingRouter代理，不经过Gateway；不转移资产且立即生效。
// owner只能通过setOperator更换非零operator，不能直接代替operator设置参数。
// operator设置24小时流出和七日净流出阈值，范围0-10000基点，默认700/2700。
// 例如700表示窗口初始池余额的7%。
function setFlowSwitchThresholds(
    uint256 dailyOutflowBps_,
    uint256 sevenDayNetOutflowBps_
) external;

// operator设置价格恢复/下跌回切比例，范围0-10000基点，默认10000/3000。
function setPriceRecoveryThresholds(
    uint256 priceRecoveryBps_,
    uint256 priceDropRecoveryBps_
) external;

// 设置每地址每日NCN累计考核额，范围0-50U；0关闭资格检查。
// 仅operator调用；默认10U。参数使用USDT原始最小单位，修改后立即追溯全部历史考核天数。
function setDailyNcnRequirement(uint256 amountUsdt) external;

// 仅owner直接调用TradingRouter代理，更换非零operator；不转移资产并立即生效。
function setOperator(address operator_) external;

// owner设置Uniswap Router、两个Pair和手续费钱包。
function setUniswapRouter(address router) external;
function setPairs(address mobPair_, address ncnPair_) external;
function setFeeWallets(address foundationWallet_, address insuranceWallet_) external;

// 关键配置和历史累计读取。
function operator() external view returns (address);
function dailyNcnRequirementUsdt() external view returns (uint256);
function dailyOutflowBps() external view returns (uint256);
function sevenDayNetOutflowBps() external view returns (uint256);
function priceRecoveryBps() external view returns (uint256);
function priceDropRecoveryBps() external view returns (uint256);
function cumulativeNcnBuyUsdt(address account) external view returns (uint256);
function cumulativeNcnSellUsdt(address account) external view returns (uint256);
function cumulativeMobReceived(address account) external view returns (uint256);
function cumulativeMobSold(address account) external view returns (uint256);
function historicalReceivedUsdt(address account) external view returns (uint256);
```
---

### MiningSystem

前端通常通过Gateway写、Dashboard读；以下直接接口用于运维、审计和必要的底层调用。

```solidity
// 直接激活矿机卡，返回positionId。
function activate(uint256 tokenId) external returns (uint256 positionId);

// 直接续缴指定仓位铸币费。
function payMintFee(uint256 positionId) external;

// 直接质押、升级和赎回加速本金。
function stakeAcceleration(uint256 positionId, uint256 mobAmount) external;
function upgradeAcceleration(uint256 positionId, uint256 additionalMob) external;
function redeemAcceleration(uint256 positionId) external;

// 单卡正常/立即提现；签名已经没有任何窗口参数。
function withdrawNormal(uint256 positionId) external returns (uint256 grossMob);
function withdrawImmediate(uint256 positionId)
    external returns (uint256 grossMob, uint256 feeMob);

// 返回单卡历史累计已消费的基础和加速USDT计价值。
function positionConsumedTotals(uint256 positionId)
    external view
    returns (uint256 consumedBaseUsdt, uint256 consumedAccelerationUsdt);

// 按矿机类型返回当前和历史数量。
// workingTotal/exitedTotal：包含真实NFT和虚拟卡；
// workingNft/exitedNft：仅真实NFT；workingVirtual/exitedVirtual：仅虚拟卡。
function miningCardTypeStats(MiningCardType cardType)
    external view
    returns (
        uint256 workingTotal,
        uint256 workingNft,
        uint256 workingVirtual,
        uint256 exitedTotal,
        uint256 exitedNft,
        uint256 exitedVirtual
    );

// 返回指定positionId的仓位字段；字段定义见UserDashboard.Position。
function positions(uint256 positionId)
    external view
    returns (
        address account,
        uint256 nftTokenId,
        uint8 cardType,
        uint8 exitMultiple,
        uint8 accelerationTier,
        uint64 activatedAt,
        uint64 activeUntil,
        uint64 lowAccelerationAt,
        uint64 highAccelerationAt,
        uint64 exitedAt,
        uint256 accelerationMob,
        uint256 exitCapUsdt,
        uint256 exitUsedUsdt,
        uint256 previous,
        uint256 next,
        bool virtualCard,
        bool accelerationUpgraded,
        bool exited
    );

// 账户和全局只读状态。
function firstActivationAt(address account) external view returns (uint256);
function withdrawalExempt(address account) external view returns (bool);
function maturityDelay() external view returns (uint256);
function workingCardCount(address account) external view returns (uint256);
function exitedCardCount(address account) external view returns (uint256);
function accountPositionCount(address account) external view returns (uint256);
function accountPositionIds(address account, uint256 index) external view returns (uint256);
function workingFaceValueUsdt(address account) external view returns (uint256);
function workingSellLimitUsdt(address account) external view returns (uint256);
function maxActiveUntil(address account) external view returns (uint256);
function earningsLedger() external view returns (address);
function accelerationClosedAt() external view returns (uint64);

// owner设置全局成熟秒数，必须大于0，修改后立即作用于全部未消费收益。
function setMaturityDelay(uint256 newDelay) external;

// owner每批最多100个地址设置或取消成熟期/NCN资格豁免。
function setWithdrawalExemptions(address[] calldata accounts, bool exempt) external;

// owner永久关闭加速；关闭后不能重新开启或新增质押，原本金仍可赎回。
function closeAccelerationPermanently() external;

// owner每批最多100条导入旧虚拟矿机卡。
// remainingUsdt是每张卡剩余可领取USDT计价值，不是已领取金额。
function importLegacyVirtualCards(
    address[] calldata accounts,
    MiningCardType[] calldata cardTypes,
    uint256[] calldata remainingUsdt
) external;
```

---

### MiningEarningsLedger

写方法只允许MiningSystem调用。前端优先使用UserDashboard，以下方法适合审计和诊断。

```solidity
// 返回指定单卡截至当前的按秒收益分类。
// maturedUntil必须由调用方按当前maturityDelay计算；普通前端应改用Dashboard避免口径错误。
function earningsStatus(uint256 positionId, uint64 maturedUntil)
    external view returns (EarningsStatus memory status);

// 返回当前加速本金和最近一次加速周期。
// principalMob：当前未赎回本金；cycleStartedAt/cycleEndedAt：最近周期起止；
// earningUntil：该周期最多产出到何时；dailyExtraUsdt：当前日加速额；earningActive：当前是否仍产出。
function positionAccelerationStatus(uint256 positionId)
    external view
    returns (
        uint256 principalMob,
        uint64 cycleStartedAt,
        uint64 cycleEndedAt,
        uint64 earningUntil,
        uint256 dailyExtraUsdt,
        bool earningActive
    );

// 返回历史累计已消费的基础/加速USDT计价值。
function consumedTotals(uint256 positionId)
    external view
    returns (uint256 baseUsdt, uint256 accelerationUsdt);

// 返回该仓位历史收益检查点数量，仅用于诊断，不代表天数或可领取次数。
function checkpointCount(uint256 positionId) external view returns (uint256);

// 全DApp当前尚未赎回的加速MÖB本金；质押增加、赎回减少。
function totalAccelerationMobStaked() external view returns (uint256);

// 全局永久关闭加速的时间；0表示未关闭。
function accelerationClosedAt() external view returns (uint64);
```

---

### MiningEarningsSettlement

```solidity
// 返回用户矿机静态收益历史结算总额。
// grossMobSettled：历史毛MÖB；mobPaid：历史实际到账MÖB；
// immediateFeeMob：历史立即提现发送到dead的MÖB。
function miningTotals(address account)
    external view
    returns (
        uint256 grossMobSettled,
        uint256 mobPaid,
        uint256 immediateFeeMob
    );
```

---

### RewardDistributor

```solidity
// 一次领取当前全部动态MÖB和USDT。
// MÖB到账后计入TradingRouter卖出额度。
function claim() external returns (uint256 mobAmount, uint256 usdtAmount);

// 当前普通动态奖励待领取余额。
function pendingMob(address account) external view returns (uint256);
function pendingUsdt(address account) external view returns (uint256);

// 历史实际领取总额。
function cumulativeClaimedMob(address account) external view returns (uint256);
function cumulativeClaimedUsdt(address account) external view returns (uint256);

// 按RewardKind返回历史记账总额；不是当前待领。
function cumulativeRewardByKind(address account, RewardKind kind)
    external view returns (uint256);

// 历史有效直属数：直属首次激活/导入后永久增加一次。
function effectiveDirectReferralCount(address account) external view returns (uint256);

// 伞下最多15层用户历史完整铸币费支付总额。
// 包含每笔交互池部分+基金会部分，不含本人支付，不是奖励余额。
function subtreeMintFeePaidUsdt(address account) external view returns (uint256);

// 返回奖励记录总数。
function rewardRecordCount(address account) external view returns (uint256);

// 分页返回动态奖励明细；limit应不超过100。
function getRewardRecords(address account, uint256 offset, uint256 limit)
    external view returns (RewardRecord[] memory records);
```

---

### OrganizationBenefit

```solidity
struct OrganizationRewardRecord {
    address source;    // 支付铸币费并触发权益的下级地址
    uint256 amount;    // 本次记入的USDT
    uint8 cardType;    // OrganizationType：0或1
    uint64 recordedAt; // 记账时间
}

// 永久质押组织卡；NFT进入OrganizationBenefit且不提供赎回入口。
function stake(uint256 tokenId) external;

// 领取当前全部组织USDT。
function claim() external returns (uint256 usdtAmount);

// 当前待领取组织USDT。
function pendingUsdt(address account) external view returns (uint256);

// 历史记入和历史实际领取组织USDT。
function cumulativeRecordedUsdt(address account) external view returns (uint256);
function cumulativeClaimedUsdt(address account) external view returns (uint256);

// 当前该地址质押的指定组织卡数量。
function stakeCount(address account, OrganizationType cardType)
    external view returns (uint256);

// 全网当前质押的指定组织卡数量。
function totalStaked(OrganizationType cardType) external view returns (uint256);

// 分页读取组织权益明细。
function getRewardRecords(address account, uint256 offset, uint256 limit)
    external view returns (OrganizationRewardRecord[] memory records);
```

---

### NodeRewardPool

```solidity
// 永久质押节点卡；NFT进入NodeRewardPool且不提供赎回入口。
function stake(uint256 tokenId) external;

// 领取四类节点当前全部MÖB和USDT。
// MÖB实际到账会增加用户MÖB卖出额度。
function claim() external returns (uint256 mobAmount, uint256 usdtAmount);

// 返回某地址、某节点类型当前待领MÖB和USDT。
// 结果包含尚未写入userRewards但可由累计每份收益计算出的部分。
function pendingRewards(address account, NodeType nodeType)
    external view returns (uint256 mobAmount, uint256 usdtAmount);

// 当前该地址质押的指定节点NFT数量。
function stakeCount(address account, NodeType nodeType)
    external view returns (uint256);

// 全网当前质押的指定节点NFT数量。
function totalStaked(NodeType nodeType) external view returns (uint256);

// 历史实际领取节点MÖB/USDT。
function cumulativeClaimedMob(address account) external view returns (uint256);
function cumulativeClaimedUsdt(address account) external view returns (uint256);

// 当前每类基础静态MÖB全网奖励比例，基点。
function networkRewardBps(NodeType nodeType) external view returns (uint256);

// 仅NodeRewardPool.operator直接调用NodeRewardPool代理，不经过Gateway；owner不能直接设置比例。
// 允许0-1000基点（0%-10%），不转移资产并立即生效；例如600表示该类型分配6%。
// 默认基金会/创世/联创/社区分别为600/300/300/500。
function setNetworkRewardBps(NodeType nodeType, uint256 rewardBps) external;

// 仅owner直接调用NodeRewardPool代理，更换非零operator；不转移资产并立即生效。
function setOperator(address operator_) external;
function operator() external view returns (address);
```
---

### InteractionPool

```solidity
// 无权限每日添加流动性，内部检查24小时间隔和池/Pair条件。
// 成功后自动检查交易模式。
function executeDailyLiquidity()
    external
    returns (uint256 usdtAmount, uint256 mobAmount, uint256 liquidity);

// 无权限Pair模式每日回购销毁，内部检查24小时间隔。
// 成功后自动检查交易模式。
function executeDailyBuyback() external;

// 当前累计业务USDT流入和MÖB卖出USDT流出，均为历史累计值。
function cumulativeUsdtInflow() external view returns (uint256);
function cumulativeSaleUsdtOutflow() external view returns (uint256);

// 三个当前operator配置，基点可大于10000。
// poolToPairRatioBps：进入Pair模式的交互池/Pair余额比；
// poolRecoveryRatioBps：返回交互池模式的余额恢复比；
// pairOutflowRecoveryMultipleBps：流量触发后Pair流出恢复倍数。
function poolToPairRatioBps() external view returns (uint256);
function poolRecoveryRatioBps() external view returns (uint256);
function pairOutflowRecoveryMultipleBps() external view returns (uint256);

// 以下倍数设置必须由InteractionPool.operator直接调用InteractionPool代理，不经过Gateway；
// owner不能直接代替operator设置。设置不转移资产并立即作用于当前模式判断。
// 三个值都必须大于0、无额外上限，默认15000/30000/60000；例如15000表示1.5倍。
function setSwitchMultipliers(
    uint256 poolToPairRatioBps_,
    uint256 poolRecoveryRatioBps_,
    uint256 pairOutflowRecoveryMultipleBps_
) external;

// 仅修改一个明确参数，避免顺序混淆；权限、范围和生效规则与组合方法相同。
function setPoolToPairExitRatioBps(uint256 ratioBps) external;
function setPoolBalanceRecoveryRatioBps(uint256 ratioBps) external;
function setPairOutflowRecoveryMultipleBps(uint256 multipleBps) external;

// 仅operator直接调用InteractionPool代理，可把任意ERC20（包括MÖB、USDT、LP）提到非零地址。
// 不能提取原生币；amount必须大于0。提走已承诺资产可能导致后续用户支付回滚。
function withdrawToken(address token, address recipient, uint256 amount) external;

// 仅owner直接调用InteractionPool代理，更换非零operator；不转移资产并立即生效。
function setOperator(address operator_) external;
function operator() external view returns (address);

// owner维护业务支付白名单、Router和Pair。
function setBusinessAuthorization(address business, bool authorized) external;
function setUniswapRouter(address router) external;
function setUniswapPair(address pair) external;
```

---

### ReferralRegistry

前端、Gateway、其他业务合约以及邀请关系导入/验收脚本统一使用部署日志中的 `REFERRAL_REGISTRY` 代理地址；`REFERRAL_REGISTRY_IMPLEMENTATION` 只用于源码验证和升级审计，不能作为业务调用地址。邀请关系存储在代理中，升级不会更换前端地址。首码写在实现合约的 immutable 参数中，因此新实现必须使用与首次部署完全相同的 `ROOT_CODE`，否则升级会回滚。

```solidity
// 用户永久绑定邀请人；同一地址只能登记一次。
function register(address inviter) external;

// 是否已经登记及永久邀请人。
function isRegistered(address account) external view returns (bool);
function inviterOf(address account) external view returns (address);

// 永久绑定直属数，未要求直属激活。
function directReferralCount(address account) external view returns (uint256);

// 按索引读取一个直属地址。
function directReferralAt(address account, uint256 index) external view returns (address);

// 分页读取直属地址；limit由调用方控制，前端应使用Dashboard的1-100校验入口。
function getDirectReferrals(address account, uint256 offset, uint256 limit)
    external view returns (address[] memory accounts);

// 返回最多depth层上级；业务规则通常使用depth=15。
function getUplines(address account, uint256 depth)
    external view returns (address[] memory uplines);

// 不可修改的首码地址。
function rootCode() external view returns (address);

// 当前已登记地址总数，包含首码本身；这是历史累计总数，不是活跃用户数。
function registeredUserCount() external view returns (uint256);

// 单批旧邀请关系导入上限，固定为100，超出必须拆分交易。
function MAX_BATCH_SIZE() external view returns (uint256);

// owner每批最多100条导入旧邀请关系。
function importReferralRelations(
    address[] calldata accounts,
    address[] calldata inviters
) external;

// Gateway专用登记入口：Gateway传入真实用户地址，普通地址直接调用会回滚。
function registerFor(address account, address inviter) external;

// 当前一次性接线的Gateway和MiningSystem代理地址；零地址表示尚未完成对应接线。
function gateway() external view returns (address);
function miningSystem() external view returns (address);

// 仅owner在部署组装阶段调用一次；目标必须是已部署合约，配置后不能再次修改。
function configureGateway(address gateway_) external;
function configureMiningSystem(address miningSystem_) external;

// 当前代理owner和统一可信Gateway。ReferralRegistry业务登记使用gateway()，trustedGateway()
// 来自公共UUPS底座，供统一代理状态和运维审计；普通前端只读即可，不应修改。
function owner() external view returns (address);
function trustedGateway() external view returns (address);

// 仅当前owner可执行UUPS升级，代理地址和邀请关系状态保持不变；新实现rootCode必须与当前首码一致。
function upgradeToAndCall(address newImplementation, bytes calldata data) external payable;

// 仅当前owner更换非零owner；立即影响邀请导入、一次性接线和后续升级权限，不转移资产。
function transferOwnership(address newOwner) external;

// 代理部署时由ERC1967Proxy构造交易原子调用；部署后已经初始化，任何人都不应再次调用。
// 再次调用会以ProxyAlreadyInitialized回滚，不会重置owner、Gateway或首码登记。
function initializeProxy(address initialOwner, address trustedGateway_) external;

// UUPS/ERC-1822运维识别接口。UPGRADE_INTERFACE_VERSION返回"5.0.0"；
// proxiableUUID只能对实现地址做兼容性检查，通过代理调用会按UUPS保护规则回滚。
function UPGRADE_INTERFACE_VERSION() external view returns (string memory);
function proxiableUUID() external view returns (bytes32);
```

---

### TeamPerformance

```solidity
// 主动刷新地址当前团队等级，返回旧等级和新等级。
// 普通用户通过Gateway.refreshMyTeamLevel()刷新自己；不是keeper任务。
function refreshLevel(address account)
    external returns (uint8 oldLevel, uint8 newLevel);

// 当前V等级，0表示无等级，1-7表示V1-V7。
function teamLevel(address account) external view returns (uint8);

// 当前团队总业绩、当前小区业绩。
function totalPerformance(address account) external view returns (uint256);
function smallAreaPerformance(address account) external view returns (uint256);

// 当前最大分区地址和业绩。
function maximumBranch(address account)
    external view returns (address branch, uint256 performance);

// 指定直属分区当前业绩。
function branchPerformance(address account, address directReferral)
    external view returns (uint256);

// owner设置V2-V7持币要求，level必须为2-7，amountUsdt为USDT计价值。
function setHoldingRequirement(uint8 level, uint256 amountUsdt) external;
function holdingRequirementUsdt(uint8 level) external view returns (uint256);
```

---

### TeamEligibilityManager

```solidity
// 无权限同步最多100条已到期V3/V4资格，返回实际处理数。
// 若isEligibilitySynchronized仍为false，可立即继续调用。
function syncExpiredEligibility() external returns (uint256 processed);

// 当前队列是否已经同步到当前时间；MÖB卖出前必须为true。
function isEligibilitySynchronized() external view returns (bool);

// 下一条有效资格到期时间；没有时按实现返回0。
function nextEligibilityExpiry() external view returns (uint256);

// 返回地址当前等级资格记录。
function eligibilityOf(address account)
    external view
    returns (
        uint8 grade,
        uint64 validUntil,
        uint64 version,
        bool counted
    );
```

---

### PriceOracle

所有报价都使用当前Uniswap V2 Pair储备，属于即时现货预览，不包含TradingRouter的MÖB卖出手续费。

```solidity
// MÖB数量换算为USDT计价值。
function quoteMobToUsdt(uint256 mobAmount) external view returns (uint256 usdtAmount);

// USDT计价值换算为MÖB数量。
function quoteUsdtToMob(uint256 usdtAmount) external view returns (uint256 mobAmount);

// NCN数量换算为USDT计价值。
function quoteNcnToUsdt(uint256 ncnAmount) external view returns (uint256 usdtAmount);

// USDT计价值换算为NCN数量。
function quoteUsdtToNcn(uint256 usdtAmount) external view returns (uint256 ncnAmount);

// owner设置经过代币校验的Pair地址。
function setMobPair(address pair) external;
function setNcnPair(address pair) external;
```

---

### Mob

```solidity
// ERC20标准方法。
function balanceOf(address account) external view returns (uint256);
function allowance(address owner, address spender) external view returns (uint256);
function approve(address spender, uint256 amount) external returns (bool);
function transfer(address recipient, uint256 amount) external returns (bool);
function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
function decimals() external view returns (uint8);

// 无权限每日销毁Pair当前MÖB余额的1%，内部检查24小时间隔。
function burnPairBalance() external;

// TradingRouter专用：成功卖出时销毁对应Pair余额；普通用户不能调用。
function burnSoldPairBalance(uint256 soldAmount) external;

// owner管理白名单、Pair和卖出销毁调用者。
function setAllowlisted(address account, bool allowed) external;
function setUniswapPair(address pair) external;
function setSaleBurner(address burner) external;

// 固定初始接收数量为1亿MÖB，原始单位。
function INITIAL_RECIPIENT_AMOUNT() external view returns (uint256);
```

---

### Ncn / IERC20 USDT

```solidity
function balanceOf(address account) external view returns (uint256);
function allowance(address owner, address spender) external view returns (uint256);
function approve(address spender, uint256 amount) external returns (bool);
function transfer(address recipient, uint256 amount) external returns (bool);
function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
function decimals() external view returns (uint8);
```

---

### MiningCard / OrganizationCard / NodeCard

三个NFT都实现ERC721标准ABI。

```solidity
function ownerOf(uint256 tokenId) external view returns (address);
function balanceOf(address owner) external view returns (uint256);
function tokenCardTypes(uint256 tokenId) external view returns (uint8);
function tokenURI(uint256 tokenId) external view returns (string memory);
function approve(address operator, uint256 tokenId) external;
function setApprovalForAll(address operator, bool approved) external;
function getApproved(uint256 tokenId) external view returns (address);
function isApprovedForAll(address owner, address operator) external view returns (bool);
function transferFrom(address from, address to, uint256 tokenId) external;
function safeTransferFrom(address from, address to, uint256 tokenId) external;
```

矿机卡额外规则：普通钱包转移时，接收人必须在发送人伞下最多15层内；MiningSystem激活转入是唯一系统放行路径。组织卡和节点卡不受该邀请链转移限制，但质押后永久锁定。

## 5. 前端调用顺序建议

1. 用CardManager预览购买金额，再分别授权CardManager并通过Gateway购买。
2. 绑定邀请关系后，授权MiningCard和USDT给MiningSystem，再激活卡牌。
3. 卡牌页面读取Dashboard的 `getWorkingCards` 和每张卡的 `getPositionEarnings`；静态收益不再存在窗口编号或窗口参数。
4. 正常提现只调用 `withdrawNormal(positionId)`；立即提现只调用 `withdrawImmediate(positionId)`。
5. 合成只传 `positionId/cardType/maxWalletMob`，不再存在窗口编号或窗口上限。
6. MÖB卖出前同时读取 `previewSellMob` 和 `getNcnSellQualification`；预览中的三个资格布尔值都应为true。
7. 动态、组织、节点的“当前待领”和“历史累计”必须分开展示，不能相加后标为余额。
8. `subtreeMintFeePaidUsdt` 是伞下历史完整支付总额，不是交互池流入额、本人支付额或可领取奖励。
9. owner/operator配置应放在独立管理页面，并在发送交易前校验当前调用钱包等于对应合约 `owner()` 或 `operator()`。
