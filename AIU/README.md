# AIU recharge contract

## address test version
- AIU: 0xEc1F0B33Df61bca7b612ea51165c3D3CD2d0371a
- Recharge: 0x9b690452060A649c3BdD9450854bef44B6cb34B8

## func
```shell
//remark从后端拿到标识传进来，或者传空跟着后端业务走
//amount充值数量，USDT的数量
//usdt需要对该合约进行授权
function recharge(string calldata remark, uint256 amount) external;
```