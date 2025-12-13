# Go-Ethereum EVM 深度解析：DEX 数学模型（第十部分）

## 第71节：AMM 基础数学原理

### 71.1 自动做市商概述

自动做市商（AMM）是 DeFi 的核心创新，用数学公式替代传统订单簿：

```
┌─────────────────────────────────────────────────────────────────┐
│                    AMM vs 订单簿对比                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  传统订单簿 (Order Book)           AMM (Automated Market Maker) │
│  ┌─────────────────────┐          ┌─────────────────────┐      │
│  │ 卖单 (Asks)          │          │                     │      │
│  │ $105 - 10 ETH       │          │   流动性池           │      │
│  │ $104 - 5 ETH        │          │  ┌───────────────┐  │      │
│  │ $103 - 8 ETH        │          │  │ ETH: 1000     │  │      │
│  ├─────────────────────┤          │  │ USDC: 2000000 │  │      │
│  │ 买单 (Bids)          │          │  └───────────────┘  │      │
│  │ $100 - 12 ETH       │          │         ↓           │      │
│  │ $99 - 20 ETH        │          │   x * y = k         │      │
│  │ $98 - 15 ETH        │          │   定价公式           │      │
│  └─────────────────────┘          └─────────────────────┘      │
│                                                                 │
│  特点：                            特点：                        │
│  - 需要做市商                       - 无需做市商                  │
│  - 离散价格点                       - 连续价格曲线                 │
│  - 流动性碎片化                     - 统一流动性池                 │
│  - 高效但复杂                       - 简单但有滑点                 │
└─────────────────────────────────────────────────────────────────┘
```

### 71.2 恒定乘积公式（Constant Product）

Uniswap V2 使用的核心公式：

```go
package dexmath

import (
    "errors"
    "math/big"
)

var (
    ErrInsufficientLiquidity = errors.New("insufficient liquidity")
    ErrInvalidInput          = errors.New("invalid input amount")
    ErrZeroAmount            = errors.New("zero amount")
)

// ConstantProductAMM 恒定乘积AMM
type ConstantProductAMM struct {
    ReserveX *big.Int // Token X 储备量
    ReserveY *big.Int // Token Y 储备量
    Fee      uint64   // 手续费（基点，如 30 = 0.3%）
}

// NewConstantProductAMM 创建新的恒定乘积AMM
func NewConstantProductAMM(reserveX, reserveY *big.Int, feeBps uint64) *ConstantProductAMM {
    return &ConstantProductAMM{
        ReserveX: new(big.Int).Set(reserveX),
        ReserveY: new(big.Int).Set(reserveY),
        Fee:      feeBps,
    }
}

// K 计算恒定乘积 k = x * y
func (amm *ConstantProductAMM) K() *big.Int {
    return new(big.Int).Mul(amm.ReserveX, amm.ReserveY)
}

// SpotPrice 即时价格 (Y/X)
// price = reserveY / reserveX
func (amm *ConstantProductAMM) SpotPrice() *big.Float {
    x := new(big.Float).SetInt(amm.ReserveX)
    y := new(big.Float).SetInt(amm.ReserveY)
    return new(big.Float).Quo(y, x)
}

// GetAmountOut 计算输出金额（考虑手续费）
// 公式：amountOut = (reserveOut * amountIn * (10000 - fee)) / (reserveIn * 10000 + amountIn * (10000 - fee))
func (amm *ConstantProductAMM) GetAmountOut(amountIn *big.Int, tokenIn int) (*big.Int, error) {
    if amountIn.Sign() <= 0 {
        return nil, ErrZeroAmount
    }

    var reserveIn, reserveOut *big.Int
    if tokenIn == 0 {
        reserveIn = amm.ReserveX
        reserveOut = amm.ReserveY
    } else {
        reserveIn = amm.ReserveY
        reserveOut = amm.ReserveX
    }

    if reserveIn.Sign() <= 0 || reserveOut.Sign() <= 0 {
        return nil, ErrInsufficientLiquidity
    }

    // amountInWithFee = amountIn * (10000 - fee)
    feeMultiplier := big.NewInt(10000 - int64(amm.Fee))
    amountInWithFee := new(big.Int).Mul(amountIn, feeMultiplier)

    // numerator = amountInWithFee * reserveOut
    numerator := new(big.Int).Mul(amountInWithFee, reserveOut)

    // denominator = reserveIn * 10000 + amountInWithFee
    denominator := new(big.Int).Mul(reserveIn, big.NewInt(10000))
    denominator.Add(denominator, amountInWithFee)

    // amountOut = numerator / denominator
    amountOut := new(big.Int).Div(numerator, denominator)

    if amountOut.Sign() <= 0 {
        return nil, ErrInsufficientLiquidity
    }

    return amountOut, nil
}

// GetAmountIn 计算需要的输入金额
// 公式：amountIn = (reserveIn * amountOut * 10000) / ((reserveOut - amountOut) * (10000 - fee)) + 1
func (amm *ConstantProductAMM) GetAmountIn(amountOut *big.Int, tokenOut int) (*big.Int, error) {
    if amountOut.Sign() <= 0 {
        return nil, ErrZeroAmount
    }

    var reserveIn, reserveOut *big.Int
    if tokenOut == 0 {
        reserveIn = amm.ReserveY
        reserveOut = amm.ReserveX
    } else {
        reserveIn = amm.ReserveX
        reserveOut = amm.ReserveY
    }

    if reserveIn.Sign() <= 0 || reserveOut.Cmp(amountOut) <= 0 {
        return nil, ErrInsufficientLiquidity
    }

    // numerator = reserveIn * amountOut * 10000
    numerator := new(big.Int).Mul(reserveIn, amountOut)
    numerator.Mul(numerator, big.NewInt(10000))

    // denominator = (reserveOut - amountOut) * (10000 - fee)
    feeMultiplier := big.NewInt(10000 - int64(amm.Fee))
    denominator := new(big.Int).Sub(reserveOut, amountOut)
    denominator.Mul(denominator, feeMultiplier)

    // amountIn = numerator / denominator + 1 (向上取整)
    amountIn := new(big.Int).Div(numerator, denominator)
    amountIn.Add(amountIn, big.NewInt(1))

    return amountIn, nil
}
```

### 71.3 价格影响计算

```go
// PriceImpact 计算价格影响
type PriceImpact struct {
    SpotPriceBefore *big.Float // 交易前即时价格
    SpotPriceAfter  *big.Float // 交易后即时价格
    ExecutionPrice  *big.Float // 实际执行价格
    ImpactPercent   *big.Float // 价格影响百分比
}

// CalculatePriceImpact 计算交易的价格影响
func (amm *ConstantProductAMM) CalculatePriceImpact(amountIn *big.Int, tokenIn int) (*PriceImpact, error) {
    amountOut, err := amm.GetAmountOut(amountIn, tokenIn)
    if err != nil {
        return nil, err
    }

    // 交易前即时价格
    var spotPriceBefore *big.Float
    if tokenIn == 0 {
        spotPriceBefore = amm.SpotPrice() // Y/X
    } else {
        spotPriceBefore = new(big.Float).Quo(
            new(big.Float).SetInt(amm.ReserveX),
            new(big.Float).SetInt(amm.ReserveY),
        ) // X/Y
    }

    // 实际执行价格
    executionPrice := new(big.Float).Quo(
        new(big.Float).SetInt(amountOut),
        new(big.Float).SetInt(amountIn),
    )

    // 计算交易后储备
    var newReserveX, newReserveY *big.Int
    if tokenIn == 0 {
        newReserveX = new(big.Int).Add(amm.ReserveX, amountIn)
        newReserveY = new(big.Int).Sub(amm.ReserveY, amountOut)
    } else {
        newReserveX = new(big.Int).Sub(amm.ReserveX, amountOut)
        newReserveY = new(big.Int).Add(amm.ReserveY, amountIn)
    }

    // 交易后即时价格
    var spotPriceAfter *big.Float
    if tokenIn == 0 {
        spotPriceAfter = new(big.Float).Quo(
            new(big.Float).SetInt(newReserveY),
            new(big.Float).SetInt(newReserveX),
        )
    } else {
        spotPriceAfter = new(big.Float).Quo(
            new(big.Float).SetInt(newReserveX),
            new(big.Float).SetInt(newReserveY),
        )
    }

    // 价格影响 = (spotPriceBefore - executionPrice) / spotPriceBefore * 100
    impact := new(big.Float).Sub(spotPriceBefore, executionPrice)
    impact.Quo(impact, spotPriceBefore)
    impact.Mul(impact, big.NewFloat(100))

    return &PriceImpact{
        SpotPriceBefore: spotPriceBefore,
        SpotPriceAfter:  spotPriceAfter,
        ExecutionPrice:  executionPrice,
        ImpactPercent:   impact,
    }, nil
}

// 价格影响公式推导:
//
// 恒定乘积: x * y = k
//
// 交易 Δx 换取 Δy:
// (x + Δx) * (y - Δy) = k
//
// 解得:
// Δy = y * Δx / (x + Δx)
//
// 执行价格:
// P_exec = Δy / Δx = y / (x + Δx)
//
// 即时价格:
// P_spot = y / x
//
// 价格影响:
// Impact = (P_spot - P_exec) / P_spot
//        = 1 - x / (x + Δx)
//        = Δx / (x + Δx)
```

### 71.4 恒定乘积曲线可视化

```
┌─────────────────────────────────────────────────────────────────┐
│              恒定乘积曲线 x * y = k                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Token Y                                                        │
│    ▲                                                            │
│    │                                                            │
│  4k┤ ╲                                                          │
│    │  ╲                                                         │
│    │   ╲                                                        │
│  3k┤    ╲                                                       │
│    │     ╲                                                      │
│    │      ╲                                                     │
│  2k┤       ╲___                                                 │
│    │           ╲___                                             │
│    │               ╲___                                         │
│  1k┤                   ╲___                                     │
│    │                       ╲___                                 │
│    │                           ╲___                             │
│    │                               ╲___                         │
│    └────┴────┴────┴────┴────┴────┴────┴────► Token X           │
│         1k   2k   3k   4k   5k   6k   7k                        │
│                                                                 │
│  特性:                                                           │
│  1. 双曲线形状                                                    │
│  2. 永不触及坐标轴（无限流动性）                                    │
│  3. 曲率决定滑点大小                                              │
│  4. k 值越大，曲线越平缓，滑点越小                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第72节：Uniswap V2 数学详解

### 72.1 Uniswap V2 完整数学模型

```go
package uniswapv2

import (
    "math/big"
)

// UniswapV2Pair Uniswap V2 交易对
type UniswapV2Pair struct {
    Token0   string   // Token0 地址
    Token1   string   // Token1 地址
    Reserve0 *big.Int // Token0 储备
    Reserve1 *big.Int // Token1 储备
}

// 常量
const (
    FeeBps      = 30   // 0.3% 手续费
    FeeBase     = 10000
    FeeMultiplier = FeeBase - FeeBps // 9970
)

// GetAmountOut Uniswap V2 标准输出计算
// Solidity 原版:
// function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) internal pure returns (uint amountOut) {
//     require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
//     require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
//     uint amountInWithFee = amountIn.mul(997);
//     uint numerator = amountInWithFee.mul(reserveOut);
//     uint denominator = reserveIn.mul(1000).add(amountInWithFee);
//     amountOut = numerator / denominator;
// }
func GetAmountOut(amountIn, reserveIn, reserveOut *big.Int) *big.Int {
    if amountIn.Sign() <= 0 || reserveIn.Sign() <= 0 || reserveOut.Sign() <= 0 {
        return big.NewInt(0)
    }

    // amountInWithFee = amountIn * 997
    amountInWithFee := new(big.Int).Mul(amountIn, big.NewInt(997))

    // numerator = amountInWithFee * reserveOut
    numerator := new(big.Int).Mul(amountInWithFee, reserveOut)

    // denominator = reserveIn * 1000 + amountInWithFee
    denominator := new(big.Int).Mul(reserveIn, big.NewInt(1000))
    denominator.Add(denominator, amountInWithFee)

    // amountOut = numerator / denominator
    return new(big.Int).Div(numerator, denominator)
}

// GetAmountIn Uniswap V2 标准输入计算
// Solidity 原版:
// function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut) internal pure returns (uint amountIn) {
//     require(amountOut > 0, 'UniswapV2Library: INSUFFICIENT_OUTPUT_AMOUNT');
//     require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
//     uint numerator = reserveIn.mul(amountOut).mul(1000);
//     uint denominator = reserveOut.sub(amountOut).mul(997);
//     amountIn = (numerator / denominator).add(1);
// }
func GetAmountIn(amountOut, reserveIn, reserveOut *big.Int) *big.Int {
    if amountOut.Sign() <= 0 || reserveIn.Sign() <= 0 || reserveOut.Cmp(amountOut) <= 0 {
        return nil // 流动性不足
    }

    // numerator = reserveIn * amountOut * 1000
    numerator := new(big.Int).Mul(reserveIn, amountOut)
    numerator.Mul(numerator, big.NewInt(1000))

    // denominator = (reserveOut - amountOut) * 997
    denominator := new(big.Int).Sub(reserveOut, amountOut)
    denominator.Mul(denominator, big.NewInt(997))

    // amountIn = numerator / denominator + 1
    amountIn := new(big.Int).Div(numerator, denominator)
    amountIn.Add(amountIn, big.NewInt(1))

    return amountIn
}

// GetAmountsOut 多跳路径输出计算
func GetAmountsOut(amountIn *big.Int, path []UniswapV2Pair, directions []bool) []*big.Int {
    amounts := make([]*big.Int, len(path)+1)
    amounts[0] = new(big.Int).Set(amountIn)

    for i, pair := range path {
        var reserveIn, reserveOut *big.Int
        if directions[i] { // token0 -> token1
            reserveIn = pair.Reserve0
            reserveOut = pair.Reserve1
        } else { // token1 -> token0
            reserveIn = pair.Reserve1
            reserveOut = pair.Reserve0
        }

        amounts[i+1] = GetAmountOut(amounts[i], reserveIn, reserveOut)
    }

    return amounts
}

// GetAmountsIn 多跳路径输入计算
func GetAmountsIn(amountOut *big.Int, path []UniswapV2Pair, directions []bool) []*big.Int {
    amounts := make([]*big.Int, len(path)+1)
    amounts[len(path)] = new(big.Int).Set(amountOut)

    for i := len(path) - 1; i >= 0; i-- {
        pair := path[i]
        var reserveIn, reserveOut *big.Int
        if directions[i] { // token0 -> token1
            reserveIn = pair.Reserve0
            reserveOut = pair.Reserve1
        } else { // token1 -> token0
            reserveIn = pair.Reserve1
            reserveOut = pair.Reserve0
        }

        amounts[i] = GetAmountIn(amounts[i+1], reserveIn, reserveOut)
        if amounts[i] == nil {
            return nil // 流动性不足
        }
    }

    return amounts
}
```

### 72.2 最优套利金额计算

```go
package arbitrage

import (
    "math"
    "math/big"
)

// 二池套利最优金额计算
//
// 场景: Pool A (价格低) -> Pool B (价格高)
//
// 数学推导:
// 设初始状态:
//   Pool A: (x_a, y_a), k_a = x_a * y_a
//   Pool B: (x_b, y_b), k_b = x_b * y_b
//
// 用 Δx 在 Pool A 买入:
//   Δy_a = y_a * Δx * 0.997 / (x_a + Δx * 0.997)
//
// 用 Δy_a 在 Pool B 卖出:
//   Δx_b = x_b * Δy_a * 0.997 / (y_b + Δy_a * 0.997)
//
// 利润:
//   Profit = Δx_b - Δx
//
// 对 Δx 求导并令导数为0，得到最优输入

// OptimalArbitrageAmount 计算两池套利最优金额
// poolA: 低价池 (买入)
// poolB: 高价池 (卖出)
// 返回: 最优输入金额
func OptimalArbitrageAmount(
    reserveAIn, reserveAOut *big.Int, // Pool A 储备
    reserveBIn, reserveBOut *big.Int, // Pool B 储备
) *big.Int {
    // 转换为 float64 进行计算（精度足够用于估算）
    xa := float64FromBigInt(reserveAIn)
    ya := float64FromBigInt(reserveAOut)
    xb := float64FromBigInt(reserveBOut)
    yb := float64FromBigInt(reserveBIn)

    fee := 0.997 // 扣除 0.3% 手续费

    // 最优输入公式（通过求导得到）:
    // optimal = (sqrt(xa * ya * xb * yb * fee^4) - xa * yb * fee^2) / (yb * fee^2 + ya * fee^2)

    sqrtPart := math.Sqrt(xa * ya * xb * yb * math.Pow(fee, 4))
    numerator := sqrtPart - xa*yb*fee*fee
    denominator := yb*fee*fee + ya*fee*fee

    if numerator <= 0 || denominator <= 0 {
        return big.NewInt(0) // 无套利机会
    }

    optimal := numerator / denominator

    return bigIntFromFloat64(optimal)
}

// OptimalArbitrageV2 更精确的二池套利计算
func OptimalArbitrageV2(
    reserveAIn, reserveAOut *big.Int, // Pool A: X进Y出
    reserveBIn, reserveBOut *big.Int, // Pool B: Y进X出
) (*big.Int, *big.Int) {
    // 使用big.Float进行高精度计算
    xa := new(big.Float).SetInt(reserveAIn)
    ya := new(big.Float).SetInt(reserveAOut)
    yb := new(big.Float).SetInt(reserveBIn)
    xb := new(big.Float).SetInt(reserveBOut)

    fee := big.NewFloat(0.997)
    fee2 := new(big.Float).Mul(fee, fee)
    fee4 := new(big.Float).Mul(fee2, fee2)

    // product = xa * ya * xb * yb * fee^4
    product := new(big.Float).Mul(xa, ya)
    product.Mul(product, xb)
    product.Mul(product, yb)
    product.Mul(product, fee4)

    // sqrt_part = sqrt(product)
    sqrtPart := bigFloatSqrt(product)

    // term1 = xa * yb * fee^2
    term1 := new(big.Float).Mul(xa, yb)
    term1.Mul(term1, fee2)

    // numerator = sqrt_part - term1
    numerator := new(big.Float).Sub(sqrtPart, term1)

    // 检查是否有套利机会
    if numerator.Sign() <= 0 {
        return big.NewInt(0), big.NewInt(0)
    }

    // denominator = (yb + ya) * fee^2
    denominator := new(big.Float).Add(yb, ya)
    denominator.Mul(denominator, fee2)

    // optimal = numerator / denominator
    optimal := new(big.Float).Quo(numerator, denominator)

    // 转换为 big.Int
    optimalInt, _ := optimal.Int(nil)

    // 计算预期利润
    profit := CalculateProfit(optimalInt, reserveAIn, reserveAOut, reserveBIn, reserveBOut)

    return optimalInt, profit
}

// CalculateProfit 计算套利利润
func CalculateProfit(
    amountIn *big.Int,
    reserveAIn, reserveAOut *big.Int, // Pool A
    reserveBIn, reserveBOut *big.Int, // Pool B
) *big.Int {
    if amountIn.Sign() <= 0 {
        return big.NewInt(0)
    }

    // 第一跳: Pool A (X -> Y)
    amountOutA := GetAmountOut(amountIn, reserveAIn, reserveAOut)

    // 第二跳: Pool B (Y -> X)
    amountOutB := GetAmountOut(amountOutA, reserveBIn, reserveBOut)

    // 利润 = 输出 - 输入
    profit := new(big.Int).Sub(amountOutB, amountIn)

    return profit
}

// 辅助函数
func float64FromBigInt(n *big.Int) float64 {
    f, _ := new(big.Float).SetInt(n).Float64()
    return f
}

func bigIntFromFloat64(f float64) *big.Int {
    bf := big.NewFloat(f)
    i, _ := bf.Int(nil)
    return i
}

func bigFloatSqrt(x *big.Float) *big.Float {
    // 牛顿迭代法计算平方根
    if x.Sign() <= 0 {
        return big.NewFloat(0)
    }

    // 初始猜测
    guess := new(big.Float).Copy(x)
    half := big.NewFloat(0.5)

    for i := 0; i < 100; i++ {
        // newGuess = (guess + x/guess) / 2
        quo := new(big.Float).Quo(x, guess)
        sum := new(big.Float).Add(guess, quo)
        newGuess := new(big.Float).Mul(sum, half)

        // 检查收敛
        diff := new(big.Float).Sub(newGuess, guess)
        diff.Abs(diff)
        if diff.Cmp(big.NewFloat(1)) < 0 {
            return newGuess
        }

        guess = newGuess
    }

    return guess
}
```

### 72.3 三角套利计算

```go
// TriangularArbitrage 三角套利
type TriangularArbitrage struct {
    TokenA   string // 起始/结束代币
    TokenB   string // 中间代币1
    TokenC   string // 中间代币2

    // 三个交易对
    PairAB *UniswapV2Pair // A-B
    PairBC *UniswapV2Pair // B-C
    PairCA *UniswapV2Pair // C-A

    // 交易方向
    DirectionAB bool // true: A->B, false: B->A
    DirectionBC bool // true: B->C, false: C->B
    DirectionCA bool // true: C->A, false: A->C
}

// 三角套利路径示例:
//
// ┌─────────────────────────────────────────────────────────────────┐
// │                    三角套利路径                                   │
// ├─────────────────────────────────────────────────────────────────┤
// │                                                                 │
// │                        ETH                                      │
// │                       /   \                                     │
// │                      /     \                                    │
// │              Pool AB/       \Pool CA                            │
// │                    /         \                                  │
// │                   ▼           ▲                                 │
// │                USDC ────────► USDT                              │
// │                     Pool BC                                     │
// │                                                                 │
// │  路径: ETH -> USDC -> USDT -> ETH                               │
// │                                                                 │
// │  条件: 如果三个池价格不一致，存在套利空间                           │
// │  例如:                                                           │
// │    Pool AB: 1 ETH = 2000 USDC                                   │
// │    Pool BC: 1 USDC = 1.001 USDT                                 │
// │    Pool CA: 1 USDT = 0.000502 ETH (即 1 ETH = 1992 USDT)        │
// │                                                                 │
// │  套利: 1 ETH → 2000 USDC → 2002 USDT → 1.005 ETH               │
// │  利润: 0.005 ETH (扣除手续费前)                                  │
// │                                                                 │
// └─────────────────────────────────────────────────────────────────┘

// CalculateTriangularProfit 计算三角套利利润
func (ta *TriangularArbitrage) CalculateTriangularProfit(amountIn *big.Int) *big.Int {
    // 第一跳: A -> B
    amountB := ta.getAmountOut(ta.PairAB, amountIn, ta.DirectionAB)
    if amountB.Sign() <= 0 {
        return big.NewInt(0)
    }

    // 第二跳: B -> C
    amountC := ta.getAmountOut(ta.PairBC, amountB, ta.DirectionBC)
    if amountC.Sign() <= 0 {
        return big.NewInt(0)
    }

    // 第三跳: C -> A
    amountA := ta.getAmountOut(ta.PairCA, amountC, ta.DirectionCA)
    if amountA.Sign() <= 0 {
        return big.NewInt(0)
    }

    // 利润 = 最终A - 初始A
    return new(big.Int).Sub(amountA, amountIn)
}

func (ta *TriangularArbitrage) getAmountOut(pair *UniswapV2Pair, amountIn *big.Int, direction bool) *big.Int {
    var reserveIn, reserveOut *big.Int
    if direction {
        reserveIn = pair.Reserve0
        reserveOut = pair.Reserve1
    } else {
        reserveIn = pair.Reserve1
        reserveOut = pair.Reserve0
    }
    return GetAmountOut(amountIn, reserveIn, reserveOut)
}

// FindOptimalTriangularAmount 查找最优三角套利金额
// 使用二分搜索找到利润最大化的输入金额
func (ta *TriangularArbitrage) FindOptimalTriangularAmount(maxAmount *big.Int) (*big.Int, *big.Int) {
    // 二分搜索边界
    low := big.NewInt(1e15)  // 最小 0.001 ETH
    high := new(big.Int).Set(maxAmount)

    var bestAmount, bestProfit *big.Int
    bestAmount = big.NewInt(0)
    bestProfit = big.NewInt(0)

    // 黄金分割搜索
    phi := 1.618033988749895

    for new(big.Int).Sub(high, low).Cmp(big.NewInt(1e15)) > 0 {
        // 计算两个测试点
        diff := new(big.Int).Sub(high, low)
        diffFloat, _ := new(big.Float).SetInt(diff).Float64()

        step1 := bigIntFromFloat64(diffFloat / phi)
        step2 := bigIntFromFloat64(diffFloat / phi / phi)

        mid1 := new(big.Int).Add(low, step2)
        mid2 := new(big.Int).Add(low, step1)

        profit1 := ta.CalculateTriangularProfit(mid1)
        profit2 := ta.CalculateTriangularProfit(mid2)

        if profit1.Cmp(profit2) > 0 {
            high = mid2
            if profit1.Cmp(bestProfit) > 0 {
                bestAmount = new(big.Int).Set(mid1)
                bestProfit = new(big.Int).Set(profit1)
            }
        } else {
            low = mid1
            if profit2.Cmp(bestProfit) > 0 {
                bestAmount = new(big.Int).Set(mid2)
                bestProfit = new(big.Int).Set(profit2)
            }
        }
    }

    return bestAmount, bestProfit
}
```

### 72.4 LP Token 数学

```go
// LPTokenMath LP代币数学计算
type LPTokenMath struct {
    TotalSupply *big.Int // LP代币总供应量
    Reserve0    *big.Int // Token0 储备
    Reserve1    *big.Int // Token1 储备
}

// 首次添加流动性:
// liquidity = sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY
// MINIMUM_LIQUIDITY = 1000 (锁定在零地址，防止除零)

// 后续添加流动性:
// liquidity = min(amount0 * totalSupply / reserve0, amount1 * totalSupply / reserve1)

// CalculateLiquidityMint 计算铸造的LP代币数量
func (lp *LPTokenMath) CalculateLiquidityMint(amount0, amount1 *big.Int) *big.Int {
    if lp.TotalSupply.Sign() == 0 {
        // 首次添加流动性
        product := new(big.Int).Mul(amount0, amount1)
        liquidity := bigIntSqrt(product)

        // 减去最小流动性（锁定）
        minLiquidity := big.NewInt(1000)
        liquidity.Sub(liquidity, minLiquidity)

        return liquidity
    }

    // 后续添加
    // liquidity0 = amount0 * totalSupply / reserve0
    liquidity0 := new(big.Int).Mul(amount0, lp.TotalSupply)
    liquidity0.Div(liquidity0, lp.Reserve0)

    // liquidity1 = amount1 * totalSupply / reserve1
    liquidity1 := new(big.Int).Mul(amount1, lp.TotalSupply)
    liquidity1.Div(liquidity1, lp.Reserve1)

    // 返回较小值
    if liquidity0.Cmp(liquidity1) < 0 {
        return liquidity0
    }
    return liquidity1
}

// CalculateUnderlyingAmounts 计算LP代币对应的底层资产
func (lp *LPTokenMath) CalculateUnderlyingAmounts(lpAmount *big.Int) (amount0, amount1 *big.Int) {
    // amount0 = lpAmount * reserve0 / totalSupply
    amount0 = new(big.Int).Mul(lpAmount, lp.Reserve0)
    amount0.Div(amount0, lp.TotalSupply)

    // amount1 = lpAmount * reserve1 / totalSupply
    amount1 = new(big.Int).Mul(lpAmount, lp.Reserve1)
    amount1.Div(amount1, lp.TotalSupply)

    return amount0, amount1
}

// CalculateRemoveLiquidityAmounts 计算移除流动性获得的代币
func (lp *LPTokenMath) CalculateRemoveLiquidityAmounts(lpAmount *big.Int) (amount0, amount1 *big.Int) {
    return lp.CalculateUnderlyingAmounts(lpAmount)
}

// bigIntSqrt 大整数平方根（牛顿法）
func bigIntSqrt(n *big.Int) *big.Int {
    if n.Sign() <= 0 {
        return big.NewInt(0)
    }

    // 初始猜测
    x := new(big.Int).Rsh(n, uint(n.BitLen()/2))
    if x.Sign() == 0 {
        x.SetInt64(1)
    }

    for {
        // x1 = (x + n/x) / 2
        x1 := new(big.Int).Div(n, x)
        x1.Add(x1, x)
        x1.Rsh(x1, 1)

        if x1.Cmp(x) >= 0 {
            return x
        }
        x = x1
    }
}
```

---

## 第73节：Uniswap V3 集中流动性数学

### 73.1 V3 核心概念

```
┌─────────────────────────────────────────────────────────────────┐
│              Uniswap V3 集中流动性原理                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  V2 流动性分布:                    V3 集中流动性:                 │
│                                                                 │
│  流动性                            流动性                        │
│    ▲                                 ▲                          │
│    │ ░░░░░░░░░░░░░░░░░░░░░░         │        ████████          │
│    │ ░░░░░░░░░░░░░░░░░░░░░░         │        ████████          │
│    │ ░░░░░░░░░░░░░░░░░░░░░░         │        ████████          │
│    │ ░░░░░░░░░░░░░░░░░░░░░░         │   █████████████████      │
│    │ ░░░░░░░░░░░░░░░░░░░░░░         │   █████████████████      │
│    └──────────────────────►         └───────────────────────►  │
│      0              ∞ 价格             Plow    Pcur    Phigh    │
│                                                                 │
│  V2: 流动性均匀分布在 (0, ∞)        V3: 流动性集中在价格区间内    │
│  资本效率: ~0.5%                   资本效率: 可达 4000x          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  V3 核心创新:                                                    │
│                                                                 │
│  1. Tick 刻度系统                                                │
│     - 价格空间被离散化为 tick                                    │
│     - tick_spacing 决定精度 (1, 10, 60, 200)                    │
│     - tick = log_{1.0001}(price)                                │
│                                                                 │
│  2. Position NFT                                                │
│     - 每个头寸是独立的 NFT                                       │
│     - 包含: tickLower, tickUpper, liquidity                     │
│                                                                 │
│  3. 虚拟储备                                                     │
│     - 在价格区间内表现如同 V2                                    │
│     - 使用虚拟储备量计算                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 73.2 Tick 数学

```go
package uniswapv3

import (
    "math"
    "math/big"
)

// Tick 相关常量
const (
    MinTick     = -887272
    MaxTick     = 887272
    TickBase    = 1.0001
    Q96         = 79228162514264337593543950336 // 2^96
)

var (
    Q96Big = new(big.Int).Exp(big.NewInt(2), big.NewInt(96), nil)
)

// TickMath Tick数学计算
type TickMath struct{}

// GetSqrtRatioAtTick 根据tick计算sqrt(price) * 2^96
// sqrt(1.0001^tick) * 2^96
func (tm *TickMath) GetSqrtRatioAtTick(tick int) *big.Int {
    absTick := tick
    if tick < 0 {
        absTick = -tick
    }

    // 使用预计算的2的幂次进行计算
    // ratio = product of 1.0001^(2^i) for each bit i that is set
    var ratio *big.Int

    // 基础值 2^128
    ratio = new(big.Int).Exp(big.NewInt(2), big.NewInt(128), nil)

    // 预计算的魔数（每个对应 sqrt(1.0001^(2^i)) * 2^128）
    magicNumbers := []string{
        "340265354078544963557816517032075149313", // 2^0
        "340248342086729790484326174814286782778", // 2^1
        "340214320654664324051920982716015181260", // 2^2
        "340146287995602323631171512101879684304", // 2^3
        "340010263488231146823593991679159461444", // 2^4
        "339738377640345403697157401104375502016", // 2^5
        "339195258003219555707034227454543997025", // 2^6
        "338111622100601834656805679988414885971", // 2^7
        "335954724994790223023589805789778977700", // 2^8
        "331682121138379247127172139078559817300", // 2^9
        "323299236684853023288211250268160618739", // 2^10
        "307163716377032989948697243942600083929", // 2^11
        "277268403626896220162999269216087595045", // 2^12
        "225923453940442621947126027127485391333", // 2^13
        "149997214084966997727330242082538205943", // 2^14
        "66119101136024775622716233608466517926",  // 2^15
        "12847376061809297530290974190478138313",  // 2^16
        "485053260817066172746253684029974020",    // 2^17
        "691415978906521570653435304214168",       // 2^18
        "1404880482679654955896180642",            // 2^19
    }

    for i := 0; i < 20; i++ {
        if (absTick & (1 << i)) != 0 {
            magic, _ := new(big.Int).SetString(magicNumbers[i], 10)
            ratio.Mul(ratio, magic)
            ratio.Rsh(ratio, 128)
        }
    }

    // 如果tick为负，取倒数
    if tick < 0 {
        maxUint256 := new(big.Int).Sub(
            new(big.Int).Exp(big.NewInt(2), big.NewInt(256), nil),
            big.NewInt(1),
        )
        ratio = new(big.Int).Div(maxUint256, ratio)
    }

    // 转换为 Q96 格式
    ratio.Rsh(ratio, 32)

    return ratio
}

// GetTickAtSqrtRatio 根据sqrtPriceX96计算tick
func (tm *TickMath) GetTickAtSqrtRatio(sqrtPriceX96 *big.Int) int {
    // tick = floor(log_{sqrt(1.0001)}(sqrtPrice / 2^96))
    // tick = floor(log(sqrtPrice / 2^96) / log(sqrt(1.0001)))
    // tick = floor(2 * log(sqrtPrice / 2^96) / log(1.0001))

    sqrtPrice := new(big.Float).SetInt(sqrtPriceX96)
    sqrtPrice.Quo(sqrtPrice, new(big.Float).SetInt(Q96Big))

    sqrtPriceFloat, _ := sqrtPrice.Float64()

    tick := math.Floor(math.Log(sqrtPriceFloat) / math.Log(math.Sqrt(TickBase)))

    return int(tick)
}

// TickToPrice 将tick转换为价格
func TickToPrice(tick int) float64 {
    return math.Pow(TickBase, float64(tick))
}

// PriceToTick 将价格转换为tick
func PriceToTick(price float64) int {
    return int(math.Floor(math.Log(price) / math.Log(TickBase)))
}

// TickToSqrtPrice 将tick转换为sqrt(price)
func TickToSqrtPrice(tick int) float64 {
    return math.Pow(TickBase, float64(tick)/2)
}
```

### 73.3 V3 Swap 数学

```go
// V3SwapMath Uniswap V3 交换数学
type V3SwapMath struct {
    SqrtPriceX96    *big.Int // 当前价格的平方根 * 2^96
    Liquidity       *big.Int // 当前tick区间的流动性
    CurrentTick     int      // 当前tick
    TickSpacing     int      // tick间距
    Fee             uint32   // 手续费 (百万分之一)
}

// ComputeSwapStep 计算单步swap
// 返回: 新价格, amountIn, amountOut, feeAmount
func ComputeSwapStep(
    sqrtRatioCurrentX96 *big.Int,
    sqrtRatioTargetX96 *big.Int,
    liquidity *big.Int,
    amountRemaining *big.Int,
    feePips uint32,
) (sqrtRatioNextX96, amountIn, amountOut, feeAmount *big.Int) {

    zeroForOne := sqrtRatioCurrentX96.Cmp(sqrtRatioTargetX96) >= 0
    exactIn := amountRemaining.Sign() >= 0

    if exactIn {
        // 扣除手续费后的金额
        amountRemainingLessFee := new(big.Int).Mul(
            amountRemaining,
            big.NewInt(1000000-int64(feePips)),
        )
        amountRemainingLessFee.Div(amountRemainingLessFee, big.NewInt(1000000))

        if zeroForOne {
            amountIn = getAmount0Delta(sqrtRatioTargetX96, sqrtRatioCurrentX96, liquidity, true)
        } else {
            amountIn = getAmount1Delta(sqrtRatioCurrentX96, sqrtRatioTargetX96, liquidity, true)
        }

        if amountRemainingLessFee.Cmp(amountIn) >= 0 {
            sqrtRatioNextX96 = sqrtRatioTargetX96
        } else {
            sqrtRatioNextX96 = getNextSqrtPriceFromInput(
                sqrtRatioCurrentX96,
                liquidity,
                amountRemainingLessFee,
                zeroForOne,
            )
        }
    } else {
        if zeroForOne {
            amountOut = getAmount1Delta(sqrtRatioTargetX96, sqrtRatioCurrentX96, liquidity, false)
        } else {
            amountOut = getAmount0Delta(sqrtRatioCurrentX96, sqrtRatioTargetX96, liquidity, false)
        }

        amountOutAbs := new(big.Int).Abs(amountRemaining)
        if amountOutAbs.Cmp(amountOut) >= 0 {
            sqrtRatioNextX96 = sqrtRatioTargetX96
        } else {
            sqrtRatioNextX96 = getNextSqrtPriceFromOutput(
                sqrtRatioCurrentX96,
                liquidity,
                amountOutAbs,
                zeroForOne,
            )
        }
    }

    max := sqrtRatioTargetX96.Cmp(sqrtRatioNextX96) == 0

    if zeroForOne {
        if !max || !exactIn {
            amountIn = getAmount0Delta(sqrtRatioNextX96, sqrtRatioCurrentX96, liquidity, true)
        }
        if !max || exactIn {
            amountOut = getAmount1Delta(sqrtRatioNextX96, sqrtRatioCurrentX96, liquidity, false)
        }
    } else {
        if !max || !exactIn {
            amountIn = getAmount1Delta(sqrtRatioCurrentX96, sqrtRatioNextX96, liquidity, true)
        }
        if !max || exactIn {
            amountOut = getAmount0Delta(sqrtRatioCurrentX96, sqrtRatioNextX96, liquidity, false)
        }
    }

    // 计算手续费
    if exactIn && sqrtRatioNextX96.Cmp(sqrtRatioTargetX96) != 0 {
        feeAmount = new(big.Int).Sub(amountRemaining, amountIn)
    } else {
        feeAmount = mulDivRoundingUp(amountIn, big.NewInt(int64(feePips)), big.NewInt(1000000-int64(feePips)))
    }

    return
}

// getAmount0Delta 计算token0数量变化
// Δx = L * (1/√P_lower - 1/√P_upper)
func getAmount0Delta(sqrtRatioAX96, sqrtRatioBX96, liquidity *big.Int, roundUp bool) *big.Int {
    if sqrtRatioAX96.Cmp(sqrtRatioBX96) > 0 {
        sqrtRatioAX96, sqrtRatioBX96 = sqrtRatioBX96, sqrtRatioAX96
    }

    numerator1 := new(big.Int).Lsh(liquidity, 96)
    numerator2 := new(big.Int).Sub(sqrtRatioBX96, sqrtRatioAX96)

    result := new(big.Int).Mul(numerator1, numerator2)
    result.Div(result, sqrtRatioBX96)

    if roundUp {
        result = divRoundingUp(result, sqrtRatioAX96)
    } else {
        result.Div(result, sqrtRatioAX96)
    }

    return result
}

// getAmount1Delta 计算token1数量变化
// Δy = L * (√P_upper - √P_lower)
func getAmount1Delta(sqrtRatioAX96, sqrtRatioBX96, liquidity *big.Int, roundUp bool) *big.Int {
    if sqrtRatioAX96.Cmp(sqrtRatioBX96) > 0 {
        sqrtRatioAX96, sqrtRatioBX96 = sqrtRatioBX96, sqrtRatioAX96
    }

    diff := new(big.Int).Sub(sqrtRatioBX96, sqrtRatioAX96)

    if roundUp {
        return mulDivRoundingUp(liquidity, diff, Q96Big)
    }

    result := new(big.Int).Mul(liquidity, diff)
    result.Div(result, Q96Big)
    return result
}

// getNextSqrtPriceFromInput 根据输入计算新价格
func getNextSqrtPriceFromInput(sqrtPriceX96, liquidity, amountIn *big.Int, zeroForOne bool) *big.Int {
    if zeroForOne {
        return getNextSqrtPriceFromAmount0RoundingUp(sqrtPriceX96, liquidity, amountIn, true)
    }
    return getNextSqrtPriceFromAmount1RoundingDown(sqrtPriceX96, liquidity, amountIn, true)
}

// getNextSqrtPriceFromOutput 根据输出计算新价格
func getNextSqrtPriceFromOutput(sqrtPriceX96, liquidity, amountOut *big.Int, zeroForOne bool) *big.Int {
    if zeroForOne {
        return getNextSqrtPriceFromAmount1RoundingDown(sqrtPriceX96, liquidity, amountOut, false)
    }
    return getNextSqrtPriceFromAmount0RoundingUp(sqrtPriceX96, liquidity, amountOut, false)
}

// 辅助数学函数
func getNextSqrtPriceFromAmount0RoundingUp(sqrtPriceX96, liquidity, amount *big.Int, add bool) *big.Int {
    if amount.Sign() == 0 {
        return sqrtPriceX96
    }

    numerator1 := new(big.Int).Lsh(liquidity, 96)

    if add {
        product := new(big.Int).Mul(amount, sqrtPriceX96)
        denominator := new(big.Int).Add(numerator1, product)
        result := mulDivRoundingUp(numerator1, sqrtPriceX96, denominator)
        return result
    } else {
        product := new(big.Int).Mul(amount, sqrtPriceX96)
        denominator := new(big.Int).Sub(numerator1, product)
        result := mulDivRoundingUp(numerator1, sqrtPriceX96, denominator)
        return result
    }
}

func getNextSqrtPriceFromAmount1RoundingDown(sqrtPriceX96, liquidity, amount *big.Int, add bool) *big.Int {
    if add {
        quotient := new(big.Int).Lsh(amount, 96)
        quotient.Div(quotient, liquidity)
        return new(big.Int).Add(sqrtPriceX96, quotient)
    } else {
        quotient := divRoundingUp(new(big.Int).Lsh(amount, 96), liquidity)
        return new(big.Int).Sub(sqrtPriceX96, quotient)
    }
}

func mulDivRoundingUp(a, b, denominator *big.Int) *big.Int {
    product := new(big.Int).Mul(a, b)
    result := new(big.Int).Div(product, denominator)

    remainder := new(big.Int).Mod(product, denominator)
    if remainder.Sign() > 0 {
        result.Add(result, big.NewInt(1))
    }

    return result
}

func divRoundingUp(a, b *big.Int) *big.Int {
    result := new(big.Int).Div(a, b)
    remainder := new(big.Int).Mod(a, b)
    if remainder.Sign() > 0 {
        result.Add(result, big.NewInt(1))
    }
    return result
}
```

### 73.4 V3 流动性计算

```go
// V3 流动性公式
//
// 当价格在区间内 [P_lower, P_upper]:
// L = Δx * √P_upper * √P / (√P_upper - √P)
// L = Δy / (√P - √P_lower)
//
// 当价格低于区间 (P < P_lower):
// 所有流动性转为 token0
// Δx = L * (1/√P_lower - 1/√P_upper)
//
// 当价格高于区间 (P > P_upper):
// 所有流动性转为 token1
// Δy = L * (√P_upper - √P_lower)

// LiquidityAmounts 流动性金额计算
type LiquidityAmounts struct{}

// GetLiquidityForAmounts 根据存入金额计算流动性
func (la *LiquidityAmounts) GetLiquidityForAmounts(
    sqrtRatioX96 *big.Int,      // 当前价格
    sqrtRatioAX96 *big.Int,     // 区间下界
    sqrtRatioBX96 *big.Int,     // 区间上界
    amount0 *big.Int,           // Token0 数量
    amount1 *big.Int,           // Token1 数量
) *big.Int {
    if sqrtRatioAX96.Cmp(sqrtRatioBX96) > 0 {
        sqrtRatioAX96, sqrtRatioBX96 = sqrtRatioBX96, sqrtRatioAX96
    }

    var liquidity *big.Int

    if sqrtRatioX96.Cmp(sqrtRatioAX96) <= 0 {
        // 当前价格低于区间，全部是 token0
        liquidity = getLiquidityForAmount0(sqrtRatioAX96, sqrtRatioBX96, amount0)
    } else if sqrtRatioX96.Cmp(sqrtRatioBX96) < 0 {
        // 当前价格在区间内
        liquidity0 := getLiquidityForAmount0(sqrtRatioX96, sqrtRatioBX96, amount0)
        liquidity1 := getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioX96, amount1)

        if liquidity0.Cmp(liquidity1) < 0 {
            liquidity = liquidity0
        } else {
            liquidity = liquidity1
        }
    } else {
        // 当前价格高于区间，全部是 token1
        liquidity = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioBX96, amount1)
    }

    return liquidity
}

// GetAmountsForLiquidity 根据流动性计算对应的代币数量
func (la *LiquidityAmounts) GetAmountsForLiquidity(
    sqrtRatioX96 *big.Int,
    sqrtRatioAX96 *big.Int,
    sqrtRatioBX96 *big.Int,
    liquidity *big.Int,
) (amount0, amount1 *big.Int) {
    if sqrtRatioAX96.Cmp(sqrtRatioBX96) > 0 {
        sqrtRatioAX96, sqrtRatioBX96 = sqrtRatioBX96, sqrtRatioAX96
    }

    if sqrtRatioX96.Cmp(sqrtRatioAX96) <= 0 {
        // 价格低于区间
        amount0 = getAmount0ForLiquidity(sqrtRatioAX96, sqrtRatioBX96, liquidity)
        amount1 = big.NewInt(0)
    } else if sqrtRatioX96.Cmp(sqrtRatioBX96) < 0 {
        // 价格在区间内
        amount0 = getAmount0ForLiquidity(sqrtRatioX96, sqrtRatioBX96, liquidity)
        amount1 = getAmount1ForLiquidity(sqrtRatioAX96, sqrtRatioX96, liquidity)
    } else {
        // 价格高于区间
        amount0 = big.NewInt(0)
        amount1 = getAmount1ForLiquidity(sqrtRatioAX96, sqrtRatioBX96, liquidity)
    }

    return
}

func getLiquidityForAmount0(sqrtRatioAX96, sqrtRatioBX96, amount0 *big.Int) *big.Int {
    if sqrtRatioAX96.Cmp(sqrtRatioBX96) > 0 {
        sqrtRatioAX96, sqrtRatioBX96 = sqrtRatioBX96, sqrtRatioAX96
    }

    intermediate := new(big.Int).Mul(sqrtRatioAX96, sqrtRatioBX96)
    intermediate.Div(intermediate, Q96Big)

    numerator := new(big.Int).Mul(amount0, intermediate)
    denominator := new(big.Int).Sub(sqrtRatioBX96, sqrtRatioAX96)

    return new(big.Int).Div(numerator, denominator)
}

func getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioBX96, amount1 *big.Int) *big.Int {
    if sqrtRatioAX96.Cmp(sqrtRatioBX96) > 0 {
        sqrtRatioAX96, sqrtRatioBX96 = sqrtRatioBX96, sqrtRatioAX96
    }

    diff := new(big.Int).Sub(sqrtRatioBX96, sqrtRatioAX96)

    return new(big.Int).Div(
        new(big.Int).Mul(amount1, Q96Big),
        diff,
    )
}

func getAmount0ForLiquidity(sqrtRatioAX96, sqrtRatioBX96, liquidity *big.Int) *big.Int {
    if sqrtRatioAX96.Cmp(sqrtRatioBX96) > 0 {
        sqrtRatioAX96, sqrtRatioBX96 = sqrtRatioBX96, sqrtRatioAX96
    }

    diff := new(big.Int).Sub(sqrtRatioBX96, sqrtRatioAX96)
    numerator := new(big.Int).Lsh(new(big.Int).Mul(liquidity, diff), 96)
    denominator := new(big.Int).Mul(sqrtRatioBX96, sqrtRatioAX96)

    return new(big.Int).Div(numerator, denominator)
}

func getAmount1ForLiquidity(sqrtRatioAX96, sqrtRatioBX96, liquidity *big.Int) *big.Int {
    if sqrtRatioAX96.Cmp(sqrtRatioBX96) > 0 {
        sqrtRatioAX96, sqrtRatioBX96 = sqrtRatioBX96, sqrtRatioAX96
    }

    diff := new(big.Int).Sub(sqrtRatioBX96, sqrtRatioAX96)

    return new(big.Int).Div(
        new(big.Int).Mul(liquidity, diff),
        Q96Big,
    )
}
```

---

## 第74节：Curve StableSwap 数学

### 74.1 StableSwap 不变量

```
┌─────────────────────────────────────────────────────────────────┐
│              Curve StableSwap 曲线                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  恒定乘积 vs 恒定和 vs StableSwap:                               │
│                                                                 │
│  Token Y                                                        │
│    ▲                                                            │
│    │   恒定和 (x + y = D)                                       │
│    │    \                                                       │
│    │     \    StableSwap                                        │
│    │      \     (混合曲线)                                       │
│    │       \   ╱                                                │
│    │        \ ╱                                                 │
│    │         X─────                                             │
│    │        ╱ \                                                 │
│    │       ╱   \                                                │
│    │      ╱     \                                               │
│    │     ╱       \  恒定乘积 (x * y = k)                        │
│    │    ╱         \                                             │
│    └───────────────────────────────────────────────► Token X   │
│                                                                 │
│  StableSwap 不变量:                                              │
│  A * n^n * Σx + D = A * D * n^n + D^(n+1) / (n^n * Πx)         │
│                                                                 │
│  其中:                                                           │
│  - A: 放大系数 (Amplification coefficient)                       │
│  - n: 代币数量                                                   │
│  - D: 总存款 (当所有代币等价时)                                   │
│  - Σx: 所有代币储备之和                                          │
│  - Πx: 所有代币储备之积                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 74.2 StableSwap 实现

```go
package curve

import (
    "errors"
    "math/big"
)

// StableSwap Curve StableSwap 池
type StableSwap struct {
    Balances   []*big.Int // 各代币余额
    A          *big.Int   // 放大系数 (实际值 * A_PRECISION)
    Fee        *big.Int   // 手续费 (基点)
    AdminFee   *big.Int   // 管理费 (手续费的百分比)
    Precision  *big.Int   // 精度乘数
    N          int        // 代币数量
}

const (
    A_PRECISION = 100     // A 的精度
    FEE_DENOMINATOR = 10000000000 // 手续费分母 (10^10)
)

var (
    APrecision     = big.NewInt(A_PRECISION)
    FeeDenominator = new(big.Int).SetUint64(FEE_DENOMINATOR)
)

// NewStableSwap 创建新的 StableSwap 池
func NewStableSwap(balances []*big.Int, a, fee, adminFee *big.Int) *StableSwap {
    return &StableSwap{
        Balances:  balances,
        A:         a,
        Fee:       fee,
        AdminFee:  adminFee,
        Precision: big.NewInt(1e18),
        N:         len(balances),
    }
}

// GetD 计算不变量 D
// 使用牛顿迭代法求解:
// D^(n+1) / (n^n * Πx) + D * (A*n^n - 1) = A * n^n * Σx
func (ss *StableSwap) GetD(balances []*big.Int) *big.Int {
    n := len(balances)
    nBig := big.NewInt(int64(n))

    // 计算 Σx
    sum := big.NewInt(0)
    for _, b := range balances {
        sum.Add(sum, b)
    }

    if sum.Sign() == 0 {
        return big.NewInt(0)
    }

    // 初始 D = Σx
    d := new(big.Int).Set(sum)

    // Ann = A * n^n
    ann := new(big.Int).Set(ss.A)
    for i := 0; i < n; i++ {
        ann.Mul(ann, nBig)
    }

    // 牛顿迭代
    for i := 0; i < 255; i++ {
        dPrev := new(big.Int).Set(d)

        // D_P = D^(n+1) / (n^n * Πx)
        dP := new(big.Int).Set(d)
        for _, balance := range balances {
            // D_P = D_P * D / (balance * n)
            dP.Mul(dP, d)
            dP.Div(dP, new(big.Int).Mul(balance, nBig))
        }

        // numerator = (Ann * sum / A_PRECISION + D_P * n) * D
        numerator := new(big.Int).Mul(ann, sum)
        numerator.Div(numerator, APrecision)
        numerator.Add(numerator, new(big.Int).Mul(dP, nBig))
        numerator.Mul(numerator, d)

        // denominator = (Ann - A_PRECISION) * D / A_PRECISION + (n + 1) * D_P
        denominator := new(big.Int).Sub(ann, APrecision)
        denominator.Mul(denominator, d)
        denominator.Div(denominator, APrecision)
        denominator.Add(denominator, new(big.Int).Mul(big.NewInt(int64(n+1)), dP))

        d.Div(numerator, denominator)

        // 检查收敛
        diff := new(big.Int).Sub(d, dPrev)
        if diff.Sign() < 0 {
            diff.Neg(diff)
        }
        if diff.Cmp(big.NewInt(1)) <= 0 {
            return d
        }
    }

    return d
}

// GetY 计算给定其他代币余额时，某代币的均衡余额
// 用于计算 swap 输出
func (ss *StableSwap) GetY(i, j int, x *big.Int, balances []*big.Int) *big.Int {
    n := len(balances)
    nBig := big.NewInt(int64(n))

    d := ss.GetD(balances)

    // Ann = A * n^n
    ann := new(big.Int).Set(ss.A)
    for k := 0; k < n; k++ {
        ann.Mul(ann, nBig)
    }

    // c = D^(n+1) / (n^n * Ann * Π(xj for j != y_index))
    c := new(big.Int).Set(d)
    sum := big.NewInt(0)

    for k := 0; k < n; k++ {
        var xk *big.Int
        if k == i {
            xk = x
        } else if k != j {
            xk = balances[k]
        } else {
            continue
        }

        sum.Add(sum, xk)
        c.Mul(c, d)
        c.Div(c, new(big.Int).Mul(xk, nBig))
    }

    c.Mul(c, d)
    c.Mul(c, APrecision)
    c.Div(c, new(big.Int).Mul(ann, nBig))

    // b = sum + D * A_PRECISION / Ann
    b := new(big.Int).Add(sum, new(big.Int).Div(
        new(big.Int).Mul(d, APrecision),
        ann,
    ))

    // 牛顿迭代求 y
    y := new(big.Int).Set(d)

    for i := 0; i < 255; i++ {
        yPrev := new(big.Int).Set(y)

        // y = (y^2 + c) / (2*y + b - D)
        numerator := new(big.Int).Mul(y, y)
        numerator.Add(numerator, c)

        denominator := new(big.Int).Mul(big.NewInt(2), y)
        denominator.Add(denominator, b)
        denominator.Sub(denominator, d)

        y.Div(numerator, denominator)

        diff := new(big.Int).Sub(y, yPrev)
        if diff.Sign() < 0 {
            diff.Neg(diff)
        }
        if diff.Cmp(big.NewInt(1)) <= 0 {
            return y
        }
    }

    return y
}

// GetDY 计算交换输出金额
func (ss *StableSwap) GetDY(i, j int, dx *big.Int) *big.Int {
    // 新的 x 余额
    newX := new(big.Int).Add(ss.Balances[i], dx)

    // 计算新的 y 余额
    newY := ss.GetY(i, j, newX, ss.Balances)

    // dy = old_y - new_y - 1 (向下取整)
    dy := new(big.Int).Sub(ss.Balances[j], newY)
    dy.Sub(dy, big.NewInt(1))

    // 扣除手续费
    fee := new(big.Int).Mul(dy, ss.Fee)
    fee.Div(fee, FeeDenominator)

    return new(big.Int).Sub(dy, fee)
}

// Exchange 执行交换
func (ss *StableSwap) Exchange(i, j int, dx *big.Int) (*big.Int, error) {
    if i == j {
        return nil, errors.New("same coin")
    }
    if i < 0 || i >= ss.N || j < 0 || j >= ss.N {
        return nil, errors.New("invalid coin index")
    }

    dy := ss.GetDY(i, j, dx)

    if dy.Sign() <= 0 {
        return nil, errors.New("insufficient output")
    }

    // 更新余额
    ss.Balances[i].Add(ss.Balances[i], dx)
    ss.Balances[j].Sub(ss.Balances[j], dy)

    return dy, nil
}
```

### 74.3 StableSwap 价格影响

```go
// 在 A=100, n=2 的情况下:
//
// 储备比例    恒定乘积滑点    StableSwap滑点
// 1:1        0%             0%
// 1.01:1     0.5%           0.005%
// 1.1:1      4.5%           0.045%
// 2:1        25%            2.5%

// CalculateStableSwapPriceImpact 计算稳定币交换的价格影响
func (ss *StableSwap) CalculateStableSwapPriceImpact(i, j int, dx *big.Int) *big.Float {
    // 小额交易的价格 (近似即时价格)
    smallDx := big.NewInt(1e15) // 0.001 token
    smallDy := ss.GetDY(i, j, smallDx)

    spotPrice := new(big.Float).Quo(
        new(big.Float).SetInt(smallDy),
        new(big.Float).SetInt(smallDx),
    )

    // 实际交易价格
    dy := ss.GetDY(i, j, dx)
    executionPrice := new(big.Float).Quo(
        new(big.Float).SetInt(dy),
        new(big.Float).SetInt(dx),
    )

    // 价格影响 = (spotPrice - executionPrice) / spotPrice * 100
    impact := new(big.Float).Sub(spotPrice, executionPrice)
    impact.Quo(impact, spotPrice)
    impact.Mul(impact, big.NewFloat(100))

    return impact
}

// StableSwap 虚拟价格
// virtual_price = D / total_supply
// 用于衡量 LP 代币的内在价值

// GetVirtualPrice 计算虚拟价格
func (ss *StableSwap) GetVirtualPrice(totalSupply *big.Int) *big.Float {
    d := ss.GetD(ss.Balances)

    if totalSupply.Sign() == 0 {
        return big.NewFloat(0)
    }

    vp := new(big.Float).Quo(
        new(big.Float).SetInt(d),
        new(big.Float).SetInt(totalSupply),
    )

    return vp
}
```

---

## 第75节：Balancer 加权池数学

### 75.1 加权常数乘积

```
┌─────────────────────────────────────────────────────────────────┐
│              Balancer 加权池模型                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  恒定加权乘积不变量:                                              │
│  Π(Bi^Wi) = k                                                   │
│                                                                 │
│  其中:                                                           │
│  - Bi: 代币 i 的余额                                             │
│  - Wi: 代币 i 的权重 (Σ Wi = 1)                                  │
│  - k: 不变量                                                     │
│                                                                 │
│  示例 (80/20 ETH/USDC 池):                                       │
│  B_ETH^0.8 * B_USDC^0.2 = k                                     │
│                                                                 │
│  与 Uniswap 对比:                                                │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Uniswap (50/50):           Balancer (80/20):             ││
│  │  B_ETH^0.5 * B_USDC^0.5 = k  B_ETH^0.8 * B_USDC^0.2 = k  ││
│  │                                                            ││
│  │  - 无常损失对称               - ETH波动影响更大            ││
│  │  - 两代币风险相等             - 可定制风险敞口              ││
│  │  - 资本效率固定               - 可优化特定策略              ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  价格公式:                                                       │
│  P_i/j = (B_j / B_i) * (W_i / W_j)                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 75.2 Balancer 数学实现

```go
package balancer

import (
    "math"
    "math/big"
)

// WeightedPool Balancer 加权池
type WeightedPool struct {
    Tokens    []string   // 代币地址
    Balances  []*big.Int // 代币余额
    Weights   []float64  // 代币权重 (归一化，总和为1)
    SwapFee   float64    // 交换手续费 (如 0.003 = 0.3%)
}

// NewWeightedPool 创建加权池
func NewWeightedPool(tokens []string, balances []*big.Int, weights []float64, swapFee float64) *WeightedPool {
    // 归一化权重
    totalWeight := 0.0
    for _, w := range weights {
        totalWeight += w
    }

    normalizedWeights := make([]float64, len(weights))
    for i, w := range weights {
        normalizedWeights[i] = w / totalWeight
    }

    return &WeightedPool{
        Tokens:   tokens,
        Balances: balances,
        Weights:  normalizedWeights,
        SwapFee:  swapFee,
    }
}

// SpotPrice 计算即时价格 (tokenIn/tokenOut)
// spotPrice = (balanceOut / weightOut) / (balanceIn / weightIn)
func (wp *WeightedPool) SpotPrice(tokenIn, tokenOut int) float64 {
    balanceIn := float64FromBigInt(wp.Balances[tokenIn])
    balanceOut := float64FromBigInt(wp.Balances[tokenOut])
    weightIn := wp.Weights[tokenIn]
    weightOut := wp.Weights[tokenOut]

    return (balanceOut / weightOut) / (balanceIn / weightIn)
}

// CalcOutGivenIn 计算给定输入的输出
// Ao = Bo * (1 - (Bi / (Bi + Ai))^(Wi/Wo))
func (wp *WeightedPool) CalcOutGivenIn(tokenIn, tokenOut int, amountIn *big.Int) *big.Int {
    balanceIn := float64FromBigInt(wp.Balances[tokenIn])
    balanceOut := float64FromBigInt(wp.Balances[tokenOut])
    weightIn := wp.Weights[tokenIn]
    weightOut := wp.Weights[tokenOut]
    amountInFloat := float64FromBigInt(amountIn)

    // 扣除手续费
    adjustedIn := amountInFloat * (1 - wp.SwapFee)

    // 计算输出
    // y = balanceOut * (1 - (balanceIn / (balanceIn + adjustedIn))^(weightIn/weightOut))
    ratio := balanceIn / (balanceIn + adjustedIn)
    power := math.Pow(ratio, weightIn/weightOut)
    amountOut := balanceOut * (1 - power)

    return bigIntFromFloat64(amountOut)
}

// CalcInGivenOut 计算给定输出需要的输入
// Ai = Bi * ((Bo / (Bo - Ao))^(Wo/Wi) - 1)
func (wp *WeightedPool) CalcInGivenOut(tokenIn, tokenOut int, amountOut *big.Int) *big.Int {
    balanceIn := float64FromBigInt(wp.Balances[tokenIn])
    balanceOut := float64FromBigInt(wp.Balances[tokenOut])
    weightIn := wp.Weights[tokenIn]
    weightOut := wp.Weights[tokenOut]
    amountOutFloat := float64FromBigInt(amountOut)

    // 计算输入 (不含手续费)
    ratio := balanceOut / (balanceOut - amountOutFloat)
    power := math.Pow(ratio, weightOut/weightIn)
    amountIn := balanceIn * (power - 1)

    // 加上手续费
    amountIn = amountIn / (1 - wp.SwapFee)

    return bigIntFromFloat64(amountIn)
}

// CalcSpotPriceAfterSwap 计算交换后的即时价格
func (wp *WeightedPool) CalcSpotPriceAfterSwap(tokenIn, tokenOut int, amountIn *big.Int) float64 {
    amountOut := wp.CalcOutGivenIn(tokenIn, tokenOut, amountIn)

    // 新余额
    newBalanceIn := new(big.Int).Add(wp.Balances[tokenIn], amountIn)
    newBalanceOut := new(big.Int).Sub(wp.Balances[tokenOut], amountOut)

    balIn := float64FromBigInt(newBalanceIn)
    balOut := float64FromBigInt(newBalanceOut)

    return (balOut / wp.Weights[tokenOut]) / (balIn / wp.Weights[tokenIn])
}
```

### 75.3 多代币交换路径

```go
// BalancerSmartOrderRouter Balancer 智能订单路由
type BalancerSmartOrderRouter struct {
    Pools []*WeightedPool
}

// SwapPath 交换路径
type SwapPath struct {
    Pools      []*WeightedPool
    TokensIn   []int
    TokensOut  []int
    AmountsIn  []*big.Int
    AmountsOut []*big.Int
    TotalIn    *big.Int
    TotalOut   *big.Int
}

// FindBestPath 找到最佳交换路径
func (sor *BalancerSmartOrderRouter) FindBestPath(
    tokenIn, tokenOut string,
    amountIn *big.Int,
    maxHops int,
) *SwapPath {
    // 简化版：直接路径搜索
    // 实际实现需要图搜索算法 (Dijkstra, BFS 等)

    var bestPath *SwapPath
    var bestOutput *big.Int

    // 搜索单跳路径
    for _, pool := range sor.Pools {
        inIdx := findTokenIndex(pool.Tokens, tokenIn)
        outIdx := findTokenIndex(pool.Tokens, tokenOut)

        if inIdx >= 0 && outIdx >= 0 {
            output := pool.CalcOutGivenIn(inIdx, outIdx, amountIn)

            if bestOutput == nil || output.Cmp(bestOutput) > 0 {
                bestOutput = output
                bestPath = &SwapPath{
                    Pools:      []*WeightedPool{pool},
                    TokensIn:   []int{inIdx},
                    TokensOut:  []int{outIdx},
                    AmountsIn:  []*big.Int{amountIn},
                    AmountsOut: []*big.Int{output},
                    TotalIn:    amountIn,
                    TotalOut:   output,
                }
            }
        }
    }

    // 搜索多跳路径...
    if maxHops > 1 {
        // 实现多跳搜索
        multiHopPaths := sor.findMultiHopPaths(tokenIn, tokenOut, amountIn, maxHops)
        for _, path := range multiHopPaths {
            if bestOutput == nil || path.TotalOut.Cmp(bestOutput) > 0 {
                bestOutput = path.TotalOut
                bestPath = path
            }
        }
    }

    return bestPath
}

func findTokenIndex(tokens []string, target string) int {
    for i, t := range tokens {
        if t == target {
            return i
        }
    }
    return -1
}

func (sor *BalancerSmartOrderRouter) findMultiHopPaths(
    tokenIn, tokenOut string,
    amountIn *big.Int,
    maxHops int,
) []*SwapPath {
    // BFS 搜索多跳路径
    var paths []*SwapPath

    // 找到所有包含 tokenIn 的池
    type searchState struct {
        currentToken string
        amount       *big.Int
        path         *SwapPath
        depth        int
    }

    queue := []searchState{}

    for _, pool := range sor.Pools {
        inIdx := findTokenIndex(pool.Tokens, tokenIn)
        if inIdx < 0 {
            continue
        }

        // 对于池中每个其他代币
        for outIdx, outToken := range pool.Tokens {
            if outIdx == inIdx {
                continue
            }

            output := pool.CalcOutGivenIn(inIdx, outIdx, amountIn)

            path := &SwapPath{
                Pools:      []*WeightedPool{pool},
                TokensIn:   []int{inIdx},
                TokensOut:  []int{outIdx},
                AmountsIn:  []*big.Int{amountIn},
                AmountsOut: []*big.Int{output},
                TotalIn:    amountIn,
                TotalOut:   output,
            }

            if outToken == tokenOut {
                paths = append(paths, path)
            } else if maxHops > 1 {
                queue = append(queue, searchState{
                    currentToken: outToken,
                    amount:       output,
                    path:         path,
                    depth:        1,
                })
            }
        }
    }

    // 继续 BFS
    for len(queue) > 0 {
        state := queue[0]
        queue = queue[1:]

        if state.depth >= maxHops {
            continue
        }

        for _, pool := range sor.Pools {
            inIdx := findTokenIndex(pool.Tokens, state.currentToken)
            if inIdx < 0 {
                continue
            }

            for outIdx, outToken := range pool.Tokens {
                if outIdx == inIdx {
                    continue
                }

                output := pool.CalcOutGivenIn(inIdx, outIdx, state.amount)

                // 复制路径并扩展
                newPath := &SwapPath{
                    Pools:      append([]*WeightedPool{}, state.path.Pools...),
                    TokensIn:   append([]int{}, state.path.TokensIn...),
                    TokensOut:  append([]int{}, state.path.TokensOut...),
                    AmountsIn:  append([]*big.Int{}, state.path.AmountsIn...),
                    AmountsOut: append([]*big.Int{}, state.path.AmountsOut...),
                    TotalIn:    state.path.TotalIn,
                }
                newPath.Pools = append(newPath.Pools, pool)
                newPath.TokensIn = append(newPath.TokensIn, inIdx)
                newPath.TokensOut = append(newPath.TokensOut, outIdx)
                newPath.AmountsIn = append(newPath.AmountsIn, state.amount)
                newPath.AmountsOut = append(newPath.AmountsOut, output)
                newPath.TotalOut = output

                if outToken == tokenOut {
                    paths = append(paths, newPath)
                } else if state.depth+1 < maxHops {
                    queue = append(queue, searchState{
                        currentToken: outToken,
                        amount:       output,
                        path:         newPath,
                        depth:        state.depth + 1,
                    })
                }
            }
        }
    }

    return paths
}
```

---

## 第76节：无常损失（Impermanent Loss）

### 76.1 无常损失原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    无常损失原理图解                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  初始状态:                                                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  LP 投入: 1 ETH + 2000 USDC                                 ││
│  │  ETH 价格: $2000                                            ││
│  │  总价值: $4000                                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  价格变化后 (ETH → $4000):                                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                                                             ││
│  │  如果只是持有 (HODL):                                        ││
│  │  1 ETH = $4000, 2000 USDC = $2000                          ││
│  │  总价值: $6000                                              ││
│  │                                                             ││
│  │  如果提供流动性:                                             ││
│  │  新储备: 0.707 ETH + 2828 USDC (因为套利者平衡了池)          ││
│  │  0.707 × $4000 + $2828 = $5656                             ││
│  │                                                             ││
│  │  无常损失: $6000 - $5656 = $344 (5.7%)                      ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  无常损失公式 (50/50 池):                                        │
│  IL = 2 * √r / (1 + r) - 1                                     │
│  其中 r = 新价格 / 原价格                                        │
│                                                                 │
│  价格变化     无常损失                                            │
│  1.25x       0.6%                                               │
│  1.50x       2.0%                                               │
│  2x          5.7%                                               │
│  3x          13.4%                                              │
│  4x          20.0%                                              │
│  5x          25.5%                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 76.2 无常损失计算实现

```go
package impermanentloss

import (
    "math"
    "math/big"
)

// ImpermanentLossCalculator 无常损失计算器
type ImpermanentLossCalculator struct{}

// Position LP 头寸
type Position struct {
    Token0Amount  *big.Int // Token0 初始数量
    Token1Amount  *big.Int // Token1 初始数量
    InitialPrice  float64  // 初始价格 (Token1/Token0)
    Weight0       float64  // Token0 权重 (默认0.5)
    Weight1       float64  // Token1 权重 (默认0.5)
}

// ImpermanentLossResult 无常损失结果
type ImpermanentLossResult struct {
    HodlValue      *big.Float // 单纯持有的价值
    LPValue        *big.Float // LP 头寸的价值
    ILAbsolute     *big.Float // 绝对无常损失
    ILPercent      float64    // 无常损失百分比
    NewToken0      *big.Float // 新的 Token0 数量
    NewToken1      *big.Float // 新的 Token1 数量
}

// Calculate 计算无常损失
func (c *ImpermanentLossCalculator) Calculate(pos Position, newPrice float64) *ImpermanentLossResult {
    // 50/50 池的无常损失公式
    // IL = 2 * sqrt(r) / (1 + r) - 1
    // 其中 r = newPrice / initialPrice

    r := newPrice / pos.InitialPrice

    // 计算 HODL 价值
    token0Value := float64FromBigInt(pos.Token0Amount) * newPrice
    token1Value := float64FromBigInt(pos.Token1Amount)
    hodlValue := token0Value + token1Value

    // 计算 LP 价值
    // 根据恒定乘积，新的代币数量:
    // new_token0 = initial_token0 * sqrt(initialPrice / newPrice)
    // new_token1 = initial_token1 * sqrt(newPrice / initialPrice)
    sqrtR := math.Sqrt(r)
    newToken0 := float64FromBigInt(pos.Token0Amount) / sqrtR
    newToken1 := float64FromBigInt(pos.Token1Amount) * sqrtR

    lpValue := newToken0*newPrice + newToken1

    // 无常损失
    ilPercent := 2*sqrtR/(1+r) - 1
    ilAbsolute := hodlValue - lpValue

    return &ImpermanentLossResult{
        HodlValue:  big.NewFloat(hodlValue),
        LPValue:    big.NewFloat(lpValue),
        ILAbsolute: big.NewFloat(ilAbsolute),
        ILPercent:  ilPercent * 100, // 转为百分比
        NewToken0:  big.NewFloat(newToken0),
        NewToken1:  big.NewFloat(newToken1),
    }
}

// CalculateWeighted 计算加权池的无常损失
// Balancer 风格的加权池
func (c *ImpermanentLossCalculator) CalculateWeighted(pos Position, newPrice float64) *ImpermanentLossResult {
    w0 := pos.Weight0
    w1 := pos.Weight1
    r := newPrice / pos.InitialPrice

    // 加权池无常损失公式:
    // IL = (r^w0 * 1^w1) / (w0 * r + w1 * 1) - 1
    // 简化为: IL = r^w0 / (w0 * r + w1) - 1

    // HODL 价值
    token0Value := float64FromBigInt(pos.Token0Amount) * newPrice
    token1Value := float64FromBigInt(pos.Token1Amount)
    hodlValue := token0Value + token1Value

    // LP 价值比例
    lpRatio := math.Pow(r, w0) / (w0*r + w1)

    // 初始 LP 价值
    initialValue := float64FromBigInt(pos.Token0Amount)*pos.InitialPrice + float64FromBigInt(pos.Token1Amount)

    // 当前 LP 价值 (按比例增长)
    lpValue := initialValue * lpRatio * (w0*r + w1)

    ilAbsolute := hodlValue - lpValue
    ilPercent := (hodlValue - lpValue) / hodlValue

    return &ImpermanentLossResult{
        HodlValue:  big.NewFloat(hodlValue),
        LPValue:    big.NewFloat(lpValue),
        ILAbsolute: big.NewFloat(ilAbsolute),
        ILPercent:  ilPercent * 100,
    }
}

// BreakEvenFees 计算需要多少手续费收入才能弥补无常损失
func (c *ImpermanentLossCalculator) BreakEvenFees(ilPercent float64, positionValue float64) float64 {
    // 需要的手续费 = IL 金额
    return positionValue * ilPercent / 100
}

// CalculateILAtPriceRange 计算价格区间内的无常损失
func (c *ImpermanentLossCalculator) CalculateILAtPriceRange(initialPrice float64, priceRange []float64) []float64 {
    results := make([]float64, len(priceRange))

    for i, price := range priceRange {
        r := price / initialPrice
        sqrtR := math.Sqrt(r)
        il := 2*sqrtR/(1+r) - 1
        results[i] = il * 100 // 百分比
    }

    return results
}

// V3 集中流动性的无常损失（更复杂）
// ConcentratedIL 计算 Uniswap V3 风格的集中流动性无常损失
func (c *ImpermanentLossCalculator) ConcentratedIL(
    initialPrice float64,
    newPrice float64,
    priceLower float64,
    priceUpper float64,
) float64 {
    // V3 无常损失取决于价格区间
    // 如果价格移出区间，损失可能更大

    // 计算初始流动性分配
    sqrtP := math.Sqrt(initialPrice)
    sqrtPa := math.Sqrt(priceLower)
    sqrtPb := math.Sqrt(priceUpper)

    // 假设初始 1 单位 token0 价值
    // L = token0 * sqrt(P) * sqrt(Pb) / (sqrt(Pb) - sqrt(P))
    // 当 P 在区间内

    if newPrice < priceLower {
        // 价格低于区间 - 全部转为 token0
        // 相当于在 priceLower 卖出了所有 token1
        return c.calculateOutOfRangeIL(initialPrice, newPrice, priceLower, true)
    } else if newPrice > priceUpper {
        // 价格高于区间 - 全部转为 token1
        return c.calculateOutOfRangeIL(initialPrice, newPrice, priceUpper, false)
    }

    // 价格仍在区间内 - 使用标准公式但放大
    r := newPrice / initialPrice
    sqrtR := math.Sqrt(r)

    // 集中流动性放大因子
    concentration := (sqrtPb - sqrtPa) / (sqrtP * (sqrtPb - sqrtP) / sqrtPb + (sqrtP - sqrtPa))

    // 放大的无常损失
    il := (2*sqrtR/(1+r) - 1) * concentration

    return il * 100
}

func (c *ImpermanentLossCalculator) calculateOutOfRangeIL(
    initialPrice, newPrice, boundaryPrice float64,
    isLower bool,
) float64 {
    // 简化计算：价格移出区间时的损失
    if isLower {
        // 全部变成 token0
        // 损失 = 持有 token1 相比持有 token0 的损失
        return (1 - newPrice/boundaryPrice) * 100
    }
    // 全部变成 token1
    return (1 - boundaryPrice/newPrice) * 100
}
```

### 76.3 无常损失对冲策略

```go
// ILHedgeStrategy 无常损失对冲策略
type ILHedgeStrategy struct {
    Position      Position
    HedgeType     string
    HedgeAmount   *big.Int
    HedgeCost     *big.Int
}

// HedgeWithOptions 使用期权对冲
// 买入看跌期权保护下行风险
func HedgeWithOptions(pos Position, strikePrice float64, premium float64) *ILHedgeStrategy {
    // 计算需要多少期权
    // 通常对冲池中 token0 的一半
    hedgeAmount := new(big.Int).Div(pos.Token0Amount, big.NewInt(2))

    // 期权成本
    cost := float64FromBigInt(hedgeAmount) * premium

    return &ILHedgeStrategy{
        Position:    pos,
        HedgeType:   "put_option",
        HedgeAmount: hedgeAmount,
        HedgeCost:   bigIntFromFloat64(cost),
    }
}

// HedgeWithPerps 使用永续合约对冲
// 开空仓对冲多头敞口
func HedgeWithPerps(pos Position, leverage float64) *ILHedgeStrategy {
    // 计算 delta (价格敏感度)
    // 对于 50/50 池，delta ≈ 0.5 (一半暴露于 token0)
    delta := 0.5

    // 需要做空的数量
    shortAmount := float64FromBigInt(pos.Token0Amount) * delta

    return &ILHedgeStrategy{
        Position:    pos,
        HedgeType:   "perp_short",
        HedgeAmount: bigIntFromFloat64(shortAmount),
    }
}

// DeltaNeutralStrategy Delta 中性策略
type DeltaNeutralStrategy struct {
    LPPosition    Position
    ShortPosition *big.Int // 做空数量
    CurrentDelta  float64  // 当前 Delta
    TargetDelta   float64  // 目标 Delta (通常为 0)
}

// RebalanceDelta 重新平衡 Delta
func (dns *DeltaNeutralStrategy) RebalanceDelta(currentPrice float64) *big.Int {
    // 计算当前 LP 的 delta
    // delta = 0.5 * (1 + IL导数)
    // 对于恒定乘积：delta ≈ token0价值 / 总价值

    token0Value := float64FromBigInt(dns.LPPosition.Token0Amount) * currentPrice
    token1Value := float64FromBigInt(dns.LPPosition.Token1Amount)
    totalValue := token0Value + token1Value

    currentLPDelta := token0Value / totalValue

    // 当前总 delta = LP delta - short delta
    shortValue := float64FromBigInt(dns.ShortPosition) * currentPrice
    shortDelta := shortValue / totalValue

    dns.CurrentDelta = currentLPDelta - shortDelta

    // 需要调整的做空数量
    deltaAdjustment := dns.CurrentDelta - dns.TargetDelta
    adjustmentAmount := deltaAdjustment * totalValue / currentPrice

    return bigIntFromFloat64(adjustmentAmount)
}
```

---

## 第77节：滑点与最优执行

### 77.1 滑点计算

```go
package slippage

import (
    "math/big"
)

// SlippageCalculator 滑点计算器
type SlippageCalculator struct{}

// SlippageResult 滑点结果
type SlippageResult struct {
    ExpectedOutput    *big.Int   // 期望输出（按即时价格）
    ActualOutput      *big.Int   // 实际输出
    SlippageAmount    *big.Int   // 滑点金额
    SlippagePercent   float64    // 滑点百分比
    PriceImpact       float64    // 价格影响
    EffectivePrice    float64    // 实际成交价格
}

// CalculateSlippage 计算滑点
func (sc *SlippageCalculator) CalculateSlippage(
    amountIn *big.Int,
    reserveIn *big.Int,
    reserveOut *big.Int,
    fee uint64, // 基点
) *SlippageResult {
    // 即时价格 = reserveOut / reserveIn
    spotPrice := new(big.Float).Quo(
        new(big.Float).SetInt(reserveOut),
        new(big.Float).SetInt(reserveIn),
    )

    // 期望输出 (无滑点)
    expectedFloat := new(big.Float).Mul(
        new(big.Float).SetInt(amountIn),
        spotPrice,
    )
    expectedOutput, _ := expectedFloat.Int(nil)

    // 实际输出 (Uniswap V2 公式)
    feeMultiplier := big.NewInt(10000 - int64(fee))
    amountInWithFee := new(big.Int).Mul(amountIn, feeMultiplier)

    numerator := new(big.Int).Mul(amountInWithFee, reserveOut)
    denominator := new(big.Int).Mul(reserveIn, big.NewInt(10000))
    denominator.Add(denominator, amountInWithFee)

    actualOutput := new(big.Int).Div(numerator, denominator)

    // 滑点
    slippageAmount := new(big.Int).Sub(expectedOutput, actualOutput)

    slippagePercent := 0.0
    if expectedOutput.Sign() > 0 {
        expectedF, _ := new(big.Float).SetInt(expectedOutput).Float64()
        slippageF, _ := new(big.Float).SetInt(slippageAmount).Float64()
        slippagePercent = slippageF / expectedF * 100
    }

    // 价格影响 (不含手续费)
    // priceImpact = amountIn / (reserveIn + amountIn)
    priceImpactDenom := new(big.Int).Add(reserveIn, amountIn)
    amountInF, _ := new(big.Float).SetInt(amountIn).Float64()
    denomF, _ := new(big.Float).SetInt(priceImpactDenom).Float64()
    priceImpact := amountInF / denomF * 100

    // 有效价格
    actualF, _ := new(big.Float).SetInt(actualOutput).Float64()
    effectivePrice := actualF / amountInF

    return &SlippageResult{
        ExpectedOutput:  expectedOutput,
        ActualOutput:    actualOutput,
        SlippageAmount:  slippageAmount,
        SlippagePercent: slippagePercent,
        PriceImpact:     priceImpact,
        EffectivePrice:  effectivePrice,
    }
}

// MaxAmountForSlippage 计算给定滑点下的最大交易量
func (sc *SlippageCalculator) MaxAmountForSlippage(
    maxSlippagePercent float64,
    reserveIn *big.Int,
    reserveOut *big.Int,
) *big.Int {
    // 价格影响 = amountIn / (reserveIn + amountIn) = slippage
    // amountIn = slippage * reserveIn / (1 - slippage)

    slippage := maxSlippagePercent / 100
    reserveInF, _ := new(big.Float).SetInt(reserveIn).Float64()

    maxAmount := slippage * reserveInF / (1 - slippage)

    return bigIntFromFloat64(maxAmount)
}
```

### 77.2 最优交易拆分

```go
// TradeSplitter 交易拆分器
type TradeSplitter struct {
    Pools []PoolInfo
}

// PoolInfo 池信息
type PoolInfo struct {
    Address   string
    ReserveIn *big.Int
    ReserveOut *big.Int
    Fee       uint64
    Protocol  string // "uniswap_v2", "sushiswap", etc.
}

// SplitResult 拆分结果
type SplitResult struct {
    Splits       []TradeSplit
    TotalOutput  *big.Int
    GasCost      *big.Int
    NetOutput    *big.Int
}

// TradeSplit 单个拆分
type TradeSplit struct {
    Pool       *PoolInfo
    AmountIn   *big.Int
    AmountOut  *big.Int
    Percentage float64
}

// OptimalSplit 计算最优拆分
// 使用拉格朗日乘数法或数值优化
func (ts *TradeSplitter) OptimalSplit(totalAmountIn *big.Int, gasPrice *big.Int) *SplitResult {
    n := len(ts.Pools)
    if n == 0 {
        return nil
    }

    if n == 1 {
        // 单池，无需拆分
        output := ts.calculateOutput(0, totalAmountIn)
        return &SplitResult{
            Splits: []TradeSplit{{
                Pool:       &ts.Pools[0],
                AmountIn:   totalAmountIn,
                AmountOut:  output,
                Percentage: 100,
            }},
            TotalOutput: output,
        }
    }

    // 多池优化 - 使用梯度下降
    // 初始均匀分配
    splits := make([]float64, n)
    for i := range splits {
        splits[i] = 1.0 / float64(n)
    }

    totalIn, _ := new(big.Float).SetInt(totalAmountIn).Float64()

    // 梯度下降迭代
    learningRate := 0.01
    for iter := 0; iter < 1000; iter++ {
        // 计算梯度
        gradients := make([]float64, n)
        epsilon := 0.001

        currentOutput := ts.calculateTotalOutput(splits, totalIn)

        for i := range splits {
            // 数值梯度
            splits[i] += epsilon
            newOutput := ts.calculateTotalOutput(splits, totalIn)
            gradients[i] = (newOutput - currentOutput) / epsilon
            splits[i] -= epsilon
        }

        // 更新权重
        for i := range splits {
            splits[i] += learningRate * gradients[i]
            if splits[i] < 0 {
                splits[i] = 0
            }
        }

        // 归一化
        total := 0.0
        for _, s := range splits {
            total += s
        }
        for i := range splits {
            splits[i] /= total
        }
    }

    // 构建结果
    result := &SplitResult{
        Splits: make([]TradeSplit, 0, n),
    }

    totalOutput := big.NewInt(0)
    for i, split := range splits {
        if split < 0.01 {
            continue // 忽略太小的拆分
        }

        amountIn := bigIntFromFloat64(totalIn * split)
        amountOut := ts.calculateOutput(i, amountIn)

        result.Splits = append(result.Splits, TradeSplit{
            Pool:       &ts.Pools[i],
            AmountIn:   amountIn,
            AmountOut:  amountOut,
            Percentage: split * 100,
        })

        totalOutput.Add(totalOutput, amountOut)
    }

    result.TotalOutput = totalOutput

    // 计算 gas 成本
    gasPerSwap := big.NewInt(150000) // 估算每次 swap 的 gas
    totalGas := new(big.Int).Mul(gasPerSwap, big.NewInt(int64(len(result.Splits))))
    result.GasCost = new(big.Int).Mul(totalGas, gasPrice)

    // 净输出
    result.NetOutput = new(big.Int).Sub(totalOutput, result.GasCost)

    return result
}

func (ts *TradeSplitter) calculateOutput(poolIdx int, amountIn *big.Int) *big.Int {
    pool := ts.Pools[poolIdx]

    feeMultiplier := big.NewInt(10000 - int64(pool.Fee))
    amountInWithFee := new(big.Int).Mul(amountIn, feeMultiplier)

    numerator := new(big.Int).Mul(amountInWithFee, pool.ReserveOut)
    denominator := new(big.Int).Mul(pool.ReserveIn, big.NewInt(10000))
    denominator.Add(denominator, amountInWithFee)

    return new(big.Int).Div(numerator, denominator)
}

func (ts *TradeSplitter) calculateTotalOutput(splits []float64, totalIn float64) float64 {
    total := 0.0
    for i, split := range splits {
        amountIn := bigIntFromFloat64(totalIn * split)
        output := ts.calculateOutput(i, amountIn)
        outputF, _ := new(big.Float).SetInt(output).Float64()
        total += outputF
    }
    return total
}
```

### 77.3 多路径路由优化

```go
// MultiPathRouter 多路径路由器
type MultiPathRouter struct {
    Graph       *DEXGraph
    MaxHops     int
    MaxSplits   int
}

// DEXGraph DEX 交易图
type DEXGraph struct {
    Nodes map[string]bool           // 代币地址
    Edges map[string][]PoolEdge     // 代币 -> 可交易池列表
}

// PoolEdge 池边
type PoolEdge struct {
    Pool         *PoolInfo
    TokenOut     string
    Direction    bool // true: token0->token1, false: token1->token0
}

// Route 路由
type Route struct {
    Path       []string      // 代币路径
    Pools      []*PoolInfo   // 使用的池
    Directions []bool        // 交易方向
    AmountIn   *big.Int
    AmountOut  *big.Int
}

// FindBestRoutes 找到最佳路由组合
func (mpr *MultiPathRouter) FindBestRoutes(
    tokenIn, tokenOut string,
    amountIn *big.Int,
) ([]*Route, *big.Int) {
    // 1. 找到所有可能的路径
    allPaths := mpr.findAllPaths(tokenIn, tokenOut, mpr.MaxHops)

    if len(allPaths) == 0 {
        return nil, big.NewInt(0)
    }

    // 2. 评估每条路径的输出
    type pathOutput struct {
        path   *Route
        output *big.Int
    }

    pathOutputs := make([]pathOutput, 0, len(allPaths))
    for _, path := range allPaths {
        output := mpr.simulatePath(path, amountIn)
        if output.Sign() > 0 {
            pathOutputs = append(pathOutputs, pathOutput{path, output})
        }
    }

    // 3. 排序选择最佳路径
    // 简化版：选择输出最高的单路径
    // 高级版：使用凸优化进行多路径拆分

    var bestPath *Route
    bestOutput := big.NewInt(0)

    for _, po := range pathOutputs {
        if po.output.Cmp(bestOutput) > 0 {
            bestOutput = po.output
            bestPath = po.path
        }
    }

    if bestPath != nil {
        bestPath.AmountIn = amountIn
        bestPath.AmountOut = bestOutput
        return []*Route{bestPath}, bestOutput
    }

    return nil, big.NewInt(0)
}

func (mpr *MultiPathRouter) findAllPaths(start, end string, maxHops int) []*Route {
    var results []*Route

    // BFS 搜索所有路径
    type searchState struct {
        current string
        path    []string
        pools   []*PoolInfo
        dirs    []bool
        depth   int
    }

    visited := make(map[string]bool)
    queue := []searchState{{
        current: start,
        path:    []string{start},
        pools:   []*PoolInfo{},
        dirs:    []bool{},
        depth:   0,
    }}

    for len(queue) > 0 {
        state := queue[0]
        queue = queue[1:]

        if state.current == end {
            results = append(results, &Route{
                Path:       state.path,
                Pools:      state.pools,
                Directions: state.dirs,
            })
            continue
        }

        if state.depth >= maxHops {
            continue
        }

        // 探索相邻节点
        for _, edge := range mpr.Graph.Edges[state.current] {
            if visited[edge.TokenOut] && edge.TokenOut != end {
                continue
            }

            newPath := append([]string{}, state.path...)
            newPath = append(newPath, edge.TokenOut)

            newPools := append([]*PoolInfo{}, state.pools...)
            newPools = append(newPools, edge.Pool)

            newDirs := append([]bool{}, state.dirs...)
            newDirs = append(newDirs, edge.Direction)

            queue = append(queue, searchState{
                current: edge.TokenOut,
                path:    newPath,
                pools:   newPools,
                dirs:    newDirs,
                depth:   state.depth + 1,
            })
        }

        visited[state.current] = true
    }

    return results
}

func (mpr *MultiPathRouter) simulatePath(route *Route, amountIn *big.Int) *big.Int {
    current := new(big.Int).Set(amountIn)

    for i, pool := range route.Pools {
        var reserveIn, reserveOut *big.Int
        if route.Directions[i] {
            reserveIn = pool.ReserveIn
            reserveOut = pool.ReserveOut
        } else {
            reserveIn = pool.ReserveOut
            reserveOut = pool.ReserveIn
        }

        feeMultiplier := big.NewInt(10000 - int64(pool.Fee))
        amountInWithFee := new(big.Int).Mul(current, feeMultiplier)

        numerator := new(big.Int).Mul(amountInWithFee, reserveOut)
        denominator := new(big.Int).Mul(reserveIn, big.NewInt(10000))
        denominator.Add(denominator, amountInWithFee)

        current = new(big.Int).Div(numerator, denominator)

        if current.Sign() <= 0 {
            return big.NewInt(0)
        }
    }

    return current
}
```

---

## 第78节：跨 DEX 套利数学

### 78.1 套利机会检测

```go
package arbitrage

import (
    "math/big"
    "sort"
)

// ArbitrageDetector 套利机会检测器
type ArbitrageDetector struct {
    Pools       map[string][]*PoolInfo // tokenPair -> pools
    MinProfit   *big.Int               // 最小利润阈值
    GasPrice    *big.Int               // 当前 gas 价格
}

// ArbitrageOpportunity 套利机会
type ArbitrageOpportunity struct {
    Type          string       // "simple", "triangular", "cross_dex"
    Path          []string     // 代币路径
    Pools         []*PoolInfo  // 使用的池
    OptimalAmount *big.Int     // 最优输入
    ExpectedProfit *big.Int    // 预期利润
    GasCost       *big.Int     // Gas 成本
    NetProfit     *big.Int     // 净利润
    ProfitPercent float64      // 利润率
}

// DetectSimpleArbitrage 检测简单双池套利
func (ad *ArbitrageDetector) DetectSimpleArbitrage(tokenA, tokenB string) []*ArbitrageOpportunity {
    pairKey := tokenA + "-" + tokenB
    pools := ad.Pools[pairKey]

    if len(pools) < 2 {
        return nil
    }

    var opportunities []*ArbitrageOpportunity

    // 比较所有池对的价格
    for i := 0; i < len(pools); i++ {
        for j := i + 1; j < len(pools); j++ {
            poolA := pools[i]
            poolB := pools[j]

            // 计算价格
            priceA := calculatePrice(poolA)
            priceB := calculatePrice(poolB)

            // 检查价格差
            priceDiff := (priceA - priceB) / priceB
            if priceDiff < 0 {
                priceDiff = -priceDiff
                poolA, poolB = poolB, poolA
            }

            // 需要至少 0.3% 的价差来覆盖双边手续费
            if priceDiff < 0.006 {
                continue
            }

            // 计算最优套利金额
            optimalAmount, profit := ad.calculateOptimalSimple(poolA, poolB)

            if profit.Cmp(ad.MinProfit) <= 0 {
                continue
            }

            // 计算 gas 成本
            gasCost := new(big.Int).Mul(ad.GasPrice, big.NewInt(300000)) // 估算 2 次 swap

            netProfit := new(big.Int).Sub(profit, gasCost)
            if netProfit.Sign() <= 0 {
                continue
            }

            profitF, _ := new(big.Float).SetInt(profit).Float64()
            amountF, _ := new(big.Float).SetInt(optimalAmount).Float64()

            opportunities = append(opportunities, &ArbitrageOpportunity{
                Type:           "simple",
                Path:           []string{tokenA, tokenB, tokenA},
                Pools:          []*PoolInfo{poolA, poolB},
                OptimalAmount:  optimalAmount,
                ExpectedProfit: profit,
                GasCost:        gasCost,
                NetProfit:      netProfit,
                ProfitPercent:  profitF / amountF * 100,
            })
        }
    }

    // 按净利润排序
    sort.Slice(opportunities, func(i, j int) bool {
        return opportunities[i].NetProfit.Cmp(opportunities[j].NetProfit) > 0
    })

    return opportunities
}

func calculatePrice(pool *PoolInfo) float64 {
    inF, _ := new(big.Float).SetInt(pool.ReserveIn).Float64()
    outF, _ := new(big.Float).SetInt(pool.ReserveOut).Float64()
    return outF / inF
}

func (ad *ArbitrageDetector) calculateOptimalSimple(poolLow, poolHigh *PoolInfo) (*big.Int, *big.Int) {
    // 使用之前推导的最优公式
    xa := float64FromBigInt(poolLow.ReserveIn)
    ya := float64FromBigInt(poolLow.ReserveOut)
    xb := float64FromBigInt(poolHigh.ReserveOut)
    yb := float64FromBigInt(poolHigh.ReserveIn)

    fee := 0.997 // 0.3% 手续费

    // 最优输入公式
    sqrtPart := math.Sqrt(xa * ya * xb * yb * math.Pow(fee, 4))
    numerator := sqrtPart - xa*yb*fee*fee
    denominator := yb*fee*fee + ya*fee*fee

    if numerator <= 0 {
        return big.NewInt(0), big.NewInt(0)
    }

    optimal := numerator / denominator
    optimalInt := bigIntFromFloat64(optimal)

    // 计算利润
    profit := ad.simulateArbitrage(poolLow, poolHigh, optimalInt)

    return optimalInt, profit
}

func (ad *ArbitrageDetector) simulateArbitrage(poolLow, poolHigh *PoolInfo, amountIn *big.Int) *big.Int {
    // 第一跳：在低价池买入
    amountOut1 := getAmountOut(amountIn, poolLow.ReserveIn, poolLow.ReserveOut, poolLow.Fee)

    // 第二跳：在高价池卖出
    amountOut2 := getAmountOut(amountOut1, poolHigh.ReserveIn, poolHigh.ReserveOut, poolHigh.Fee)

    // 利润
    return new(big.Int).Sub(amountOut2, amountIn)
}

func getAmountOut(amountIn, reserveIn, reserveOut *big.Int, fee uint64) *big.Int {
    feeMultiplier := big.NewInt(10000 - int64(fee))
    amountInWithFee := new(big.Int).Mul(amountIn, feeMultiplier)

    numerator := new(big.Int).Mul(amountInWithFee, reserveOut)
    denominator := new(big.Int).Mul(reserveIn, big.NewInt(10000))
    denominator.Add(denominator, amountInWithFee)

    return new(big.Int).Div(numerator, denominator)
}
```

### 78.2 动态套利优化

```go
import "math"

// DynamicArbitrageOptimizer 动态套利优化器
type DynamicArbitrageOptimizer struct {
    MaxIterations int
    Tolerance     float64
}

// OptimizeMultiPool 多池套利优化
// 使用牛顿法求解最优输入
func (dao *DynamicArbitrageOptimizer) OptimizeMultiPool(
    pools []*PoolInfo,
    directions []bool,
) (*big.Int, *big.Int) {
    // 利润函数 P(x) = output(x) - x
    // 求解 P'(x) = 0

    // 使用数值方法
    // 初始猜测：第一个池储备的 1%
    x := float64FromBigInt(pools[0].ReserveIn) * 0.01

    for iter := 0; iter < dao.MaxIterations; iter++ {
        // 计算当前利润
        profit := dao.calculateProfit(pools, directions, x)

        // 数值导数
        epsilon := x * 0.0001
        profitPlus := dao.calculateProfit(pools, directions, x+epsilon)
        profitMinus := dao.calculateProfit(pools, directions, x-epsilon)

        derivative := (profitPlus - profitMinus) / (2 * epsilon)
        secondDerivative := (profitPlus - 2*profit + profitMinus) / (epsilon * epsilon)

        // 牛顿更新
        if math.Abs(secondDerivative) < 1e-10 {
            break
        }

        step := derivative / secondDerivative
        x = x - step

        if x < 0 {
            x = float64FromBigInt(pools[0].ReserveIn) * 0.001
        }

        // 检查收敛
        if math.Abs(step) < dao.Tolerance {
            break
        }
    }

    optimalAmount := bigIntFromFloat64(x)
    profit := bigIntFromFloat64(dao.calculateProfit(pools, directions, x))

    return optimalAmount, profit
}

func (dao *DynamicArbitrageOptimizer) calculateProfit(
    pools []*PoolInfo,
    directions []bool,
    amountIn float64,
) float64 {
    current := amountIn

    for i, pool := range pools {
        var reserveIn, reserveOut float64
        if directions[i] {
            reserveIn = float64FromBigInt(pool.ReserveIn)
            reserveOut = float64FromBigInt(pool.ReserveOut)
        } else {
            reserveIn = float64FromBigInt(pool.ReserveOut)
            reserveOut = float64FromBigInt(pool.ReserveIn)
        }

        fee := 1.0 - float64(pool.Fee)/10000.0

        // amountOut = reserveOut * amountIn * fee / (reserveIn + amountIn * fee)
        current = reserveOut * current * fee / (reserveIn + current*fee)
    }

    return current - amountIn
}

// BinarySearchOptimal 二分搜索最优金额
func (dao *DynamicArbitrageOptimizer) BinarySearchOptimal(
    pools []*PoolInfo,
    directions []bool,
    maxAmount *big.Int,
) (*big.Int, *big.Int) {
    low := 1e15  // 最小 0.001
    high := float64FromBigInt(maxAmount)

    var bestAmount, bestProfit float64

    // 三分搜索找到利润峰值
    for high-low > 1e15 {
        mid1 := low + (high-low)/3
        mid2 := high - (high-low)/3

        profit1 := dao.calculateProfit(pools, directions, mid1)
        profit2 := dao.calculateProfit(pools, directions, mid2)

        if profit1 < profit2 {
            low = mid1
            if profit2 > bestProfit {
                bestProfit = profit2
                bestAmount = mid2
            }
        } else {
            high = mid2
            if profit1 > bestProfit {
                bestProfit = profit1
                bestAmount = mid1
            }
        }
    }

    return bigIntFromFloat64(bestAmount), bigIntFromFloat64(bestProfit)
}
```

### 78.3 实时套利监控

```go
// ArbitrageMonitor 套利监控器
type ArbitrageMonitor struct {
    Detector      *ArbitrageDetector
    Optimizer     *DynamicArbitrageOptimizer
    TokenPairs    []string
    UpdateChan    chan *PoolUpdate
    OpportunityChan chan *ArbitrageOpportunity
}

// PoolUpdate 池更新
type PoolUpdate struct {
    Pool      *PoolInfo
    NewReserveIn  *big.Int
    NewReserveOut *big.Int
    BlockNumber   uint64
    Timestamp     int64
}

// Start 启动监控
func (am *ArbitrageMonitor) Start() {
    go am.monitorLoop()
}

func (am *ArbitrageMonitor) monitorLoop() {
    for update := range am.UpdateChan {
        // 更新池状态
        am.updatePool(update)

        // 检查相关的套利机会
        opportunities := am.checkOpportunities(update.Pool)

        for _, opp := range opportunities {
            // 验证机会仍然有效
            if am.validateOpportunity(opp) {
                am.OpportunityChan <- opp
            }
        }
    }
}

func (am *ArbitrageMonitor) updatePool(update *PoolUpdate) {
    update.Pool.ReserveIn = update.NewReserveIn
    update.Pool.ReserveOut = update.NewReserveOut
}

func (am *ArbitrageMonitor) checkOpportunities(pool *PoolInfo) []*ArbitrageOpportunity {
    var allOpportunities []*ArbitrageOpportunity

    // 检查所有涉及该池的代币对
    for _, pair := range am.TokenPairs {
        pools := am.Detector.Pools[pair]

        for _, p := range pools {
            if p.Address == pool.Address {
                // 该池更新了，检查套利机会
                tokens := splitPair(pair)
                opps := am.Detector.DetectSimpleArbitrage(tokens[0], tokens[1])
                allOpportunities = append(allOpportunities, opps...)
                break
            }
        }
    }

    return allOpportunities
}

func (am *ArbitrageMonitor) validateOpportunity(opp *ArbitrageOpportunity) bool {
    // 重新计算利润
    var totalProfit *big.Int

    switch opp.Type {
    case "simple":
        _, profit := am.Optimizer.BinarySearchOptimal(
            opp.Pools,
            []bool{true, false}, // 买入后卖出
            opp.OptimalAmount,
        )
        totalProfit = profit
    }

    // 检查利润是否仍然超过阈值
    return totalProfit.Cmp(am.Detector.MinProfit) > 0
}

func splitPair(pair string) []string {
    // 简单实现，实际应该正确解析
    for i := range pair {
        if pair[i] == '-' {
            return []string{pair[:i], pair[i+1:]}
        }
    }
    return nil
}

// ProfitabilityAnalysis 盈利能力分析
type ProfitabilityAnalysis struct {
    Opportunity     *ArbitrageOpportunity
    GasEstimate     uint64
    GasPriceGwei    float64
    EthPrice        float64
    SlippageRisk    float64
    CompetitionRisk float64
    SuccessProbability float64
    ExpectedValue   float64
}

// AnalyzeProfitability 分析盈利能力
func AnalyzeProfitability(opp *ArbitrageOpportunity, gasPrice, ethPrice float64) *ProfitabilityAnalysis {
    // Gas 估算
    gasEstimate := uint64(150000 * len(opp.Pools))

    // Gas 成本 (ETH)
    gasCostEth := float64(gasEstimate) * gasPrice * 1e-9

    // Gas 成本 (USD)
    gasCostUsd := gasCostEth * ethPrice

    // 预期利润 (假设以 ETH 计价)
    profitEth, _ := new(big.Float).SetInt(opp.ExpectedProfit).Float64()
    profitEth /= 1e18

    // 滑点风险 (根据交易大小估算)
    amountF, _ := new(big.Float).SetInt(opp.OptimalAmount).Float64()
    reserveF, _ := new(big.Float).SetInt(opp.Pools[0].ReserveIn).Float64()
    slippageRisk := amountF / reserveF * 100 // 占池子的百分比

    // 竞争风险 (简化估算)
    competitionRisk := 0.5 // 假设 50% 的机会被抢先

    // 成功概率
    successProb := (1 - competitionRisk) * (1 - slippageRisk/100)

    // 期望值
    expectedValue := profitEth*ethPrice*successProb - gasCostUsd

    return &ProfitabilityAnalysis{
        Opportunity:        opp,
        GasEstimate:        gasEstimate,
        GasPriceGwei:       gasPrice,
        EthPrice:           ethPrice,
        SlippageRisk:       slippageRisk,
        CompetitionRisk:    competitionRisk,
        SuccessProbability: successProb,
        ExpectedValue:      expectedValue,
    }
}
```

### 78.4 辅助函数

```go
import "math/big"

func float64FromBigInt(n *big.Int) float64 {
    f, _ := new(big.Float).SetInt(n).Float64()
    return f
}

func bigIntFromFloat64(f float64) *big.Int {
    bf := big.NewFloat(f)
    i, _ := bf.Int(nil)
    return i
}
```

---

## 第79节：流动性聚合器数学

### 79.1 聚合器架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEX 聚合器架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户交易请求                                                     │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              聚合器路由引擎                                │   │
│  │                                                          │   │
│  │  1. 获取所有 DEX 价格                                     │   │
│  │  2. 计算最优路由                                          │   │
│  │  3. 拆分订单到多个池                                      │   │
│  │  4. 考虑 gas 成本                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     DEX 池层                             │   │
│  │                                                          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │Uniswap  │  │Sushiswap│  │ Curve   │  │Balancer │    │   │
│  │  │ V2/V3   │  │         │  │         │  │         │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │   │
│  │                                                          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │ DODO    │  │Bancor   │  │Kyber    │  │  0x     │    │   │
│  │  │         │  │         │  │         │  │         │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  优化目标:                                                       │
│  - 最大化输出金额                                                │
│  - 最小化 gas 成本                                              │
│  - 最小化滑点                                                   │
│  - 平衡执行速度                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 79.2 凸优化拆分算法

```go
package aggregator

import (
    "math"
    "math/big"
    "sort"
)

// ConvexOptimizer 凸优化器
// 用于解决流动性拆分问题
type ConvexOptimizer struct {
    Pools         []*LiquidityPool
    GasPerSwap    uint64
    GasPrice      *big.Int
    MaxIterations int
    Tolerance     float64
}

// LiquidityPool 流动性池
type LiquidityPool struct {
    ID          string
    Protocol    string
    ReserveIn   *big.Int
    ReserveOut  *big.Int
    Fee         float64 // 0.003 = 0.3%
    GasEstimate uint64  // 该池 swap 的 gas 估算
}

// SplitSolution 拆分解决方案
type SplitSolution struct {
    Allocations   []float64  // 每个池的分配比例
    AmountsIn     []*big.Int // 每个池的输入金额
    AmountsOut    []*big.Int // 每个池的输出金额
    TotalOutput   *big.Int   // 总输出
    TotalGasCost  *big.Int   // 总 gas 成本
    NetOutput     *big.Int   // 净输出 (考虑 gas)
}

// Optimize 执行凸优化
// 问题形式: max Σf_i(x_i) - gas_cost
// 约束: Σx_i = X (总输入金额)
// 其中 f_i 是第 i 个池的输出函数
func (co *ConvexOptimizer) Optimize(totalAmount *big.Int) *SplitSolution {
    n := len(co.Pools)
    if n == 0 {
        return nil
    }

    totalIn, _ := new(big.Float).SetInt(totalAmount).Float64()

    // 使用 ADMM (Alternating Direction Method of Multipliers)
    // 或简化的梯度投影法

    // 初始化: 按流动性比例分配
    totalLiquidity := 0.0
    for _, pool := range co.Pools {
        totalLiquidity += float64FromBigInt(pool.ReserveIn)
    }

    allocations := make([]float64, n)
    for i, pool := range co.Pools {
        allocations[i] = float64FromBigInt(pool.ReserveIn) / totalLiquidity
    }

    // 梯度投影迭代
    for iter := 0; iter < co.MaxIterations; iter++ {
        // 计算梯度 (边际输出率)
        gradients := make([]float64, n)
        for i, pool := range co.Pools {
            amount := totalIn * allocations[i]
            gradients[i] = co.marginalOutput(pool, amount)
        }

        // 找到最大和最小梯度
        maxGrad, minGrad := gradients[0], gradients[0]
        maxIdx, minIdx := 0, 0
        for i, g := range gradients {
            if g > maxGrad {
                maxGrad = g
                maxIdx = i
            }
            if g < minGrad {
                minGrad = g
                minIdx = i
            }
        }

        // 检查收敛
        if maxGrad-minGrad < co.Tolerance {
            break
        }

        // 从低边际池移动到高边际池
        step := 0.01 // 每次移动 1%
        allocations[minIdx] -= step
        allocations[maxIdx] += step

        // 确保非负
        if allocations[minIdx] < 0 {
            allocations[maxIdx] += allocations[minIdx]
            allocations[minIdx] = 0
        }
    }

    // 移除太小的分配
    for i := range allocations {
        if allocations[i] < 0.01 {
            allocations[i] = 0
        }
    }

    // 重新归一化
    total := 0.0
    for _, a := range allocations {
        total += a
    }
    for i := range allocations {
        allocations[i] /= total
    }

    // 构建解决方案
    return co.buildSolution(allocations, totalAmount)
}

// marginalOutput 计算边际输出 (输出对输入的导数)
func (co *ConvexOptimizer) marginalOutput(pool *LiquidityPool, currentInput float64) float64 {
    // 对于恒定乘积 AMM: dy/dx = y*x / (x + Δx)^2 * (1-fee)

    x := float64FromBigInt(pool.ReserveIn)
    y := float64FromBigInt(pool.ReserveOut)
    fee := 1 - pool.Fee

    // 边际输出
    denom := x + currentInput*fee
    marginal := y * x * fee / (denom * denom)

    return marginal
}

func (co *ConvexOptimizer) buildSolution(allocations []float64, totalAmount *big.Int) *SplitSolution {
    solution := &SplitSolution{
        Allocations: allocations,
        AmountsIn:   make([]*big.Int, len(co.Pools)),
        AmountsOut:  make([]*big.Int, len(co.Pools)),
    }

    totalOut := big.NewInt(0)
    totalGas := uint64(0)
    totalInF, _ := new(big.Float).SetInt(totalAmount).Float64()

    for i, pool := range co.Pools {
        if allocations[i] < 0.001 {
            solution.AmountsIn[i] = big.NewInt(0)
            solution.AmountsOut[i] = big.NewInt(0)
            continue
        }

        amountIn := bigIntFromFloat64(totalInF * allocations[i])
        amountOut := co.calculateOutput(pool, amountIn)

        solution.AmountsIn[i] = amountIn
        solution.AmountsOut[i] = amountOut

        totalOut.Add(totalOut, amountOut)
        totalGas += pool.GasEstimate
    }

    solution.TotalOutput = totalOut
    solution.TotalGasCost = new(big.Int).Mul(big.NewInt(int64(totalGas)), co.GasPrice)
    solution.NetOutput = new(big.Int).Sub(totalOut, solution.TotalGasCost)

    return solution
}

func (co *ConvexOptimizer) calculateOutput(pool *LiquidityPool, amountIn *big.Int) *big.Int {
    x := pool.ReserveIn
    y := pool.ReserveOut

    fee := big.NewFloat(1 - pool.Fee)
    amountInF := new(big.Float).SetInt(amountIn)
    amountInWithFee := new(big.Float).Mul(amountInF, fee)

    xF := new(big.Float).SetInt(x)
    yF := new(big.Float).SetInt(y)

    // amountOut = y * amountInWithFee / (x + amountInWithFee)
    denom := new(big.Float).Add(xF, amountInWithFee)
    numerator := new(big.Float).Mul(yF, amountInWithFee)
    result := new(big.Float).Quo(numerator, denom)

    out, _ := result.Int(nil)
    return out
}
```

### 79.3 带 Gas 成本的优化

```go
// GasAwareOptimizer 考虑 gas 成本的优化器
type GasAwareOptimizer struct {
    Pools         []*LiquidityPool
    GasPrice      *big.Int
    EthPrice      float64 // ETH/USD 价格
    TokenPrice    float64 // 输出代币/USD 价格
}

// OptimizeWithGas 考虑 gas 成本的优化
func (gao *GasAwareOptimizer) OptimizeWithGas(totalAmount *big.Int) *SplitSolution {
    n := len(gao.Pools)

    // 计算每个池的效率 (输出 - gas成本)
    type poolEfficiency struct {
        index      int
        efficiency float64
        maxAmount  *big.Int
    }

    efficiencies := make([]poolEfficiency, n)
    gasPriceF, _ := new(big.Float).SetInt(gao.GasPrice).Float64()

    for i, pool := range gao.Pools {
        // 计算 1 单位输入的边际效率
        testAmount := big.NewInt(1e18) // 1 token
        output := gao.calculateOutput(pool, testAmount)
        outputF, _ := new(big.Float).SetInt(output).Float64()

        // Gas 成本转换为输出代币
        gasCostWei := float64(pool.GasEstimate) * gasPriceF
        gasCostEth := gasCostWei / 1e18
        gasCostUsd := gasCostEth * gao.EthPrice
        gasCostInToken := gasCostUsd / gao.TokenPrice * 1e18

        efficiency := outputF - gasCostInToken

        // 该池的最大有效输入 (超过后边际收益为负)
        maxAmount := gao.findMaxProfitableAmount(pool, gasCostInToken)

        efficiencies[i] = poolEfficiency{
            index:      i,
            efficiency: efficiency,
            maxAmount:  maxAmount,
        }
    }

    // 按效率排序
    sort.Slice(efficiencies, func(i, j int) bool {
        return efficiencies[i].efficiency > efficiencies[j].efficiency
    })

    // 贪心分配
    remaining := new(big.Int).Set(totalAmount)
    allocations := make([]float64, n)
    totalInF, _ := new(big.Float).SetInt(totalAmount).Float64()

    for _, eff := range efficiencies {
        if eff.efficiency <= 0 {
            continue // 无利可图的池
        }

        pool := gao.Pools[eff.index]
        maxForPool := eff.maxAmount

        // 分配给该池的金额
        var allocation *big.Int
        if remaining.Cmp(maxForPool) > 0 {
            allocation = new(big.Int).Set(maxForPool)
        } else {
            allocation = new(big.Int).Set(remaining)
        }

        allocF, _ := new(big.Float).SetInt(allocation).Float64()
        allocations[eff.index] = allocF / totalInF

        remaining.Sub(remaining, allocation)

        if remaining.Sign() <= 0 {
            break
        }
    }

    // 如果还有剩余，分配给最大的池
    if remaining.Sign() > 0 {
        var maxIdx int
        maxLiquidity := big.NewInt(0)
        for i, pool := range gao.Pools {
            if pool.ReserveIn.Cmp(maxLiquidity) > 0 {
                maxLiquidity = pool.ReserveIn
                maxIdx = i
            }
        }
        remF, _ := new(big.Float).SetInt(remaining).Float64()
        allocations[maxIdx] += remF / totalInF
    }

    // 构建解决方案
    return gao.buildSolution(allocations, totalAmount)
}

func (gao *GasAwareOptimizer) findMaxProfitableAmount(pool *LiquidityPool, gasCostInToken float64) *big.Int {
    // 二分搜索找到边际收益等于 0 的点
    low := 0.0
    high := float64FromBigInt(pool.ReserveIn) * 0.5 // 最多用池子的一半

    for high-low > 1e15 {
        mid := (low + high) / 2
        marginal := gao.marginalOutput(pool, mid)

        // 边际收益 > 0 时继续增加
        if marginal > 0 {
            low = mid
        } else {
            high = mid
        }
    }

    return bigIntFromFloat64(low)
}

func (gao *GasAwareOptimizer) marginalOutput(pool *LiquidityPool, currentInput float64) float64 {
    x := float64FromBigInt(pool.ReserveIn)
    y := float64FromBigInt(pool.ReserveOut)
    fee := 1 - pool.Fee

    denom := x + currentInput*fee
    return y * x * fee / (denom * denom)
}

func (gao *GasAwareOptimizer) calculateOutput(pool *LiquidityPool, amountIn *big.Int) *big.Int {
    x := pool.ReserveIn
    y := pool.ReserveOut

    fee := big.NewFloat(1 - pool.Fee)
    amountInF := new(big.Float).SetInt(amountIn)
    amountInWithFee := new(big.Float).Mul(amountInF, fee)

    xF := new(big.Float).SetInt(x)
    yF := new(big.Float).SetInt(y)

    denom := new(big.Float).Add(xF, amountInWithFee)
    numerator := new(big.Float).Mul(yF, amountInWithFee)
    result := new(big.Float).Quo(numerator, denom)

    out, _ := result.Int(nil)
    return out
}

func (gao *GasAwareOptimizer) buildSolution(allocations []float64, totalAmount *big.Int) *SplitSolution {
    solution := &SplitSolution{
        Allocations: allocations,
        AmountsIn:   make([]*big.Int, len(gao.Pools)),
        AmountsOut:  make([]*big.Int, len(gao.Pools)),
    }

    totalOut := big.NewInt(0)
    totalGas := uint64(0)
    totalInF, _ := new(big.Float).SetInt(totalAmount).Float64()

    activePoolCount := 0
    for i, pool := range gao.Pools {
        if allocations[i] < 0.001 {
            solution.AmountsIn[i] = big.NewInt(0)
            solution.AmountsOut[i] = big.NewInt(0)
            continue
        }

        activePoolCount++
        amountIn := bigIntFromFloat64(totalInF * allocations[i])
        amountOut := gao.calculateOutput(pool, amountIn)

        solution.AmountsIn[i] = amountIn
        solution.AmountsOut[i] = amountOut

        totalOut.Add(totalOut, amountOut)
        totalGas += pool.GasEstimate
    }

    solution.TotalOutput = totalOut
    solution.TotalGasCost = new(big.Int).Mul(big.NewInt(int64(totalGas)), gao.GasPrice)
    solution.NetOutput = new(big.Int).Sub(totalOut, solution.TotalGasCost)

    return solution
}
```

### 79.4 实时报价引擎

```go
// QuoteEngine 报价引擎
type QuoteEngine struct {
    Optimizer     *GasAwareOptimizer
    PriceCache    map[string]*CachedPrice
    CacheDuration int64 // 缓存时间(秒)
}

// CachedPrice 缓存的价格
type CachedPrice struct {
    Price     float64
    Timestamp int64
}

// Quote 报价
type Quote struct {
    AmountIn       *big.Int
    AmountOut      *big.Int
    MinAmountOut   *big.Int // 考虑滑点保护
    Path           []*PoolInfo
    Splits         []float64
    PriceImpact    float64
    GasCost        *big.Int
    ValidUntil     int64
}

// GetQuote 获取报价
func (qe *QuoteEngine) GetQuote(
    tokenIn, tokenOut string,
    amountIn *big.Int,
    slippageTolerance float64, // 0.005 = 0.5%
) *Quote {
    // 1. 获取最优路由
    solution := qe.Optimizer.OptimizeWithGas(amountIn)

    // 2. 计算价格影响
    spotPrice := qe.getSpotPrice(tokenIn, tokenOut)
    executionPrice := float64FromBigInt(solution.TotalOutput) / float64FromBigInt(amountIn)
    priceImpact := (spotPrice - executionPrice) / spotPrice * 100

    // 3. 计算最小输出 (考虑滑点)
    slippageMultiplier := 1 - slippageTolerance
    minOutF := float64FromBigInt(solution.TotalOutput) * slippageMultiplier
    minAmountOut := bigIntFromFloat64(minOutF)

    // 4. 构建路径
    var path []*PoolInfo
    var splits []float64
    for i, pool := range qe.Optimizer.Pools {
        if solution.Allocations[i] > 0.001 {
            path = append(path, &PoolInfo{
                Address:    pool.ID,
                Protocol:   pool.Protocol,
                ReserveIn:  pool.ReserveIn,
                ReserveOut: pool.ReserveOut,
                Fee:        uint64(pool.Fee * 10000),
            })
            splits = append(splits, solution.Allocations[i])
        }
    }

    return &Quote{
        AmountIn:     amountIn,
        AmountOut:    solution.TotalOutput,
        MinAmountOut: minAmountOut,
        Path:         path,
        Splits:       splits,
        PriceImpact:  priceImpact,
        GasCost:      solution.TotalGasCost,
        ValidUntil:   time.Now().Unix() + 30, // 30秒有效
    }
}

func (qe *QuoteEngine) getSpotPrice(tokenIn, tokenOut string) float64 {
    // 从缓存或计算即时价格
    key := tokenIn + "-" + tokenOut
    if cached, ok := qe.PriceCache[key]; ok {
        if time.Now().Unix()-cached.Timestamp < qe.CacheDuration {
            return cached.Price
        }
    }

    // 计算加权平均即时价格
    var totalWeight, weightedPrice float64
    for _, pool := range qe.Optimizer.Pools {
        weight := float64FromBigInt(pool.ReserveIn)
        price := float64FromBigInt(pool.ReserveOut) / float64FromBigInt(pool.ReserveIn)
        weightedPrice += price * weight
        totalWeight += weight
    }

    spotPrice := weightedPrice / totalWeight

    // 缓存
    qe.PriceCache[key] = &CachedPrice{
        Price:     spotPrice,
        Timestamp: time.Now().Unix(),
    }

    return spotPrice
}

import "time"
```

---

## 第80节：高级数学优化技术

### 80.1 矩阵形式的套利检测

```go
package advanced

import (
    "math"
    "math/big"
)

// MatrixArbitrageDetector 矩阵套利检测器
// 使用对数价格矩阵检测套利机会
type MatrixArbitrageDetector struct {
    Tokens     []string            // 代币列表
    LogPrices  [][]float64         // 对数价格矩阵 log(P_ij)
    Pools      map[string]*PoolInfo // token pair -> pool
}

// 套利检测原理:
// 如果存在套利机会，则存在一个循环使得:
// P_AB * P_BC * P_CA > 1
// 取对数: log(P_AB) + log(P_BC) + log(P_CA) > 0
// 即在对数价格图中存在正权重循环

// DetectArbitrage 使用 Bellman-Ford 检测负循环 (取反后为正循环)
func (mad *MatrixArbitrageDetector) DetectArbitrage() [][]int {
    n := len(mad.Tokens)

    // 构建边列表 (取负对数价格)
    type edge struct {
        from, to int
        weight   float64
    }

    var edges []edge
    for i := 0; i < n; i++ {
        for j := 0; j < n; j++ {
            if i != j && mad.LogPrices[i][j] != 0 {
                // 边权重为 -log(price) ，这样负循环对应套利
                edges = append(edges, edge{i, j, -mad.LogPrices[i][j]})
            }
        }
    }

    // Bellman-Ford 从每个顶点开始
    var arbitrageCycles [][]int

    for start := 0; start < n; start++ {
        dist := make([]float64, n)
        parent := make([]int, n)
        for i := range dist {
            dist[i] = math.Inf(1)
            parent[i] = -1
        }
        dist[start] = 0

        // 松弛 n-1 次
        for k := 0; k < n-1; k++ {
            for _, e := range edges {
                if dist[e.from]+e.weight < dist[e.to] {
                    dist[e.to] = dist[e.from] + e.weight
                    parent[e.to] = e.from
                }
            }
        }

        // 第 n 次检测负循环
        for _, e := range edges {
            if dist[e.from]+e.weight < dist[e.to] {
                // 找到负循环
                cycle := mad.extractCycle(parent, e.to, n)
                if len(cycle) > 0 {
                    arbitrageCycles = append(arbitrageCycles, cycle)
                }
            }
        }
    }

    // 去重
    return mad.deduplicateCycles(arbitrageCycles)
}

func (mad *MatrixArbitrageDetector) extractCycle(parent []int, start, n int) []int {
    // 回溯 n 步确保在循环内
    node := start
    for i := 0; i < n; i++ {
        node = parent[node]
        if node == -1 {
            return nil
        }
    }

    // 提取循环
    cycle := []int{node}
    for next := parent[node]; next != node; next = parent[next] {
        cycle = append(cycle, next)
        if len(cycle) > n {
            return nil // 防止无限循环
        }
    }

    // 反转
    for i, j := 0, len(cycle)-1; i < j; i, j = i+1, j-1 {
        cycle[i], cycle[j] = cycle[j], cycle[i]
    }

    return cycle
}

func (mad *MatrixArbitrageDetector) deduplicateCycles(cycles [][]int) [][]int {
    seen := make(map[string]bool)
    var unique [][]int

    for _, cycle := range cycles {
        // 规范化循环 (从最小元素开始)
        normalized := mad.normalizeCycle(cycle)
        key := fmt.Sprintf("%v", normalized)

        if !seen[key] {
            seen[key] = true
            unique = append(unique, normalized)
        }
    }

    return unique
}

func (mad *MatrixArbitrageDetector) normalizeCycle(cycle []int) []int {
    if len(cycle) == 0 {
        return cycle
    }

    // 找到最小元素的位置
    minIdx := 0
    for i, v := range cycle {
        if v < cycle[minIdx] {
            minIdx = i
        }
    }

    // 从最小元素开始重排
    result := make([]int, len(cycle))
    for i := 0; i < len(cycle); i++ {
        result[i] = cycle[(minIdx+i)%len(cycle)]
    }

    return result
}

import "fmt"
```

### 80.2 凸优化闭式解

```go
// ClosedFormSolution 闭式解计算
// 对于简单的双池套利问题，存在闭式最优解

// TwoPoolOptimal 双池套利闭式最优解
// Pool A: (x_a, y_a), price = y_a/x_a
// Pool B: (x_b, y_b), price = y_b/x_b
// 假设 price_A < price_B (在 A 买，在 B 卖)
func TwoPoolOptimal(
    xa, ya *big.Int, // Pool A 储备
    xb, yb *big.Int, // Pool B 储备
    feeA, feeB float64, // 手续费率
) (*big.Int, *big.Int) {
    // 最优输入公式:
    // Δx* = (√(xa * ya * xb * yb * γa² * γb²) - xa * yb * γa * γb) / (yb * γa * γb + ya * γa)
    // 其中 γ = 1 - fee

    gammaA := 1 - feeA
    gammaB := 1 - feeB

    xaF := float64FromBigInt(xa)
    yaF := float64FromBigInt(ya)
    xbF := float64FromBigInt(xb)
    ybF := float64FromBigInt(yb)

    // 计算根号内的值
    sqrtArg := xaF * yaF * xbF * ybF * gammaA * gammaA * gammaB * gammaB
    sqrtVal := math.Sqrt(sqrtArg)

    numerator := sqrtVal - xaF*ybF*gammaA*gammaB
    denominator := ybF*gammaA*gammaB + yaF*gammaA

    if numerator <= 0 || denominator <= 0 {
        return big.NewInt(0), big.NewInt(0) // 无套利机会
    }

    optimalInput := numerator / denominator

    // 计算利润
    // 第一跳输出
    outputA := yaF * optimalInput * gammaA / (xaF + optimalInput*gammaA)

    // 第二跳输出
    outputB := xbF * outputA * gammaB / (ybF + outputA*gammaB)

    profit := outputB - optimalInput

    return bigIntFromFloat64(optimalInput), bigIntFromFloat64(profit)
}

// ThreePoolOptimal 三池套利 (数值解)
// 由于闭式解复杂，使用拉格朗日乘数法的数值近似
func ThreePoolOptimal(
    pools [3]*LiquidityPool,
    directions [3]bool, // 交易方向
) (*big.Int, *big.Int) {
    // 使用黄金分割搜索

    // 估算搜索范围
    maxInput := float64FromBigInt(pools[0].ReserveIn) * 0.1

    // 利润函数
    profitFunc := func(input float64) float64 {
        current := input
        for i, pool := range pools {
            var resIn, resOut float64
            if directions[i] {
                resIn = float64FromBigInt(pool.ReserveIn)
                resOut = float64FromBigInt(pool.ReserveOut)
            } else {
                resIn = float64FromBigInt(pool.ReserveOut)
                resOut = float64FromBigInt(pool.ReserveIn)
            }
            gamma := 1 - pool.Fee
            current = resOut * current * gamma / (resIn + current*gamma)
        }
        return current - input
    }

    // 黄金分割搜索
    phi := (math.Sqrt(5) - 1) / 2
    a, b := 1e15, maxInput
    c := b - phi*(b-a)
    d := a + phi*(b-a)

    for b-a > 1e15 {
        if profitFunc(c) > profitFunc(d) {
            b = d
        } else {
            a = c
        }
        c = b - phi*(b-a)
        d = a + phi*(b-a)
    }

    optimal := (a + b) / 2
    profit := profitFunc(optimal)

    return bigIntFromFloat64(optimal), bigIntFromFloat64(profit)
}
```

### 80.3 动态规划路径优化

```go
// DPPathOptimizer 动态规划路径优化器
// 用于找到多跳最优路径
type DPPathOptimizer struct {
    Graph      map[string][]Edge // 代币图
    Tokens     []string
    TokenIndex map[string]int
}

// Edge 图边
type Edge struct {
    To     string
    Pool   *LiquidityPool
    Direct bool // true: token0->token1, false: token1->token0
}

// OptimalPath 最优路径结果
type OptimalPath struct {
    Path       []string
    Pools      []*LiquidityPool
    Directions []bool
    AmountOut  *big.Int
}

// FindOptimalPath 使用 DP 找到最优路径
// dp[hop][token] = 到达 token 的最大输出
func (dpo *DPPathOptimizer) FindOptimalPath(
    tokenIn, tokenOut string,
    amountIn *big.Int,
    maxHops int,
) *OptimalPath {
    n := len(dpo.Tokens)

    // DP 表
    // dp[hop][token] = {最大输出, 前驱代币, 使用的池, 方向}
    type dpEntry struct {
        amount   *big.Int
        prevToken string
        pool     *LiquidityPool
        direct   bool
    }

    dp := make([]map[string]*dpEntry, maxHops+1)
    for i := range dp {
        dp[i] = make(map[string]*dpEntry)
    }

    // 初始化
    dp[0][tokenIn] = &dpEntry{amount: amountIn}

    // 填充 DP 表
    for hop := 0; hop < maxHops; hop++ {
        for token, entry := range dp[hop] {
            if entry == nil || entry.amount.Sign() <= 0 {
                continue
            }

            // 遍历所有出边
            for _, edge := range dpo.Graph[token] {
                output := dpo.calculateSwapOutput(edge.Pool, entry.amount, edge.Direct)

                if output.Sign() <= 0 {
                    continue
                }

                // 更新 DP 值
                if prev := dp[hop+1][edge.To]; prev == nil || output.Cmp(prev.amount) > 0 {
                    dp[hop+1][edge.To] = &dpEntry{
                        amount:    output,
                        prevToken: token,
                        pool:      edge.Pool,
                        direct:    edge.Direct,
                    }
                }
            }
        }
    }

    // 找到最优解
    var bestEntry *dpEntry
    var bestHop int

    for hop := 1; hop <= maxHops; hop++ {
        if entry := dp[hop][tokenOut]; entry != nil {
            if bestEntry == nil || entry.amount.Cmp(bestEntry.amount) > 0 {
                bestEntry = entry
                bestHop = hop
            }
        }
    }

    if bestEntry == nil {
        return nil
    }

    // 回溯构建路径
    path := make([]string, bestHop+1)
    pools := make([]*LiquidityPool, bestHop)
    directions := make([]bool, bestHop)

    path[bestHop] = tokenOut
    for i := bestHop; i > 0; i-- {
        entry := dp[i][path[i]]
        path[i-1] = entry.prevToken
        pools[i-1] = entry.pool
        directions[i-1] = entry.direct
    }

    return &OptimalPath{
        Path:       path,
        Pools:      pools,
        Directions: directions,
        AmountOut:  bestEntry.amount,
    }
}

func (dpo *DPPathOptimizer) calculateSwapOutput(pool *LiquidityPool, amountIn *big.Int, direct bool) *big.Int {
    var resIn, resOut *big.Int
    if direct {
        resIn = pool.ReserveIn
        resOut = pool.ReserveOut
    } else {
        resIn = pool.ReserveOut
        resOut = pool.ReserveIn
    }

    gamma := 1 - pool.Fee
    amountInF := float64FromBigInt(amountIn) * gamma

    resInF := float64FromBigInt(resIn)
    resOutF := float64FromBigInt(resOut)

    output := resOutF * amountInF / (resInF + amountInF)

    return bigIntFromFloat64(output)
}
```

### 80.4 蒙特卡洛模拟

```go
import "math/rand"

// MonteCarloSimulator 蒙特卡洛模拟器
// 用于评估策略在随机市场条件下的表现
type MonteCarloSimulator struct {
    Pools            []*LiquidityPool
    NumSimulations   int
    PriceVolatility  float64 // 价格波动率
    TimeHorizon      int     // 模拟时间步数
}

// SimulationResult 模拟结果
type SimulationResult struct {
    MeanProfit      float64
    StdDevProfit    float64
    MaxProfit       float64
    MinProfit       float64
    WinRate         float64 // 盈利交易比例
    SharpeRatio     float64
    MaxDrawdown     float64
    ProfitDistribution []float64 // 利润分布
}

// RunSimulation 运行蒙特卡洛模拟
func (mcs *MonteCarloSimulator) RunSimulation(strategy ArbitrageStrategy) *SimulationResult {
    profits := make([]float64, mcs.NumSimulations)
    wins := 0

    for sim := 0; sim < mcs.NumSimulations; sim++ {
        // 复制池状态
        simPools := mcs.copyPools()

        totalProfit := 0.0
        peakValue := 0.0
        maxDrawdown := 0.0

        for t := 0; t < mcs.TimeHorizon; t++ {
            // 模拟价格变动
            mcs.simulatePriceMove(simPools)

            // 执行策略
            profit := strategy.Execute(simPools)
            totalProfit += profit

            // 计算回撤
            if totalProfit > peakValue {
                peakValue = totalProfit
            }
            drawdown := peakValue - totalProfit
            if drawdown > maxDrawdown {
                maxDrawdown = drawdown
            }
        }

        profits[sim] = totalProfit
        if totalProfit > 0 {
            wins++
        }
    }

    // 计算统计量
    return mcs.calculateStats(profits, wins)
}

func (mcs *MonteCarloSimulator) copyPools() []*LiquidityPool {
    copied := make([]*LiquidityPool, len(mcs.Pools))
    for i, p := range mcs.Pools {
        copied[i] = &LiquidityPool{
            ID:          p.ID,
            Protocol:    p.Protocol,
            ReserveIn:   new(big.Int).Set(p.ReserveIn),
            ReserveOut:  new(big.Int).Set(p.ReserveOut),
            Fee:         p.Fee,
            GasEstimate: p.GasEstimate,
        }
    }
    return copied
}

func (mcs *MonteCarloSimulator) simulatePriceMove(pools []*LiquidityPool) {
    for _, pool := range pools {
        // 几何布朗运动
        drift := 0.0 // 无漂移
        diffusion := mcs.PriceVolatility * rand.NormFloat64()

        priceChange := math.Exp(drift + diffusion)

        // 调整储备以反映价格变化
        // 保持 k 不变，调整价格
        resInF := float64FromBigInt(pool.ReserveIn)
        resOutF := float64FromBigInt(pool.ReserveOut)

        // 新价格 = 旧价格 * priceChange
        // 新的 y/x = (y/x) * priceChange
        // 保持 x*y = k
        // 新 x = sqrt(k / new_price) = sqrt(k) / sqrt(new_price)
        // 新 y = sqrt(k * new_price) = sqrt(k) * sqrt(new_price)

        k := resInF * resOutF
        sqrtK := math.Sqrt(k)
        sqrtNewPrice := math.Sqrt(resOutF / resInF * priceChange)

        newResIn := sqrtK / sqrtNewPrice
        newResOut := sqrtK * sqrtNewPrice

        pool.ReserveIn = bigIntFromFloat64(newResIn)
        pool.ReserveOut = bigIntFromFloat64(newResOut)
    }
}

func (mcs *MonteCarloSimulator) calculateStats(profits []float64, wins int) *SimulationResult {
    n := float64(len(profits))

    // 均值
    sum := 0.0
    for _, p := range profits {
        sum += p
    }
    mean := sum / n

    // 标准差
    sumSqDiff := 0.0
    for _, p := range profits {
        diff := p - mean
        sumSqDiff += diff * diff
    }
    stdDev := math.Sqrt(sumSqDiff / n)

    // 最大最小
    maxP, minP := profits[0], profits[0]
    for _, p := range profits {
        if p > maxP {
            maxP = p
        }
        if p < minP {
            minP = p
        }
    }

    // Sharpe 比率 (假设无风险利率为 0)
    sharpe := 0.0
    if stdDev > 0 {
        sharpe = mean / stdDev
    }

    return &SimulationResult{
        MeanProfit:         mean,
        StdDevProfit:       stdDev,
        MaxProfit:          maxP,
        MinProfit:          minP,
        WinRate:            float64(wins) / n,
        SharpeRatio:        sharpe,
        ProfitDistribution: profits,
    }
}

// ArbitrageStrategy 套利策略接口
type ArbitrageStrategy interface {
    Execute(pools []*LiquidityPool) float64
}
```

### 80.5 数值精度处理

```go
// PrecisionMath 高精度数学运算
// 在 DeFi 中，精度至关重要
type PrecisionMath struct{}

// 使用 big.Int 进行无损计算
// 避免浮点数精度问题

// MulDiv 精确的乘除运算 (a * b / c)
func (pm *PrecisionMath) MulDiv(a, b, c *big.Int, roundUp bool) *big.Int {
    // 先乘后除
    product := new(big.Int).Mul(a, b)

    if !roundUp {
        return new(big.Int).Div(product, c)
    }

    // 向上取整: (a * b + c - 1) / c
    result := new(big.Int).Add(product, c)
    result.Sub(result, big.NewInt(1))
    result.Div(result, c)

    return result
}

// Sqrt 大整数平方根
func (pm *PrecisionMath) Sqrt(n *big.Int) *big.Int {
    if n.Sign() <= 0 {
        return big.NewInt(0)
    }

    // 牛顿法
    x := new(big.Int).Rsh(n, uint(n.BitLen()/2))
    if x.Sign() == 0 {
        x.SetInt64(1)
    }

    for {
        x1 := new(big.Int).Div(n, x)
        x1.Add(x1, x)
        x1.Rsh(x1, 1)

        if x1.Cmp(x) >= 0 {
            return x
        }
        x = x1
    }
}

// 固定点数学 (避免浮点误差)
// Q96 格式: 值 * 2^96
const Q96Shift = 96

// ToQ96 转换为 Q96 格式
func (pm *PrecisionMath) ToQ96(value *big.Int) *big.Int {
    return new(big.Int).Lsh(value, Q96Shift)
}

// FromQ96 从 Q96 格式转换
func (pm *PrecisionMath) FromQ96(value *big.Int) *big.Int {
    return new(big.Int).Rsh(value, Q96Shift)
}

// MulQ96 Q96 乘法
func (pm *PrecisionMath) MulQ96(a, b *big.Int) *big.Int {
    product := new(big.Int).Mul(a, b)
    return new(big.Int).Rsh(product, Q96Shift)
}

// DivQ96 Q96 除法
func (pm *PrecisionMath) DivQ96(a, b *big.Int) *big.Int {
    shifted := new(big.Int).Lsh(a, Q96Shift)
    return new(big.Int).Div(shifted, b)
}

// 示例：使用高精度计算 Uniswap V2 输出
func (pm *PrecisionMath) GetAmountOutPrecise(
    amountIn, reserveIn, reserveOut *big.Int,
    feeNumerator, feeDenominator int64,
) *big.Int {
    // amountOut = reserveOut * amountIn * (1 - fee) / (reserveIn + amountIn * (1 - fee))
    // 使用整数运算: amountOut = reserveOut * amountIn * feeNumerator / (reserveIn * feeDenominator + amountIn * feeNumerator)

    feeNum := big.NewInt(feeNumerator)     // 例如 997
    feeDenom := big.NewInt(feeDenominator) // 例如 1000

    // numerator = reserveOut * amountIn * feeNumerator
    numerator := new(big.Int).Mul(reserveOut, amountIn)
    numerator.Mul(numerator, feeNum)

    // denominator = reserveIn * feeDenominator + amountIn * feeNumerator
    term1 := new(big.Int).Mul(reserveIn, feeDenom)
    term2 := new(big.Int).Mul(amountIn, feeNum)
    denominator := new(big.Int).Add(term1, term2)

    // 结果
    return new(big.Int).Div(numerator, denominator)
}
```

---

## 总结

本文档详细介绍了 DEX 数学模型的核心概念和实现：

1. **AMM 基础数学** (第71节)
   - 恒定乘积公式
   - 价格影响计算
   - 输入/输出计算

2. **Uniswap V2** (第72节)
   - 完整数学模型
   - 最优套利金额公式
   - 三角套利计算
   - LP Token 数学

3. **Uniswap V3** (第73节)
   - Tick 数学
   - 集中流动性计算
   - Swap 步骤计算
   - 流动性数量计算

4. **Curve StableSwap** (第74节)
   - 不变量计算
   - 牛顿迭代求解
   - 虚拟价格

5. **Balancer** (第75节)
   - 加权常数乘积
   - 多代币池计算
   - 智能订单路由

6. **无常损失** (第76节)
   - 计算公式
   - V3 集中流动性 IL
   - 对冲策略

7. **滑点与执行** (第77节)
   - 滑点计算
   - 交易拆分优化
   - 多路径路由

8. **跨 DEX 套利** (第78节)
   - 套利检测
   - 动态优化
   - 实时监控

9. **流动性聚合** (第79节)
   - 凸优化拆分
   - Gas 感知优化
   - 报价引擎

10. **高级技术** (第80节)
    - 矩阵套利检测
    - 闭式解计算
    - DP 路径优化
    - 蒙特卡洛模拟
    - 数值精度处理

这些数学模型是构建高效 DeFi 应用和套利机器人的基础。

---

## 附录：Curve 高级数学模型

### A.1 Curve Meta Pool 数学

Meta Pool 是基于基础池（如 3pool）的二级池：

```go
package dexmath

import (
    "math/big"
)

// MetaPool Curve Meta Pool (如 FRAX/3CRV)
type MetaPool struct {
    *StableSwap
    basePool      *StableSwap
    basePoolLP    *big.Int       // 基础池 LP 数量
    baseCoinIndex int            // 基础池 LP 在 Meta Pool 中的索引
    baseCoins     int            // 基础池代币数量
}

// NewMetaPool 创建 Meta Pool
func NewMetaPool(
    balances []*big.Int,
    a, fee, adminFee *big.Int,
    basePool *StableSwap,
    baseCoinIndex int,
) *MetaPool {
    return &MetaPool{
        StableSwap:    NewStableSwap(balances, a, fee, adminFee),
        basePool:      basePool,
        basePoolLP:    balances[baseCoinIndex],
        baseCoinIndex: baseCoinIndex,
        baseCoins:     len(basePool.Balances),
    }
}

// GetDyUnderlying 获取底层代币交换输出
// 支持跨基础池和 Meta Pool 的交换
func (mp *MetaPool) GetDyUnderlying(i, j int, dx *big.Int) (*big.Int, error) {
    metaCoins := len(mp.Balances)
    baseCoins := mp.baseCoins

    // 确定交换类型
    iIsMeta := i < metaCoins-1  // 是否是 Meta 代币 (非 LP)
    jIsMeta := j < metaCoins-1
    iIsBase := i >= metaCoins-1 && i < metaCoins-1+baseCoins
    jIsBase := j >= metaCoins-1 && j < metaCoins-1+baseCoins

    if iIsMeta && jIsMeta {
        // Meta -> Meta: 直接在 Meta Pool 交换
        return mp.GetDY(i, j, dx), nil
    }

    if iIsMeta && jIsBase {
        // Meta -> Base: 先换成 LP，再在基础池换
        // 步骤1: Meta -> LP
        lpOut := mp.GetDY(i, mp.baseCoinIndex, dx)

        // 步骤2: LP -> Base coin (计算 LP 对应的基础代币)
        baseJ := j - (metaCoins - 1)
        return mp.calcWithdrawOneCoin(lpOut, baseJ), nil
    }

    if iIsBase && jIsMeta {
        // Base -> Meta: 先存入基础池获得 LP，再在 Meta Pool 换
        baseI := i - (metaCoins - 1)

        // 步骤1: Base -> LP
        lpIn := mp.calcAddLiquidity(baseI, dx)

        // 步骤2: LP -> Meta
        return mp.GetDY(mp.baseCoinIndex, j, lpIn), nil
    }

    if iIsBase && jIsBase {
        // Base -> Base: 直接在基础池交换
        baseI := i - (metaCoins - 1)
        baseJ := j - (metaCoins - 1)
        return mp.basePool.GetDY(baseI, baseJ, dx), nil
    }

    return nil, fmt.Errorf("invalid coin indices")
}

// calcWithdrawOneCoin 计算单币提取
func (mp *MetaPool) calcWithdrawOneCoin(lpAmount *big.Int, coinIndex int) *big.Int {
    // 使用基础池的 calc_withdraw_one_coin 逻辑
    // 简化实现
    d0 := mp.basePool.GetD(mp.basePool.Balances)

    // 计算新的 D
    totalSupply := mp.getTotalSupply()
    d1 := new(big.Int).Mul(d0, new(big.Int).Sub(totalSupply, lpAmount))
    d1.Div(d1, totalSupply)

    // 计算新余额
    newY := mp.basePool.GetYD(coinIndex, d1)
    dy := new(big.Int).Sub(mp.basePool.Balances[coinIndex], newY)

    // 扣除费用
    fee := new(big.Int).Mul(dy, mp.basePool.Fee)
    fee.Div(fee, big.NewInt(10000000000))
    dy.Sub(dy, fee)

    return dy
}

// calcAddLiquidity 计算添加流动性获得的 LP
func (mp *MetaPool) calcAddLiquidity(coinIndex int, amount *big.Int) *big.Int {
    // 简化实现：单币添加流动性
    d0 := mp.basePool.GetD(mp.basePool.Balances)

    // 新余额
    newBalances := make([]*big.Int, len(mp.basePool.Balances))
    for i := range newBalances {
        newBalances[i] = new(big.Int).Set(mp.basePool.Balances[i])
    }
    newBalances[coinIndex].Add(newBalances[coinIndex], amount)

    d1 := mp.basePool.GetD(newBalances)

    // 计算 LP 数量
    totalSupply := mp.getTotalSupply()
    mintAmount := new(big.Int).Mul(totalSupply, new(big.Int).Sub(d1, d0))
    mintAmount.Div(mintAmount, d0)

    return mintAmount
}

// getTotalSupply 获取总供应量 (简化)
func (mp *MetaPool) getTotalSupply() *big.Int {
    return big.NewInt(1e18) // 实际需要从合约获取
}

// GetYD 根据 D 计算 Y
func (ss *StableSwap) GetYD(j int, d *big.Int) *big.Int {
    n := len(ss.Balances)
    nBI := big.NewInt(int64(n))

    // c = D^(n+1) / (n^n * prod(x[i]) for i != j)
    c := new(big.Int).Set(d)
    s := big.NewInt(0)

    for i := 0; i < n; i++ {
        if i == j {
            continue
        }
        s.Add(s, ss.Balances[i])
        c.Mul(c, d)
        c.Div(c, new(big.Int).Mul(ss.Balances[i], nBI))
    }

    c.Mul(c, d)
    c.Div(c, new(big.Int).Mul(new(big.Int).Mul(ss.A, nBI), nBI))

    b := new(big.Int).Add(s, new(big.Int).Div(d, new(big.Int).Mul(ss.A, nBI)))

    // 牛顿迭代
    y := new(big.Int).Set(d)
    for i := 0; i < 255; i++ {
        yPrev := new(big.Int).Set(y)

        // y = (y^2 + c) / (2*y + b - D)
        numerator := new(big.Int).Add(new(big.Int).Mul(y, y), c)
        denominator := new(big.Int).Sub(
            new(big.Int).Add(new(big.Int).Mul(y, big.NewInt(2)), b),
            d,
        )

        y.Div(numerator, denominator)

        if new(big.Int).Abs(new(big.Int).Sub(y, yPrev)).Cmp(big.NewInt(1)) <= 0 {
            break
        }
    }

    return y
}
```

### A.2 Curve CryptoSwap (V2) 数学

Curve V2 使用动态 peg 支持非稳定资产对：

```go
package dexmath

import (
    "math/big"
)

// CryptoSwap Curve V2 CryptoSwap 池
type CryptoSwap struct {
    Balances      []*big.Int
    PriceScale    []*big.Int  // 内部价格缩放因子
    PriceOracle   []*big.Int  // 预言机价格 (EMA)
    LastPrices    []*big.Int  // 最后交易价格
    D             *big.Int    // 当前 D 值
    A             *big.Int    // 放大系数
    Gamma         *big.Int    // Gamma 参数 (控制曲线形状)
    FeeGamma      *big.Int    // 动态费用 Gamma
    MidFee        *big.Int    // 中间费用
    OutFee        *big.Int    // 边缘费用
    FutureAGamma  *big.Int    // 未来 A*Gamma
    AdjustmentStep *big.Int   // 调整步长
    MaTime        *big.Int    // 移动平均时间
}

// NewCryptoSwap 创建 CryptoSwap 池
func NewCryptoSwap(
    balances []*big.Int,
    a, gamma, midFee, outFee *big.Int,
) *CryptoSwap {
    n := len(balances)
    priceScale := make([]*big.Int, n-1)
    for i := range priceScale {
        priceScale[i] = big.NewInt(1e18) // 初始价格 1:1
    }

    return &CryptoSwap{
        Balances:   balances,
        PriceScale: priceScale,
        A:          a,
        Gamma:      gamma,
        MidFee:     midFee,
        OutFee:     outFee,
    }
}

// GetD CryptoSwap 的 D 计算
// 使用修改的不变量: D^(n-1) * sum(x_i) + D^n / (A * n^n * prod(x_i) * gamma^(n-1)) = D^n / gamma^(n-1)
func (cs *CryptoSwap) GetD(xp []*big.Int) *big.Int {
    n := len(xp)
    nBI := big.NewInt(int64(n))

    // S = sum(x)
    s := big.NewInt(0)
    for _, x := range xp {
        s.Add(s, x)
    }

    // 初始 D 猜测
    d := new(big.Int).Set(s)

    // 牛顿迭代求解
    for i := 0; i < 255; i++ {
        dPrev := new(big.Int).Set(d)

        // K0 = D^n / (n^n * prod(x))
        k0 := new(big.Int).Exp(d, nBI, nil)
        for _, x := range xp {
            k0.Div(k0, new(big.Int).Mul(x, nBI))
        }

        // g = gamma / 1e18
        // K = A * K0 * gamma^2 / (gamma + 1 - K0)^2
        gammaPlusOne := new(big.Int).Add(cs.Gamma, big.NewInt(1e18))
        gammaPlusOneMinusK0 := new(big.Int).Sub(gammaPlusOne, k0)

        k := new(big.Int).Mul(cs.A, k0)
        k.Mul(k, cs.Gamma)
        k.Mul(k, cs.Gamma)
        denominator := new(big.Int).Mul(gammaPlusOneMinusK0, gammaPlusOneMinusK0)
        k.Div(k, denominator)
        k.Div(k, big.NewInt(1e36)) // 调整精度

        // f = K * D / (K - 1) - S
        // f' = K / (K - 1)
        // D_new = D - f / f'

        kMinus1 := new(big.Int).Sub(k, big.NewInt(1e18))
        if kMinus1.Sign() == 0 {
            break
        }

        numerator := new(big.Int).Mul(k, d)
        numerator.Div(numerator, big.NewInt(1e18))
        f := new(big.Int).Sub(new(big.Int).Div(numerator, kMinus1), s)

        fPrime := new(big.Int).Div(k, kMinus1)

        dDelta := new(big.Int).Mul(f, big.NewInt(1e18))
        dDelta.Div(dDelta, fPrime)
        d.Sub(d, dDelta)

        if new(big.Int).Abs(new(big.Int).Sub(d, dPrev)).Cmp(big.NewInt(1)) <= 0 {
            break
        }
    }

    return d
}

// GetY CryptoSwap 的 Y 计算
func (cs *CryptoSwap) GetY(i, j int, x *big.Int, xp []*big.Int) *big.Int {
    d := cs.GetD(xp)
    n := len(xp)
    nBI := big.NewInt(int64(n))

    // 创建新余额数组
    newXp := make([]*big.Int, n)
    for k := range newXp {
        if k == i {
            newXp[k] = x
        } else if k == j {
            newXp[k] = big.NewInt(0) // 待求解
        } else {
            newXp[k] = new(big.Int).Set(xp[k])
        }
    }

    // 使用牛顿迭代求解 y
    y := new(big.Int).Set(d)

    for iter := 0; iter < 255; iter++ {
        yPrev := new(big.Int).Set(y)

        // 计算 K0
        k0 := new(big.Int).Exp(d, nBI, nil)
        for k := 0; k < n; k++ {
            var val *big.Int
            if k == j {
                val = y
            } else {
                val = newXp[k]
            }
            k0.Div(k0, new(big.Int).Mul(val, nBI))
        }

        // 简化的牛顿迭代
        // 实际实现更复杂，需要考虑 gamma 参数
        s := big.NewInt(0)
        for k := 0; k < n; k++ {
            if k == j {
                continue
            }
            s.Add(s, newXp[k])
        }

        c := new(big.Int).Mul(d, d)
        c.Div(c, new(big.Int).Mul(big.NewInt(2), y))

        b := new(big.Int).Add(s, y)

        // y = (y^2 + c) / (2y + b - d)
        numerator := new(big.Int).Add(new(big.Int).Mul(y, y), c)
        denominator := new(big.Int).Sub(new(big.Int).Add(new(big.Int).Mul(y, big.NewInt(2)), b), d)

        if denominator.Sign() <= 0 {
            break
        }

        y.Div(numerator, denominator)

        if new(big.Int).Abs(new(big.Int).Sub(y, yPrev)).Cmp(big.NewInt(1)) <= 0 {
            break
        }
    }

    return y
}

// GetDynamicFee 计算动态费用
func (cs *CryptoSwap) GetDynamicFee(xp []*big.Int) *big.Int {
    n := len(xp)

    // 计算价格偏离度
    // 当价格接近 peg 时使用 midFee，远离时使用 outFee
    s := big.NewInt(0)
    for _, x := range xp {
        s.Add(s, x)
    }

    avg := new(big.Int).Div(s, big.NewInt(int64(n)))

    // 计算方差
    variance := big.NewInt(0)
    for _, x := range xp {
        diff := new(big.Int).Sub(x, avg)
        diff.Mul(diff, diff)
        variance.Add(variance, diff)
    }
    variance.Div(variance, big.NewInt(int64(n)))

    // 根据方差在 midFee 和 outFee 之间插值
    // fee = midFee + (outFee - midFee) * variance / (variance + feeGamma)
    feeDiff := new(big.Int).Sub(cs.OutFee, cs.MidFee)
    fee := new(big.Int).Mul(feeDiff, variance)
    fee.Div(fee, new(big.Int).Add(variance, cs.FeeGamma))
    fee.Add(fee, cs.MidFee)

    return fee
}

// ExchangeUnderlying 交换底层代币
func (cs *CryptoSwap) ExchangeUnderlying(i, j int, dx *big.Int) (*big.Int, error) {
    // 调整输入金额到内部精度
    xp := cs.getScaledBalances()

    var scaledDx *big.Int
    if i > 0 {
        scaledDx = new(big.Int).Mul(dx, cs.PriceScale[i-1])
        scaledDx.Div(scaledDx, big.NewInt(1e18))
    } else {
        scaledDx = dx
    }

    newX := new(big.Int).Add(xp[i], scaledDx)
    newY := cs.GetY(i, j, newX, xp)

    dy := new(big.Int).Sub(xp[j], newY)

    // 计算动态费用
    fee := cs.GetDynamicFee(xp)
    feeAmount := new(big.Int).Mul(dy, fee)
    feeAmount.Div(feeAmount, big.NewInt(10000000000))
    dy.Sub(dy, feeAmount)

    // 反向缩放
    if j > 0 {
        dy.Mul(dy, big.NewInt(1e18))
        dy.Div(dy, cs.PriceScale[j-1])
    }

    return dy, nil
}

// getScaledBalances 获取缩放后的余额
func (cs *CryptoSwap) getScaledBalances() []*big.Int {
    xp := make([]*big.Int, len(cs.Balances))
    for i, balance := range cs.Balances {
        if i == 0 {
            xp[i] = new(big.Int).Set(balance)
        } else {
            xp[i] = new(big.Int).Mul(balance, cs.PriceScale[i-1])
            xp[i].Div(xp[i], big.NewInt(1e18))
        }
    }
    return xp
}
```

### A.3 Curve TriCrypto 数学

TriCrypto 池 (如 USDT/WBTC/WETH) 的三代币 CryptoSwap：

```go
// TriCryptoPool 三代币 CryptoSwap
type TriCryptoPool struct {
    *CryptoSwap
    Precisions [3]*big.Int // 代币精度
}

// NewTriCryptoPool 创建 TriCrypto 池
func NewTriCryptoPool(
    balances [3]*big.Int,
    priceScale [2]*big.Int,
    a, gamma *big.Int,
) *TriCryptoPool {
    return &TriCryptoPool{
        CryptoSwap: &CryptoSwap{
            Balances:   balances[:],
            PriceScale: priceScale[:],
            A:          a,
            Gamma:      gamma,
            MidFee:     big.NewInt(3000000),  // 0.03%
            OutFee:     big.NewInt(30000000), // 0.3%
            FeeGamma:   big.NewInt(500000000000000), // 0.0005
        },
        Precisions: [3]*big.Int{
            big.NewInt(1e12),  // USDT (6 decimals)
            big.NewInt(1e10),  // WBTC (8 decimals)
            big.NewInt(1),     // WETH (18 decimals)
        },
    }
}

// GetTriCryptoOutput 计算三币池输出
func (tp *TriCryptoPool) GetTriCryptoOutput(i, j int, dx *big.Int) *big.Int {
    // 精度调整
    xp := make([]*big.Int, 3)
    for k := 0; k < 3; k++ {
        xp[k] = new(big.Int).Mul(tp.Balances[k], tp.Precisions[k])
        if k > 0 {
            xp[k].Mul(xp[k], tp.PriceScale[k-1])
            xp[k].Div(xp[k], big.NewInt(1e18))
        }
    }

    // 调整输入
    scaledDx := new(big.Int).Mul(dx, tp.Precisions[i])
    if i > 0 {
        scaledDx.Mul(scaledDx, tp.PriceScale[i-1])
        scaledDx.Div(scaledDx, big.NewInt(1e18))
    }

    newX := new(big.Int).Add(xp[i], scaledDx)
    newY := tp.GetY(i, j, newX, xp)

    dy := new(big.Int).Sub(xp[j], newY)

    // 扣除费用
    fee := tp.GetDynamicFee(xp)
    feeAmount := new(big.Int).Mul(dy, fee)
    feeAmount.Div(feeAmount, big.NewInt(10000000000))
    dy.Sub(dy, feeAmount)

    // 反向精度调整
    if j > 0 {
        dy.Mul(dy, big.NewInt(1e18))
        dy.Div(dy, tp.PriceScale[j-1])
    }
    dy.Div(dy, tp.Precisions[j])

    return dy
}
```

### A.4 套利优化示例

```go
// CurveArbitrageOptimizer Curve 套利优化器
type CurveArbitrageOptimizer struct {
    pools map[string]interface{} // StableSwap 或 CryptoSwap
}

// FindCurveArbitrage 寻找 Curve 套利机会
func (o *CurveArbitrageOptimizer) FindCurveArbitrage(
    token0, token1 string,
    externalPrice *big.Float,
) (*ArbitrageOpportunity, error) {
    // 获取 Curve 内部价格
    curvePrice := o.getCurvePrice(token0, token1)

    // 计算价差
    diff := new(big.Float).Sub(externalPrice, curvePrice)
    ratio := new(big.Float).Quo(diff, curvePrice)
    ratioFloat, _ := ratio.Float64()

    // 检查是否有利可图 (考虑费用)
    if ratioFloat < 0.001 && ratioFloat > -0.001 {
        return nil, nil // 无套利机会
    }

    // 计算最优交易量
    optimalAmount := o.calculateOptimalAmount(token0, token1, ratioFloat)

    return &ArbitrageOpportunity{
        Type:          "curve_external",
        TokenIn:       token0,
        TokenOut:      token1,
        Amount:        optimalAmount,
        ExpectedProfit: o.calculateProfit(optimalAmount, ratioFloat),
    }, nil
}

// getCurvePrice 获取 Curve 价格
func (o *CurveArbitrageOptimizer) getCurvePrice(token0, token1 string) *big.Float {
    // 使用 dy/dx 计算边际价格
    dx := big.NewInt(1e18) // 1 个代币

    // 根据池类型调用相应方法
    // 返回价格
    return big.NewFloat(1.0)
}

// calculateOptimalAmount 计算最优交易量
func (o *CurveArbitrageOptimizer) calculateOptimalAmount(
    token0, token1 string,
    priceDiff float64,
) *big.Int {
    // 使用二分搜索或解析解找最优量
    // 考虑价格影响和费用

    return big.NewInt(1e18)
}

// ArbitrageOpportunity 套利机会
type ArbitrageOpportunity struct {
    Type           string
    TokenIn        string
    TokenOut       string
    Amount         *big.Int
    ExpectedProfit *big.Int
}

// calculateProfit 计算利润
func (o *CurveArbitrageOptimizer) calculateProfit(amount *big.Int, diff float64) *big.Int {
    profit := new(big.Float).Mul(
        new(big.Float).SetInt(amount),
        big.NewFloat(diff),
    )
    profitInt, _ := profit.Int(nil)
    return profitInt
}
```

## 附录 B: Balancer 与 Maverick 数学模型

### B.1 Balancer 加权池数学

Balancer 使用加权恒定乘积公式，允许任意权重的多代币池。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Balancer 加权池恒定值公式                      │
│                                                                 │
│  不变量: V = ∏(Bi^Wi) = 常数                                    │
│                                                                 │
│  其中:                                                          │
│  - Bi = 代币 i 的余额                                           │
│  - Wi = 代币 i 的权重 (所有权重之和为 1)                         │
│                                                                 │
│  示例: 80/20 ETH/USDC 池                                        │
│  V = ETH^0.8 × USDC^0.2                                        │
│                                                                 │
│  交换公式:                                                       │
│  Ao = Bo × (1 - (Bi / (Bi + Ai))^(Wi/Wo))                      │
│                                                                 │
│  其中:                                                          │
│  - Ai = 输入代币数量                                            │
│  - Ao = 输出代币数量                                            │
│  - Bi = 输入代币余额                                            │
│  - Bo = 输出代币余额                                            │
│  - Wi = 输入代币权重                                            │
│  - Wo = 输出代币权重                                            │
│                                                                 │
│  边际价格 (Spot Price):                                         │
│  SP = (Bi / Wi) / (Bo / Wo)                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```go
package balancer

import (
    "math"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
)

// BalancerWeightedPool Balancer 加权池
type BalancerWeightedPool struct {
    Address    common.Address
    Tokens     []common.Address
    Balances   []*big.Int
    Weights    []float64  // 权重, 总和为 1
    SwapFee    *big.Int   // 18 位小数, 如 0.003e18 = 0.3%
    Decimals   []uint8
}

// BalancerMath Balancer 数学库
type BalancerMath struct{}

// CalcOutGivenIn 计算给定输入时的输出量
// Ao = Bo × (1 - (Bi / (Bi + Ai × (1 - fee)))^(Wi/Wo))
func (m *BalancerMath) CalcOutGivenIn(
    balanceIn *big.Int,
    weightIn float64,
    balanceOut *big.Int,
    weightOut float64,
    amountIn *big.Int,
    swapFee *big.Int,
) *big.Int {
    // 扣除手续费
    feeComplement := new(big.Int).Sub(big.NewInt(1e18), swapFee)
    adjustedIn := new(big.Int).Mul(amountIn, feeComplement)
    adjustedIn.Div(adjustedIn, big.NewInt(1e18))

    // 计算 Bi / (Bi + Ai)
    newBalanceIn := new(big.Int).Add(balanceIn, adjustedIn)

    // 使用浮点数进行幂运算
    balanceInF := new(big.Float).SetInt(balanceIn)
    newBalanceInF := new(big.Float).SetInt(newBalanceIn)
    balanceOutF := new(big.Float).SetInt(balanceOut)

    // ratio = Bi / (Bi + Ai)
    ratio, _ := new(big.Float).Quo(balanceInF, newBalanceInF).Float64()

    // exponent = Wi / Wo
    exponent := weightIn / weightOut

    // (ratio)^exponent
    power := math.Pow(ratio, exponent)

    // Ao = Bo × (1 - power)
    amountOutF := new(big.Float).Mul(balanceOutF, big.NewFloat(1-power))
    amountOut, _ := amountOutF.Int(nil)

    return amountOut
}

// CalcInGivenOut 计算给定输出时的输入量
// Ai = Bi × ((Bo / (Bo - Ao))^(Wo/Wi) - 1) / (1 - fee)
func (m *BalancerMath) CalcInGivenOut(
    balanceIn *big.Int,
    weightIn float64,
    balanceOut *big.Int,
    weightOut float64,
    amountOut *big.Int,
    swapFee *big.Int,
) *big.Int {
    // 计算 Bo / (Bo - Ao)
    newBalanceOut := new(big.Int).Sub(balanceOut, amountOut)

    balanceInF := new(big.Float).SetInt(balanceIn)
    balanceOutF := new(big.Float).SetInt(balanceOut)
    newBalanceOutF := new(big.Float).SetInt(newBalanceOut)

    // ratio = Bo / (Bo - Ao)
    ratio, _ := new(big.Float).Quo(balanceOutF, newBalanceOutF).Float64()

    // exponent = Wo / Wi
    exponent := weightOut / weightIn

    // (ratio)^exponent - 1
    power := math.Pow(ratio, exponent) - 1

    // Ai = Bi × power
    amountInF := new(big.Float).Mul(balanceInF, big.NewFloat(power))
    amountIn, _ := amountInF.Int(nil)

    // 加上手续费
    feeComplement := new(big.Int).Sub(big.NewInt(1e18), swapFee)
    amountIn.Mul(amountIn, big.NewInt(1e18))
    amountIn.Div(amountIn, feeComplement)

    return amountIn
}

// CalcSpotPrice 计算即期价格
// SP = (Bi / Wi) / (Bo / Wo) × (1 / (1 - fee))
func (m *BalancerMath) CalcSpotPrice(
    balanceIn *big.Int,
    weightIn float64,
    balanceOut *big.Int,
    weightOut float64,
    swapFee *big.Int,
) *big.Float {
    balanceInF := new(big.Float).SetInt(balanceIn)
    balanceOutF := new(big.Float).SetInt(balanceOut)

    // (Bi / Wi)
    numerator := new(big.Float).Quo(balanceInF, big.NewFloat(weightIn))

    // (Bo / Wo)
    denominator := new(big.Float).Quo(balanceOutF, big.NewFloat(weightOut))

    // 基础价格
    price := new(big.Float).Quo(numerator, denominator)

    // 加上手续费调整
    swapFeeF := new(big.Float).SetInt(swapFee)
    feeComplement := new(big.Float).Sub(big.NewFloat(1e18), swapFeeF)
    feeMultiplier := new(big.Float).Quo(big.NewFloat(1e18), feeComplement)
    price.Mul(price, feeMultiplier)

    return price
}
```

### B.2 Balancer V2 Composable Stable Pool

```go
// ComposableStablePool Balancer V2 Composable Stable Pool
// 结合了 Curve StableSwap 和 Balancer 的设计
type ComposableStablePool struct {
    Address       common.Address
    Tokens        []common.Address
    Balances      []*big.Int
    BptIndex      int        // BPT 代币在数组中的索引
    Amp           *big.Int   // 放大系数
    SwapFee       *big.Int
    ScalingFactors []*big.Int // 各代币的缩放因子
}

// ComposableStableMath Composable Stable 数学库
type ComposableStableMath struct{}

// CalcOutGivenIn 计算输出 (排除 BPT)
func (m *ComposableStableMath) CalcOutGivenIn(
    pool *ComposableStablePool,
    tokenInIndex int,
    tokenOutIndex int,
    amountIn *big.Int,
) *big.Int {
    // 1. 获取除 BPT 外的余额
    balances := m.dropBptItem(pool.Balances, pool.BptIndex)

    // 调整索引
    adjustedInIndex := m.adjustIndex(tokenInIndex, pool.BptIndex)
    adjustedOutIndex := m.adjustIndex(tokenOutIndex, pool.BptIndex)

    // 2. 应用缩放因子
    scaledBalances := m.upscaleBalances(balances, pool.ScalingFactors)
    scaledAmountIn := m.upscale(amountIn, pool.ScalingFactors[tokenInIndex])

    // 3. 扣除手续费
    feeComplement := new(big.Int).Sub(big.NewInt(1e18), pool.SwapFee)
    scaledAmountIn.Mul(scaledAmountIn, feeComplement)
    scaledAmountIn.Div(scaledAmountIn, big.NewInt(1e18))

    // 4. 计算不变量
    invariant := m.calculateInvariant(pool.Amp, scaledBalances)

    // 5. 计算输出
    scaledAmountOut := m.calcOutGivenInStable(
        scaledBalances,
        adjustedInIndex,
        adjustedOutIndex,
        scaledAmountIn,
        invariant,
        pool.Amp,
    )

    // 6. 反向缩放
    return m.downscale(scaledAmountOut, pool.ScalingFactors[tokenOutIndex])
}

// calculateInvariant 计算 StableSwap 不变量
func (m *ComposableStableMath) calculateInvariant(amp *big.Int, balances []*big.Int) *big.Int {
    n := len(balances)
    sum := big.NewInt(0)
    for _, b := range balances {
        sum.Add(sum, b)
    }

    if sum.Sign() == 0 {
        return big.NewInt(0)
    }

    // 牛顿迭代求解 D
    ampTimesNpowN := new(big.Int).Mul(amp, big.NewInt(int64(n)))
    for i := 1; i < n; i++ {
        ampTimesNpowN.Mul(ampTimesNpowN, big.NewInt(int64(n)))
    }

    D := new(big.Int).Set(sum)
    prevD := big.NewInt(0)

    for i := 0; i < 255; i++ {
        dP := new(big.Int).Set(D)
        for _, b := range balances {
            dP.Mul(dP, D)
            dP.Div(dP, new(big.Int).Mul(b, big.NewInt(int64(n))))
        }

        prevD.Set(D)

        numerator := new(big.Int).Mul(ampTimesNpowN, sum)
        numerator.Add(numerator, new(big.Int).Mul(big.NewInt(int64(n)), dP))
        numerator.Mul(numerator, D)

        denominator := new(big.Int).Sub(ampTimesNpowN, big.NewInt(1))
        denominator.Mul(denominator, D)
        denominator.Add(denominator, new(big.Int).Mul(big.NewInt(int64(n+1)), dP))

        D.Div(numerator, denominator)

        diff := new(big.Int).Sub(D, prevD)
        diff.Abs(diff)
        if diff.Cmp(big.NewInt(1)) <= 0 {
            return D
        }
    }

    return D
}

func (m *ComposableStableMath) dropBptItem(items []*big.Int, bptIndex int) []*big.Int {
    result := make([]*big.Int, 0, len(items)-1)
    for i, item := range items {
        if i != bptIndex {
            result = append(result, item)
        }
    }
    return result
}

func (m *ComposableStableMath) adjustIndex(index, bptIndex int) int {
    if index > bptIndex {
        return index - 1
    }
    return index
}

func (m *ComposableStableMath) upscale(amount *big.Int, scalingFactor *big.Int) *big.Int {
    return new(big.Int).Mul(amount, scalingFactor)
}

func (m *ComposableStableMath) downscale(amount *big.Int, scalingFactor *big.Int) *big.Int {
    return new(big.Int).Div(amount, scalingFactor)
}

func (m *ComposableStableMath) upscaleBalances(balances []*big.Int, factors []*big.Int) []*big.Int {
    result := make([]*big.Int, len(balances))
    for i, b := range balances {
        result[i] = m.upscale(b, factors[i])
    }
    return result
}
```

### B.3 Maverick AMM 数学

Maverick 使用创新的"流动性仓位"(Liquidity Bin) 模型，允许流动性提供者选择单向或双向流动性，并可以根据价格变动自动调整位置。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Maverick 流动性仓位模型                        │
│                                                                 │
│  Maverick 将价格空间划分为离散的 "bins"                          │
│  每个 bin 有固定的价格范围 [tickLower, tickUpper]                 │
│                                                                 │
│  价格与 tick 的关系:                                             │
│  price = 1.0001^tick                                            │
│                                                                 │
│  流动性类型:                                                     │
│  - Mode 0: Static (固定位置)                                     │
│  - Mode 1: Right (价格上涨时跟随)                                │
│  - Mode 2: Left (价格下跌时跟随)                                 │
│  - Mode 3: Both (双向跟随)                                       │
│                                                                 │
│  单个 Bin 内的交换:                                              │
│  使用恒定和公式: L = x + y/p                                     │
│  输出量: dy = dx × p × (1 - fee)                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```go
package maverick

import (
    "math"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
)

// LiquidityMode 流动性模式
type LiquidityMode uint8

const (
    ModeStatic LiquidityMode = 0 // 固定位置
    ModeRight  LiquidityMode = 1 // 价格上涨跟随
    ModeLeft   LiquidityMode = 2 // 价格下跌跟随
    ModeBoth   LiquidityMode = 3 // 双向跟随
)

// MaverickPool Maverick 池
type MaverickPool struct {
    Address      common.Address
    TokenA       common.Address
    TokenB       common.Address
    Fee          *big.Int     // 18 位小数
    TickSpacing  int64        // tick 间距
    ActiveTick   int64        // 当前活跃 tick
    Bins         map[int64]*Bin
}

// Bin 流动性仓位
type Bin struct {
    ID           int64
    LowerTick    int64
    UpperTick    int64
    ReserveA     *big.Int
    ReserveB     *big.Int
}

// MaverickMath Maverick 数学库
type MaverickMath struct {
    tickBase float64 // 1.0001
}

// NewMaverickMath 创建数学库
func NewMaverickMath() *MaverickMath {
    return &MaverickMath{tickBase: 1.0001}
}

// TickToPrice 将 tick 转换为价格
func (m *MaverickMath) TickToPrice(tick int64) *big.Float {
    price := math.Pow(m.tickBase, float64(tick))
    return big.NewFloat(price)
}

// PriceToTick 将价格转换为 tick
func (m *MaverickMath) PriceToTick(price float64) int64 {
    tick := math.Log(price) / math.Log(m.tickBase)
    return int64(math.Floor(tick))
}

// CalcSwapInBin 计算单个 bin 内的交换
func (m *MaverickMath) CalcSwapInBin(
    bin *Bin,
    amountIn *big.Int,
    tokenAIn bool,
    fee *big.Int,
) (amountOut *big.Int, remainingIn *big.Int) {
    // 获取 bin 的价格
    price := m.TickToPrice((bin.LowerTick + bin.UpperTick) / 2)
    priceF, _ := price.Float64()

    // 扣除手续费
    feeComplement := new(big.Int).Sub(big.NewInt(1e18), fee)
    effectiveIn := new(big.Int).Mul(amountIn, feeComplement)
    effectiveIn.Div(effectiveIn, big.NewInt(1e18))

    effectiveInF := new(big.Float).SetInt(effectiveIn)

    if tokenAIn {
        // A -> B
        maxAIn := new(big.Float).SetInt(bin.ReserveB)
        maxAIn.Quo(maxAIn, big.NewFloat(priceF))

        maxAInInt, _ := maxAIn.Int(nil)

        if effectiveIn.Cmp(maxAInInt) <= 0 {
            amountOutF := new(big.Float).Mul(effectiveInF, big.NewFloat(priceF))
            amountOut, _ = amountOutF.Int(nil)
            return amountOut, big.NewInt(0)
        } else {
            amountOut = new(big.Int).Set(bin.ReserveB)
            remainingIn = new(big.Int).Sub(effectiveIn, maxAInInt)
            remainingIn.Mul(remainingIn, big.NewInt(1e18))
            remainingIn.Div(remainingIn, feeComplement)
            return amountOut, remainingIn
        }
    } else {
        // B -> A
        maxBIn := new(big.Float).SetInt(bin.ReserveA)
        maxBIn.Mul(maxBIn, big.NewFloat(priceF))

        maxBInInt, _ := maxBIn.Int(nil)

        if effectiveIn.Cmp(maxBInInt) <= 0 {
            amountOutF := new(big.Float).Quo(effectiveInF, big.NewFloat(priceF))
            amountOut, _ = amountOutF.Int(nil)
            return amountOut, big.NewInt(0)
        } else {
            amountOut = new(big.Int).Set(bin.ReserveA)
            remainingIn = new(big.Int).Sub(effectiveIn, maxBInInt)
            remainingIn.Mul(remainingIn, big.NewInt(1e18))
            remainingIn.Div(remainingIn, feeComplement)
            return amountOut, remainingIn
        }
    }
}

// CalcSwapAcrossBins 计算跨多个 bin 的交换
func (m *MaverickMath) CalcSwapAcrossBins(
    pool *MaverickPool,
    amountIn *big.Int,
    tokenAIn bool,
) *big.Int {
    totalOut := big.NewInt(0)
    remaining := new(big.Int).Set(amountIn)

    activeBins := m.getActiveBins(pool, tokenAIn)

    for _, binID := range activeBins {
        if remaining.Sign() == 0 {
            break
        }

        bin := pool.Bins[binID]
        if bin == nil {
            continue
        }

        out, newRemaining := m.CalcSwapInBin(bin, remaining, tokenAIn, pool.Fee)
        totalOut.Add(totalOut, out)
        remaining = newRemaining
    }

    return totalOut
}

// getActiveBins 获取活跃的 bins (按价格排序)
func (m *MaverickMath) getActiveBins(pool *MaverickPool, tokenAIn bool) []int64 {
    var bins []int64

    for binID := range pool.Bins {
        bins = append(bins, binID)
    }

    // A->B: 升序, B->A: 降序
    if tokenAIn {
        for i := 0; i < len(bins)-1; i++ {
            for j := i + 1; j < len(bins); j++ {
                if bins[i] > bins[j] {
                    bins[i], bins[j] = bins[j], bins[i]
                }
            }
        }
    } else {
        for i := 0; i < len(bins)-1; i++ {
            for j := i + 1; j < len(bins); j++ {
                if bins[i] < bins[j] {
                    bins[i], bins[j] = bins[j], bins[i]
                }
            }
        }
    }

    return bins
}
```

### B.4 Balancer 套利检测

```go
// BalancerArbitrageDetector Balancer 套利检测器
type BalancerArbitrageDetector struct {
    math *BalancerMath
}

// FindArbitrageOpportunities 查找套利机会
func (d *BalancerArbitrageDetector) FindArbitrageOpportunities(
    pool *BalancerWeightedPool,
    externalPrices map[string]*big.Float,
) []BalancerArbitrageOpportunity {
    var opportunities []BalancerArbitrageOpportunity

    for i := 0; i < len(pool.Tokens); i++ {
        for j := 0; j < len(pool.Tokens); j++ {
            if i == j {
                continue
            }

            spotPrice := d.math.CalcSpotPrice(
                pool.Balances[i],
                pool.Weights[i],
                pool.Balances[j],
                pool.Weights[j],
                pool.SwapFee,
            )

            pairKey := pool.Tokens[i].Hex() + "-" + pool.Tokens[j].Hex()
            externalPrice, ok := externalPrices[pairKey]
            if !ok {
                continue
            }

            diff := new(big.Float).Sub(externalPrice, spotPrice)
            ratio := new(big.Float).Quo(diff, spotPrice)
            ratioF, _ := ratio.Float64()

            if ratioF > 0.001 || ratioF < -0.001 {
                optimalAmount := d.calculateOptimalAmount(pool, i, j, ratioF)
                profit := d.calculateExpectedProfit(pool, i, j, optimalAmount)

                opportunities = append(opportunities, BalancerArbitrageOpportunity{
                    PoolAddress:    pool.Address,
                    TokenIn:        pool.Tokens[i],
                    TokenOut:       pool.Tokens[j],
                    OptimalAmount:  optimalAmount,
                    ExpectedProfit: profit,
                    SpotPrice:      spotPrice,
                    ExternalPrice:  externalPrice,
                    PriceDiffBps:   int(ratioF * 10000),
                })
            }
        }
    }

    return opportunities
}

// BalancerArbitrageOpportunity 套利机会
type BalancerArbitrageOpportunity struct {
    PoolAddress    common.Address
    TokenIn        common.Address
    TokenOut       common.Address
    OptimalAmount  *big.Int
    ExpectedProfit *big.Int
    SpotPrice      *big.Float
    ExternalPrice  *big.Float
    PriceDiffBps   int
}

func (d *BalancerArbitrageDetector) calculateOptimalAmount(
    pool *BalancerWeightedPool,
    tokenInIndex int,
    tokenOutIndex int,
    priceDiff float64,
) *big.Int {
    balanceIn := pool.Balances[tokenInIndex]

    low := big.NewInt(0)
    high := new(big.Int).Div(balanceIn, big.NewInt(2))

    maxProfit := big.NewInt(0)
    optimalAmount := big.NewInt(0)

    for iterations := 0; iterations < 100; iterations++ {
        mid := new(big.Int).Add(low, high)
        mid.Div(mid, big.NewInt(2))

        if mid.Cmp(low) == 0 {
            break
        }

        profit := d.calculateExpectedProfit(pool, tokenInIndex, tokenOutIndex, mid)

        if profit.Cmp(maxProfit) > 0 {
            maxProfit.Set(profit)
            optimalAmount.Set(mid)
        }

        higherMid := new(big.Int).Add(mid, new(big.Int).Sub(high, mid).Div(new(big.Int).Sub(high, mid), big.NewInt(2)))
        higherProfit := d.calculateExpectedProfit(pool, tokenInIndex, tokenOutIndex, higherMid)

        if higherProfit.Cmp(profit) > 0 {
            low.Set(mid)
        } else {
            high.Set(mid)
        }
    }

    return optimalAmount
}

func (d *BalancerArbitrageDetector) calculateExpectedProfit(
    pool *BalancerWeightedPool,
    tokenInIndex int,
    tokenOutIndex int,
    amountIn *big.Int,
) *big.Int {
    amountOut := d.math.CalcOutGivenIn(
        pool.Balances[tokenInIndex],
        pool.Weights[tokenInIndex],
        pool.Balances[tokenOutIndex],
        pool.Weights[tokenOutIndex],
        amountIn,
        pool.SwapFee,
    )

    return new(big.Int).Sub(amountOut, amountIn)
}
```

### B.5 Maverick 套利策略

```go
// MaverickArbitrageStrategy Maverick 套利策略
type MaverickArbitrageStrategy struct {
    math *MaverickMath
}

// FindArbitrageOpportunity 查找套利机会
func (s *MaverickArbitrageStrategy) FindArbitrageOpportunity(
    pool *MaverickPool,
    marketPrice *big.Float,
    minProfitBps int,
) *MaverickArbitrageOpportunity {
    maverickPrice := s.math.TickToPrice(pool.ActiveTick)

    marketPriceF, _ := marketPrice.Float64()
    maverickPriceF, _ := maverickPrice.Float64()

    priceDiff := (marketPriceF - maverickPriceF) / maverickPriceF

    if priceDiff > float64(minProfitBps)/10000 {
        // 市场价格高: B -> A
        return s.optimizeArbitrage(pool, false, marketPrice)
    } else if priceDiff < -float64(minProfitBps)/10000 {
        // 市场价格低: A -> B
        return s.optimizeArbitrage(pool, true, marketPrice)
    }

    return nil
}

// MaverickArbitrageOpportunity Maverick 套利机会
type MaverickArbitrageOpportunity struct {
    Pool           *MaverickPool
    TokenAIn       bool
    AmountIn       *big.Int
    ExpectedOut    *big.Int
    EffectivePrice *big.Float
    MarketPrice    *big.Float
    ProfitBps      int
}

func (s *MaverickArbitrageStrategy) optimizeArbitrage(
    pool *MaverickPool,
    tokenAIn bool,
    marketPrice *big.Float,
) *MaverickArbitrageOpportunity {
    var maxReserve *big.Int
    if tokenAIn {
        maxReserve = s.getTotalReserveB(pool)
    } else {
        maxReserve = s.getTotalReserveA(pool)
    }

    low := big.NewInt(1e15)
    high := new(big.Int).Div(maxReserve, big.NewInt(2))

    bestOpp := &MaverickArbitrageOpportunity{}
    bestProfit := big.NewInt(0)

    for iterations := 0; iterations < 50; iterations++ {
        mid := new(big.Int).Add(low, high)
        mid.Div(mid, big.NewInt(2))

        if mid.Cmp(low) == 0 {
            break
        }

        amountOut := s.math.CalcSwapAcrossBins(pool, mid, tokenAIn)
        profit := s.calculateProfit(mid, amountOut, tokenAIn, marketPrice)

        if profit.Cmp(bestProfit) > 0 {
            bestProfit.Set(profit)

            effectivePrice := new(big.Float).Quo(
                new(big.Float).SetInt(amountOut),
                new(big.Float).SetInt(mid),
            )

            marketPriceF, _ := marketPrice.Float64()
            effectivePriceF, _ := effectivePrice.Float64()

            profitBps := int((effectivePriceF - marketPriceF) / marketPriceF * 10000)

            bestOpp = &MaverickArbitrageOpportunity{
                Pool:           pool,
                TokenAIn:       tokenAIn,
                AmountIn:       new(big.Int).Set(mid),
                ExpectedOut:    amountOut,
                EffectivePrice: effectivePrice,
                MarketPrice:    marketPrice,
                ProfitBps:      profitBps,
            }
        }

        higherMid := new(big.Int).Add(mid, new(big.Int).Div(new(big.Int).Sub(high, mid), big.NewInt(2)))
        higherOut := s.math.CalcSwapAcrossBins(pool, higherMid, tokenAIn)
        higherProfit := s.calculateProfit(higherMid, higherOut, tokenAIn, marketPrice)

        if higherProfit.Cmp(profit) > 0 {
            low.Set(mid)
        } else {
            high.Set(mid)
        }
    }

    if bestProfit.Sign() > 0 {
        return bestOpp
    }

    return nil
}

func (s *MaverickArbitrageStrategy) calculateProfit(
    amountIn *big.Int,
    amountOut *big.Int,
    tokenAIn bool,
    marketPrice *big.Float,
) *big.Int {
    amountInF := new(big.Float).SetInt(amountIn)
    amountOutF := new(big.Float).SetInt(amountOut)

    if tokenAIn {
        marketPriceF, _ := marketPrice.Float64()
        valueInA := new(big.Float).Quo(amountOutF, big.NewFloat(marketPriceF))
        profit := new(big.Float).Sub(valueInA, amountInF)
        profitInt, _ := profit.Int(nil)
        return profitInt
    } else {
        marketPriceF, _ := marketPrice.Float64()
        valueInB := new(big.Float).Mul(amountOutF, big.NewFloat(marketPriceF))
        profit := new(big.Float).Sub(valueInB, amountInF)
        profitInt, _ := profit.Int(nil)
        return profitInt
    }
}

func (s *MaverickArbitrageStrategy) getTotalReserveA(pool *MaverickPool) *big.Int {
    total := big.NewInt(0)
    for _, bin := range pool.Bins {
        total.Add(total, bin.ReserveA)
    }
    return total
}

func (s *MaverickArbitrageStrategy) getTotalReserveB(pool *MaverickPool) *big.Int {
    total := big.NewInt(0)
    for _, bin := range pool.Bins {
        total.Add(total, bin.ReserveB)
    }
    return total
}
```

### B.6 总结

Balancer 和 Maverick 代表了 DeFi 中两种创新的 AMM 设计：

1. **Balancer 加权池**: 公式 `V = ∏(Bi^Wi)` 允许非对称权重
2. **Balancer Composable Stable**: 结合 Curve StableSwap，适用于稳定资产
3. **Maverick Bin 模型**: 流动性可根据价格变动自动调整位置

关键数学要点：
- Balancer: `Ao = Bo × (1 - (Bi / (Bi + Ai))^(Wi/Wo))`
- Maverick: 单 bin 内 `dy = dx × p × (1 - fee)`，跨 bin 需累加