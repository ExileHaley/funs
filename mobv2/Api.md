# MÖB V2 前端合约 API
## 合约地址
```
MOB_V2_GATEWAY_PROXY 0x45f4A06621e94A6A8553323898e5f2FC2c53dd62
  INTERACTION_POOL 0xbAa9fc4BA8bA39AEFCBB3DA6B8997913a8DBdC69
  MOB 0xb72b968823d8c4029CDeafcCb7cBd256100acC8d
  NCN 0x098772A9fcDFe07914a8c1f96D63eb3a502f526a
  PRICE_ORACLE 0x63C5f5594D742E74B38EE3f8e4A14451B92F8234
  REFERRAL_REGISTRY 0x7e95aB5e74Ad5C2B794c15463Fb8859Dee6d022B
  CARD_MANAGER 0xA53D6E0837dC857EC363c3BbE313Bd828380eF3a
  MINING_EARNINGS_SETTLEMENT 0x3241565d79FB3115d5D1FDD2D4453BE4A2812064
  MINING_SYSTEM 0xbc240C4251517138ca5502Be66DF41f8c0270700
  CARD_COMPOSER 0x89BD99a7B9ce7fdb5C2582B9041848269E764859
  TRADING_ROUTER 0x1cB785B324693FC43c7D0f2527211214759bc34a
  USER_DASHBOARD 0x083F44c5EF9D0859CeC6D446D5D7d58A2acCD585
  REWARD_DISTRIBUTOR 0xc1D3644Dbdf917F9FdCb4DfD744F725bfF4A09b1
  TEAM_PERFORMANCE 0x68BbA37929F6ED02A6dBCD1dbB5D1ab49fddECB9
  TEAM_ELIGIBILITY 0x526800Df2D0F55b632d5C0E9a64F1A49ecc73A35
  ORGANIZATION_BENEFIT 0xEc1BB9Abf58e5B2F30958E19dA37DA50b2875711
  NODE_REWARD_POOL 0x9b5E444120C73E49D98F13fd74e7b392A71AEe3f
  MINING_CARD 0x4e6e9521d2a160e0787592C058c378dC7324A71a
  ORGANIZATION_CARD 0x9fc57aC5aC8db3Fa9Ef5B9d377A470Db291ee3E7
  NODE_CARD 0xc919750DA49353F778352aC4065D7cf214e2597e
  MOB_V2_GATEWAY_IMPLEMENTATION 0x42631d74Beed4CFb2606A0f3037b09Da012D27D8
  INTERACTION_POOL_IMPLEMENTATION 0x8A71153dbA180FD0677025d5eeb0ff02D38B6B13
  CARD_MANAGER_IMPLEMENTATION 0x1DC2dee56b7A954A6D327363d5ea7DF311C76FC3
  ORGANIZATION_BENEFIT_IMPLEMENTATION 0x240563E420ee5B4Af3439D32eBBf98F76d1C8BDc
  NODE_REWARD_POOL_IMPLEMENTATION 0x3aa9808B5E4D278712a18F553eFF57C70c511e25
  REWARD_DISTRIBUTOR_IMPLEMENTATION 0x6b54E3c5c8e8af5f34B9Ff76d59e84e91A1eABf9
  TEAM_PERFORMANCE_IMPLEMENTATION 0x3482dD898E493f585a1d418a0540b2e1dD8b3fd0
  TEAM_ELIGIBILITY_IMPLEMENTATION 0x58e6392CD2fd455C2Bc6E7a7ab0293946D95bb36
  MINING_EARNINGS_SETTLEMENT_IMPLEMENTATION 0x337d7153f0229892Cc9275950c8Ae6150558f6fa
  MINING_SYSTEM_IMPLEMENTATION 0x1145eD32cE44f7e7365BD56003b82a2921Bb0472
  CARD_COMPOSER_IMPLEMENTATION 0xFf727d096f1e21610D52e555Aa26D1D37E5A2B2C
  TRADING_ROUTER_IMPLEMENTATION 0x5725Ade05eca4b1ace19B9280c9512E990372b1f
  USER_DASHBOARD_IMPLEMENTATION 0xADebFFAC26fA24e6A8931D50dd28536FEEeBD3e3
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

// 静态收益立即提现：只处理positionId指定的一张卡，收益包含该卡基础产出和该卡MÖB质押加速增量，
// 不包含直推、层级、团队、组织卡或节点奖励。最多处理200个连续24小时窗口截至当前已产生且尚未消费的收益。
// startWindowIndex、endWindowIndex为闭区间；逐窗口扣减账本后，把整批基础与加速收益分别汇总，
// 按汇总额只执行一次价格换算、层级/团队奖励、节点奖励和MÖB支付，并从整批毛MÖB中销毁20%。
// grossMob是扣费前毛MÖB，feeMob是本次销毁的20%，用户实际到账为grossMob-feeMob。
function withdrawImmediate(uint256 positionId, uint256 startWindowIndex, uint256 endWindowIndex)
    external
    returns (uint256 grossMob, uint256 feeMob);

// 静态收益正常提现：只处理positionId指定的一张卡，收益包含该卡基础产出和该卡MÖB质押加速增量，
// 不包含任何动态、组织卡或节点奖励。普通用户所选窗口均须在窗口结束后经过
// MiningSystem.maturityDelay()；只按实际提取到非零收益的窗口数消费NCN额度：每个窗口10U，
// 其中每窗口买入消费额严格大于5U。逐窗口扣账后，价格换算、奖励和支付均按整批汇总额执行一次。
// 前端应先调用TradingRouter.ncnWithdrawalQuotaStatus(account)展示当前可用额度和资格。
// MiningSystem.withdrawalExempt(account)为true时跳过成熟周期和NCN额度条件，只领取截至当前已产生的收益。
// grossMob是本次实际进入用户钱包的MÖB；正常提现不收20%立即提现销毁费。
function withdrawNormal(uint256 positionId, uint256 startWindowIndex, uint256 endWindowIndex)
    external
    returns (uint256 grossMob);

// 使用调用者指定的一张矿机卡，从startWindowIndex到endWindowIndex按旧到新消费连续窗口静态收益，
// 单次最多200个闭区间窗口；每个窗口先用基础收益、再用加速收益，达到目标卡面值后停止。
// 逐窗口扣减账本和出局额度，但层级、团队、节点奖励及Pair价格换算均按实际使用总额执行一次。
// 不足部分由钱包MÖB补足且不得超过maxWalletMob；未使用收益保留在原卡原窗口。
function composeMiningCard(
    uint256 positionId,
    uint256 startWindowIndex,
    uint256 endWindowIndex,
    MiningCardType cardType,
    uint256 maxWalletMob
)
    external
    returns (uint256 tokenId);

// 永久质押组织卡以取得后续铸币费权益；质押后不提供NFT赎回
function stakeOrganizationCard(uint256 tokenId) external;

// 领取调用者当前全部可领取的组织卡USDT权益；成功后当前待领取余额清零并增加历史已领取总额
function claimOrganizationReward() external returns (uint256 amount);

// 永久质押节点卡以取得后续节点权益；质押后不提供NFT赎回
function stakeNodeCard(uint256 tokenId) external;

// 领取调用者四类节点卡当前全部可领取权益；MÖB来自全网基础静态收益，USDT来自卖出手续费分配
function claimNodeReward()
    external
    returns (uint256 mobAmount, uint256 usdtAmount);

// 动态收益领取：一次领取当前全部直推、层级、团队MÖB，以及铸币费团队和V3/V4卖出均分USDT。
// 不传positionId或窗口，不会提取任何单张卡的基础/加速静态收益，也不会领取组织卡或节点奖励。
// 返回值是本次实际由交互池支付到用户钱包的MÖB和USDT；成功后对应pending清零并累计到历史已领取。
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

下列为无权限状态维护入口。资格同步和每日任务通常由keeper发起，但任何地址都可以调用；交易模式检查会在每次`sellMob()`、购卡、矿机卡激活、续费产生真实USDT入池以及两个每日任务完成资产操作后自动执行，也允许前端或keeper在没有这些交易时主动检查。

```solidity
// 固定最多处理100条已到期V3/V4资格；若仍未同步到当前时间可立即重复调用
function syncExpiredEligibility() external returns (uint256 processed);

// 按实时余额、资金流量和价格检查交互池/Pair交易模式是否需要切换。
// sellMob、购卡、矿机卡激活、续费以及两个每日任务完成资产操作后都会自动执行；本方法用于没有相关交易时补充检查。
function checkAndSwitchMode() external;

// 每24小时最多执行一次：条件满足时使用交互池1% USDT和等值MÖB添加流动性，LP留在交互池；完成后自动检查并按现有规则切换交易模式
function executeDailyLiquidity()
    external
    returns (uint256 usedUsdt, uint256 usedMob, uint256 liquidity);

// 每24小时最多执行一次：仅在Pair模式及资金比例满足时，使用交互池0.1% USDT买入MÖB并销毁；完成后自动检查并按现有规则切换交易模式
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
// 返回邀请人、注册时间、卡牌数量、伞下历史铸币费、团队业绩、钱包余额及各类收益汇总。
// 收益字段已明确拆分为历史累计、当前可领取和历史已领取，读取本方法不会执行结算或领取。
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

// 按查询时Pair价格把单卡窗口剩余收益折算为预计MÖB。
// withdrawalMob按基础和加速分别报价后相加；compositionMob按两项USDT合计后一次报价。
function getPositionWindowMobPreview(uint256 positionId, uint256 windowIndex)
    external
    view
    returns (
        uint256 baseMob,
        uint256 accelerationMob,
        uint256 withdrawalMob,
        uint256 compositionMob
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

    // 直属用户中至少真实激活过一次或导入过虚拟卡的人数；永久累计，用于决定层级奖励深度
    uint256 effectiveDirectReferralCount;

    // 历史累计值：用户伞下最多15层所有下级成功支付的完整铸币费总额，不包含本人支付，单位为USDT最小精度。
    // 只在激活或续费的USDT支付成功后增加，卡牌到期、出局或下级失效均不扣减。
    // 例如A推荐B、B推荐C，B累计支付90U、C累计支付100U，则B为100U，A为190U。
    uint256 subtreeMintFeePaidUsdt;

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

    // 历史累计值：立即提现与正常提现的静态收益毛MÖB总额，包含卡牌基础收益和MÖB加速收益。
    // 不包含用于合成的收益、动态奖励、组织卡奖励或节点奖励；该字段尚未扣除立即提现20%销毁费。
    uint256 miningGrossMob;

    // 历史已到账值：静态收益提现实际进入用户钱包的MÖB总额，等于历史正常提现到账加立即提现净到账。
    // 不包含用于合成并销毁的收益，也不表示当前还有多少静态收益可提取。
    uint256 miningPaidMob;

    // 历史立即提现过程中被销毁的20% MÖB手续费总额；正常提现和合成不会增加此值
    uint256 miningImmediateFeeMob;

    // 当前可领取值：尚未从RewardDistributor领取的直推、层级和团队MÖB合计，领取成功后清零
    uint256 dynamicPendingMob;

    // 当前可领取值：铸币费团队USDT与V3/V4卖出均分USDT合计；包含查询时尚未写入pending的实时索引差额
    uint256 dynamicPendingUsdt;

    // 历史已领取值：已经从RewardDistributor实际领取并到账的MÖB总额，不包含当前pending
    uint256 dynamicClaimedMob;

    // 历史已领取值：已经从RewardDistributor实际领取并到账的USDT总额，不包含当前pending
    uint256 dynamicClaimedUsdt;

    // 历史成功记入用户账户的组织卡USDT权益总额，包含待领取和已领取部分
    uint256 organizationRecordedUsdt;

    // 当前可以领取的组织卡USDT权益
    uint256 organizationPendingUsdt;

    // 历史已经领取并到账的组织卡USDT权益总额
    uint256 organizationClaimedUsdt;

    // 当前可领取值：四种节点卡MÖB权益的实时合计，包含尚未结算进节点pending存储的累计索引差额
    uint256 nodePendingMob;

    // 当前可领取值：四种节点卡USDT卖出手续费权益的实时合计
    uint256 nodePendingUsdt;

    // 历史已领取值：节点奖励领取时已经实际到账的MÖB总额，不包含当前pending
    uint256 nodeClaimedMob;

    // 历史已领取值：节点奖励领取时已经实际到账的USDT总额，不包含当前pending
    uint256 nodeClaimedUsdt;
}

struct DynamicRewardTotals {
    // 历史成功记账的5%直推MÖB奖励总额，包含待领取和已领取部分
    uint256 directMob;

    // 历史成功记账的最多15层层级MÖB奖励总额，包含待领取和已领取部分
    uint256 levelMob;

    // 历史成功记账的V1-V7团队级差MÖB奖励总额，包含待领取和已领取部分
    uint256 teamMob;

    // 历史成功记账的铸币费团队级差USDT奖励总额，包含待领取和已领取部分
    uint256 teamUsdt;

    // 历史已经惰性结算并成功记账的V3/V4卖出手续费均分USDT总额，包含待领取和已领取部分；
    // 尚未结算的实时索引差额只会体现在UserOverview.dynamicPendingUsdt中
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
    // MiningSystem中的全局仓位编号；提现、收益查询和合成都必须用它指定单张卡
    uint256 positionId;

    // 该矿机仓位当前的完整状态
    Position position;

    // 历史已消费值：该卡全部窗口中已经被提现或用于合成的基础静态收益USDT价值，不代表当前可领取额
    uint256 consumedBaseUsdt;

    // 历史已消费值：该卡全部窗口中已经被提现或用于合成的加速增量USDT价值，不代表当前可领取额
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

// 预览指定用户卖出MÖB时的当前模式、预计毛USDT、手续费、净到账和当前单次卖出上限。
// 覆盖新用户20%、团队保护、200%后的30%、Pair动态费率和跨保护额度混合费率，但不修改状态。
function previewSellMob(address seller, uint256 mobAmount)
    external
    view
    returns (
        bool pairMode,
        uint256 grossUsdt,
        uint256 feeUsdt,
        uint256 receivedUsdt,
        uint256 sellLimitUsdt
    );

// 返回长期累计且尚未被正常提现消费的NCN买入、卖出和总USDT额度。
// qualified表示总额至少10U且买入额度严格大于5U；每次withdrawNormal成功消费10U。
function ncnWithdrawalQuotaStatus(address account)
    external
    view
    returns (uint256 buyUsdt, uint256 sellUsdt, uint256 totalUsdt, bool qualified);

// 返回用户历史MÖB卖出毛USDT累计额；该值不会因卡牌出局而清零
function historicalReceivedUsdt(address account) external view returns (uint256);
```

---

### MiningSystem

矿机页面补充只读查询。写操作应调用 Gateway。矿机提现历史已由内部 `MiningEarningsSettlement` 负责累计，前端继续统一通过 `UserDashboard.getUserOverview` 读取，不需要持有或调用该内部结算代理。

```solidity
// 返回用户当前工作卡最大铸币费有效截止时间
function maxActiveUntil(address account) external view returns (uint256);

// 返回用户是否享有正常提现NCN资格豁免
function withdrawalExempt(address account) external view returns (bool);

// 返回正常提现当前使用的全局成熟周期秒数；默认33天，owner修改后立即作用于全部未提现窗口
function maturityDelay() external view returns (uint256);

// owner按秒修改全局成熟周期；必须大于0
function setMaturityDelay(uint256 newDelay) external;

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

正常提现成熟时间为该窗口结束时间再加当前 `MiningSystem.maturityDelay()`；默认值为 `33 days`。

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

### RewardDistributor

前端一般通过 `UserDashboard.getUserOverview` 读取以下两个累计值，也可直接读取奖励合约。

```solidity
// 返回用户直属地址中已经至少真实激活一次或导入过虚拟矿机卡的永久有效人数
function effectiveDirectReferralCount(address account) external view returns (uint256);

// 返回用户伞下最多15层所有下级历史成功支付的完整铸币费总额，不含本人支付，单位为USDT最小精度。
// 该值只增不减；例如A推荐B、B推荐C，B累计支付90U、C累计支付100U，则B返回100U、A返回190U。
function subtreeMintFeePaidUsdt(address account) external view returns (uint256);
```

---

### InteractionPool

以下为交互池管理查询和操作接口。operator复用部署环境变量 `OPERATOR`，owner可更换；这些管理操作不通过Gateway。三个倍数只改变模式判断阈值，不改变7%/27%流出限制、每日任务比例或24小时间隔。

```solidity
// 返回当前交互池operator地址
function operator() external view returns (address);

// owner设置非零operator地址
function setOperator(address operator_) external;

// 返回交互池切到Pair时使用的资金比例基点，默认15000表示1.5倍
function poolToPairRatioBps() external view returns (uint256);

// 返回因资金比例切出后回到交互池模式所需的余额比例基点，默认30000表示3倍
function poolRecoveryRatioBps() external view returns (uint256);

// 返回Pair流出触发回切的倍数基点，默认60000表示6倍
function pairOutflowRecoveryMultipleBps() external view returns (uint256);

// operator一次设置三个不会混淆的模式参数：
// poolToPairRatioBps_：交互池USDT相对Pair USDT的切出比例，默认15000=1.5倍；
// poolRecoveryRatioBps_：资金比例切出后允许回到交互池模式的余额比例，默认30000=3倍；
// pairOutflowRecoveryMultipleBps_：Pair流出后的恢复倍数，默认60000=6倍。
// 三项均使用基点、必须大于0且修改后立即全局生效。
function setSwitchMultipliers(
    uint256 poolToPairRatioBps_,
    uint256 poolRecoveryRatioBps_,
    uint256 pairOutflowRecoveryMultipleBps_
) external;

// operator提取交互池持有的任意ERC20到非零接收地址；不预留用户负债，余额不足时整笔回滚
function withdrawToken(address token, address recipient, uint256 amount) external;
```

---

### NodeRewardPool

节点卡质押和领取应调用Gateway。比例管理直接调用NodeRewardPool：owner只设置operator，只有operator能调整比例。首次部署的operator复用 `.env` 的 `OPERATOR`。

```solidity
// 返回当前有权调整四类节点全网基础静态MÖB奖励比例的operator地址
function operator() external view returns (address);

// owner设置或更换非零operator；owner本身不能绕过operator直接修改节点比例
function setOperator(address operator_) external;

// 查询某张节点卡的永久质押人、节点类型和质押时间；staker为零地址表示尚未质押
function stakes(uint256 tokenId)
    external
    view
    returns (address staker, NodeType nodeType, uint64 stakedAt);

// 查询用户质押的指定类型节点卡份数；同一地址质押多张时按份数参与均分
function stakeCount(address account, NodeType nodeType) external view returns (uint256);

// 查询全网当前质押的指定类型节点卡总份数
function totalStaked(NodeType nodeType) external view returns (uint256);

// 查询指定节点类型从已提现或已用于合成的基础静态收益中取得的奖励比例。
// 单位为基点；默认基金会600、创世300、联创300、社区500，分别表示6%、3%、3%、5%。
function networkRewardBps(NodeType nodeType) external view returns (uint256);

// operator设置单个节点类型的全网基础静态MÖB奖励比例；允许0至1000基点，即0%-10%。
// 修改后只影响后续recordNetworkReward记账，不追溯重算历史已分配奖励。
function setNetworkRewardBps(NodeType nodeType, uint256 rewardBps) external;

// 查询用户在指定节点类型当前可领取的MÖB与USDT；这是实时待领取值，包含尚未落入pending存储的索引差额。
// MÖB来自全网基础静态收益分配，USDT仅来自该节点类型有权取得的MÖB卖出手续费分配。
function pendingRewards(address account, NodeType nodeType)
    external
    view
    returns (uint256 mobAmount, uint256 usdtAmount);

// 查询用户历史已经领取并到账的全部节点MÖB权益；不包含当前pendingRewards
function cumulativeClaimedMob(address account) external view returns (uint256);

// 查询用户历史已经领取并到账的全部节点USDT权益；不包含当前pendingRewards
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

    // 本笔历史成功记入账户的USDT权益数量；记录本身不会因领取而删除，也不代表当前仍可领取
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
// 每日添加流动性；Gateway转发至InteractionPool.executeDailyLiquidity，资产操作完成后自动检查并按现有规则切换交易模式
function executeDailyLiquidity()
    external
    returns (uint256 usedUsdt, uint256 usedMob, uint256 liquidity);

// Pair模式每日使用交互池0.1% USDT买入并销毁MÖB；Gateway转发至InteractionPool.executeDailyBuyback，完成后自动检查并按现有规则切换交易模式
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
3. `withdrawImmediate`、`withdrawNormal`和`composeMiningCard`都必须指定单张卡的`positionId`及最多200个连续窗口的闭区间。窗口起点读取`UserDashboard.firstActivationAt(account)`，单卡USDT收益和MÖB预览分别读取`getPositionWindowEarnings`与`getPositionWindowMobPreview`；前端无需直接读取`MiningSystem`内部依赖。
4. Dashboard分页的下一页参数必须使用合约返回的 `nextCursor` 或 `nextOffset`，不要自行按返回数组长度推断已出局卡分页位置。
5. 组织卡和节点卡质押为永久锁定；矿机卡激活后同样由挖矿合约永久托管，出局后不返还NFT。
6. MÖB卖出前若资格队列未同步，交易会回滚；此时先调用 `syncExpiredEligibility()`，直到待处理到期记录全部同步后再卖出。
7. 付款预览使用调用时的Pair储备，只代表当前应付数量。发起购买时仍应按用户确认的容忍度设置`maxMob`或`maxNcn`，防止报价变化导致超额支付。
