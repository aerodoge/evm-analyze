# Go-Ethereum EVM 详解（九）：闪电贷深度剖析

## 目录

- [65. 闪电贷基础概念](#65-闪电贷基础概念)
- [66. 主流闪电贷协议](#66-主流闪电贷协议)
- [67. 闪电贷套利实现](#67-闪电贷套利实现)
- [68. 闪电贷清算](#68-闪电贷清算)
- [69. 高级闪电贷技巧](#69-高级闪电贷技巧)
- [70. 闪电贷风险与防护](#70-闪电贷风险与防护)

---

## 65. 闪电贷基础概念

### 65.1 什么是闪电贷？

```
┌─────────────────────────────────────────────────────────────────┐
│                     闪电贷核心概念                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  闪电贷 = 在一个交易内借款并还款的无抵押贷款                           │
│                                                                 │
│  核心特性：                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 原子性：借款和还款必须在同一交易内完成                    │   │
│  │  2. 无抵押：不需要任何抵押品                                │   │
│  │  3. 无风险（对贷方）：不还款则整个交易回滚                    │   │
│  │  4. 即时：无需等待，立即可用                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  交易流程：                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                         │   │
│  │  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐           │   │
│  │  │ 借款 │───▶ │ 使用 │───▶ │ 获利 │───▶│ 还款 │           │   │
│  │  │ 100  │    │资金做 │    │      │    │100+  │           │   │
│  │  │ ETH  │    │ 套利  │    │      │    │ 手续费│          │   │
│  │  └──────┘    └──────┘    └──────┘    └──────┘          │   │
│  │      │                                   │              │   │
│  │      └────────── 同一交易 ────────────────┘              │   │
│  │                                                         │   │
│  │  如果还款失败 → 整个交易回滚 → 借款从未发生                   │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 65.2 闪电贷的原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    闪电贷工作原理                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EVM 原子性保证：                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  function flashLoan(amount) {                           │   │
│  │      // 1. 记录初始余额                                  │   │
│  │      uint256 balanceBefore = token.balanceOf(this);     │   │
│  │                                                          │   │
│  │      // 2. 转账给借款人                                  │   │
│  │      token.transfer(borrower, amount);                  │   │
│  │                                                          │   │
│  │      // 3. 调用借款人的回调函数                          │   │
│  │      borrower.executeOperation(amount, fee, params);    │   │
│  │                                                          │   │
│  │      // 4. 检查还款（核心！）                            │   │
│  │      uint256 balanceAfter = token.balanceOf(this);      │   │
│  │      require(balanceAfter >= balanceBefore + fee);      │   │
│  │      // 如果检查失败，整个交易 revert                    │   │
│  │  }                                                       │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  关键点：                                                       │
│  • 在回调函数中，借款人可以做任何事情                          │
│  • 只要最后余额检查通过，交易就成功                            │
│  • 如果余额不够，require 失败，所有状态回滚                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 65.3 闪电贷的应用场景

```
┌─────────────────────────────────────────────────────────────────┐
│                   闪电贷应用场景                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 套利（Arbitrage）                                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  借款 ──▶ 低价买入 ──▶ 高价卖出 ──▶ 还款 + 利润              │    │
│  │  无需本金，利用价格差异获利                                  │   │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  2. 清算（Liquidation）                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  借款 ──▶ 偿还不良贷款 ──▶ 获得抵押品 ──▶ 卖出还款            │   │
│  │  无需持有债务代币，获取清算奖励                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. 抵押品置换（Collateral Swap）                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  借款 ──▶ 还清贷款 ──▶ 取出抵押品 ──▶ 换新抵押品              │   │
│  │       ──▶ 重新借贷 ──▶ 还闪电贷                            │   │
│  │  一笔交易完成抵押品更换                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  4. 杠杆调整（Leverage Adjustment）                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  增加杠杆：借款 ──▶ 买入更多抵押品 ──▶ 存入 ──▶ 借出还款       │   │
│  │  减少杠杆：借款 ──▶ 还部分贷款 ──▶ 取出抵押品 ──▶ 卖出还款     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  5. 自清算（Self-Liquidation）                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  借款 ──▶ 还清贷款 ──▶ 取出抵押品 ──▶ 卖出还款               │   │
│  │  避免被他人清算损失清算罚金                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 66. 主流闪电贷协议

### 66.1 Aave V3 闪电贷

```go
// Aave V3 闪电贷接口

// IPool 接口
type AavePool interface {
// 简单闪电贷（单资产）
FlashLoanSimple(
receiverAddress common.Address,
asset common.Address,
amount *big.Int,
params []byte,
referralCode uint16,
) error

// 多资产闪电贷
FlashLoan(
receiverAddress common.Address,
assets []common.Address,
amounts []*big.Int,
interestRateModes []*big.Int, // 0=不借款, 1=稳定利率, 2=浮动利率
onBehalfOf common.Address,
params []byte,
referralCode uint16,
) error
}

// IFlashLoanSimpleReceiver 回调接口
type IFlashLoanSimpleReceiver interface {
ExecuteOperation(
asset common.Address,
amount *big.Int,
premium *big.Int, // 手续费
initiator common.Address,
params []byte,
) (bool, error)
}

// Aave V3 闪电贷费用
const AaveV3FlashLoanFee = 5 // 0.05% (5 basis points)

// Aave V3 闪电贷合约地址（主网）
var (
AaveV3Pool = common.HexToAddress("0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2")
AaveV3PoolDataProvider = common.HexToAddress("0x7B4EB56E7CD4b454BA8ff71E4518426369a138a3")
)

// 实现 Aave 闪电贷接收器
type AaveFlashLoanReceiver struct {
pool      common.Address
initiator common.Address
}

// ExecuteOperation Aave 回调函数
func (r *AaveFlashLoanReceiver) ExecuteOperation(
asset common.Address,
amount *big.Int,
premium *big.Int,
initiator common.Address,
params []byte,
) (bool, error) {
// 验证调用者
// require(msg.sender == pool)
// require(initiator == r.initiator)

// ============================================
// 在这里执行套利/清算等操作
// ============================================

// 例如：DEX 套利
profit, err := r.executeArbitrage(asset, amount, params)
if err != nil {
return false, err
}

// 计算还款金额
amountOwed := new(big.Int).Add(amount, premium)

// 确保有足够余额还款
// 需要先 approve 给 Pool
// token.approve(pool, amountOwed)

return true, nil
}

// Solidity 合约示例
const AaveFlashLoanReceiverSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@aave/v3-core/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";
import "@aave/v3-core/contracts/interfaces/IPoolAddressesProvider.sol";

contract MyFlashLoanReceiver is FlashLoanSimpleReceiverBase {
    constructor(IPoolAddressesProvider provider)
        FlashLoanSimpleReceiverBase(provider)
    {}

    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        // 1. 解码参数
        (address targetDex, bytes memory swapData) = abi.decode(params, (address, bytes));

        // 2. 执行套利操作
        // ...

        // 3. 计算还款金额
        uint256 amountOwed = amount + premium;

        // 4. 批准还款
        IERC20(asset).approve(address(POOL), amountOwed);

        return true;
    }

    // 发起闪电贷
    function requestFlashLoan(address asset, uint256 amount, bytes calldata params) external {
        POOL.flashLoanSimple(
            address(this),  // receiverAddress
            asset,
            amount,
            params,
            0  // referralCode
        );
    }
}
`
```

### 66.2 Uniswap V2/V3 Flash Swap

```go
// Uniswap V2 Flash Swap

// IUniswapV2Callee 回调接口
type IUniswapV2Callee interface {
UniswapV2Call(
sender common.Address,
amount0 *big.Int,
amount1 *big.Int,
data []byte,
)
}

// Uniswap V2 闪电贷原理
// swap 函数允许先接收代币，再在回调中支付
/*
   function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external {
       // 1. 转出代币
       if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out);
       if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out);

       // 2. 如果 data 非空，触发回调
       if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);

       // 3. 检查余额（K 值）
       uint balance0 = IERC20(_token0).balanceOf(address(this));
       uint balance1 = IERC20(_token1).balanceOf(address(this));

       // 关键检查：扣除手续费后 K 值必须不减少
       require(balance0 * balance1 >= reserve0 * reserve1, 'UniswapV2: K');
   }
*/

// Uniswap V2 闪电贷实现
type UniswapV2FlashSwap struct {
factory common.Address
weth    common.Address
}

// UniswapV2Call 回调函数
func (u *UniswapV2FlashSwap) UniswapV2Call(
sender common.Address,
amount0 *big.Int,
amount1 *big.Int,
data []byte,
) {
// 1. 解析调用数据
// 确定借入的是哪个代币

// 2. 执行套利操作
// ...

// 3. 计算还款金额（考虑 0.3% 手续费）
// amountRequired = (amountBorrowed * 1000) / 997 + 1
// 或者用另一个代币还款

// 4. 转账还款
}

// Uniswap V2 闪电贷的特点
/*
   优点：
   - 手续费只有 0.3%（如果用同币种还款）
   - 可以用另一种代币还款
   - 不需要额外的闪电贷手续费

   缺点：
   - 需要计算精确的还款金额
   - K 值检查可能比较复杂
*/

// Uniswap V3 Flash
type IUniswapV3Pool interface {
Flash(
recipient common.Address,
amount0 *big.Int,
amount1 *big.Int,
data []byte,
) error
}

// IUniswapV3FlashCallback 回调接口
type IUniswapV3FlashCallback interface {
UniswapV3FlashCallback(
fee0 *big.Int,
fee1 *big.Int,
data []byte,
)
}

// V3 闪电贷 Solidity 示例
const UniswapV3FlashSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@uniswap/v3-core/contracts/interfaces/IUniswapV3Pool.sol";
import "@uniswap/v3-core/contracts/interfaces/callback/IUniswapV3FlashCallback.sol";

contract UniswapV3Flash is IUniswapV3FlashCallback {
    IUniswapV3Pool public pool;

    function initiateFlash(uint256 amount0, uint256 amount1) external {
        pool.flash(
            address(this),
            amount0,
            amount1,
            abi.encode(msg.sender)
        );
    }

    function uniswapV3FlashCallback(
        uint256 fee0,
        uint256 fee1,
        bytes calldata data
    ) external override {
        require(msg.sender == address(pool), "not pool");

        // 执行操作...

        // 还款：本金 + 手续费
        if (fee0 > 0) {
            IERC20(pool.token0()).transfer(address(pool), amount0 + fee0);
        }
        if (fee1 > 0) {
            IERC20(pool.token1()).transfer(address(pool), amount1 + fee1);
        }
    }
}
`
```

### 66.3 Balancer 闪电贷

```go
// Balancer 闪电贷（免费！）

// IBalancerVault 接口
type IBalancerVault interface {
FlashLoan(
recipient IFlashLoanRecipient,
tokens []common.Address,
amounts []*big.Int,
userData []byte,
) error
}

// IFlashLoanRecipient 回调接口
type IFlashLoanRecipient interface {
ReceiveFlashLoan(
tokens []common.Address,
amounts []*big.Int,
feeAmounts []*big.Int,
userData []byte,
)
}

// Balancer Vault 地址（所有链相同）
var BalancerVault = common.HexToAddress("0xBA12222222228d8Ba445958a75a0704d566BF2C8")

// Balancer 闪电贷特点
/*
   优点：
   - 免费！没有手续费（除非协议治理改变）
   - 支持多资产同时借款
   - 流动性充足

   缺点：
   - 并非所有代币都有流动性
   - 需要检查可用流动性
*/

// Balancer 闪电贷 Solidity 示例
const BalancerFlashLoanSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@balancer-labs/v2-interfaces/contracts/vault/IVault.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IFlashLoanRecipient.sol";

contract BalancerFlashLoan is IFlashLoanRecipient {
    IVault public vault;

    constructor(IVault _vault) {
        vault = _vault;
    }

    function initiateFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        bytes memory userData
    ) external {
        vault.flashLoan(this, tokens, amounts, userData);
    }

    function receiveFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        uint256[] memory feeAmounts,
        bytes memory userData
    ) external override {
        require(msg.sender == address(vault), "not vault");

        // 执行套利操作
        // ...

        // 还款（Balancer 通常免费，feeAmounts 为 0）
        for (uint256 i = 0; i < tokens.length; i++) {
            tokens[i].transfer(address(vault), amounts[i] + feeAmounts[i]);
        }
    }
}
`

// 比较各协议闪电贷
type FlashLoanComparison struct {
Protocol    string
Fee         string
MaxAmount   string
MultiAsset  bool
Notes       string
}

var FlashLoanProviders = []FlashLoanComparison{
{"Aave V3", "0.05%", "池中可用流动性", true, "最常用，支持多资产"},
{"Uniswap V2", "0.3%（swap费）", "池中储备量", false, "通过flash swap实现"},
{"Uniswap V3", "池的费率", "池中流动性", true, "费率取决于池子"},
{"Balancer", "0%", "Vault中流动性", true, "免费！推荐优先使用"},
{"dYdX", "0%", "池中可用", false, "也是免费，但流动性较少"},
{"MakerDAO", "0%", "Dai供应量", false, "只能借Dai"},
}
```

---

## 67. 闪电贷套利实现

### 67.1 基础套利流程

```go
// FlashLoanArbitrage 闪电贷套利
type FlashLoanArbitrage struct {
// 闪电贷提供者
aavePool    common.Address
balancerVault common.Address

// DEX 路由器
uniswapRouter  common.Address
sushiRouter    common.Address

// 配置
minProfit   *big.Int
maxGasPrice *big.Int

client *ethclient.Client
}

// ArbitrageParams 套利参数
type ArbitrageParams struct {
// 借款信息
LoanToken   common.Address
LoanAmount  *big.Int
LoanProvider string // "aave", "balancer", "uniswap"

// 套利路径
BuyDex      string
SellDex     string
BuyPath     []common.Address
SellPath    []common.Address

// 预期结果
ExpectedProfit *big.Int
MaxSlippage    uint64 // basis points
}

// ExecuteArbitrage 执行套利
func (f *FlashLoanArbitrage) ExecuteArbitrage(ctx context.Context, params *ArbitrageParams) (*types.Transaction, error) {
// 1. 验证套利机会仍然存在
currentProfit, err := f.simulateArbitrage(params)
if err != nil {
return nil, fmt.Errorf("simulation failed: %w", err)
}
if currentProfit.Cmp(f.minProfit) < 0 {
return nil, fmt.Errorf("profit too low: %s", currentProfit.String())
}

// 2. 构建套利合约调用数据
callData, err := f.buildArbitrageCalldata(params)
if err != nil {
return nil, err
}

// 3. 估算 Gas
gasLimit, err := f.estimateGas(callData)
if err != nil {
return nil, err
}

// 4. 构建并发送交易
tx, err := f.sendTransaction(ctx, callData, gasLimit)
if err != nil {
return nil, err
}

return tx, nil
}

// simulateArbitrage 模拟套利
func (f *FlashLoanArbitrage) simulateArbitrage(params *ArbitrageParams) (*big.Int, error) {
// 模拟完整的套利流程

// 1. 计算在 BuyDex 的输出
buyOutput, err := f.getAmountOut(params.BuyDex, params.LoanAmount, params.BuyPath)
if err != nil {
return nil, err
}

// 2. 计算在 SellDex 的输出
sellOutput, err := f.getAmountOut(params.SellDex, buyOutput, params.SellPath)
if err != nil {
return nil, err
}

// 3. 计算闪电贷费用
fee := f.calculateFlashLoanFee(params.LoanProvider, params.LoanAmount)

// 4. 计算净利润
// profit = sellOutput - loanAmount - fee
profit := new(big.Int).Sub(sellOutput, params.LoanAmount)
profit.Sub(profit, fee)

return profit, nil
}

// buildArbitrageCalldata 构建套利调用数据
func (f *FlashLoanArbitrage) buildArbitrageCalldata(params *ArbitrageParams) ([]byte, error) {
// 编码套利参数，传递给闪电贷回调
innerParams := encodeArbitrageParams(params)

switch params.LoanProvider {
case "balancer":
return f.buildBalancerFlashLoan(params.LoanToken, params.LoanAmount, innerParams)
case "aave":
return f.buildAaveFlashLoan(params.LoanToken, params.LoanAmount, innerParams)
default:
return nil, fmt.Errorf("unknown loan provider: %s", params.LoanProvider)
}
}
```

### 67.2 套利合约实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@aave/v3-core/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";
import "@uniswap/v2-periphery/contracts/interfaces/IUniswapV2Router02.sol";

contract FlashLoanArbitrage is FlashLoanSimpleReceiverBase {
    address public owner;

    // DEX 路由器
    IUniswapV2Router02 public uniswapRouter;
    IUniswapV2Router02 public sushiRouter;

    constructor(
        IPoolAddressesProvider provider,
        address _uniswapRouter,
        address _sushiRouter
    ) FlashLoanSimpleReceiverBase(provider) {
        owner = msg.sender;
        uniswapRouter = IUniswapV2Router02(_uniswapRouter);
        sushiRouter = IUniswapV2Router02(_sushiRouter);
    }

    struct ArbitrageParams {
        address buyRouter;
        address sellRouter;
        address[] buyPath;
        address[] sellPath;
        uint256 minProfit;
    }

    // 发起闪电贷套利
    function executeArbitrage(
        address asset,
        uint256 amount,
        ArbitrageParams calldata arbParams
    ) external {
        require(msg.sender == owner, "not owner");

        bytes memory params = abi.encode(arbParams);

        POOL.flashLoanSimple(
            address(this),
            asset,
            amount,
            params,
            0
        );
    }

    // 闪电贷回调
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        require(msg.sender == address(POOL), "not pool");
        require(initiator == address(this), "not initiator");

        ArbitrageParams memory arbParams = abi.decode(params, (ArbitrageParams));

        // 1. 在第一个 DEX 买入
        IERC20(asset).approve(arbParams.buyRouter, amount);

        uint256[] memory buyAmounts = IUniswapV2Router02(arbParams.buyRouter)
            .swapExactTokensForTokens(
            amount,
            0,  // 接受任何数量（实际应设置滑点保护）
            arbParams.buyPath,
            address(this),
            block.timestamp
        );

        uint256 boughtAmount = buyAmounts[buyAmounts.length - 1];

        // 2. 在第二个 DEX 卖出
        address middleToken = arbParams.buyPath[arbParams.buyPath.length - 1];
        IERC20(middleToken).approve(arbParams.sellRouter, boughtAmount);

        uint256[] memory sellAmounts = IUniswapV2Router02(arbParams.sellRouter)
            .swapExactTokensForTokens(
            boughtAmount,
            0,
            arbParams.sellPath,
            address(this),
            block.timestamp
        );

        uint256 finalAmount = sellAmounts[sellAmounts.length - 1];

        // 3. 检查利润
        uint256 amountOwed = amount + premium;
        require(finalAmount >= amountOwed + arbParams.minProfit, "not profitable");

        // 4. 批准还款
        IERC20(asset).approve(address(POOL), amountOwed);

        // 5. 利润留在合约中（稍后提取）
        return true;
    }

    // 提取利润
    function withdrawProfit(address token) external {
        require(msg.sender == owner, "not owner");
        uint256 balance = IERC20(token).balanceOf(address(this));
        IERC20(token).transfer(owner, balance);
    }

    // 紧急提取
    function emergencyWithdraw(address token, uint256 amount) external {
        require(msg.sender == owner, "not owner");
        IERC20(token).transfer(owner, amount);
    }
}
```

### 67.3 多协议闪电贷选择

```go
// FlashLoanSelector 选择最优闪电贷提供者
type FlashLoanSelector struct {
providers map[string]*FlashLoanProvider
}

type FlashLoanProvider struct {
Name        string
Address     common.Address
GetFee      func(amount *big.Int) *big.Int
GetMaxLoan  func (token common.Address) *big.Int
IsAvailable func (token common.Address) bool
}

// SelectBestProvider 选择最优提供者
func (s *FlashLoanSelector) SelectBestProvider(token common.Address, amount *big.Int) (string, error) {
var bestProvider string
var lowestFee *big.Int

for name, provider := range s.providers {
// 检查是否支持该代币
if !provider.IsAvailable(token) {
continue
}

// 检查流动性是否足够
maxLoan := provider.GetMaxLoan(token)
if maxLoan.Cmp(amount) < 0 {
continue
}

// 计算费用
fee := provider.GetFee(amount)

// 选择费用最低的
if lowestFee == nil || fee.Cmp(lowestFee) < 0 {
lowestFee = fee
bestProvider = name
}
}

if bestProvider == "" {
return "", errors.New("no available flash loan provider")
}

return bestProvider, nil
}

// 初始化提供者
func NewFlashLoanSelector(client *ethclient.Client) *FlashLoanSelector {
return &FlashLoanSelector{
providers: map[string]*FlashLoanProvider{
"balancer": {
Name:    "Balancer",
Address: BalancerVault,
GetFee: func (amount *big.Int) *big.Int {
return big.NewInt(0) // 免费
},
// ...
},
"aave": {
Name:    "Aave V3",
Address: AaveV3Pool,
GetFee: func (amount *big.Int) *big.Int {
// 0.05%
fee := new(big.Int).Mul(amount, big.NewInt(5))
return fee.Div(fee, big.NewInt(10000))
},
// ...
},
},
}
}
```

---

## 68. 闪电贷清算

### 68.1 清算原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    闪电贷清算原理                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  传统清算问题：                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 需要持有债务代币才能清算                                 │   │
│  │  • 资金效率低（大量资金闲置）                                │   │
│  │  • 大额清算需要大量本金                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  闪电贷清算流程：                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  1. 借款（债务代币）                                       │   │
│  │     │                                                    │   │
│  │     ▼                                                    │   │
│  │  2. 执行清算（偿还债务）                                    │   │
│  │     │                                                    │   │
│  │     ▼                                                    │   │
│  │  3. 获得抵押品 + 清算奖励                                   │   │
│  │     │                                                    │   │
│  │     ▼                                                    │   │
│  │  4. 卖出抵押品换成债务代币                                  │   │
│  │     │                                                    │   │
│  │     ▼                                                    │   │
│  │  5. 还闪电贷 + 保留利润                                     │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  利润来源：                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  清算奖励（通常 5-10%）- 闪电贷费用 - 交易滑点 - Gas    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 68.2 Aave 清算实现

```go
// AaveLiquidator Aave 闪电贷清算
type AaveLiquidator struct {
pool          common.Address
oracle        common.Address
dataProvider  common.Address
swapRouter    common.Address

client        *ethclient.Client
}

// LiquidationParams 清算参数
type LiquidationParams struct {
User            common.Address // 被清算用户
CollateralAsset common.Address // 抵押品代币
DebtAsset       common.Address // 债务代币
DebtToCover     *big.Int        // 要偿还的债务数量
ReceiveAToken   bool            // 是否接收 aToken
}

// ExecuteLiquidation 执行清算
func (l *AaveLiquidator) ExecuteLiquidation(ctx context.Context, params *LiquidationParams) error {
// 1. 验证清算机会
isLiquidatable, err := l.checkLiquidatable(params.User)
if err != nil || !isLiquidatable {
return errors.New("position not liquidatable")
}

// 2. 计算最优清算金额
optimalAmount, expectedProfit, err := l.calculateOptimalLiquidation(params)
if err != nil {
return err
}

// 3. 使用 Balancer 闪电贷（免费）
userData := l.encodeLiquidationParams(params)

// 4. 构建闪电贷交易
tx, err := l.buildFlashLoanTx(params.DebtAsset, optimalAmount, userData)
if err != nil {
return err
}

// 5. 发送交易
return l.sendTransaction(ctx, tx)
}

// 清算合约 Solidity
const AaveLiquidatorSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@balancer-labs/v2-interfaces/contracts/vault/IVault.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IFlashLoanRecipient.sol";
import "@aave/v3-core/contracts/interfaces/IPool.sol";
import "@uniswap/v2-periphery/contracts/interfaces/IUniswapV2Router02.sol";

contract AaveLiquidator is IFlashLoanRecipient {
    IVault public vault;
    IPool public aavePool;
    IUniswapV2Router02 public swapRouter;
    address public owner;

    struct LiquidationParams {
        address user;
        address collateralAsset;
        address debtAsset;
        uint256 debtToCover;
    }

    constructor(
        address _vault,
        address _aavePool,
        address _swapRouter
    ) {
        vault = IVault(_vault);
        aavePool = IPool(_aavePool);
        swapRouter = IUniswapV2Router02(_swapRouter);
        owner = msg.sender;
    }

    // 发起清算
    function liquidate(LiquidationParams calldata params) external {
        require(msg.sender == owner, "not owner");

        IERC20[] memory tokens = new IERC20[](1);
        tokens[0] = IERC20(params.debtAsset);

        uint256[] memory amounts = new uint256[](1);
        amounts[0] = params.debtToCover;

        vault.flashLoan(
            this,
            tokens,
            amounts,
            abi.encode(params)
        );
    }

    // 闪电贷回调
    function receiveFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        uint256[] memory feeAmounts,
        bytes memory userData
    ) external override {
        require(msg.sender == address(vault), "not vault");

        LiquidationParams memory params = abi.decode(userData, (LiquidationParams));

        // 1. 批准 Aave Pool
        tokens[0].approve(address(aavePool), amounts[0]);

        // 2. 执行清算
        aavePool.liquidationCall(
            params.collateralAsset,
            params.debtAsset,
            params.user,
            params.debtToCover,
            false  // receiveAToken = false, 直接获得抵押品
        );

        // 3. 获得的抵押品数量
        uint256 collateralReceived = IERC20(params.collateralAsset)
            .balanceOf(address(this));

        // 4. 将抵押品换成债务代币
        IERC20(params.collateralAsset).approve(address(swapRouter), collateralReceived);

        address[] memory path = new address[](2);
        path[0] = params.collateralAsset;
        path[1] = params.debtAsset;

        swapRouter.swapExactTokensForTokens(
            collateralReceived,
            amounts[0] + feeAmounts[0],  // 最少要能还款
            path,
            address(this),
            block.timestamp
        );

        // 5. 还款给 Balancer（免费）
        tokens[0].transfer(address(vault), amounts[0] + feeAmounts[0]);

        // 6. 剩余利润留在合约中
    }

    // 提取利润
    function withdrawProfit(address token) external {
        require(msg.sender == owner, "not owner");
        uint256 balance = IERC20(token).balanceOf(address(this));
        if (balance > 0) {
            IERC20(token).transfer(owner, balance);
        }
    }
}
`
```

---

## 69. 高级闪电贷技巧

### 69.1 多协议组合闪电贷

```go
// 组合多个闪电贷提供者以获得更大的借款额度

type CompositeFlashLoan struct {
// 可以从多个来源借款
providers []FlashLoanProvider
}

// 策略：嵌套闪电贷
/*
场景：需要借 200 ETH，但单个协议最多只能借 100 ETH

解决方案：
1. 从 Aave 借 100 ETH
2. 在 Aave 回调中，从 Balancer 再借 100 ETH
3. 使用 200 ETH 执行操作
4. 在 Balancer 回调中还款
5. 在 Aave 回调中还款
*/

const NestedFlashLoanSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract NestedFlashLoan is IFlashLoanSimpleReceiver, IFlashLoanRecipient {
    IPool public aavePool;
    IVault public balancerVault;

    // Aave 闪电贷入口
    function initiateNestedFlashLoan(uint256 aaveAmount, uint256 balancerAmount) external {
        bytes memory params = abi.encode(balancerAmount);

        aavePool.flashLoanSimple(
            address(this),
            WETH,
            aaveAmount,
            params,
            0
        );
    }

    // Aave 回调
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        uint256 balancerAmount = abi.decode(params, (uint256));

        // 在 Aave 回调中发起 Balancer 闪电贷
        IERC20[] memory tokens = new IERC20[](1);
        tokens[0] = IERC20(asset);

        uint256[] memory amounts = new uint256[](1);
        amounts[0] = balancerAmount;

        // 传递 Aave 还款信息
        bytes memory userData = abi.encode(amount, premium);

        balancerVault.flashLoan(this, tokens, amounts, userData);

        // Balancer 回调完成后，批准 Aave 还款
        IERC20(asset).approve(address(aavePool), amount + premium);

        return true;
    }

    // Balancer 回调
    function receiveFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        uint256[] memory feeAmounts,
        bytes memory userData
    ) external override {
        // 现在有 Aave + Balancer 的总借款
        uint256 totalBorrowed = IERC20(address(tokens[0])).balanceOf(address(this));

        // 执行需要大量资金的操作
        executeHighValueOperation(totalBorrowed);

        // 解码 Aave 还款信息
        (uint256 aaveAmount, uint256 aavePremium) = abi.decode(userData, (uint256, uint256));

        // 还 Balancer（免费）
        tokens[0].transfer(address(balancerVault), amounts[0]);

        // Aave 还款在 Aave 回调中处理
    }
}
`
```

### 69.2 闪电贷 + 杠杆

```go
// 使用闪电贷创建杠杆仓位

/*
目标：用 1 ETH 创建 3x 杠杆仓位

传统方式：
1. 存 1 ETH
2. 借出 0.75 ETH 等值的 USDC
3. 将 USDC 换成 0.75 ETH
4. 存入 0.75 ETH
5. 借出 0.56 ETH...
6. 重复多次...

闪电贷方式：
1. 闪电贷借 2 ETH
2. 存入 3 ETH（1 自有 + 2 借的）
3. 借出 2 ETH 等值的 USDC
4. 将 USDC 换成 2 ETH
5. 还闪电贷

一笔交易完成 3x 杠杆！
*/

const LeveragePositionSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LeverageCreator is IFlashLoanRecipient {
    IVault public vault;
    IPool public aavePool;
    ISwapRouter public swapRouter;

    // 创建杠杆仓位
    function createLeveragedPosition(
        address collateral,  // 抵押品（如 WETH）
        address debt,        // 债务（如 USDC）
        uint256 ownAmount,   // 自有资金
        uint256 leverage     // 杠杆倍数（如 3 表示 3x）
    ) external {
        // 计算需要闪电贷的金额
        uint256 flashAmount = ownAmount * (leverage - 1);

        // 先转入用户的自有资金
        IERC20(collateral).transferFrom(msg.sender, address(this), ownAmount);

        // 发起闪电贷
        IERC20[] memory tokens = new IERC20[](1);
        tokens[0] = IERC20(collateral);

        uint256[] memory amounts = new uint256[](1);
        amounts[0] = flashAmount;

        bytes memory userData = abi.encode(msg.sender, collateral, debt, ownAmount, leverage);

        vault.flashLoan(this, tokens, amounts, userData);
    }

    function receiveFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        uint256[] memory feeAmounts,
        bytes memory userData
    ) external override {
        (
            address user,
            address collateral,
            address debt,
            uint256 ownAmount,
            uint256 leverage
        ) = abi.decode(userData, (address, address, address, uint256, uint256));

        uint256 totalCollateral = ownAmount + amounts[0];

        // 1. 存入全部抵押品到 Aave
        IERC20(collateral).approve(address(aavePool), totalCollateral);
        aavePool.supply(collateral, totalCollateral, user, 0);

        // 2. 代表用户借出债务代币
        uint256 debtAmount = calculateDebtAmount(amounts[0], collateral, debt);
        aavePool.borrow(debt, debtAmount, 2, 0, user);

        // 3. 将债务代币换成抵押品代币
        IERC20(debt).approve(address(swapRouter), debtAmount);
        uint256 swappedAmount = swapRouter.exactInputSingle(
            ISwapRouter.ExactInputSingleParams({
                tokenIn: debt,
                tokenOut: collateral,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: debtAmount,
                amountOutMinimum: amounts[0],  // 至少要能还闪电贷
                sqrtPriceLimitX96: 0
            })
        );

        // 4. 还闪电贷
        tokens[0].transfer(address(vault), amounts[0] + feeAmounts[0]);
    }
}
`
```

### 69.3 闪电铸造（Flash Minting）

```go
// 某些代币支持闪电铸造（如 DAI）
// 可以在一个交易内铸造任意数量的代币

/*
DAI 闪电铸造特点：
- 无上限（只要能还回来）
- 费用：0%（治理可调整）
- 直接从协议铸造，不依赖池子流动性
*/

// IDSS Flash 接口（MakerDAO）
type IDssFlash interface {
FlashLoan(
receiver IFlashReceiver,
token common.Address,
amount *big.Int,
data []byte,
) error

MaxFlashLoan(token common.Address) *big.Int
FlashFee(token common.Address, amount *big.Int) *big.Int
}

const DaiFlashMintSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDssFlash {
    function flashLoan(
        address receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool);
}

interface IERC3156FlashBorrower {
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32);
}

contract DaiFlashMint is IERC3156FlashBorrower {
    IDssFlash public dssFlash = IDssFlash(0x1EB4CF3A948E7D72A198fe073cCb8C7a948cD853);
    address public dai = 0x6B175474E89094C44Da98b954EesdeCD73E01Deb;

    bytes32 constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    function flashMint(uint256 amount, bytes calldata data) external {
        dssFlash.flashLoan(address(this), dai, amount, data);
    }

    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external override returns (bytes32) {
        require(msg.sender == address(dssFlash), "not dss flash");
        require(initiator == address(this), "not initiator");

        // 执行操作...
        // 可以铸造任意数量的 DAI

        // 还款
        IERC20(dai).approve(address(dssFlash), amount + fee);

        return CALLBACK_SUCCESS;
    }
}
`
```

---

## 70. 闪电贷风险与防护

### 70.1 闪电贷攻击类型

```
┌─────────────────────────────────────────────────────────────────┐
│                   闪电贷攻击类型                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 价格操纵攻击                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  攻击流程：                                              │   │
│  │  借款 ──▶ 大量交易操纵价格 ──▶ 利用错误价格获利 ──▶ 还款 │   │
│  │                                                          │   │
│  │  目标：使用 AMM 价格作为预言机的协议                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  2. 重入攻击 + 闪电贷                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  闪电贷提供大量资金                                      │   │
│  │  在状态更新前重复调用                                    │   │
│  │  放大重入攻击的效果                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. 治理攻击                                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  借款获得大量治理代币                                    │   │
│  │  在同一交易内投票                                        │   │
│  │  通过恶意提案                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. 清算攻击                                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  操纵价格使仓位可清算                                    │   │
│  │  闪电贷执行清算                                          │   │
│  │  获取清算奖励                                            │   │
│  │  恢复价格                                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 70.2 防护措施

```go
// 协议防护措施

// 1. 使用时间加权平均价格（TWAP）
type TWAPOracle struct {
observations []PriceObservation
period       uint32 // 观察期（如 30 分钟）
}

func (t *TWAPOracle) GetPrice() *big.Int {
// 计算过去 period 时间内的平均价格
// 单个交易无法操纵
return calculateTWAP(t.observations, t.period)
}

// 2. 使用外部预言机
type ChainlinkOracle struct {
aggregator common.Address
}

func (c *ChainlinkOracle) GetPrice() (*big.Int, error) {
// Chainlink 价格不受 DEX 操纵
return c.fetchLatestPrice()
}

// 3. 延迟执行
type DelayedExecution struct {
delay time.Duration
}

// 重要操作需要等待，防止闪电贷攻击
func (d *DelayedExecution) Execute(action Action) {
// 提案时记录
d.queueAction(action)

// 延迟后才能执行
time.Sleep(d.delay)

d.executeAction(action)
}

// 4. 快照治理
type SnapshotGovernance struct {
snapshotBlock uint64
}

// 使用历史快照进行投票，防止闪电贷治理攻击
func (s *SnapshotGovernance) Vote(proposal uint64, voter common.Address) error {
// 检查快照区块时的余额
balance := s.getBalanceAtBlock(voter, s.snapshotBlock)
// 闪电贷无法影响历史快照
return nil
}

// 5. 闪电贷保护修饰符
const FlashLoanGuardSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract FlashLoanGuard {
    // 记录每个区块的首次调用
    mapping(address => uint256) private lastBlock;

    modifier noFlashLoan() {
        require(lastBlock[msg.sender] != block.number, "no flash loan");
        lastBlock[msg.sender] = block.number;
        _;
    }

    // 敏感函数使用此修饰符
    function sensitiveOperation() external noFlashLoan {
        // 同一区块内不能重复调用
    }
}
`

// 6. 借贷协议保护
const LendingProtectionSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LendingProtection {
    // 使用多个价格源
    IOracle public chainlinkOracle;
    IOracle public uniswapTWAP;
    IOracle public bandOracle;

    function getPrice(address asset) public view returns (uint256) {
        uint256 chainlinkPrice = chainlinkOracle.getPrice(asset);
        uint256 twapPrice = uniswapTWAP.getPrice(asset);
        uint256 bandPrice = bandOracle.getPrice(asset);

        // 取中位数
        return median(chainlinkPrice, twapPrice, bandPrice);
    }

    // 价格偏差检查
    function checkPriceDeviation(address asset) public view returns (bool) {
        uint256 spotPrice = getSpotPrice(asset);
        uint256 oraclePrice = getPrice(asset);

        uint256 deviation = abs(spotPrice - oraclePrice) * 10000 / oraclePrice;

        // 如果偏差超过 5%，可能存在操纵
        return deviation <= 500;
    }
}
`
```

### 70.3 闪电贷攻击案例分析

```go
// 著名闪电贷攻击案例

type FlashLoanAttack struct {
Name        string
Date        string
Loss        string
Protocol    string
Description string
TxHash      string
}

var NotableAttacks = []FlashLoanAttack{
{
Name:     "bZx Attack #1",
Date:     "2020-02-15",
Loss:     "$350K",
Protocol: "bZx",
Description: `
            1. 从 dYdX 借 10,000 ETH
            2. 5,500 ETH 存入 Compound 借 112 WBTC
            3. 1,300 ETH 存入 bZx 开 5x 空单
            4. 空单推高 Uniswap WBTC 价格
            5. 在 Uniswap 高价卖出 112 WBTC
            6. 还款并获利
        `,
},
{
Name:     "Harvest Finance",
Date:     "2020-10-26",
Loss:     "$34M",
Protocol: "Harvest",
Description: `
            1. 借大量 USDC/USDT
            2. 操纵 Curve Y Pool 价格
            3. 低价存入 Harvest，获得更多份额
            4. 恢复价格
            5. 高价取出
            6. 重复多次
        `,
},
{
Name:     "Cream Finance",
Date:     "2021-10-27",
Loss:     "$130M",
Protocol: "Cream",
Description: `
            1. 闪电贷借大量资金
            2. 利用 yUSD 价格预言机漏洞
            3. 操纵 yUSD 价格
            4. 用被高估的 yUSD 作为抵押借出其他资产
            5. 获利并还款
        `,
},
{
Name:     "Euler Finance",
Date:     "2023-03-13",
Loss:     "$197M",
Protocol: "Euler",
Description: `
            1. 闪电贷借 DAI
            2. 存入 Euler
            3. 利用清算逻辑漏洞
            4. 自清算获取超额抵押品
            5. 重复并获利
        `,
},
}

// 从攻击中学习
type AttackLessons struct {
Lesson      string
Prevention  string
}

var Lessons = []AttackLessons{
{
Lesson:     "不要使用现货价格作为预言机",
Prevention: "使用 TWAP 或 Chainlink",
},
{
Lesson:     "价格更新需要延迟",
Prevention: "使用时间加权或多区块确认",
},
{
Lesson:     "治理需要快照机制",
Prevention: "使用历史区块余额投票",
},
{
Lesson:     "清算逻辑需要仔细审计",
Prevention: "考虑闪电贷场景的边界条件",
},
}
```

---

## 总结

本文档详细介绍了闪电贷机制：

1. **基础概念**：原子性、无抵押、即时借款
2. **主流协议**：Aave、Uniswap、Balancer 的闪电贷实现
3. **套利实现**：完整的闪电贷套利合约和流程
4. **清算应用**：无本金清算的实现方式
5. **高级技巧**：嵌套闪电贷、杠杆创建、闪电铸造
6. **风险防护**：攻击类型和防护措施

闪电贷是 DeFi 最强大的工具之一，善用可以实现无本金套利，但也要了解其风险。

下一篇将介绍 DEX 数学模型。

---

## 附录 A：新兴借贷协议闪电贷

### A.1 Morpho 闪电贷

Morpho 是建立在 Aave/Compound 之上的借贷优化层，提供了独特的闪电贷接口。

```go
// Morpho 闪电贷客户端
package flashloan

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Morpho Blue 合约地址
var (
	MorphoBlueAddress = common.HexToAddress("0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb")
)

// MorphoFlashLoan Morpho Blue 闪电贷实现
type MorphoFlashLoan struct {
	client   *ethclient.Client
	morpho   *bind.BoundContract
	morphoABI abi.ABI
}

// Morpho Blue 闪电贷接口
const MorphoBlueABI = `[
	{
		"name": "flashLoan",
		"type": "function",
		"inputs": [
			{"name": "token", "type": "address"},
			{"name": "assets", "type": "uint256"},
			{"name": "data", "type": "bytes"}
		],
		"outputs": []
	},
	{
		"name": "idleMarketParams",
		"type": "function",
		"inputs": [{"name": "id", "type": "bytes32"}],
		"outputs": [
			{"name": "loanToken", "type": "address"},
			{"name": "collateralToken", "type": "address"},
			{"name": "oracle", "type": "address"},
			{"name": "irm", "type": "address"},
			{"name": "lltv", "type": "uint256"}
		]
	}
]`

func NewMorphoFlashLoan(client *ethclient.Client) (*MorphoFlashLoan, error) {
	parsedABI, err := abi.JSON(strings.NewReader(MorphoBlueABI))
	if err != nil {
		return nil, err
	}

	contract := bind.NewBoundContract(MorphoBlueAddress, parsedABI, client, client, client)

	return &MorphoFlashLoan{
		client:    client,
		morpho:    contract,
		morphoABI: parsedABI,
	}, nil
}

// FlashLoan 执行 Morpho Blue 闪电贷
// Morpho Blue 特点：零手续费闪电贷
func (m *MorphoFlashLoan) FlashLoan(
	ctx context.Context,
	token common.Address,
	amount *big.Int,
	receiver common.Address,
	callbackData []byte,
	opts *bind.TransactOpts,
) (*types.Transaction, error) {
	// Morpho Blue 闪电贷完全免费
	// 只需在同一交易内还款即可

	return m.morpho.Transact(opts, "flashLoan", token, amount, callbackData)
}

// GetAvailableLiquidity 获取可用流动性
func (m *MorphoFlashLoan) GetAvailableLiquidity(
	ctx context.Context,
	token common.Address,
) (*big.Int, error) {
	// 查询 token 在 Morpho Blue 中的总供应
	balance, err := m.getTokenBalance(ctx, token)
	if err != nil {
		return nil, err
	}
	return balance, nil
}

// MorphoFlashLoanCallback Morpho 闪电贷回调合约
const MorphoFlashLoanCallbackSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {IMorpho} from "@morpho-org/morpho-blue/src/interfaces/IMorpho.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract MorphoFlashLoanReceiver {
    IMorpho public immutable morpho;

    constructor(address _morpho) {
        morpho = IMorpho(_morpho);
    }

    // Morpho Blue 闪电贷回调
    function onMorphoFlashLoan(uint256 assets, bytes calldata data) external {
        require(msg.sender == address(morpho), "not morpho");

        // 解码数据
        (address token, address target, bytes memory payload) =
            abi.decode(data, (address, address, bytes));

        // 执行套利逻辑
        (bool success,) = target.call(payload);
        require(success, "arbitrage failed");

        // 还款 - Morpho Blue 零手续费
        IERC20(token).approve(address(morpho), assets);
    }

    // 执行闪电贷套利
    function executeFlashLoan(
        address token,
        uint256 amount,
        address target,
        bytes calldata payload
    ) external {
        bytes memory data = abi.encode(token, target, payload);
        morpho.flashLoan(token, amount, data);
    }
}
`
```

### A.2 Euler Finance 闪电贷

Euler V2 采用模块化设计，提供灵活的闪电贷功能。

```go
// Euler V2 闪电贷客户端
package flashloan

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Euler V2 合约地址
var (
	EulerVaultFactoryAddress = common.HexToAddress("0x29a56a1b8214D9Cf7c5561811750D5cBDb45CC8e")
)

// EulerFlashLoan Euler V2 闪电贷
type EulerFlashLoan struct {
	client    *ethclient.Client
	vaultABI  abi.ABI
}

// Euler Vault 闪电贷接口
const EulerVaultABI = `[
	{
		"name": "flashLoan",
		"type": "function",
		"inputs": [
			{"name": "amount", "type": "uint256"},
			{"name": "data", "type": "bytes"}
		],
		"outputs": []
	},
	{
		"name": "asset",
		"type": "function",
		"inputs": [],
		"outputs": [{"name": "", "type": "address"}]
	},
	{
		"name": "totalAssets",
		"type": "function",
		"inputs": [],
		"outputs": [{"name": "", "type": "uint256"}]
	},
	{
		"name": "maxFlashLoan",
		"type": "function",
		"inputs": [{"name": "token", "type": "address"}],
		"outputs": [{"name": "", "type": "uint256"}]
	},
	{
		"name": "flashFee",
		"type": "function",
		"inputs": [
			{"name": "token", "type": "address"},
			{"name": "amount", "type": "uint256"}
		],
		"outputs": [{"name": "", "type": "uint256"}]
	}
]`

// EulerVault 单个 Vault 的闪电贷封装
type EulerVault struct {
	Address common.Address
	Asset   common.Address
	client  *EulerFlashLoan
	contract *bind.BoundContract
}

func NewEulerFlashLoan(client *ethclient.Client) (*EulerFlashLoan, error) {
	parsedABI, err := abi.JSON(strings.NewReader(EulerVaultABI))
	if err != nil {
		return nil, err
	}

	return &EulerFlashLoan{
		client:   client,
		vaultABI: parsedABI,
	}, nil
}

// GetVault 获取特定资产的 Vault
func (e *EulerFlashLoan) GetVault(vaultAddress common.Address) (*EulerVault, error) {
	contract := bind.NewBoundContract(vaultAddress, e.vaultABI, e.client, e.client, e.client)

	// 获取 asset
	var results []interface{}
	err := contract.Call(nil, &results, "asset")
	if err != nil {
		return nil, err
	}

	return &EulerVault{
		Address:  vaultAddress,
		Asset:    results[0].(common.Address),
		client:   e,
		contract: contract,
	}, nil
}

// MaxFlashLoan 获取最大可借额度
func (v *EulerVault) MaxFlashLoan(ctx context.Context) (*big.Int, error) {
	var results []interface{}
	err := v.contract.Call(&bind.CallOpts{Context: ctx}, &results, "maxFlashLoan", v.Asset)
	if err != nil {
		return nil, err
	}
	return results[0].(*big.Int), nil
}

// FlashFee 获取闪电贷费用
func (v *EulerVault) FlashFee(ctx context.Context, amount *big.Int) (*big.Int, error) {
	var results []interface{}
	err := v.contract.Call(&bind.CallOpts{Context: ctx}, &results, "flashFee", v.Asset, amount)
	if err != nil {
		return nil, err
	}
	return results[0].(*big.Int), nil
}

// Euler V2 遵循 EIP-3156 标准
const EulerFlashLoanReceiverSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import {IERC3156FlashLender} from "@openzeppelin/contracts/interfaces/IERC3156FlashLender.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract EulerFlashLoanReceiver is IERC3156FlashBorrower {
    // EIP-3156 回调函数签名
    bytes32 public constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    // Euler Vault 地址映射
    mapping(address => address) public vaults;  // token => vault

    // EIP-3156 标准回调
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external override returns (bytes32) {
        require(vaults[token] == msg.sender, "untrusted lender");
        require(initiator == address(this), "untrusted initiator");

        // 解码并执行套利
        (address target, bytes memory payload) = abi.decode(data, (address, bytes));
        (bool success,) = target.call(payload);
        require(success, "arbitrage failed");

        // 还款 = 本金 + 手续费
        IERC20(token).approve(msg.sender, amount + fee);

        return CALLBACK_SUCCESS;
    }

    // 发起闪电贷
    function flashBorrow(
        address token,
        uint256 amount,
        address target,
        bytes calldata payload
    ) external {
        address vault = vaults[token];
        require(vault != address(0), "no vault");

        bytes memory data = abi.encode(target, payload);
        IERC3156FlashLender(vault).flashLoan(
            IERC3156FlashBorrower(address(this)),
            token,
            amount,
            data
        );
    }
}
`
```

### A.3 dYdX 闪电贷

dYdX Solo Margin 提供了独特的闪电贷机制，通过操作序列实现。

```go
// dYdX Solo Margin 闪电贷
package flashloan

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// dYdX Solo Margin 地址 (Ethereum Mainnet)
var (
	DYDXSoloMarginAddress = common.HexToAddress("0x1E0447b19BB6EcFdAe1e4AE1694b0C3659614e4e")
)

// dYdX 市场 ID
const (
	DYDX_MARKET_WETH = 0
	DYDX_MARKET_DAI  = 1  // 实际是 SAI
	DYDX_MARKET_USDC = 2
	DYDX_MARKET_DAI2 = 3  // 真正的 DAI
)

// DYDXFlashLoan dYdX 闪电贷客户端
type DYDXFlashLoan struct {
	client    *ethclient.Client
	soloABI   abi.ABI
	solo      *bind.BoundContract
}

// dYdX Solo Margin ABI
const DYDXSoloABI = `[
	{
		"name": "operate",
		"type": "function",
		"inputs": [
			{
				"name": "accounts",
				"type": "tuple[]",
				"components": [
					{"name": "owner", "type": "address"},
					{"name": "number", "type": "uint256"}
				]
			},
			{
				"name": "actions",
				"type": "tuple[]",
				"components": [
					{"name": "actionType", "type": "uint8"},
					{"name": "accountId", "type": "uint256"},
					{"name": "amount", "type": "tuple", "components": [
						{"name": "sign", "type": "bool"},
						{"name": "denomination", "type": "uint8"},
						{"name": "ref", "type": "uint8"},
						{"name": "value", "type": "uint256"}
					]},
					{"name": "primaryMarketId", "type": "uint256"},
					{"name": "secondaryMarketId", "type": "uint256"},
					{"name": "otherAddress", "type": "address"},
					{"name": "otherAccountId", "type": "uint256"},
					{"name": "data", "type": "bytes"}
				]
			}
		],
		"outputs": []
	},
	{
		"name": "getMarketTokenAddress",
		"type": "function",
		"inputs": [{"name": "marketId", "type": "uint256"}],
		"outputs": [{"name": "", "type": "address"}]
	}
]`

// ActionType dYdX 操作类型
type ActionType uint8

const (
	ActionDeposit    ActionType = 0
	ActionWithdraw   ActionType = 1
	ActionTransfer   ActionType = 2
	ActionBuy        ActionType = 3
	ActionSell       ActionType = 4
	ActionTrade      ActionType = 5
	ActionLiquidate  ActionType = 6
	ActionVaporize   ActionType = 7
	ActionCall       ActionType = 8  // 闪电贷用这个
)

// AccountInfo dYdX 账户信息
type AccountInfo struct {
	Owner  common.Address
	Number *big.Int
}

// AssetAmount 资产数量
type AssetAmount struct {
	Sign         bool     // true = positive, false = negative
	Denomination uint8    // 0 = Wei, 1 = Par
	Ref          uint8    // 0 = Delta, 1 = Target
	Value        *big.Int
}

// ActionArgs 操作参数
type ActionArgs struct {
	ActionType        ActionType
	AccountId         *big.Int
	Amount            AssetAmount
	PrimaryMarketId   *big.Int
	SecondaryMarketId *big.Int
	OtherAddress      common.Address
	OtherAccountId    *big.Int
	Data              []byte
}

func NewDYDXFlashLoan(client *ethclient.Client) (*DYDXFlashLoan, error) {
	parsedABI, err := abi.JSON(strings.NewReader(DYDXSoloABI))
	if err != nil {
		return nil, err
	}

	contract := bind.NewBoundContract(DYDXSoloMarginAddress, parsedABI, client, client, client)

	return &DYDXFlashLoan{
		client:  client,
		soloABI: parsedABI,
		solo:    contract,
	}, nil
}

// BuildFlashLoanActions 构建闪电贷操作序列
// dYdX 闪电贷原理：
// 1. Withdraw - 借出资金
// 2. Call - 执行自定义逻辑
// 3. Deposit - 还款
// 三个操作在同一交易内原子执行
func (d *DYDXFlashLoan) BuildFlashLoanActions(
	marketId *big.Int,
	amount *big.Int,
	callee common.Address,
	data []byte,
) []ActionArgs {
	actions := make([]ActionArgs, 3)

	// Action 1: Withdraw (借出)
	actions[0] = ActionArgs{
		ActionType: ActionWithdraw,
		AccountId:  big.NewInt(0),
		Amount: AssetAmount{
			Sign:         false, // negative = withdraw
			Denomination: 0,     // Wei
			Ref:          0,     // Delta
			Value:        amount,
		},
		PrimaryMarketId:   marketId,
		SecondaryMarketId: big.NewInt(0),
		OtherAddress:      callee,
		OtherAccountId:    big.NewInt(0),
		Data:              nil,
	}

	// Action 2: Call (执行自定义逻辑)
	actions[1] = ActionArgs{
		ActionType:        ActionCall,
		AccountId:         big.NewInt(0),
		Amount:            AssetAmount{},
		PrimaryMarketId:   big.NewInt(0),
		SecondaryMarketId: big.NewInt(0),
		OtherAddress:      callee,
		OtherAccountId:    big.NewInt(0),
		Data:              data,
	}

	// Action 3: Deposit (还款)
	// dYdX 闪电贷需要还款 amount + 2 wei (防止舍入误差)
	repayAmount := new(big.Int).Add(amount, big.NewInt(2))
	actions[2] = ActionArgs{
		ActionType: ActionDeposit,
		AccountId:  big.NewInt(0),
		Amount: AssetAmount{
			Sign:         true, // positive = deposit
			Denomination: 0,
			Ref:          0,
			Value:        repayAmount,
		},
		PrimaryMarketId:   marketId,
		SecondaryMarketId: big.NewInt(0),
		OtherAddress:      callee,
		OtherAccountId:    big.NewInt(0),
		Data:              nil,
	}

	return actions
}

// dYdX 闪电贷回调合约
const DYDXFlashLoanReceiverSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

// dYdX Solo Margin 接口
interface ISoloMargin {
    struct AccountInfo {
        address owner;
        uint256 number;
    }

    struct AssetAmount {
        bool sign;
        uint8 denomination;
        uint8 ref;
        uint256 value;
    }

    struct ActionArgs {
        uint8 actionType;
        uint256 accountId;
        AssetAmount amount;
        uint256 primaryMarketId;
        uint256 secondaryMarketId;
        address otherAddress;
        uint256 otherAccountId;
        bytes data;
    }

    function operate(
        AccountInfo[] calldata accounts,
        ActionArgs[] calldata actions
    ) external;

    function getMarketTokenAddress(uint256 marketId) external view returns (address);
}

// dYdX Callee 接口
interface ICallee {
    function callFunction(
        address sender,
        AccountInfo calldata accountInfo,
        bytes calldata data
    ) external;
}

contract DYDXFlashLoanReceiver is ICallee {
    ISoloMargin public immutable soloMargin;

    // 市场 ID
    uint256 constant MARKET_WETH = 0;
    uint256 constant MARKET_USDC = 2;
    uint256 constant MARKET_DAI = 3;

    constructor(address _soloMargin) {
        soloMargin = ISoloMargin(_soloMargin);
    }

    // dYdX 回调函数
    function callFunction(
        address sender,
        AccountInfo calldata accountInfo,
        bytes calldata data
    ) external override {
        require(msg.sender == address(soloMargin), "not solo margin");
        require(sender == address(this), "not this contract");

        // 解码套利参数
        (
            uint256 marketId,
            uint256 amount,
            address target,
            bytes memory payload
        ) = abi.decode(data, (uint256, uint256, address, bytes));

        // 执行套利逻辑
        (bool success,) = target.call(payload);
        require(success, "arbitrage failed");

        // 还款准备 - dYdX 需要额外 2 wei
        address token = soloMargin.getMarketTokenAddress(marketId);
        uint256 repayAmount = amount + 2;
        IERC20(token).approve(address(soloMargin), repayAmount);
    }

    // 发起 dYdX 闪电贷
    function initiateFlashLoan(
        uint256 marketId,
        uint256 amount,
        address target,
        bytes calldata payload
    ) external {
        // 构建账户
        ISoloMargin.AccountInfo[] memory accounts = new ISoloMargin.AccountInfo[](1);
        accounts[0] = ISoloMargin.AccountInfo({
            owner: address(this),
            number: 1
        });

        // 构建操作序列
        ISoloMargin.ActionArgs[] memory actions = new ISoloMargin.ActionArgs[](3);

        // 1. Withdraw
        actions[0] = ISoloMargin.ActionArgs({
            actionType: 1, // Withdraw
            accountId: 0,
            amount: ISoloMargin.AssetAmount({
                sign: false,
                denomination: 0,
                ref: 0,
                value: amount
            }),
            primaryMarketId: marketId,
            secondaryMarketId: 0,
            otherAddress: address(this),
            otherAccountId: 0,
            data: ""
        });

        // 2. Call
        bytes memory callData = abi.encode(marketId, amount, target, payload);
        actions[1] = ISoloMargin.ActionArgs({
            actionType: 8, // Call
            accountId: 0,
            amount: ISoloMargin.AssetAmount({
                sign: false,
                denomination: 0,
                ref: 0,
                value: 0
            }),
            primaryMarketId: 0,
            secondaryMarketId: 0,
            otherAddress: address(this),
            otherAccountId: 0,
            data: callData
        });

        // 3. Deposit
        actions[2] = ISoloMargin.ActionArgs({
            actionType: 0, // Deposit
            accountId: 0,
            amount: ISoloMargin.AssetAmount({
                sign: true,
                denomination: 0,
                ref: 0,
                value: amount + 2
            }),
            primaryMarketId: marketId,
            secondaryMarketId: 0,
            otherAddress: address(this),
            otherAccountId: 0,
            data: ""
        });

        soloMargin.operate(accounts, actions);
    }
}
`
```

### A.4 MakerDAO DAI Flash Mint

MakerDAO 提供了 DAI 的闪电铸造功能，可以在单笔交易内无限铸造 DAI。

```go
// MakerDAO Flash Mint 客户端
package flashloan

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// MakerDAO 合约地址
var (
	DaiFlashMintAddress = common.HexToAddress("0x1EB4CF3A948E7D72A198fe073cCb8C7a948cD853")
	DaiTokenAddress     = common.HexToAddress("0x6B175474E89094C44Da98b954EescdeCB5c3cD")
)

// DaiFlashMint DAI 闪电铸造客户端
type DaiFlashMint struct {
	client      *ethclient.Client
	flashMint   *bind.BoundContract
	flashABI    abi.ABI
}

// DAI Flash Mint ABI (EIP-3156)
const DaiFlashMintABI = `[
	{
		"name": "flashLoan",
		"type": "function",
		"inputs": [
			{"name": "receiver", "type": "address"},
			{"name": "token", "type": "address"},
			{"name": "amount", "type": "uint256"},
			{"name": "data", "type": "bytes"}
		],
		"outputs": [{"name": "", "type": "bool"}]
	},
	{
		"name": "maxFlashLoan",
		"type": "function",
		"inputs": [{"name": "token", "type": "address"}],
		"outputs": [{"name": "", "type": "uint256"}]
	},
	{
		"name": "flashFee",
		"type": "function",
		"inputs": [
			{"name": "token", "type": "address"},
			{"name": "amount", "type": "uint256"}
		],
		"outputs": [{"name": "", "type": "uint256"}]
	},
	{
		"name": "dai",
		"type": "function",
		"inputs": [],
		"outputs": [{"name": "", "type": "address"}]
	},
	{
		"name": "vat",
		"type": "function",
		"inputs": [],
		"outputs": [{"name": "", "type": "address"}]
	},
	{
		"name": "max",
		"type": "function",
		"inputs": [],
		"outputs": [{"name": "", "type": "uint256"}]
	},
	{
		"name": "toll",
		"type": "function",
		"inputs": [],
		"outputs": [{"name": "", "type": "uint256"}]
	}
]`

func NewDaiFlashMint(client *ethclient.Client) (*DaiFlashMint, error) {
	parsedABI, err := abi.JSON(strings.NewReader(DaiFlashMintABI))
	if err != nil {
		return nil, err
	}

	contract := bind.NewBoundContract(DaiFlashMintAddress, parsedABI, client, client, client)

	return &DaiFlashMint{
		client:    client,
		flashMint: contract,
		flashABI:  parsedABI,
	}, nil
}

// MaxFlashLoan 获取最大可铸造 DAI 数量
// 这是一个很大的数字，理论上可以铸造任意数量（受限于合约配置）
func (d *DaiFlashMint) MaxFlashLoan(ctx context.Context) (*big.Int, error) {
	var results []interface{}
	err := d.flashMint.Call(&bind.CallOpts{Context: ctx}, &results, "maxFlashLoan", DaiTokenAddress)
	if err != nil {
		return nil, err
	}
	return results[0].(*big.Int), nil
}

// FlashFee 获取闪电铸造费用
// DAI Flash Mint 费用 = amount * toll / WAD
// 当前 toll = 0，即免费
func (d *DaiFlashMint) FlashFee(ctx context.Context, amount *big.Int) (*big.Int, error) {
	var results []interface{}
	err := d.flashMint.Call(&bind.CallOpts{Context: ctx}, &results, "flashFee", DaiTokenAddress, amount)
	if err != nil {
		return nil, err
	}
	return results[0].(*big.Int), nil
}

// GetToll 获取费率
func (d *DaiFlashMint) GetToll(ctx context.Context) (*big.Int, error) {
	var results []interface{}
	err := d.flashMint.Call(&bind.CallOpts{Context: ctx}, &results, "toll")
	if err != nil {
		return nil, err
	}
	return results[0].(*big.Int), nil
}

// DAI Flash Mint 回调合约
const DaiFlashMintReceiverSolidity = `
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import {IERC3156FlashLender} from "@openzeppelin/contracts/interfaces/IERC3156FlashLender.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract DaiFlashMintReceiver is IERC3156FlashBorrower {
    bytes32 public constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    IERC3156FlashLender public immutable daiFlashMint;
    address public immutable dai;

    constructor(address _daiFlashMint, address _dai) {
        daiFlashMint = IERC3156FlashLender(_daiFlashMint);
        dai = _dai;
    }

    // EIP-3156 回调
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external override returns (bytes32) {
        require(msg.sender == address(daiFlashMint), "not dai flash mint");
        require(initiator == address(this), "not this contract");
        require(token == dai, "not dai");

        // 解码并执行套利
        (address target, bytes memory payload) = abi.decode(data, (address, bytes));

        // 此时我们有大量 DAI 可用
        // 可以用于：
        // 1. 大额清算
        // 2. 价格操纵套利
        // 3. 杠杆创建
        (bool success,) = target.call(payload);
        require(success, "arbitrage failed");

        // 还款（当前免费，fee = 0）
        IERC20(dai).approve(address(daiFlashMint), amount + fee);

        return CALLBACK_SUCCESS;
    }

    // 发起 DAI 闪电铸造
    function flashMint(
        uint256 amount,
        address target,
        bytes calldata payload
    ) external {
        bytes memory data = abi.encode(target, payload);
        daiFlashMint.flashLoan(
            IERC3156FlashBorrower(address(this)),
            dai,
            amount,
            data
        );
    }

    // 大额清算示例
    function flashMintLiquidate(
        uint256 daiAmount,
        address lendingPool,
        address collateralAsset,
        address debtAsset,
        address user,
        uint256 debtToCover
    ) external {
        // 编码清算参数
        bytes memory liquidateCall = abi.encodeWithSignature(
            "liquidationCall(address,address,address,uint256,bool)",
            collateralAsset,
            debtAsset,
            user,
            debtToCover,
            false
        );

        bytes memory data = abi.encode(lendingPool, liquidateCall);

        // 铸造大量 DAI 进行清算
        daiFlashMint.flashLoan(
            IERC3156FlashBorrower(address(this)),
            dai,
            daiAmount,
            data
        );
    }
}
`

// DAI Flash Mint 的独特优势
type DaiFlashMintAdvantages struct {
	Feature     string
	Description string
}

var DaiFlashMintFeatures = []DaiFlashMintAdvantages{
	{
		Feature:     "超大额度",
		Description: "理论上可以铸造任意数量的 DAI，不受池子流动性限制",
	},
	{
		Feature:     "零费用",
		Description: "当前 toll = 0，闪电铸造完全免费",
	},
	{
		Feature:     "原生铸造",
		Description: "直接铸造新 DAI，不是从池子借出，不影响其他用户",
	},
	{
		Feature:     "稳定币优势",
		Description: "DAI 是稳定币，价格稳定，适合套利计算",
	},
	{
		Feature:     "EIP-3156 兼容",
		Description: "遵循标准接口，易于集成",
	},
}
```

### A.5 闪电贷协议对比

```go
// 闪电贷协议综合对比
package flashloan

import (
	"math/big"
)

// FlashLoanProtocol 闪电贷协议特性
type FlashLoanProtocol struct {
	Name           string
	Fee            string      // 费率
	MaxLoan        string      // 最大借款
	Assets         []string    // 支持资产
	Standard       string      // 接口标准
	GasEstimate    uint64      // Gas 估算
	Advantages     []string
	Disadvantages  []string
}

var FlashLoanProtocols = []FlashLoanProtocol{
	{
		Name:        "Aave V3",
		Fee:         "0.05% (可豁免)",
		MaxLoan:     "池子可用流动性",
		Assets:      []string{"ETH", "USDC", "DAI", "WBTC", "等 20+"},
		Standard:    "自定义",
		GasEstimate: 250000,
		Advantages: []string{
			"资产种类最多",
			"跨链支持",
			"机构级安全",
		},
		Disadvantages: []string{
			"费率相对较高",
			"需要实现特定接口",
		},
	},
	{
		Name:        "Uniswap V3",
		Fee:         "池子交易费率 (0.01%-1%)",
		MaxLoan:     "单个池子流动性",
		Assets:      []string{"任意 ERC20 对"},
		Standard:    "自定义",
		GasEstimate: 200000,
		Advantages: []string{
			"支持任意代币对",
			"可获取任意池子流动性",
		},
		Disadvantages: []string{
			"费率取决于池子",
			"需要先 swap 再还款",
		},
	},
	{
		Name:        "Balancer",
		Fee:         "0% (协议可调)",
		MaxLoan:     "Vault 总余额",
		Assets:      []string{"Vault 内所有资产"},
		Standard:    "自定义",
		GasEstimate: 180000,
		Advantages: []string{
			"零费用",
			"可借多种资产",
			"Gas 效率高",
		},
		Disadvantages: []string{
			"资产种类有限",
		},
	},
	{
		Name:        "Morpho Blue",
		Fee:         "0%",
		MaxLoan:     "市场供应量",
		Assets:      []string{"市场支持资产"},
		Standard:    "自定义",
		GasEstimate: 150000,
		Advantages: []string{
			"完全免费",
			"Gas 最低",
			"简洁接口",
		},
		Disadvantages: []string{
			"相对较新",
			"资产有限",
		},
	},
	{
		Name:        "Euler V2",
		Fee:         "可配置 (默认 0%)",
		MaxLoan:     "Vault 可用余额",
		Assets:      []string{"Vault 支持资产"},
		Standard:    "EIP-3156",
		GasEstimate: 170000,
		Advantages: []string{
			"EIP-3156 标准",
			"模块化设计",
			"灵活配置",
		},
		Disadvantages: []string{
			"需要找到对应 Vault",
		},
	},
	{
		Name:        "dYdX",
		Fee:         "2 wei",
		MaxLoan:     "协议流动性",
		Assets:      []string{"WETH", "USDC", "DAI"},
		Standard:    "自定义 (operate)",
		GasEstimate: 300000,
		Advantages: []string{
			"几乎免费",
			"久经考验",
		},
		Disadvantages: []string{
			"接口复杂",
			"资产有限",
			"Gas 较高",
		},
	},
	{
		Name:        "MakerDAO Flash Mint",
		Fee:         "0% (可调)",
		MaxLoan:     "理论无限 (配置限制)",
		Assets:      []string{"DAI"},
		Standard:    "EIP-3156",
		GasEstimate: 160000,
		Advantages: []string{
			"超大额度",
			"免费",
			"标准接口",
		},
		Disadvantages: []string{
			"仅支持 DAI",
		},
	},
}

// FlashLoanSelector 闪电贷选择器
type FlashLoanSelector struct {
	protocols map[string]*FlashLoanProtocol
}

// SelectBestProtocol 根据需求选择最佳协议
func (s *FlashLoanSelector) SelectBestProtocol(
	asset string,
	amount *big.Int,
	prioritizeFee bool,
) *FlashLoanProtocol {
	var candidates []*FlashLoanProtocol

	// 按优先级筛选
	for _, p := range FlashLoanProtocols {
		if s.supportsAsset(&p, asset) {
			candidates = append(candidates, &p)
		}
	}

	if len(candidates) == 0 {
		return nil
	}

	// 如果优先考虑费用，选择免费的
	if prioritizeFee {
		for _, p := range candidates {
			if p.Fee == "0%" || p.Fee == "2 wei" {
				return p
			}
		}
	}

	// 返回第一个匹配的
	return candidates[0]
}

// 选择策略推荐
type FlashLoanRecommendation struct {
	Scenario    string
	Recommended string
	Reason      string
}

var Recommendations = []FlashLoanRecommendation{
	{
		Scenario:    "大额 DAI 套利",
		Recommended: "MakerDAO Flash Mint",
		Reason:      "无限额度，零费用",
	},
	{
		Scenario:    "多资产闪电贷",
		Recommended: "Balancer",
		Reason:      "一次借多种资产，零费用",
	},
	{
		Scenario:    "ETH/主流币套利",
		Recommended: "Morpho Blue",
		Reason:      "零费用，Gas 最低",
	},
	{
		Scenario:    "任意代币对套利",
		Recommended: "Uniswap V3",
		Reason:      "支持任意池子",
	},
	{
		Scenario:    "需要最大流动性",
		Recommended: "Aave V3",
		Reason:      "最大资产池",
	},
	{
		Scenario:    "EIP-3156 兼容需求",
		Recommended: "Euler V2 / MakerDAO",
		Reason:      "标准接口",
	},
}
```

### A.6 多协议闪电贷聚合器

```go
// 多协议闪电贷聚合器
package flashloan

import (
	"context"
	"math/big"
	"sort"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// FlashLoanAggregator 闪电贷聚合器
type FlashLoanAggregator struct {
	client    *ethclient.Client
	aave      *AaveFlashLoan
	morpho    *MorphoFlashLoan
	euler     *EulerFlashLoan
	balancer  *BalancerFlashLoan
	daiMint   *DaiFlashMint
	dydx      *DYDXFlashLoan
}

// FlashLoanOption 闪电贷选项
type FlashLoanOption struct {
	Protocol    string
	Asset       common.Address
	MaxAmount   *big.Int
	Fee         *big.Int
	FeePercent  float64
	GasEstimate uint64
	Score       float64  // 综合评分
}

// NewFlashLoanAggregator 创建聚合器
func NewFlashLoanAggregator(client *ethclient.Client) (*FlashLoanAggregator, error) {
	agg := &FlashLoanAggregator{client: client}

	var err error

	// 初始化各协议
	agg.aave, _ = NewAaveFlashLoan(client)
	agg.morpho, _ = NewMorphoFlashLoan(client)
	agg.euler, _ = NewEulerFlashLoan(client)
	agg.daiMint, _ = NewDaiFlashMint(client)
	agg.dydx, _ = NewDYDXFlashLoan(client)

	return agg, err
}

// GetBestFlashLoan 获取最佳闪电贷选项
func (a *FlashLoanAggregator) GetBestFlashLoan(
	ctx context.Context,
	asset common.Address,
	amount *big.Int,
) (*FlashLoanOption, error) {
	options, err := a.GetAllOptions(ctx, asset, amount)
	if err != nil {
		return nil, err
	}

	if len(options) == 0 {
		return nil, fmt.Errorf("no flash loan available for asset")
	}

	// 按评分排序
	sort.Slice(options, func(i, j int) bool {
		return options[i].Score > options[j].Score
	})

	return options[0], nil
}

// GetAllOptions 获取所有可用选项
func (a *FlashLoanAggregator) GetAllOptions(
	ctx context.Context,
	asset common.Address,
	amount *big.Int,
) ([]*FlashLoanOption, error) {
	var options []*FlashLoanOption

	// 检查 Morpho Blue
	if a.morpho != nil {
		maxAmount, _ := a.morpho.GetAvailableLiquidity(ctx, asset)
		if maxAmount != nil && maxAmount.Cmp(amount) >= 0 {
			options = append(options, &FlashLoanOption{
				Protocol:    "Morpho Blue",
				Asset:       asset,
				MaxAmount:   maxAmount,
				Fee:         big.NewInt(0),
				FeePercent:  0,
				GasEstimate: 150000,
				Score:       calculateScore(0, 150000, maxAmount, amount),
			})
		}
	}

	// 检查 DAI Flash Mint (仅 DAI)
	if asset == DaiTokenAddress && a.daiMint != nil {
		maxAmount, _ := a.daiMint.MaxFlashLoan(ctx)
		fee, _ := a.daiMint.FlashFee(ctx, amount)
		if maxAmount != nil && maxAmount.Cmp(amount) >= 0 {
			options = append(options, &FlashLoanOption{
				Protocol:    "MakerDAO Flash Mint",
				Asset:       asset,
				MaxAmount:   maxAmount,
				Fee:         fee,
				FeePercent:  0, // 当前免费
				GasEstimate: 160000,
				Score:       calculateScore(0, 160000, maxAmount, amount),
			})
		}
	}

	// 检查 Aave
	if a.aave != nil {
		maxAmount, _ := a.aave.GetAvailableLiquidity(ctx, asset)
		if maxAmount != nil && maxAmount.Cmp(amount) >= 0 {
			// Aave 费率 0.05%
			feePercent := 0.0005
			fee := new(big.Int).Div(
				new(big.Int).Mul(amount, big.NewInt(5)),
				big.NewInt(10000),
			)
			options = append(options, &FlashLoanOption{
				Protocol:    "Aave V3",
				Asset:       asset,
				MaxAmount:   maxAmount,
				Fee:         fee,
				FeePercent:  feePercent,
				GasEstimate: 250000,
				Score:       calculateScore(feePercent, 250000, maxAmount, amount),
			})
		}
	}

	return options, nil
}

// calculateScore 计算综合评分
// 考虑因素：费率、Gas、额度
func calculateScore(feePercent float64, gas uint64, maxAmount, needAmount *big.Int) float64 {
	// 基础分 100
	score := 100.0

	// 费率扣分（每 0.01% 扣 1 分）
	score -= feePercent * 10000

	// Gas 扣分（每 10000 gas 扣 1 分）
	score -= float64(gas) / 10000

	// 额度奖励（充足额度加分）
	if maxAmount.Cmp(needAmount) > 0 {
		ratio := new(big.Float).Quo(
			new(big.Float).SetInt(maxAmount),
			new(big.Float).SetInt(needAmount),
		)
		ratioFloat, _ := ratio.Float64()
		if ratioFloat > 10 {
			score += 5 // 额度充足奖励
		}
	}

	return score
}

// 聚合器使用示例
const AggregatorUsageExample = `
func main() {
    client, _ := ethclient.Dial("https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY")

    agg, _ := NewFlashLoanAggregator(client)

    // 查找最佳 USDC 闪电贷
    usdcAddress := common.HexToAddress("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48")
    amount := new(big.Int).Mul(big.NewInt(1000000), big.NewInt(1e6)) // 100万 USDC

    best, _ := agg.GetBestFlashLoan(context.Background(), usdcAddress, amount)

    fmt.Printf("最佳协议: %s\n", best.Protocol)
    fmt.Printf("费用: %s (%.4f%%)\n", best.Fee, best.FeePercent*100)
    fmt.Printf("Gas 估算: %d\n", best.GasEstimate)
    fmt.Printf("评分: %.2f\n", best.Score)
}
`
```
