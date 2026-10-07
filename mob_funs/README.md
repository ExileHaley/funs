### MOB:0xd8339aDBA626bbB395bb10c92F8599E6E7021408
### Mining NFT:0x8e723f48EC1c32eE9EE56D6A77bC2C9f9542B046
### Purchase NFT:0xD034A38B783E17864fdD8940C24d39e1037939Fc

### Purchase方法列表
```solidity
enum MiningCardType {
    U100, //100usdt的卡牌
    U300, //300usdt的卡牌
    U1000,//1000usdt的卡牌
    U3000,//3000usdt的卡牌
    U10000//10000usdt的卡牌
}

//查询邀请人地址
function inviterOf(address) external view returns(address);
//查询卡牌价格，cardType是卡牌类型，ncnAmount所需NCN代币数量，usdtAmount所需usdt数量
function paymentAmounts(MiningCardType cardType) public pure returns (uint256 ncnAmount, uint256 usdtAmount);
//购买卡牌
function purchaseCard(MiningCardType cardType) external;
//操作员免费铸造，不需要支付，管理功能
function operatorMint(MiningCardType cardType) external;
//操作员批量发送空投，管理功能
//token要发送的代币合约地址
//recipients要发送给哪些地址
//amount每个地址要发送多少代币
function batchTransferErc20(address token, address[] calldata recipients, uint256 amount) external;
//邀请功能，通过inviterOf查询inviter对应的地址是否为0判断inviter是否有邀请资格，然后就可以执行
function bindInviter(address inviter) external;
```
