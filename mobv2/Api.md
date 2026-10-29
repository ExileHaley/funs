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

下列地址是加入 ReferralRegistry 与 TradingModeController 独立UUPS代理之前的历史部署记录，不能用于当前源码。重新全量部署后，所有业务调用地址必须以新日志为准，并完整记录两个新增代理及implementation；正式交付前应整体替换本节，不能局部沿用历史地址。

```
  MOB_V2_GATEWAY_PROXY 0x5f49263A569d69Dc04Fafee69fDE27748339F323
  INTERACTION_POOL 0x2c9FacC971A2808B6eDe71c149b1bA4eC8809d87
  MOB 0x9a04237A8f43Ad18483dEB06d37E32421FE0a053
  MOB_PAIR 0xE465483BBf7197F86CD4838852283424da734F85
  NCN 0xa25011c3F82332CcCDEda813E049699AF0ab9818
  PRICE_ORACLE 0x5b254B7d7FA7540F901705a34F4B8721a9c2586D
  REFERRAL_REGISTRY 0xb6DD045c33f5eB2D8C51bfD0Cd82C2214045EE27
  CARD_MANAGER 0x15E7ED2AFd868Cd74cF9AFa215C21878Fd69a598
  MINING_EARNINGS_SETTLEMENT 0x65396e8cDe4686a8fD4F16E7426F391D8c9dB6F2
  MINING_EARNINGS_LEDGER 0x98fF85E2cacEe7Bca80bDE8F9BC3455dfB3adD04
  MINING_SYSTEM 0xEb2FF59EDdA81bd3AC748863dcb4202CEa2A045e
  CARD_COMPOSER 0xf526aa853Ba6B8008F50e8Cd3EC5a2F7987B21B3
  TRADING_ROUTER 0x6482D7F8f0499E512E12E84dfd4e718Fb0Faa73F
  TRADING_MODE_CONTROLLER 0xD464038d543d17B99c6c7C42676bc700275a1488
  USER_DASHBOARD 0xc8F816564b3E689B0EC590838122Dd13E2002559
  REWARD_DISTRIBUTOR 0x687D20F2c7cD3C50C05aa390Db6F3D379F288Dfa
  TEAM_PERFORMANCE 0x088acA843E6B6C5f394864b54ECeaB0FCc7950E5
  TEAM_ELIGIBILITY 0x5278A714d56d1674b62dB0bEc650404d94A2a982
  ORGANIZATION_BENEFIT 0x805CFE426381BCFfed83415C0c064A99Df9E6cAD
  NODE_REWARD_POOL 0xAa327E9a66e6FF0aAaeeef9867E0E96CF861F4f1
  MINING_CARD 0xC05dA3396aaB03548B497018E9B8C11a693d8231
  ORGANIZATION_CARD 0x40701dcd42d07443d7eAFEf00aB27D38f5844ECF
  NODE_CARD 0x08573C7F23af5CC92CA9B5BE845e399D106D8026
  MOB_V2_GATEWAY_IMPLEMENTATION 0xdB8D27Cd7E7a66F34fDa4D6D367DfEd6CFc5B152
  INTERACTION_POOL_IMPLEMENTATION 0xEb552F11B2A4Adc754897f51bbeA879C42acdF31
  CARD_MANAGER_IMPLEMENTATION 0x51aFDA34535EFbe290942CBb049113519Deba267
  ORGANIZATION_BENEFIT_IMPLEMENTATION 0x47CD36eEe13352c13A08bb3cFc7EFB4A42541415
  NODE_REWARD_POOL_IMPLEMENTATION 0x96E24B2f09BE9Ec8A8FFf00679f4c22261AB8B7B
  REWARD_DISTRIBUTOR_IMPLEMENTATION 0x6bD73f7FE8EeA5a66E8667bADd1318a7e0e6a02A
  TEAM_PERFORMANCE_IMPLEMENTATION 0x594fcc72233E84B76c1a21a7A465400A24827855
  TEAM_ELIGIBILITY_IMPLEMENTATION 0xe3714621F24FB7823f9d96FbDfD518B199210AE5
  MINING_EARNINGS_SETTLEMENT_IMPLEMENTATION 0x0cB923Ff3e1bcD27b7E120E66c4422564A677a2A
  MINING_EARNINGS_LEDGER_IMPLEMENTATION 0x56f93027F1E4a834841eDBcD5AF66D42389D0601
  MINING_SYSTEM_IMPLEMENTATION 0xAb2ac3B3F709E741d3e7ABF43Bf9f50AAc25208A
  CARD_COMPOSER_IMPLEMENTATION 0xf483D673Ad9a7eaACDc2CC45aE5c58162c95968e
  TRADING_ROUTER_IMPLEMENTATION 0xfE4a2d497DCE1e59fBc6d53d210C979dc95883ca
  USER_DASHBOARD_IMPLEMENTATION 0xe47a18A88f592C3d4D95723aDe79F3F4c77996b9
  REFERRAL_REGISTRY_IMPLEMENTATION 0x8C928539b412e9E1944E5768192D9A4a07c4fC9D
  TRADING_MODE_CONTROLLER_IMPLEMENTATION 0xc6E4556C85847F6EB5215B34367f556bFc962d55
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
// 豁免地址的owner导入虚拟卡仍需调用，但支付0 USDT且不记录铸币费相关奖励；真实NFT卡不免费。
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
// 提现豁免地址跳过NCN资格且手续费最高20%，但不跳过两类卖出限额。
function sellMob(uint256 mobAmount) external returns (uint256 receivedUsdt);

// 主动按当前业绩和持币条件刷新调用者团队等级；不是keeper方法。
function refreshMyTeamLevel() external returns (uint8 oldLevel, uint8 newLevel);

// 无权限补充检查交易模式。没有自动检查交易发生时可由任意地址调用。
function checkAndSwitchMode() external;

// 无权限每日加LP；合约内检查独立24小时间隔和固定2倍池/Pair条件。
// 返回使用的USDT、MÖB和获得的LP数量。
function executeDailyLiquidity()
    external
    returns (uint256 usdtAmount, uint256 mobAmount, uint256 liquidity);

// 无权限每日回购销毁；Pair和交互池模式均可执行，合约内检查独立24小时间隔和固定2倍池/Pair条件。
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
    uint256 workingMobSellLimitUsdt;        // 已激活或曾激活最高面值卡牌对应的一笔MÖB卖出毛USDT上限，不随出局降低
}

struct DynamicRewardTotals {
    uint256 directMob;    // 历史记账直推MÖB，不是当前待领
    uint256 levelMob;     // 历史记账层级MÖB，不是当前待领
    uint256 teamMob;      // 历史记账团队MÖB，不是当前待领
    uint256 teamUsdt;     // 历史记账铸币费团队USDT，不是当前待领
    uint256 gradeSaleUsdt;// 兼容字段；V3/V4卖出手续费不再写入动态收益记录，当前返回0
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
    TradingModeStatus status;                    // TradingModeController原始模式状态；字段定义见TradingModeController章节
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
    bool usesPoolBalanceRecoveryRatio;           // 当前切入原因是否使用余额恢复倍数；仅PoolRatio原因的Pair模式为true
    bool usesPairOutflowRecoveryMultiple;        // 当前切入原因是否使用Pair流出倍数；仅流量原因Pair模式为true
    bool usesPriceRecoveryRatio;                 // 当前切入原因是否使用价格恢复比例；仅流量原因Pair模式为true
    bool usesPriceDropRecoveryRatio;             // 当前切入原因是否使用价格下跌比例；仅流量原因Pair模式为true
}

// 返回用户聚合总览。字段中历史累计、当前待领和当前余额必须按注释分别展示。
function getUserOverview(address account)
    external view returns (UserOverview memory overview);

// 查询该地址是否满足MÖB卖出前的累计NCN交易考核，以及当前还差多少USDT交易额。
// 考核从首张真实矿机卡激活或首张虚拟卡导入时起算，按累计天数乘operator设置的每日X计算累计要求；
// 天数包含首日，卡牌出局期间仍继续计时；修改X会追溯影响全部累计天数，历史买卖记录不清零。
// 买入和卖出NCN都计入累计交易额，但买入必须严格大于累计要求的一半；卖出MÖB不会消费NCN交易记录。
// qualified=true表示可通过NCN资格项，但实际卖MÖB仍需同时满足单笔工作卡卖出限额和剩余MÖB卖出额度。
// missingTotalUsdt是总交易额缺口，missingBuyUsdt是严格超过总要求50%的买入缺口；补足一个缺口不代表另一个也已满足。
// dailyRequirementUsdt=0时关闭NCN资格检查；exempt=true的豁免地址也直接通过，两个缺口均返回0。
// 所有金额均为USDT最小单位；这是只读实时状态，不会锁定或消费额度。字段完整定义见TradingRouter章节。
function getNcnSellQualification(address account)
    external view returns (NcnSellQualificationStatus memory status);

// 一次返回当前模式为何切换、切换基数、七项operator配置、所有回切目标和当前差额。
// activeReason=PoolRatio时只看usesPoolBalanceRecoveryRatio对应的余额恢复路径。
// activeReason=DailyOutflow或SevenDayNetOutflow时，仅Pair流出倍数、价格恢复、价格下跌三个uses字段为true，三条路径任一满足即可回切；3倍余额路径不适用。
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

// 查询指定矿机仓位positionId的基础卡牌收益及MÖB质押加速静态收益；每次只查一张卡。
// status内的收益账本以USDT计价并按秒累计：produced为历史累计产出，consumed为历史已提现/合成消费；
// availableMatured/availableUnmatured是完整账本未消费值，不按剩余出局额度裁剪。
// 下方五个MÖB操作预览与真实执行一致，按exitCapUsdt-exitUsedUsdt剩余额度裁剪；已出局时全部为0。
// maturedWithdrawalMob是成熟基础+加速收益按当前Pair价格折算的正常提现毛MÖB；正常提现不收20%费用。
// immediateGrossMob仅包含未成熟的基础+加速收益折算毛MÖB；immediateFeeMob为其20%费用，
// immediateNetMob=gross-fee，是按当前价格估算的立即提现到账MÖB。立即提现不会把已成熟部分重复计入。
// compositionMob是该仓位当前全部未消费基础+加速收益折算的MÖB总值，包含成熟和未成熟部分，
// 仅供合成预览，不等于可提现金额；实际合成还受目标卡面值和钱包补差规则影响。
// 该接口只读，不会锁价或消费账本；返回值是当前估算，实际交易会按结算时价格计算。
function getPositionEarnings(uint256 positionId)
    external view returns (
        EarningsStatus memory status,
        uint256 maturedWithdrawalMob,
        uint256 immediateGrossMob,
        uint256 immediateFeeMob,
        uint256 immediateNetMob,
        uint256 compositionMob
    );

// 返回动态奖励的历史记账总额，不是当前待领。
// V3/V4卖出手续费不再写入动态收益记录，gradeSaleUsdt保持为0。
function getDynamicRewardTotals(address account)
    external view returns (DynamicRewardTotals memory totals);

// 返回当前地址可领取的V3/V4卖出手续费USDT合计。
// 地址同一时刻只能处于一个有效等级，但降级、到期或V3/V4切换不会丢失已结算待领金额。
// 返回值=过往等级已结算未领取+当前V3/V4累计索引未结算金额；claim成功后清零。
// 从未产生且当前也不是V3/V4的地址返回0；该方法不修改链上状态。
function getGradeSalePendingUsdt(address account)
    external view returns (uint256 amount);

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

// 分页读取OrganizationBenefit现有组织卡USDT收益明细，不新增或修改收益账本。
// 每条记录包含支付铸币费的下级地址、记入的USDT数量、组织卡类型和记账时间；字段见OrganizationBenefit章节。
// limit必须为1-100；nextOffset等于记录总数时表示已经读取完毕。
function getOrganizationRewardRecords(address account, uint256 offset, uint256 limit)
    external view returns (OrganizationRewardRecord[] memory records, uint256 nextOffset);
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

// 查询四类节点卡的累计发行量。购买、operator单张铸造和operator批量铸造均计入同一计数；
// minted/cap/remaining分别是已发行张数、该类型固定发行上限和还可发行张数。
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
    bool exempt;                      // 是否为提现豁免地址；同时跳过NCN资格并将MÖB卖出手续费封顶为20%
}

struct MobSellPreview {
    bool pairMode;                  // true使用Uniswap Pair，false由交互池报价/支付
    uint256 grossUsdt;              // 扣手续费前预计毛USDT
    uint256 feeUsdt;                // 按当前所有条件预计扣除的USDT手续费；提现豁免地址最高为毛额20%
    uint256 receivedUsdt;           // 预计用户实收USDT=grossUsdt-feeUsdt
    uint256 workingSellLimitUsdt;   // 已激活或曾激活最高面值卡牌对应的一笔毛USDT卖出上限，不随出局降低
    uint256 cumulativeMobReceived;  // 历史实际领取并计入卖出额度的MÖB总量，不含转入或购买所得
    uint256 cumulativeMobSold;      // 历史通过TradingRouter成功卖出的MÖB总量
    uint256 remainingMobAllowance;  // 当前奖励来源剩余可卖MÖB数量
    bool passesWorkingSellLimit;    // 当前预览毛USDT是否不超过工作卡限额
    bool passesMobAllowance;        // 本次mobAmount是否不超过剩余MÖB额度
    bool ncnQualified;              // 当前NCN累计资格是否满足
}
```

### TradingModeController

该合约是独立UUPS代理，保存全部交易模式状态。前端、operator、keeper、Gateway、InteractionPool和TradingRouter都必须连接代理地址，不能连接implementation。

```solidity

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
    bool poolRatioRecoveryMet;               // 当前是否满足3倍余额条件；只有activeReason=PoolRatio时才是有效回切依据
    bool canSwitch;                          // 现在调用checkAndSwitchMode是否会切换
    uint8 prospectiveReason;                 // 非Pair模式下若切换将采用的SwitchReason
}

// 返回模式切换所需的全部当前数据和判断结果。
function getTradingModeStatus()
    external view returns (TradingModeStatus memory status);

// 返回当前是否Pair模式。
function isPairMode() external view returns (bool);

// 无权限按当前状态检查并切换模式；可由keeper直接调用，也可经Gateway调用。
function checkAndSwitchMode() external;

// 以下两个参数方法必须直接调用TradingModeController代理，不经过Gateway；不转移资产且立即生效。
// owner只能通过setOperator更换非零operator，不能直接代替operator设置参数。
// 四个参数范围均为0-10000基点，0是有效阈值；默认分别为700/2700/10000/3000。
function setFlowSwitchThresholds(
    uint256 dailyOutflowBps_,
    uint256 sevenDayNetOutflowBps_
) external;

function setPriceRecoveryThresholds(
    uint256 priceRecoveryBps_,
    uint256 priceDropRecoveryBps_
) external;

// 仅owner更换控制器operator。
function setOperator(address operator_) external;

// 关键模式状态与配置读取。
function operator() external view returns (address);
function tradingRouter() external view returns (address); // 唯一可提交卖出流量的TradingRouter代理
function switchedAt() external view returns (uint64);
function switchPrice() external view returns (uint256);
function switchTriggerUsdt() external view returns (uint256);
function pairOutflowSinceSwitch() external view returns (uint256);
function dailyOutflowBps() external view returns (uint256);
function sevenDayNetOutflowBps() external view returns (uint256);
function priceRecoveryBps() external view returns (uint256);
function priceDropRecoveryBps() external view returns (uint256);
```

`prepareSale()` 与 `recordPairSaleAndCheck(uint256)` 是TradingRouter专用写入口，其他地址调用会回滚；前端和keeper不得调用。前者只在卖出路径确定前刷新到期窗口，后者在整笔Pair卖出完成后记录真实毛USDT流出并检查切换。

### TradingRouter

```solidity

// 买入NCN，返回实际收到的NCN；累计买入USDT增加。
function buyNcn(uint256 usdtAmount) external returns (uint256 ncnReceived);

// 卖出NCN，返回实际收到的USDT；累计卖出USDT增加。
function sellNcn(uint256 ncnAmount) external returns (uint256 usdtReceived);

// 卖出MÖB，返回实际收到USDT。
// 所有资格、手续费、销毁、奖励和模式检查在同一交易内原子执行。
function sellMob(uint256 mobAmount) external returns (uint256 receivedUsdt);

// 完整预览MÖB卖出。覆盖以最近一次Pair切换价格为基准、回切交互池后仍连续适用的20%-70%手续费、豁免地址20%封顶、保护额度分段和三项前置资格。
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

// 设置每地址每日NCN累计考核额，范围0-50U；0关闭资格检查。
// 仅operator调用；默认10U。参数使用USDT原始最小单位，修改后立即追溯全部历史考核天数。
function setDailyNcnRequirement(uint256 amountUsdt) external;

// 仅operator设置新用户固定20%手续费保护期，参数单位为整数小时，最小非零值为1小时，默认1680（70天）；0表示关闭。
// 修改后按每个非旧系统用户原始firstActivationAt立即追溯生效；旧系统虚拟卡用户不适用。
function setNewUserProtectionHours(uint256 protectionHours) external;

// 仅owner直接调用TradingRouter代理，更换非零operator；不转移资产并立即生效。
function setOperator(address operator_) external;

// owner设置Uniswap Router、两个Pair和手续费钱包。
function setUniswapRouter(address router) external;
function setPairs(address mobPair_, address ncnPair_) external;
function setFeeWallets(address foundationWallet_, address insuranceWallet_) external;

// 关键配置和历史累计读取。
function operator() external view returns (address);
function dailyNcnRequirementUsdt() external view returns (uint256);
function newUserProtectionPeriod() external view returns (uint256); // 当前保护期秒数，默认70 days
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

// 直接续缴指定仓位铸币费。豁免地址的owner导入虚拟卡支付0 USDT并增加30天；真实NFT卡正常收费。
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

// 查询五类矿机卡中指定类型的当前工作/质押中数量和累计出局数量；这是全局O(1)计数，不遍历卡牌。
// workingTotal/exitedTotal：包含真实NFT和导入的虚拟卡；
// workingNft/exitedNft：仅真实NFT；workingVirtual/exitedVirtual：仅虚拟卡。
// 用户确认的5张特殊U3000虚拟卡永久不计入上述六个展示字段，其他业务记录不变。
// 注意：这是MiningSystem直接查询，不经过UserDashboard；工作中即当前已激活且未出局。
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

// 查询指定仓位是否被专用导入入口永久排除在类型展示统计之外。
function cardTypeStatsExcluded(uint256 positionId) external view returns (bool);

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
// 返回已激活或曾激活最高面值卡牌对应的单笔毛USDT卖出上限；多卡不累加，出局后仍保留。
function maxActivatedSellLimitUsdt(address account) external view returns (uint256);
function maxActiveUntil(address account) external view returns (uint256);
function earningsLedger() external view returns (address);
function accelerationClosedAt() external view returns (uint64);

// owner设置全局成熟秒数，必须大于0，修改后立即作用于全部未消费收益。
function setMaturityDelay(uint256 newDelay) external;

// owner每批最多100个地址设置或取消成熟期/NCN资格豁免、MÖB卖出手续费20%封顶，
// 以及owner导入虚拟卡的0 USDT续费资格。
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

// 仅owner依据用户确认的特殊批次调用；卡牌业务完整导入，但永久不计入miningCardTypeStats。
// 生产脚本只对data/legacy-cards/batch-031.json调用此方法，普通前端不得调用。
function importLegacyVirtualCardsExcludedFromTypeStats(
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

// 使用与consumeRange完全相同的最旧优先和已消费区间跳过规则，预览时间范围内最多maximumUsdt的消费结果。
// 只读且不写入消费区间；UserDashboard用它对齐MiningSystem剩余出局额度裁剪。
function previewConsumeRange(
    uint256 positionId,
    uint64 producedFrom,
    uint64 producedUntil,
    uint256 maximumUsdt
) external view returns (uint256 baseUsdt, uint256 accelerationUsdt);

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

// 当前合并动态奖励待领余额；pendingUsdt包含已结算V3/V4手续费。
function pendingMob(address account) external view returns (uint256);
function pendingUsdt(address account) external view returns (uint256);

// pendingUsdt中属于V3/V4卖出手续费的已结算未领取子集。
// 降级、到期或V3/V4切换后仍保留；claim领取合并余额成功后同步清零。
function pendingGradeSaleUsdt(address account) external view returns (uint256);

// 历史实际领取总额。
function cumulativeClaimedMob(address account) external view returns (uint256);
function cumulativeClaimedUsdt(address account) external view returns (uint256);

// 按RewardKind返回历史记账总额；不是当前待领。
function cumulativeRewardByKind(address account, RewardKind kind)
    external view returns (uint256);

// 层级奖励钱包MÖB最低持有价值，单位为整U；当前固定为30，按Pair即时储备比估值。
function MIN_HOLDING_USDT() external view returns (uint256);

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
// 无权限每日添加流动性，内部检查独立24小时间隔和固定2倍池/Pair条件。
// 成功后自动检查交易模式。
function executeDailyLiquidity()
    external
    returns (uint256 usdtAmount, uint256 mobAmount, uint256 liquidity);

// 无权限每日回购销毁，Pair和交互池模式均可执行，内部检查独立24小时间隔和固定2倍池/Pair条件。
// 成功后自动检查交易模式。
function executeDailyBuyback() external;

// 当前累计业务USDT流入和MÖB卖出USDT流出，均为历史累计值。
function cumulativeUsdtInflow() external view returns (uint256);
function cumulativeSaleUsdtOutflow() external view returns (uint256);

// 三个当前operator配置，基点可大于10000。
// poolToPairRatioBps：进入Pair模式的交互池/Pair余额比；
// poolRecoveryRatioBps：返回1.5倍余额原因切出后的交互池余额恢复比，不用于单日或七日流量原因；
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
function tradingMode() external view returns (address);   // TradingModeController代理
function tradingRouter() external view returns (address); // 完整Pair卖出结束后统一提交模式检查的TradingRouter代理

// owner维护业务支付白名单、Router和Pair。
function setBusinessAuthorization(address business, bool authorized) external;
function setUniswapRouter(address router) external;
function setUniswapPair(address pair) external;

// 仅首次组装由owner调用一次；五个地址依次为MÖB、PriceOracle、TradingModeController代理、
// TradingRouter代理和MÖB/USDT Pair。配置后不可重复执行。
function configureDependencies(
    address mob_,
    address oracle_,
    address tradingMode_,
    address tradingRouter_,
    address pair_
) external;
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

// 仅TeamPerformance.operator设置V2-V7持币要求；level必须为2-7，amountUsdt为USDT最小单位计价值。
// 默认均为0，0表示关闭对应等级持币门槛；修改后在下一次业绩变化或refreshLevel时按当前Pair储备比生效。
function setHoldingRequirement(uint8 level, uint256 amountUsdt) external;
function holdingRequirementUsdt(uint8 level) external view returns (uint256);

// 仅owner直接调用TeamPerformance代理，更换非零operator；owner不能直接设置持币门槛。
function setOperator(address operator_) external;
function operator() external view returns (address);
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

`quoteMobToUsdt`/`quoteUsdtToMob`使用Uniswap V2标准输出公式，包含0.3%手续费和输入数量造成的价格影响，供真实swap及MÖB卖出路径使用。
`quoteMobToUsdtSpot`/`quoteUsdtToMobSpot`只按当前Pair储备比换算，不包含手续费和价格冲击，供内部估值、非真实swap兑换及添加流动性配比使用。

```solidity
// MÖB数量换算为USDT计价值。
function quoteMobToUsdt(uint256 mobAmount) external view returns (uint256 usdtAmount);

// USDT计价值换算为MÖB数量。
function quoteUsdtToMob(uint256 usdtAmount) external view returns (uint256 mobAmount);

// 仅按当前MÖB/USDT Pair储备比计算MÖB对应USDT，不包含0.3%手续费和价格冲击。
function quoteMobToUsdtSpot(uint256 mobAmount) external view returns (uint256 usdtAmount);

// 仅按当前MÖB/USDT Pair储备比计算USDT对应MÖB，不包含0.3%手续费和价格冲击。
function quoteUsdtToMobSpot(uint256 usdtAmount) external view returns (uint256 mobAmount);

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

// 无权限销毁Pair当前MÖB余额的1%。周期从部署时间起按固定86400秒倍数推进；
// 错过多个周期时单次仍只销毁一次1%，不会补销毁。
function burnPairBalance() external;

// 最近已经消费的固定24小时周期边界，以及固定周期秒数86400。
// 下一次允许执行时间为lastPairBurnAt + PAIR_BURN_INTERVAL。
function lastPairBurnAt() external view returns (uint256);
function PAIR_BURN_INTERVAL() external view returns (uint256);

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

三类卡牌的转账规则一致：铸造到用户钱包不检查邀请链；普通钱包之间转移时，接收人必须在发送人伞下最多15层内，禁止转给上级、旁系或伞下之外地址。MiningCard进入MiningSystem、OrganizationCard进入OrganizationBenefit、NodeCard进入NodeRewardPool分别是唯一的系统质押/激活放行路径；三类卡牌进入对应系统后均永久锁定。

// 以下配置方法仅由部署owner调用一次，配置后不可重复配置。
function configureTransferRules(address referrals, address miningSystem) external;
function configureTransferRules(address referrals, address organizationBenefit) external;
function configureTransferRules(address referrals, address nodeRewardPool) external;

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

## 6. 备注
1. 读一下INTERACTION_POOL合约里面的usdt数量展示到管理端
2. 读一下mob/usdt pair池子中usdt的数量展示到管理端
