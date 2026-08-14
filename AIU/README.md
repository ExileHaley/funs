# AIU recharge contract

## address test version
- AIU: 0x3aAA386076C4D180314fD3e43ED51df7F893e579
- Recharge: 0x383959c16Ce178Af2b06F8D6EaC8B18E9b8AB691
- pool: 0x32B18B95A4964917Ae8caf1C220870B9E89Abe9A
- exchange: 0xe757F8439AC52d50B74E5D34C96f3CfE6dbb36a8
## func
```shell
//remark从后端拿到标识传进来，或者传空跟着后端业务走
//token要充值的代币合约地址，支持aiu和usdt都要对该合约进行授权
//amount充值数量，USDT的数量

function recharge(string calldata remark, address token, uint256 amount) external;
```
