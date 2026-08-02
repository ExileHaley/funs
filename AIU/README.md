# AIU recharge contract

## address test version
- AIU: 0x13915222E551d294a47D2cC37871A4A2aB950aa1
- Recharge: 0x567827970376b386137e464d28084fcA424E1CAA
- pool: 0xa06f6B163B7Ba8624D052E631Bc48AD9e764Ed88
- exchange: 0xBD005451D85A237c7866089a9E7bc96d9BB48E25
## func
```shell
//remark从后端拿到标识传进来，或者传空跟着后端业务走
//token要充值的代币合约地址，支持aiu和usdt都要对该合约进行授权
//amount充值数量，USDT的数量

function recharge(string calldata remark, address token, uint256 amount) external;
```
