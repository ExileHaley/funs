### == online contract address ==
  #### Lex token: 0xBf9CfFd9523b8F9808058321e3521121c1194bad
  #### Lex`s pair: 0x22C98318c5cd13B842e8E65Cfc89A56e6ff51172
  #### Leo token: 0xc9ADcCAa0984796431E3352cb7423e2309357ba9
  #### Leo`s pair: 0x284E03fC6c8364E3147ADA4d457cB2763677d895
  #### USDT test token: 0x55d398326f99059fF775485246999027B3197955
  =========================================================
  #### Treasury: 0xA5cA72bE7bba63260b9BbF5a0A645D97D11E5D54
  #### TreasuryLiquidity: 0xBDbd2D9040f000572710d91BEe669015DD7FbD01
  #### NodeDividends: 0x18A749dc4488c46733F86Ef747edc0BA955B6cEc
  #### Queue: 0xc816b2C2522EB06DBf36fFE902c3313F76B533E4
  #### Referrals: 0x310e1E50b0f42Ade0898B14D83839da374697C06
  #### Router: 0x3DcB9452D460321E0ba99a002C51a0Abf4E2D748
  #### Exchange: 0xaf91451c13ABCA2480aC90EC59Efb450467FA179
  #### Payback: 0x5B66C805BE6F14Ed9aAf6Bd7Abcc98fB6e659550
==================================================================================================================
### == final test version ==
  #### Lex token: 0x975c6B9A2D8755e33C99042348150b4D2cA5163B
  #### Lex`s pair: 0xe6DA957092Dc3b64B5Af689A935128423468667C
  #### Leo token: 0x238Ac53Be4cC0936Eb2AdAA2Cda2429723C6Da83
  #### Leo`s pair: 0xF2187B9F6D6216C291aaB84762E092a8C1661422
  #### USDT test token: 0x55d398326f99059fF775485246999027B3197955
  =========================================================
  #### Treasury: 0x18f285cd6d75caDf2eA44f74FDcD5d3368898117
  #### TreasuryLiquidity: 0xb8C3D9157e8a30cF4DAC519704f3d3302656B5d0
  #### NodeDividends: 0xD0c3E7c81B09f0C44ed5521De360F26Aa04ceb8A
  #### Queue: 0x1AeF06CFEAC82aE4e3FAA18a282FD6A306B786A1
  #### Referrals: 0x71f014e4aFEf2e30cdA7b6f2Aee08ccC035566a8
  #### Router: 0xa16EC857AbB697eaFADB8C7bBdd13B193786244A
  #### Exchange: 0x3866882De8A78D4e685d1b9C0808bc8854FD0e4c
  #### Payback: 0x97F23F6f0aEC7A6f04680473cDA9d0520EA8f6d5
  =========================================================
### 用户方法列表
#### router func list
```solidity
//查询当前地址剩余的总质押额度
function getRemainingQuota(address user) public view returns(uint256);
//获取当前地址的体验单剩余质押额度
function getTrialRemainingQuota(address user) public view returns(uint256);
// 获取首码地址
function rootAddr() external view returns(address);
// 验证当前地址是否有邀请资格
function verifyReferralCode(address user) external view returns(bool);
// 获取USDT地址
function USDT() external view returns(address);
// 绑定邀请关系，parent是邀请人的地址
function referral(address parent) external;
// amount要质押的数量，stakeIndex要小于4，0/1/2/3代表1天、15天、30天、90天，这里质押的代币是usdt，包括赎回等
function stake(uint256 amount, uint8 stakeIndex) external;
// 通过orderIndex订单编号赎回订单
function unstake(uint256 orderIndex) external;
// 通过orderIndex订单编号和newStakeIndex(质押类型索引)对已有订单进行重新质押
function restake(uint256 orderIndex, uint8 newStakeIndex) external;
// 取消排队订单，根据排队订单编号queueOrderId取消排队订单
function cancelQueue(uint256 queueOrderId) external;
// 根据订单编号orderIndex提取订单收益
function claim(uint256 orderIndex) external;
struct Order {
        uint128 amount; //质押数量
        uint128 claimed; //正常收益已提取部分
        uint32  claimedPeriods; 
        uint32  startTime;  //订单开始时间
        uint8   stakeIndex; //订单类型，0、1、2、3分别是1天，15天，30天，90天
        uint8   status; // 0代表质押中、1代表已赎回

        uint32  createdAt; //订单创建时间

        uint128 frozenReward; //冻结的总奖励，熔断后
        uint128 frozenClaimed; //冻结奖励释放并且已提取的数量
        uint32  freezeStart; //冻结时间
        uint32  freezeRound; //冻结轮次
}
// TreasuryRules.Order[]的索引就是订单Id(orderIndex)
function getUserOrders(address user) external view returns(TreasuryRules.Order[] memory);
// 通过订单编号和用户地址获取订单倒计时和状态等
struct RuleResult {
        uint256 claimCountdown; //收益提取倒计时
        uint256 unstakeCountdown; //赎回倒计时
        bool canClaim;  //是否可以提取收益，可用作按钮置灰判断
        bool canUnstake; //是否可以赎回，可用作按钮置灰判断
        bool canRestake; //是否可以重新质押，可用作按钮置灰判断
        bool isFrozen; //当前订单是否熔断或冻结
        bool isExpired; //当前订单是否超过12个月
        bool isMatured; //当前订单是否到期
}
//是当前订单真实可提取收益，result是上述结构体
function getOrderStatus(address user, uint256 orderIndex) 
        external 
        view 
        returns(
            uint256 truthAward,
            Models.RuleResult memory result
);
// 获取用户的所有订单，返回order结构体数组
function getUserOrders(address user) external view returns(TreasuryRules.Order[] memory);
// 根据排队订单编号获取获取排队详情
function getQueueInfo(uint256 queueId) 
        external 
        view 
        returns(
            address user,   //当前排队订单拥有者
            uint256 amount, //当前排队数量
            uint256 orderIndex, //订单编号，重新质押并进入排队的订单才会有值，通过isRestake判断
            uint8 stakeIndex,//当前排队订单要质押的类型或者说期限，这里类型代表的就是不同的期限
            uint256 createdAt,//当前排队订单创建时间
            bool isRestake, //是否是旧订单重新质押
            uint8 status    //当前排队订单的状态，0排队中，1已经质押，2已取消
        );
//获取当前时间的质押额度以及赎回额度
function getCurrentQuota() 
        external 
        view 
        returns(
            uint256 stakeUsed, //当前已使用的质押额度
            uint256 stakeRemaining, //当前剩余的可质押额度
            uint256 unstakeUsed, //当前已使用的赎回额度
            uint256 unstakeRemaining //当前剩余的可赎回额度
        );
//获取fomo池奖励
function getFomoInfo() 
        external 
        view 
        returns(
            uint256 roundId,    //当前属于第几轮奖励
            address[21] memory lastQualified, //当前轮游资格参与奖励的21个用户地址，有可能不足21个
            uint256 count,  //当前有资格参与奖励的地址数量
            uint256 poolBalance //当前轮次的fomo奖池总量
        );

struct DirectReferral{
        address user; //用户地址
        uint256 staked; //当前用户的总质押
        uint256 performance; //当前用户的伞下业绩
}
//获取直推信息
function getDirectReferralInfo(address user) 
        external 
        view 
    returns(Models.DirectReferral[] memory);
//获取邀请信息
enum LevelType{INVALID, L1, L2, L3, L4, L5, L6, L7}
function getReferralInfo(address user) 
        external 
        view 
        returns(
            address parent, //当前用户的邀请人地址
            Models.LevelType level,//当前用户的级别，INVALID就是没有级别
            uint16 referralNum, //当前用户的总邀请人数
            uint256 totalStaked, //当前用户的总质押数量
            uint256 performance //当前用户的伞下业绩
        );

//获取排队订单的首尾序号
function getQueueRange() external view returns(uint256 head, uint256 tail);
//duration质押的期限，claimInterval多长时间领取一次收益，window领取收益的时间窗口，rate是收益计算比例不用展示
function getStakePlans(uint8 stakeIndex) 
        external 
        view returns(
            uint32 duration,
            uint32 claimInterval,
            uint32 window,
            uint64 rate
        );
struct FomoAwards{
        uint256 rounds; //奖励来源的轮次
        uint256 amount; //本轮次奖励的数量
    }
//fomo奖励详情
function getFomoAwardsInfo(address user) external view returns(Models.FomoAwards[] memory);
```


#### nodeDividends func list
```solidity
//如何计算累计？
//1.代币手续费累积目前分红 = getTaxFeeAward + taxExtracted
//2.代币盈利税累积目前分红 = getProfitFeeAward + profitExtracted
//3.质押手续费累积目前分红 = getStakeFeeAward + stakeExtracted
////invalid(0/无效)，envoy(1/大使)，director(2/股东)，partner(3/合伙人)
enum NodeType{INVALID, ENVOY, DIRECTOR, PARTNER}
function userInfo(address user) external view returns(
        Models.NodeType nodeType, //节点类型
        uint256 stakingAmount, //节点对应的价格
        uint128 taxExtracted, //已提取的代币滑点分红
        uint128 profitExtracted, //已提取的代币盈利税分红
        uint128 stakeExtracted, //已提取的质押分红数量
        uint128 extracted, //总共已提取
        uint128 debtTax,
        uint128 debtProfit,
        uint128 debtStake,
        bool isOut //是否出局
);
//获取用户可提取收益
function getUserAward(address user) public view returns(uint256);
//用户获得的代币手续费分红
function getTaxFeeAward(address user) public view returns(uint256);
//用户获得的盈利税分红
function getProfitFeeAward(address user) public view returns(uint256);
//用户获得的做市商手续费分红
function getStakeFeeAward(address user) public view returns(uint256);
//提取收益
function claim() external;


```


#### exchange func list
```solidity
// 当前用户拥有的token购买额度，根据用户地址以及代币地址获取
function availableLimit(address user, address token) external view returns(uint256);
// 预估兑换结果，比如买入lex，就是fromToken==USDT，toToken==lex，fromAmount == usdt的数量
function getAmountOut(address fromToken, address toToken, uint256 fromAmount) external view returns(uint256);
// 卖出子币，这里用于卖出leo，获取预估结果的时候就是fromToken==leo，toToken==USDT， fromAmount == leo的数量，这里需要leo授权
function sellSubCoin(uint256 amountLeo) external;
// 买入代币，token这里是要买入的代币地址，比如lex或者leo，amountUsdt是买入多少USDT的代币，所以这里需要USDT授权
function buy(address token, uint256 amountUsdt) external;
```


#### payback func list
```solidity
//用户提取收益
function claim() external;
//用户信息查询，质押总额staking(USDT)，已提取的extracted(leo)，usdtValue已提取的leo价值多少USDT，当前真实可提取收益truthAward(leo)
function getUserInfo(address user) external view returns(uint256 staking, uint256 usdtValue, uint256 extracted, uint256 truthAward);

```





### 管理员方法列表
#### queue合约
```solidity
//获取管理地址
function admin() external view returns(address);
//管理员方法，用来设置质押和赎回额度的比例

//这里20%就是 2000 / 10000，分母是万，这里没有精度
function setQuotaRatios(uint256 _stakeRatio, uint256 _unstakeRatio) external;

//设置质押手续费和取消手续费
function setFees(uint256 _stakeFee, uint256 _cancelFee) external;

//fomo池管理员开奖方法
function drawFomoRewards() external;

//手动把排队订单按照额度和顺序刷进国库合约质押，每次执行5个有效排队订单，额度不足则一个都刷不进去
function emergencyProcessQueue() external;
```

#### router合约
```solidity
//获取管理员地址
function owner() external view returns(address);
//设置封顶质押数量，amount带上18位精度进来
function setPersonalQuota(uint256 amount) external;
//查询当前设置的个人总额度
function totalPersonalQuota() external view returns(uint256);
```

#### exchange合约
```solidity
///////////////////////////////////////管理员方法////////////////////////////////////////////////
//获取管理员地址
function admin() external view returns(address);
//管理员批量减少token购买额度，users[i]的token对应购买额度减少amountUsdt，amountUsdt带上18位精度进来
function subAvailableLimit(address token, address[] memory users, uint256 amountUsdt) external;
//管理员批量增加token购买额度，users[i]的token对应购买额度增加amountUsdt，amountUsdt带上18位精度进来
function addAvailableLimit(address token, address[] memory users, uint256 amountUsdt) external;
```

#### referrals合约
```solidity
//这里是个view方法谁都能调用，直接给对接到管理端即可
//获取级别从L1到L7的所有用户地址
function getLevelUsers() external view returns(address[] memory);
//通过级别用户的地址，批量获取对应地址的小区额度
function getEffectivePerformance(address[] memory users)
        external
        view
        returns (Models.Effective[] memory);
```

#### payback合约
```solidity
//获取管理员地址
function admin() external view returns(address);
//给指定地址增加数量，amount把精度带进来18位，user用户地址
function add(address user, uint256 amount) external;
//给指定地址减少数量，amount把精度带进来18位，user用户地址
function reduce(address user, uint256 amount) external;
//设置分红开关，true表示开启leo分红，false代表关闭leo分红
function setIsDividends(bool _isDividends) external;
```

#### lex合约
```solidity
//查询管理员地址
function owner() external view returns(address);
//开关lex的pancake买入功能，true表示开启，false表示关闭
function switchBuy(bool _b) external;
//开关lex的pancake卖出功能，true表示开启，false表示关闭
function switchSell(bool _b) external;
//设置买入卖出比例，这里没有精度，_buy买入手续费，_sell卖出手续费，都不允许大于15，只能输入整数
function setRate(uint256 _buy, uint256 _sell) external;
//查询买入手续费百分比
function buyRate() external view returns(uint256);
////查询卖出手续费百分比
function sellRate() external view returns(uint256);
//管理员分发买入手续费比例
function issueBuyTaxFee() external;
```
#### leo合约
```solidity
//查询管理员地址
function owner() external view returns(address);
//这里输入的数字没有精度，小于100代表开启买入，大于等于100代表关闭买入交易
function setRate(uint256 _buyRate) external;
//查询买入手续费百分比，目前是100%，禁止买入
function buyRate() external view returns(uint256);
//管理员分发买入收手续费
function issueBuyTaxFee() external;
//管理员销毁每天0.5%
function burnFromPair() external;
```
