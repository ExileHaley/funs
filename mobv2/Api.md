# MÖB V2 前端合约 API
## 合约地址
```
  MOB_V2_GATEWAY_PROXY 0x8Fd5fcA1D9C769570247Af9E78603381505a279e
  INTERACTION_POOL 0x46fA61F5D84D11196b17647c1CAEC1d0719c2122
  MOB 0x552b28379A920fa53cB6D618316C62ea9d5cD990
  NCN 0x920C05487e03677fed6B88859b2c014aaa6B2EBD
  PRICE_ORACLE 0xE26bf82Fe51CEFDE96C19cD58a33B2f1B870b01F
  REFERRAL_REGISTRY 0x18fbE770caC45Fb1E614BB81fa3755d5548E5861
  CARD_MANAGER 0x4e2DA80C542dF792C44D8e31A887E48b0Fa1D296
  MINING_SYSTEM 0x580C3BcAa08241472912f459D8bec35462B79bA7
  CARD_COMPOSER 0x2a1B953b11bd07898C9b6B9558D410E92d6fA9c7
  TRADING_ROUTER 0x8607455DD3299cAF6EF054f39f1F892CEF676F69
  USER_DASHBOARD 0x735b4FA663a41d33E7653B4a353C50CA5E4B1aCb
  REWARD_DISTRIBUTOR 0x0aB8738b2508cF4456aDeE2763dACA6469e0884D
  TEAM_PERFORMANCE 0x4Ef1FF0ef62F17C812508A286BF1F5B647e860f2
  TEAM_ELIGIBILITY 0xe7c474F470b08D4Ea82a8Fb6202D92c783a529Ba
  ORGANIZATION_BENEFIT 0x39f55263Df540Fe4bDEF9CF61B2F66DfBF2A1d09
  NODE_REWARD_POOL 0xf7e927ad9c19050b3Bc124ac8ac88307A7A018C6
  MINING_CARD 0x59987A7FA35229541e2C05eb43E3Bc1BBf305F06
  ORGANIZATION_CARD 0x9b0fB18EeAd7290b3D7Ade4A3A53f67a9346aFa4
  NODE_CARD 0xE5f740E69d0925398d4FCa1AeF0509Cb186e4144
  USDX:0x2ebA9Cc440487554f861f217B4eF963aC16dbB9d
```
```
在这些对应的地址里找ABI
MOB_V2_GATEWAY_IMPLEMENTATION 0xf2a8FbbE96DD38198222cDaf6Dfb1F561f4e463B
  INTERACTION_POOL_IMPLEMENTATION 0xb032E09C84eeBEB72E05D471C8E9B65d6c753FA9
  CARD_MANAGER_IMPLEMENTATION 0x2e27d5049fa9dB284B9B0157bAB8903e42F41644
  ORGANIZATION_BENEFIT_IMPLEMENTATION 0xF8e0AA72b9F1befBBFE1fdAAFB8a6714723Fd29b
  NODE_REWARD_POOL_IMPLEMENTATION 0x032B30e4b075B9e527261d0A9D82a1b4094eff19
  REWARD_DISTRIBUTOR_IMPLEMENTATION 0x74319a68e677e3456b27509b59B250E07F3a09fc
  TEAM_PERFORMANCE_IMPLEMENTATION 0x23C7da6fa5Ece119471697859f51aC43079F299B
  TEAM_ELIGIBILITY_IMPLEMENTATION 0xBCef3Af8467a3C4c358A12663a2aba19df498c5f
  MINING_SYSTEM_IMPLEMENTATION 0xD88f23022232c55E21AF707411393FB1d50695F5
  CARD_COMPOSER_IMPLEMENTATION 0xB58f344977ee7FfF6E0Daf1219661D357f988687
  TRADING_ROUTER_IMPLEMENTATION 0xd6533443Fb08cBdfC20E3683E9837200b0d7D9E5
  USER_DASHBOARD_IMPLEMENTATION 0x8dfaf92242ECB1D7adcD0Ac74F84c5bAEb2a4D3A
```
## 枚举值
```solidity
enum MiningCardType {
    U100,   // 0
    U300,   // 1
    U1000,  // 2
    U3000,  // 3
    U10000  // 4
}

enum MiningPaymentMethod {
    NcnAndUsdt, // 0，仅100U矿机卡支持
    MobAndUsdt  // 1，100U为100% MÖB；其他矿机卡按当前MÖB支付比例拆分
}

enum OrganizationType {
    RingOrganization, // 0，环组织令牌
    MobiusRing         // 1，莫比乌斯环令牌
}

enum NodeType {
    Foundation, // 0，基金会节点，不开放用户购买
    Genesis,    // 1，创世节点
    CoFounder,  // 2，联创节点
    Community   // 3，社区节点
}
```

## 授权对象速查

| 用户操作 | 资产 | `spender` / NFT `operator` |
| --- | --- | --- |
| 购买矿机卡、节点卡 | MÖB、NCN、USDT | `CardManager` 代理 |
| 激活矿机卡、续费 | MiningCard、USDT | `MiningSystem` 代理 |
| 质押或升级加速 | MÖB | `MiningSystem` 代理 |
| 合成时钱包补足 | MÖB | `CardComposer` 代理 |
| 永久质押组织卡 | OrganizationCard | `OrganizationBenefit` 代理 |
| 永久质押节点卡 | NodeCard | `NodeRewardPool` 代理 |
| 买入 NCN | USDT | `TradingRouter` 代理 |
| 卖出 NCN | NCN | `TradingRouter` 代理 |
| 卖出 MÖB | MÖB | `TradingRouter` 代理 |

---

### MobV2Gateway
```solidity
// 绑定永久邀请关系；普通邀请人必须已经登记，并拥有至少一张未出局且铸币费有效的工作卡。
// 首码不要求工作卡资格。绑定前可组合查询ReferralRegistry.isRegistered、ReferralRegistry.rootCode
// 和UserDashboard.isRewardEligible判断邀请人是否满足条件，前端无需直接读取MiningSystem。
function register(address inviter) external;

// 购买矿机卡；maxMob/maxNcn是价格保护上限，返回新NFT的tokenId
function buyMiningCard(
    MiningCardType cardType,
    MiningPaymentMethod method,
    uint256 maxMob,
    uint256 maxNcn
) external returns (uint256 tokenId);

// 购买创世、联创或社区节点卡；基金会节点不可购买，maxMob是价格保护上限
function buyNodeCard(NodeType nodeType, uint256 maxMob) external returns (uint256 tokenId);

// 永久质押矿机NFT、创建仓位并支付首个30天铸币费；调用前需已绑定邀请关系
function activateMiningCard(uint256 tokenId) external returns (uint256 positionId);

// 为指定矿机仓位续缴一个30天铸币费周期；可以在当前有效期结束前续费
function payMintFee(uint256 positionId) external;

// 为单张矿机仓位首次质押MÖB加速本金；金额必须达到该卡低档或高档门槛
function stakeAcceleration(uint256 positionId, uint256 mobAmount) external;

// 将低档加速一次性补足到高档；按补仓时Pair价格判断原本金与新增本金总价值
function upgradeAcceleration(uint256 positionId, uint256 additionalMob) external;

// 一次性赎回指定仓位的全部加速MÖB；赎回后该仓位不能再次质押加速
function redeemAcceleration(uint256 positionId) external;

// 立即提取用户指定24小时收益窗口截至当前已经产生且尚未消费的基础和加速收益，
// 并从毛MÖB收益中扣除20%发送到dead。windowIndex从0开始，以
// UserDashboard.firstActivationAt(account)作为窗口0起点；前端可通过
// UserDashboard.getPositionWindowEarnings(positionId, windowIndex)查看单张卡在该窗口的收益组成。
function withdrawImmediate(uint256 windowIndex)
    external
    returns (uint256 grossMob, uint256 feeMob);

// 正常提取指定24小时收益窗口。普通用户必须等待窗口结束后33天，并满足该窗口的NCN交易资格；
// 前端应先调用TradingRouter.ncnWindowStatus(account, windowIndex)展示是否合格。
// MiningSystem.withdrawalExempt(account)为true时，无需等待33天且无需NCN资格，只能领取截至当前已产生的收益。
function withdrawNormal(uint256 windowIndex) external returns (uint256 grossMob);

// 将指定过去或当前24小时窗口中截至调用时已经产生、尚未提现或合成消费的
// 矿机基础静态收益和MÖB质押加速收益合并换算为MÖB，计入合成额度。
// 本方法不消费直推、层级、团队、组织卡或节点卡奖励；返回值是本次新增的合成MÖB额度。
function accumulateCompositionEarnings(uint256 windowIndex)
    external
    returns (uint256 mobAmount);

// 使用已累计合成额度并从钱包补足MÖB合成矿机卡；maxWalletMob是钱包补足上限
function composeMiningCard(MiningCardType cardType, uint256 maxWalletMob)
    external
    returns (uint256 tokenId);

// 永久质押组织卡以取得后续铸币费权益；质押后不提供NFT赎回
function stakeOrganizationCard(uint256 tokenId) external;

// 领取调用者全部已记账的组织卡USDT权益
function claimOrganizationReward() external returns (uint256 amount);

// 永久质押节点卡以取得后续节点权益；质押后不提供NFT赎回
function stakeNodeCard(uint256 tokenId) external;

// 领取调用者四类节点卡累计的MÖB和USDT权益
function claimNodeReward()
    external
    returns (uint256 mobAmount, uint256 usdtAmount);

// 领取调用者全部直推、层级、团队MÖB及团队/V3/V4 USDT奖励
function claimDynamicReward()
    external
    returns (uint256 mobAmount, uint256 usdtAmount);

// 使用USDT经Uniswap V2 Router买入NCN，并记录正常提现所需交易量
function buyNcn(uint256 usdtAmount) external returns (uint256 ncnReceived);

// 卖出NCN换取USDT，并记录正常提现所需交易量；不收系统交易手续费
function sellNcn(uint256 ncnAmount) external returns (uint256 usdtReceived);

// 按当前交互池/Pair模式卖出MÖB；执行卖出额度、资格同步和手续费规则
function sellMob(uint256 mobAmount) external returns (uint256 usdtReceived);

// 当团队业绩未变化、但调用者钱包MÖB余额变化时，主动重新计算调用者自己的V1-V7等级。
// 卡牌激活或出局导致的上级15层业绩及等级变化已由合约自动处理，本方法不是keeper任务。
function refreshMyTeamLevel() external returns (uint8 oldLevel, uint8 newLevel);
```

下列为无权限状态维护入口。资格同步和每日任务通常由keeper发起，但任何地址都可以调用；交易模式检查会在每次`sellMob()`以及购卡、矿机卡激活、续费产生真实USDT入池时自动执行，也允许前端或keeper在没有这些用户交易时主动检查。

```solidity
// 固定最多处理100条已到期V3/V4资格；若仍未同步到当前时间可立即重复调用
function syncExpiredEligibility() external returns (uint256 processed);

// 按实时余额、资金流量和价格检查交互池/Pair交易模式是否需要切换。
// sellMob以及购卡、矿机卡激活和续费产生真实USDT入池时都会自动执行；本方法用于没有相关用户交易时补充检查。
function checkAndSwitchMode() external;

// 每24小时最多执行一次：条件满足时使用交互池1% USDT和等值MÖB添加流动性，LP留在交互池
function executeDailyLiquidity()
    external
    returns (uint256 usedUsdt, uint256 usedMob, uint256 liquidity);

// 每24小时最多执行一次：仅在Pair模式及资金比例满足时，使用交互池0.1% USDT买入MÖB并销毁
function executeDailyBuyback() external;
```

Pair中MÖB余额每日1%销毁不经过Gateway，需要直接调用本文`Mob.burnPairBalance()`。

Gateway目标地址查询：

```solidity
// 返回Gateway绑定的11个实际业务代理地址；可用于取得各授权spender地址
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
```

---

### UserDashboard

前端个人数据的主要只读聚合入口。分页参数 `limit` / `scanLimit` 必须为 `1-100`。

```solidity
// 返回邀请人、注册时间、卡牌数量、团队等级与业绩、钱包余额及各类收益汇总
function getUserOverview(address account)
    external
    view
    returns (UserOverview memory overview);

// 返回用户首张矿机卡激活时间，也是24小时收益窗口的统一起点；0表示尚未激活
// 该方法由Dashboard读取MiningSystem，前端无需持有或直接调用MiningSystem地址
function firstActivationAt(address account) external view returns (uint256 activatedAt);

// 判断地址当前是否至少有一张未出局且铸币费有效的工作卡
// 普通地址只有返回true时才能成为新邀请关系的邀请人；首码不受该条件限制
function isRewardEligible(address account) external view returns (bool eligible);

// 返回直推、层级、团队MÖB、铸币费团队USDT和V3/V4卖出USDT的历史累计记账额
function getDynamicRewardTotals(address account)
    external
    view
    returns (DynamicRewardTotals memory totals);

// 从工作卡FIFO游标分页读取未出局卡；首页cursorPositionId传0，nextCursor为0表示结束
function getWorkingCards(address account, uint256 cursorPositionId, uint256 limit)
    external
    view
    returns (CardView[] memory cards, uint256 nextCursor);

// 按创建顺序分页读取真实卡和虚拟卡的全部历史；首页offset传0
function getCardHistory(address account, uint256 offset, uint256 limit)
    external
    view
    returns (CardView[] memory cards, uint256 nextOffset);

// 在最多scanLimit条历史中筛选已出局卡；即使本页为空也要使用nextOffset继续翻页
function getExitedCards(address account, uint256 offset, uint256 scanLimit)
    external
    view
    returns (CardView[] memory cards, uint256 nextOffset);

// 分页返回直接邀请地址；首页offset传0
function getDirectReferrals(address account, uint256 offset, uint256 limit)
    external
    view
    returns (address[] memory accounts, uint256 nextOffset);

// 分页返回直推、层级、团队及V3/V4卖出均分奖励的逐笔历史
function getDynamicRewardRecords(address account, uint256 offset, uint256 limit)
    external
    view
    returns (RewardRecord[] memory records, uint256 nextOffset);

// 预览单张卡在指定24小时窗口中已产生、已消费和当前可用的基础/加速USDT收益
function getPositionWindowEarnings(uint256 positionId, uint256 windowIndex)
    external
    view
    returns (
        uint256 totalBaseUsdt,
        uint256 totalAccelerationUsdt,
        uint256 claimedBaseUsdt,
        uint256 claimedAccelerationUsdt,
        uint256 availableBaseUsdt,
        uint256 availableAccelerationUsdt
    );
```

主要返回结构：

```solidity
struct UserOverview {
    // 用户永久绑定的直接邀请人；未绑定邀请关系或查询首码时为零地址
    address inviter;

    // 用户第一张矿机卡的激活时间，也是收益24小时窗口的统一起点；0表示尚未激活
    uint256 registrationAt;

    // 用户累计直接邀请的地址数量，不会因为下级卡牌出局而减少
    uint256 directReferralCount;

    // 当前已质押且尚未出局的矿机卡数量；这些卡仍计入业绩和卖出额度
    uint256 workingCardCount;

    // 用户创建过的全部矿机仓位数量，包括工作卡、已出局卡和旧系统虚拟卡
    uint256 totalCardCount;

    // 已经达到收益上限并停止产出的卡牌数量
    uint256 exitedCardCount;

    // 当前团队等级；0表示无等级，1至7分别表示V1至V7
    uint8 teamLevel;

    // 当前业绩最高的直接邀请分区地址；没有直推分区时为零地址
    address maximumBranch;

    // 最大分区内最多向下15层的未出局矿机卡面值总和，单位为USDT最小精度
    uint256 maximumBranchPerformance;

    // 所有直推分区的团队业绩总和，不包含用户自己的矿机卡
    uint256 totalPerformance;

    // 小区业绩，等于总团队业绩减去最大分区业绩，是团队升级的主要考核值
    uint256 smallAreaPerformance;

    // 查询时用户钱包内实际持有的MÖB余额，不包含合约内待领取奖励
    uint256 walletMob;

    // 查询时用户钱包内实际持有的USDT余额，不包含合约内待领取奖励
    uint256 walletUsdt;

    // 历史通过立即提现和正常提现结算的矿机毛MÖB总额，尚未扣除立即提现销毁费
    uint256 miningGrossMob;

    // 历史矿机提现实际进入用户钱包的MÖB总额
    uint256 miningPaidMob;

    // 历史立即提现过程中被销毁的20% MÖB手续费总额
    uint256 miningImmediateFeeMob;

    // 已从基础/加速收益窗口转出、但尚未用于合成卡牌的MÖB内部额度，不是钱包余额
    uint256 compositionMobCredit;

    // 当前可领取的直推、层级和团队MÖB奖励合计
    uint256 dynamicPendingMob;

    // 当前可领取的铸币费团队USDT和V3/V4卖出手续费USDT奖励合计
    uint256 dynamicPendingUsdt;

    // 历史已经从动态奖励合约领取并到账的MÖB总额
    uint256 dynamicClaimedMob;

    // 历史已经从动态奖励合约领取并到账的USDT总额
    uint256 dynamicClaimedUsdt;

    // 历史成功记入用户账户的组织卡USDT权益总额，包含待领取和已领取部分
    uint256 organizationRecordedUsdt;

    // 当前可以领取的组织卡USDT权益
    uint256 organizationPendingUsdt;

    // 历史已经领取并到账的组织卡USDT权益总额
    uint256 organizationClaimedUsdt;

    // 四种节点卡当前可以领取的MÖB权益合计
    uint256 nodePendingMob;

    // 四种节点卡当前可以领取的USDT卖出手续费权益合计
    uint256 nodePendingUsdt;

    // 历史已经领取并到账的节点MÖB权益总额
    uint256 nodeClaimedMob;

    // 历史已经领取并到账的节点USDT权益总额
    uint256 nodeClaimedUsdt;
}

struct DynamicRewardTotals {
    // 历史成功记账的5%直推MÖB奖励总额，包含待领取和已领取部分
    uint256 directMob;

    // 历史成功记账的最多15层层级MÖB奖励总额
    uint256 levelMob;

    // 历史成功记账的V1-V7团队级差MÖB奖励总额
    uint256 teamMob;

    // 历史成功记账的铸币费团队级差USDT奖励总额
    uint256 teamUsdt;

    // 历史已经结算记账的V3/V4卖出手续费均分USDT总额
    uint256 gradeSaleUsdt;
}

struct RewardRecord {
    // 触发本笔奖励的下级地址；V3/V4全局卖出均分没有单一来源时为零地址
    address source;

    // 本笔实际成功记账的代币数量，已按出局剩余额度裁剪
    uint256 amount;

    // 本笔奖励使用的资产合约地址，用于区分MÖB和USDT
    address token;

    // 奖励类型：0直推MÖB、1层级MÖB、2团队MÖB、3铸币费团队USDT、4 V3/V4卖出USDT
    uint8 kind;

    // 本笔奖励写入链上账本的时间
    uint64 recordedAt;
}

struct CardView {
    // MiningSystem中的全局仓位编号，提现等窗口操作不是使用此编号
    uint256 positionId;

    // 该矿机仓位当前的完整状态
    Position position;

    // 该卡全部收益窗口中已经被提现或转入合成额度的基础静态收益USDT价值
    uint256 consumedBaseUsdt;

    // 该卡全部收益窗口中已经被提现或转入合成额度的加速增量收益USDT价值
    uint256 consumedAccelerationUsdt;
}

struct Position {
    // 该矿机仓位所属的用户地址
    address account;

    // 对应MiningCard的NFT编号；旧系统导入的虚拟卡固定为0
    uint256 nftTokenId;

    // 矿机卡面值类型：100U、300U、1000U、3000U或10000U
    MiningCardType cardType;

    // 该卡的收益出局倍数，只会是3倍或5倍
    uint8 exitMultiple;

    // 该卡曾达到的加速档位：0未加速、1低档、2高档；赎回后仍保留历史档位
    uint8 accelerationTier;

    // 卡牌质押并建立仓位的时间；虚拟旧卡为导入时间
    uint64 activatedAt;

    // 当前已缴铸币费覆盖到的截止时间；超过后停止产生新收益
    uint64 activeUntil;

    // 首次进入加速状态的时间；0表示从未进行加速质押
    uint64 lowAccelerationAt;

    // 开始按高档加速计算收益的时间；0表示从未达到高档
    uint64 highAccelerationAt;

    // 加速收益停止计算的时间，通常是赎回全部加速本金的时间；0表示尚未停止
    uint64 accelerationEndedAt;

    // 达到出局上限并停止产出的时间；0表示尚未出局
    uint64 exitedAt;

    // 当前仍存放在交互池、尚未赎回的加速MÖB本金，不代表加速收益
    uint256 accelerationMob;

    // 该卡允许累计获得的最大收益价值，等于卡牌面值乘以3倍或5倍
    uint256 exitCapUsdt;

    // 已经占用的出局额度，包含基础、加速、直推、层级和团队等计入该卡的收益价值
    uint256 exitUsedUsdt;

    // 工作卡FIFO中上一张卡的仓位编号；0表示没有上一张或已移出工作队列
    uint256 previous;

    // 工作卡FIFO中下一张卡的仓位编号；0表示没有下一张或已移出工作队列
    uint256 next;

    // true表示旧系统导入的虚拟卡，不存在可转移的MiningCard NFT
    bool virtualCard;

    // true表示该卡已经执行过一次从低档补到高档，不能再次补档
    bool accelerationUpgraded;

    // true表示该卡的加速本金已经全部赎回，之后不能重新质押加速
    bool accelerationRedeemed;

    // true表示该卡已达到出局额度、停止产出并移出工作卡队列
    bool exited;
}
```

---

### PriceOracle

价格只读入口，报价来自当前 Uniswap V2 Pair，包含标准输出公式、0.3%费用和当前报价数量的价格影响。报价会随链上储备变化，交易提交时仍应设置合理的 `maxMob` / `maxNcn`。

```solidity
// 估算卖出指定MÖB可得到的USDT
function quoteMobToUsdt(uint256 mobAmount)
    external
    view
    returns (uint256 usdtAmount);

// 估算取得指定USDT价值需要支付的MÖB
function quoteUsdtToMob(uint256 usdtAmount)
    external
    view
    returns (uint256 mobAmount);

// 估算卖出指定NCN可得到的USDT
function quoteNcnToUsdt(uint256 ncnAmount)
    external
    view
    returns (uint256 usdtAmount);

// 估算取得指定USDT价值需要支付的NCN
function quoteUsdtToNcn(uint256 usdtAmount)
    external
    view
    returns (uint256 ncnAmount);
```

---

### CardManager

卡牌价格、支付比例、完整付款拆分和节点供应量的只读查询。实际购买应调用 Gateway；付款预览直接读取 CardManager。

```solidity
// 返回矿机卡USDT面值
function miningFaceValue(MiningCardType cardType) external view returns (uint256);

// 返回节点卡USDT面值；基金会节点返回0
function nodeFaceValue(NodeType nodeType) external view returns (uint256);

// 返回指定节点类型的总发行上限
function nodeSupplyCap(NodeType nodeType) external pure returns (uint256);

// 返回指定节点类型当前累计铸造数量，包括operator铸造和用户购买
function nodeMinted(NodeType nodeType) external view returns (uint256);

// 返回300U及以上矿机卡当前MÖB支付比例，单位为基点；5000表示50%
function miningMobBps() external view returns (uint256);

// 按当前Pair报价和矿机卡付款比例返回实际购买所需的MÖB、NCN和USDT数量；未使用的资产返回0
function previewBuyMiningCard(MiningCardType cardType, MiningPaymentMethod method)
    external
    view
    returns (uint256 mobAmount, uint256 ncnAmount, uint256 usdtAmount);

// 按当前Pair报价返回节点卡付款拆分：销毁的MÖB、进入交互池的USDT、支付指定收款人的USDT
// 基金会节点不开放购买，查询该类型会回滚
function previewBuyNodeCard(NodeType nodeType)
    external
    view
    returns (uint256 mobAmount, uint256 poolUsdtAmount, uint256 recipientUsdtAmount);
```

---

### TradingRouter

交易页面补充只读查询。实际交易应调用 Gateway。

```solidity
// 返回当前MÖB卖出模式；true为外部Pair，false为交互池
function isPairMode() external view returns (bool);

// 返回用户指定NCN交易窗口的总额、买入额、是否已消费及当前是否合格
function ncnWindowStatus(address account, uint256 windowIndex)
    external
    view
    returns (uint256 totalUsdt, uint256 buyUsdt, bool used, bool qualified);

// 返回用户NCN连续24小时窗口起点；首张矿机卡激活或虚拟卡导入时建立，0表示尚未建立
function ncnWindowOrigin(address account) external view returns (uint256);

// 返回用户历史MÖB卖出毛USDT累计额；该值不会因卡牌出局而清零
function historicalReceivedUsdt(address account) external view returns (uint256);
```

---

### MiningSystem

矿机页面补充只读查询。写操作应调用 Gateway。

```solidity
// 返回用户当前工作卡最大铸币费有效截止时间
function maxActiveUntil(address account) external view returns (uint256);

// 返回用户是否享有正常提现NCN资格豁免
function withdrawalExempt(address account) external view returns (bool);

// 返回矿机NFT对应的仓位编号；0表示尚未激活
function positionOfNft(uint256 tokenId) external view returns (uint256);

// 返回指定仓位、收益窗口已经消费的基础和加速USDT收益
function claimedByWindow(uint256 positionId, uint256 windowIndex)
    external
    view
    returns (uint256 baseUsdt, uint256 accelerationUsdt);
```

窗口编号计算：

```text
windowIndex = floor((目标时间 - UserDashboard.firstActivationAt(account)) / 24 hours)
```

正常提现成熟时间为该窗口结束时间再加 `33 days`。

---

### TeamPerformance

团队等级一般在卡牌激活或出局引起业绩变化时自动更新到上级15层。下列方法仅处理钱包MÖB余额变化引起的持币条件变化，不属于keeper任务。

```solidity
// 按当前小区业绩和钱包MÖB价值重新计算指定地址的等级；任何地址都可帮助目标用户刷新
function refreshLevel(address account)
    external
    returns (uint8 oldLevel, uint8 newLevel);
```

普通用户刷新自己的等级时，优先调用`MobV2Gateway.refreshMyTeamLevel()`。

---

### ReferralRegistry

邀请关系状态补充查询。邀请登记写操作应调用 Gateway。

```solidity
// 查询地址是否已经永久登记邀请关系；首码地址部署时即为true
function isRegistered(address account) external view returns (bool);

// 查询用户永久绑定的直接邀请人；未注册用户及首码返回零地址
function inviterOf(address account) external view returns (address);

// 返回邀请树首码地址；该地址禁止激活矿机卡
function rootCode() external view returns (address);
```

邀请人资格判断：

```text
inviter == ReferralRegistry.rootCode()
或
ReferralRegistry.isRegistered(inviter) == true
且 UserDashboard.isRewardEligible(inviter) == true
```

---
### NodeRewardPool

节点卡领取写操作应调用Gateway。以下接口用于按节点类型展示质押数量、比例和待领取权益；`NodeType`取值见本文枚举说明。

```solidity
// 查询某张节点卡的永久质押人、节点类型和质押时间；staker为零地址表示尚未质押
function stakes(uint256 tokenId)
    external
    view
    returns (address staker, NodeType nodeType, uint64 stakedAt);

// 查询用户质押的指定类型节点卡份数；同一地址质押多张时按份数参与均分
function stakeCount(address account, NodeType nodeType) external view returns (uint256);

// 查询全网当前质押的指定类型节点卡总份数
function totalStaked(NodeType nodeType) external view returns (uint256);

// 查询指定节点类型从全网基础静态收益中取得的奖励比例，单位为基点；600表示6%
function networkRewardBps(NodeType nodeType) external view returns (uint256);

// 查询用户在指定节点类型当前可领取的MÖB与USDT；包含尚未结算进pending字段的实时份额
function pendingRewards(address account, NodeType nodeType)
    external
    view
    returns (uint256 mobAmount, uint256 usdtAmount);

// 查询用户历史已经领取并到账的全部节点MÖB权益
function cumulativeClaimedMob(address account) external view returns (uint256);

// 查询用户历史已经领取并到账的全部节点USDT权益
function cumulativeClaimedUsdt(address account) external view returns (uint256);
```

---

### OrganizationBenefit

组织权益历史补充查询；总览和领取分别优先使用 Dashboard 与 Gateway。

```solidity
// 查询用户永久质押的指定类型组织卡份数
function stakeCount(address account, OrganizationType cardType) external view returns (uint256);

// 返回用户组织权益历史记录总数
function rewardRecordCount(address account) external view returns (uint256);

// 分页返回组织权益记录；offset从0开始，limit为0或offset越界时返回空数组
function getRewardRecords(address account, uint256 offset, uint256 limit)
    external
    view
    returns (RewardRecord[] memory records);
```

```solidity
struct RewardRecord {
    // 支付铸币费并触发这笔组织权益的下级用户地址
    address source;

    // 本笔成功记入账户的USDT权益数量
    uint256 amount;

    // 产生本笔权益的组织卡类型：环组织令牌或莫比乌斯环令牌
    OrganizationType cardType;

    // 本笔权益写入链上账本的时间
    uint64 recordedAt;
}
```

---

### Mob

MÖB的Pair余额每日销毁入口不经过Gateway。该方法无调用者权限限制，通常由keeper直接调用MÖB代币合约。

```solidity
// 距离上次成功执行满24小时后，将Pair当前MÖB余额的1%转入dead并调用Pair.sync；一次不补多个周期
function burnPairBalance() external;

// 返回最近一次成功执行Pair余额1%销毁的时间，用于计算下次最早执行时间
function lastPairBurnAt() external view returns (uint256);

// 返回两次Pair余额销毁之间要求的最短间隔，当前固定为24小时
function PAIR_BURN_INTERVAL() external view returns (uint256);
```

另外两个每日任务通过Gateway调用：

```solidity
// 每日添加流动性；Gateway转发至InteractionPool.executeDailyLiquidity
function executeDailyLiquidity()
    external
    returns (uint256 usedUsdt, uint256 usedMob, uint256 liquidity);

// Pair模式每日使用交互池0.1% USDT买入并销毁MÖB；Gateway转发至InteractionPool.executeDailyBuyback
function executeDailyBuyback() external;
```

---

### IERC20（MÖB、NCN、USDT）

MÖB、NCN 和外部 USDT 均通过标准 ERC20 接口交互。

```solidity
// 查询账户代币余额
function balanceOf(address account) external view returns (uint256);

// 查询owner已经授权给spender的剩余额度
function allowance(address owner, address spender) external view returns (uint256);

// 授权下游业务代理支出代币；spender按本文“授权对象速查”选择
function approve(address spender, uint256 amount) external returns (bool);

// 普通钱包转账；不用于系统购买、质押或交易入口
function transfer(address to, uint256 amount) external returns (bool);

// 查询代币精度
function decimals() external view returns (uint8);

// 查询代币名称
function name() external view returns (string memory);

// 查询代币符号
function symbol() external view returns (string memory);
```

---

### MiningCard / OrganizationCard / NodeCard

三种卡牌均实现标准 ERC721。矿机卡激活、组织卡质押和节点卡质押前，需要分别授权对应的业务代理。

```solidity
// 查询tokenId当前持有人
function ownerOf(uint256 tokenId) external view returns (address owner);

// 查询某地址持有的NFT数量
function balanceOf(address owner) external view returns (uint256 balance);

// 授权指定业务代理操作单张NFT
function approve(address to, uint256 tokenId) external;

// 批量授权或取消授权指定业务代理操作调用者的全部同类NFT
function setApprovalForAll(address operator, bool approved) external;

// 查询单张NFT当前授权地址
function getApproved(uint256 tokenId) external view returns (address operator);

// 查询owner是否已经批量授权operator
function isApprovedForAll(address owner, address operator) external view returns (bool);

// 在普通钱包之间安全转移尚未质押的NFT；已质押NFT永久锁定，无法转移
function safeTransferFrom(address from, address to, uint256 tokenId) external;

// 查询卡牌元数据URI
function tokenURI(uint256 tokenId) external view returns (string memory);

// MiningCard：查询tokenId对应的矿机卡类型
function tokenCardTypes(uint256 tokenId) external view returns (MiningCardType cardType);

// OrganizationCard：查询tokenId对应的组织卡类型
function tokenCardTypes(uint256 tokenId) external view returns (OrganizationType cardType);

// NodeCard：查询tokenId对应的节点卡类型
function tokenCardTypes(uint256 tokenId) external view returns (NodeType cardType);
```

## 前端调用注意事项

1. 购买、交易、质押和合成前，先查询余额与 allowance/approval；授权目标必须是实际下游业务代理。
2. `maxMob`、`maxNcn` 和 `maxWalletMob` 是用户价格保护上限，不是固定应付数量。应先读取 `PriceOracle` 报价，再按前端确认的容忍度设置。
3. `withdrawImmediate`、`withdrawNormal` 和 `accumulateCompositionEarnings` 的参数都是用户统一24小时收益窗口编号，不是仓位编号。窗口起点读取`UserDashboard.firstActivationAt(account)`，单张卡窗口收益读取`UserDashboard.getPositionWindowEarnings(positionId, windowIndex)`；前端无需直接查询`MiningSystem`。
4. Dashboard分页的下一页参数必须使用合约返回的 `nextCursor` 或 `nextOffset`，不要自行按返回数组长度推断已出局卡分页位置。
5. 组织卡和节点卡质押为永久锁定；矿机卡激活后同样由挖矿合约永久托管，出局后不返还NFT。
6. MÖB卖出前若资格队列未同步，交易会回滚；此时先调用 `syncExpiredEligibility()`，直到待处理到期记录全部同步后再卖出。
7. 付款预览使用调用时的Pair储备，只代表当前应付数量。发起购买时仍应按用户确认的容忍度设置`maxMob`或`maxNcn`，防止报价变化导致超额支付。
