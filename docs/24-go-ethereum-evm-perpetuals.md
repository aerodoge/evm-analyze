# Go-Ethereum EVM 深入解析：永续合约套利

## 概述

链上永续合约协议（dYdX、GMX、Perpetual Protocol 等）提供了独特的套利机会，特别是资金费率套利。本文详细介绍永续合约的工作原理和套利策略。

## 1. 永续合约基础

### 1.1 核心机制

```
┌─────────────────────────────────────────────────────────────┐
│                    永续合约核心机制                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  价格机制                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  标记价格 (Mark Price) = 预言机价格 + 溢价基差        │   │
│  │  指数价格 (Index Price) = 外部预言机聚合价格          │   │
│  │  资金费率 = (标记价格 - 指数价格) / 指数价格 × 系数    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  资金费率                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  正资金费率: 多头支付空头 (期货溢价)                  │   │
│  │  负资金费率: 空头支付多头 (期货折价)                  │   │
│  │  结算周期: 每 1/8 小时 (dYdX) 或实时 (GMX)           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  保证金模式                                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  隔离保证金: 每个仓位独立保证金                       │   │
│  │  全仓保证金: 所有仓位共享保证金                       │   │
│  │  初始保证金: 开仓所需保证金                           │   │
│  │  维持保证金: 防止清算的最低保证金                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心数据结构

```go
package perpetual

import (
	"context"
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

// PerpetualPosition 永续合约仓位
type PerpetualPosition struct {
	Market           string       `json:"market"`
	Side             PositionSide `json:"side"`
	Size             *big.Int     `json:"size"`
	EntryPrice       *big.Int     `json:"entryPrice"`
	MarkPrice        *big.Int     `json:"markPrice"`
	Leverage         uint64       `json:"leverage"`
	Margin           *big.Int     `json:"margin"`
	UnrealizedPnL    *big.Int     `json:"unrealizedPnL"`
	RealizedPnL      *big.Int     `json:"realizedPnL"`
	LiquidationPrice *big.Int     `json:"liquidationPrice"`
	FundingPayment   *big.Int     `json:"fundingPayment"`
	OpenTimestamp    time.Time    `json:"openTimestamp"`
}

type PositionSide int

const (
	Long PositionSide = iota
	Short
)

// FundingRate 资金费率
type FundingRate struct {
	Market          string        `json:"market"`
	Rate            *big.Int      `json:"rate"` // 18 decimals
	NextFundingTime time.Time     `json:"nextFundingTime"`
	Interval        time.Duration `json:"interval"`
	AnnualizedRate  float64       `json:"annualizedRate"`
}

// PerpetualExchange 永续合约交易所接口
type PerpetualExchange interface {
	// 市场信息
	GetMarkets(ctx context.Context) ([]Market, error)
	GetMarkPrice(ctx context.Context, market string) (*big.Int, error)
	GetIndexPrice(ctx context.Context, market string) (*big.Int, error)
	GetFundingRate(ctx context.Context, market string) (*FundingRate, error)

	// 交易
	OpenPosition(ctx context.Context, params *OpenPositionParams) (*PerpetualPosition, error)
	ClosePosition(ctx context.Context, params *ClosePositionParams) error
	UpdateMargin(ctx context.Context, market string, amount *big.Int, isAdd bool) error

	// 查询
	GetPositions(ctx context.Context, account common.Address) ([]*PerpetualPosition, error)
	GetAccountInfo(ctx context.Context, account common.Address) (*AccountInfo, error)
}

type Market struct {
	Symbol          string
	BaseAsset       string
	QuoteAsset      string
	ContractAddress common.Address
	MinOrderSize    *big.Int
	MaxLeverage     uint64
	MakerFee        *big.Int // basis points
	TakerFee        *big.Int
	FundingInterval time.Duration
}

type OpenPositionParams struct {
	Market     string
	Side       PositionSide
	Size       *big.Int
	Leverage   uint64
	MarginType MarginType
	TakeProfit *big.Int // optional
	StopLoss   *big.Int // optional
}

type ClosePositionParams struct {
	Market string
	Size   *big.Int // nil = close all
}

type MarginType int

const (
	CrossMargin MarginType = iota
	IsolatedMargin
)

type AccountInfo struct {
	Account            common.Address
	TotalCollateral    *big.Int
	FreeCollateral     *big.Int
	TotalPositionValue *big.Int
	UnrealizedPnL      *big.Int
	MarginRatio        *big.Int
}
```

## 2. GMX 集成

### 2.1 GMX V2 客户端

```go
package gmx

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// GMX V2 合约地址 (Arbitrum)
var (
	Router         = common.HexToAddress("0x7C68C7866A64FA2160F78EEaE12217FFbf871fa8")
	ExchangeRouter = common.HexToAddress("0x7C68C7866A64FA2160F78EEaE12217FFbf871fa8")
	PositionRouter = common.HexToAddress("0xb87a436B93fFE9D75c5cFA7bAcFff96430b09868")
	Reader         = common.HexToAddress("0xf60becbba223EEA9495Da3f606753867eC10d139")
	DataStore      = common.HexToAddress("0xFD70de6b91282D8017aA4E741e9Ae325CAb992d8")

	// 市场代币
	GMXWETH = common.HexToAddress("0x70d95587d40A2caf56bd97485aB3Eec10Bee6336") // ETH/USD
	GMXBTC  = common.HexToAddress("0x47c031236e19d024b42f8AE6780E44A573170703") // BTC/USD
)

// GMXClient GMX 客户端
type GMXClient struct {
	ethClient *ethclient.Client
	readerABI abi.ABI
	routerABI abi.ABI
	chainID   *big.Int
}

// NewGMXClient 创建客户端
func NewGMXClient(ethClient *ethclient.Client) (*GMXClient, error) {
	readerABI, err := abi.JSON(strings.NewReader(GMXReaderABI))
	if err != nil {
		return nil, err
	}

	routerABI, err := abi.JSON(strings.NewReader(GMXExchangeRouterABI))
	if err != nil {
		return nil, err
	}

	chainID, _ := ethClient.ChainID(context.Background())

	return &GMXClient{
		ethClient: ethClient,
		readerABI: readerABI,
		routerABI: routerABI,
		chainID:   chainID,
	}, nil
}

// GetMarketInfo 获取市场信息
func (c *GMXClient) GetMarketInfo(ctx context.Context, marketToken common.Address) (*GMXMarketInfo, error) {
	// 调用 Reader.getMarket
	callData, _ := c.readerABI.Pack("getMarket", DataStore, marketToken)

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &Reader,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	// 解析结果
	var market GMXMarketInfo
	if err := c.readerABI.UnpackIntoInterface(&market, "getMarket", result); err != nil {
		return nil, err
	}

	return &market, nil
}

// GetPositionInfo 获取仓位信息
func (c *GMXClient) GetPositionInfo(ctx context.Context, account common.Address, marketToken common.Address, isLong bool) (*GMXPosition, error) {
	// 构建 position key
	positionKey := c.getPositionKey(account, marketToken, isLong)

	// 调用 Reader.getPositionInfo
	callData, _ := c.readerABI.Pack("getPositionInfo", DataStore, positionKey)

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &Reader,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	var position GMXPosition
	if err := c.readerABI.UnpackIntoInterface(&position, "getPositionInfo", result); err != nil {
		return nil, err
	}

	return &position, nil
}

// GetFundingFees 获取资金费率
func (c *GMXClient) GetFundingFees(ctx context.Context, marketToken common.Address) (*GMXFundingFees, error) {
	// GMX V2 使用 borrowing fees 而不是传统资金费率
	callData, _ := c.readerABI.Pack("getMarketTokenPrice", DataStore, marketToken)

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &Reader,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	// 解析 funding 相关数据
	var fees GMXFundingFees
	// ... 解析逻辑

	return &fees, nil
}

// CreateIncreasePosition 创建/增加仓位
func (c *GMXClient) CreateIncreasePosition(ctx context.Context, params *GMXIncreaseParams) (common.Hash, error) {
	// 构建 multicall 数据
	// 1. sendWnt (如果使用 ETH)
	// 2. sendTokens (发送抵押品)
	// 3. createOrder

	var calls [][]byte

	// 如果使用 ETH 作为抵押品
	if params.CollateralToken == common.Address {
	}
	{
		sendWntData, _ := c.routerABI.Pack("sendWnt", PositionRouter, params.CollateralAmount)
		calls = append(calls, sendWntData)
	}

	// 创建订单
	createOrderParams := GMXCreateOrderParams{
		Receiver:                     params.Account,
		CallbackContract:             common.Address{},
		UiFeeReceiver:                common.Address{},
		Market:                       params.Market,
		InitialCollateralToken:       params.CollateralToken,
		SwapPath:                     []common.Address{},
		SizeDeltaUsd:                 params.SizeUSD,
		InitialCollateralDeltaAmount: params.CollateralAmount,
		TriggerPrice:                 big.NewInt(0),
		AcceptablePrice:              params.AcceptablePrice,
		ExecutionFee:                 params.ExecutionFee,
		CallbackGasLimit:             big.NewInt(0),
		MinOutputAmount:              big.NewInt(0),
		OrderType:                    OrderTypeMarketIncrease,
		DecreasePositionSwapType:     0,
		IsLong:                       params.IsLong,
		ShouldUnwrapNativeToken:      false,
		Referrer:                     common.Address{},
	}

	createOrderData, _ := c.routerABI.Pack("createOrder", createOrderParams)
	calls = append(calls, createOrderData)

	// multicall
	multicallData, _ := c.routerABI.Pack("multicall", calls)

	// 发送交易
	tx, err := c.sendTransaction(ctx, ExchangeRouter, multicallData, params.ExecutionFee)
	if err != nil {
		return common.Hash{}, err
	}

	return tx.Hash(), nil
}

// CreateDecreasePosition 减少/关闭仓位
func (c *GMXClient) CreateDecreasePosition(ctx context.Context, params *GMXDecreaseParams) (common.Hash, error) {
	createOrderParams := GMXCreateOrderParams{
		Receiver:               params.Account,
		Market:                 params.Market,
		InitialCollateralToken: params.CollateralToken,
		SizeDeltaUsd:           params.SizeUSD,
		AcceptablePrice:        params.AcceptablePrice,
		ExecutionFee:           params.ExecutionFee,
		OrderType:              OrderTypeMarketDecrease,
		IsLong:                 params.IsLong,
	}

	createOrderData, _ := c.routerABI.Pack("createOrder", createOrderParams)

	tx, err := c.sendTransaction(ctx, ExchangeRouter, createOrderData, params.ExecutionFee)
	if err != nil {
		return common.Hash{}, err
	}

	return tx.Hash(), nil
}

func (c *GMXClient) getPositionKey(account common.Address, market common.Address, isLong bool) [32]byte {
	// keccak256(abi.encode(account, market, collateralToken, isLong))
	data := make([]byte, 128)
	copy(data[12:32], account.Bytes())
	copy(data[44:64], market.Bytes())
	// collateralToken 和 isLong...
	return common.BytesToHash(crypto.Keccak256(data))
}

type GMXMarketInfo struct {
	MarketToken common.Address
	IndexToken  common.Address
	LongToken   common.Address
	ShortToken  common.Address
}

type GMXPosition struct {
	Account                                 common.Address
	Market                                  common.Address
	CollateralToken                         common.Address
	IsLong                                  bool
	SizeInUsd                               *big.Int
	SizeInTokens                            *big.Int
	CollateralAmount                        *big.Int
	BorrowingFactor                         *big.Int
	FundingFeeAmountPerSize                 *big.Int
	LongTokenClaimableFundingAmountPerSize  *big.Int
	ShortTokenClaimableFundingAmountPerSize *big.Int
}

type GMXFundingFees struct {
	LongFundingFeeAmountPerSize  *big.Int
	ShortFundingFeeAmountPerSize *big.Int
	ClaimableLongTokenAmount     *big.Int
	ClaimableShortTokenAmount    *big.Int
}

type GMXIncreaseParams struct {
	Account          common.Address
	Market           common.Address
	CollateralToken  common.Address
	CollateralAmount *big.Int
	SizeUSD          *big.Int
	IsLong           bool
	AcceptablePrice  *big.Int
	ExecutionFee     *big.Int
}

type GMXDecreaseParams struct {
	Account         common.Address
	Market          common.Address
	CollateralToken common.Address
	SizeUSD         *big.Int
	IsLong          bool
	AcceptablePrice *big.Int
	ExecutionFee    *big.Int
}

const (
	OrderTypeMarketSwap = iota
	OrderTypeLimitSwap
	OrderTypeMarketIncrease
	OrderTypeLimitIncrease
	OrderTypeMarketDecrease
	OrderTypeLimitDecrease
	OrderTypeStopLossDecrease
	OrderTypeLiquidation
)

type GMXCreateOrderParams struct {
	Receiver                     common.Address
	CallbackContract             common.Address
	UiFeeReceiver                common.Address
	Market                       common.Address
	InitialCollateralToken       common.Address
	SwapPath                     []common.Address
	SizeDeltaUsd                 *big.Int
	InitialCollateralDeltaAmount *big.Int
	TriggerPrice                 *big.Int
	AcceptablePrice              *big.Int
	ExecutionFee                 *big.Int
	CallbackGasLimit             *big.Int
	MinOutputAmount              *big.Int
	OrderType                    uint8
	DecreasePositionSwapType     uint8
	IsLong                       bool
	ShouldUnwrapNativeToken      bool
	Referrer                     common.Address
}
```

## 3. 资金费率套利

### 3.1 基差套利策略

```go
package arbitrage

import (
	"context"
	"math/big"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

// FundingRateArbitrage 资金费率套利
type FundingRateArbitrage struct {
	// 永续合约交易所
	exchanges map[string]PerpetualExchange

	// 现货交易所
	spotExchange SpotExchange

	// 配置
	config FundingArbConfig
}

type FundingArbConfig struct {
	MinFundingRateAPR  float64       // 最小年化资金费率
	MaxPositionUSD     float64       // 最大仓位
	MinHoldingPeriod   time.Duration // 最小持仓时间
	RebalanceThreshold float64       // 再平衡阈值
	SupportedMarkets   []string      // 支持的市场
}

// FundingOpportunity 资金费率套利机会
type FundingOpportunity struct {
	Exchange        string
	Market          string
	FundingRate     *big.Int
	AnnualizedRate  float64
	Side            PositionSide // 收取资金费率的方向
	IndexPrice      *big.Int
	MarkPrice       *big.Int
	Premium         float64 // 期货溢价/折价
	EstimatedProfit float64
}

// FindOpportunities 寻找资金费率套利机会
func (a *FundingRateArbitrage) FindOpportunities(ctx context.Context) ([]*FundingOpportunity, error) {
	var opportunities []*FundingOpportunity
	var mu sync.Mutex
	var wg sync.WaitGroup

	for name, exchange := range a.exchanges {
		wg.Add(1)
		go func(n string, ex PerpetualExchange) {
			defer wg.Done()

			for _, market := range a.config.SupportedMarkets {
				opp := a.evaluateMarket(ctx, n, ex, market)
				if opp != nil && opp.AnnualizedRate >= a.config.MinFundingRateAPR {
					mu.Lock()
					opportunities = append(opportunities, opp)
					mu.Unlock()
				}
			}
		}(name, exchange)
	}

	wg.Wait()
	return opportunities, nil
}

// evaluateMarket 评估单个市场
func (a *FundingRateArbitrage) evaluateMarket(ctx context.Context, exchange string, ex PerpetualExchange, market string) *FundingOpportunity {
	// 获取资金费率
	fundingRate, err := ex.GetFundingRate(ctx, market)
	if err != nil {
		return nil
	}

	// 获取价格
	markPrice, err := ex.GetMarkPrice(ctx, market)
	if err != nil {
		return nil
	}

	indexPrice, err := ex.GetIndexPrice(ctx, market)
	if err != nil {
		return nil
	}

	// 计算年化利率
	// APR = fundingRate * (365 * 24 * 60 / fundingInterval.Minutes())
	intervalsPerYear := float64(365*24*60) / fundingRate.Interval.Minutes()
	rateFloat, _ := new(big.Float).SetInt(fundingRate.Rate).Float64()
	annualizedRate := rateFloat / 1e18 * intervalsPerYear

	// 确定方向
	var side PositionSide
	if fundingRate.Rate.Sign() > 0 {
		// 正资金费率，做空收取
		side = Short
	} else {
		// 负资金费率，做多收取
		side = Long
		annualizedRate = -annualizedRate
	}

	// 计算溢价
	premium, _ := new(big.Float).Quo(
		new(big.Float).Sub(
			new(big.Float).SetInt(markPrice),
			new(big.Float).SetInt(indexPrice),
		),
		new(big.Float).SetInt(indexPrice),
	).Float64()

	return &FundingOpportunity{
		Exchange:        exchange,
		Market:          market,
		FundingRate:     fundingRate.Rate,
		AnnualizedRate:  annualizedRate * 100, // 转为百分比
		Side:            side,
		IndexPrice:      indexPrice,
		MarkPrice:       markPrice,
		Premium:         premium * 100,
		EstimatedProfit: a.estimateProfit(annualizedRate),
	}
}

// ExecuteHedgedPosition 执行对冲仓位
func (a *FundingRateArbitrage) ExecuteHedgedPosition(ctx context.Context, opp *FundingOpportunity, amount *big.Int) error {
	exchange := a.exchanges[opp.Exchange]

	// 1. 在永续合约开仓（收取资金费率方向）
	perpPosition, err := exchange.OpenPosition(ctx, &OpenPositionParams{
		Market:   opp.Market,
		Side:     opp.Side,
		Size:     amount,
		Leverage: 1, // 1x 杠杆
	})
	if err != nil {
		return fmt.Errorf("open perp position failed: %w", err)
	}

	// 2. 在现货市场开反向仓位对冲
	var spotSide string
	if opp.Side == Long {
		// 永续做多，现货做空（卖出）
		spotSide = "sell"
	} else {
		// 永续做空，现货做多（买入）
		spotSide = "buy"
	}

	// 将市场名转换为代币
	baseToken := a.marketToToken(opp.Market)

	err = a.spotExchange.ExecuteOrder(ctx, &SpotOrderParams{
		Token:  baseToken,
		Side:   spotSide,
		Amount: amount,
	})
	if err != nil {
		// 需要平掉永续仓位
		exchange.ClosePosition(ctx, &ClosePositionParams{Market: opp.Market})
		return fmt.Errorf("open spot position failed: %w", err)
	}

	// 记录仓位
	log.Printf("Hedged position opened: %s %s perp %s + spot %s, size: %s",
		opp.Exchange, opp.Market, opp.Side, spotSide, amount.String())

	return nil
}

// MonitorAndRebalance 监控并再平衡
func (a *FundingRateArbitrage) MonitorAndRebalance(ctx context.Context) error {
	ticker := time.NewTicker(1 * time.Minute)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
			// 检查所有对冲仓位
			for name, ex := range a.exchanges {
				positions, err := ex.GetPositions(ctx, a.account)
				if err != nil {
					continue
				}

				for _, pos := range positions {
					// 检查是否需要再平衡
					if a.needsRebalance(pos) {
						a.rebalancePosition(ctx, name, ex, pos)
					}

					// 检查是否应该关闭（资金费率反转）
					opp := a.evaluateMarket(ctx, name, ex, pos.Market)
					if opp == nil || opp.Side != pos.Side {
						a.closeHedgedPosition(ctx, name, ex, pos)
					}
				}
			}
		}
	}
}

func (a *FundingRateArbitrage) needsRebalance(pos *PerpetualPosition) bool {
	// 检查 delta 是否超出阈值
	// 对冲仓位应该保持 delta neutral
	return false // 简化
}

func (a *FundingRateArbitrage) estimateProfit(annualizedRate float64) float64 {
	// 考虑交易费用、借贷成本等
	tradingCost := 0.001 * 2 // 开仓 + 平仓
	borrowingCost := 0.05    // 年化借贷成本

	netAPR := annualizedRate - tradingCost*365 - borrowingCost
	return netAPR * a.config.MaxPositionUSD
}
```

### 3.2 跨交易所资金费率套利

```go
package arbitrage

import (
	"context"
	"math/big"
)

// CrossExchangeFundingArbitrage 跨交易所资金费率套利
type CrossExchangeFundingArbitrage struct {
	exchanges map[string]PerpetualExchange
	config    CrossFundingConfig
}

type CrossFundingConfig struct {
	MinRateDiffAPR float64 // 最小费率差
	MaxPositionUSD float64
}

// CrossFundingOpportunity 跨交易所资金费率机会
type CrossFundingOpportunity struct {
	LongExchange     string
	ShortExchange    string
	Market           string
	LongFundingRate  float64
	ShortFundingRate float64
	NetAPR           float64
	OptimalSize      *big.Int
}

// FindCrossExchangeOpportunities 寻找跨交易所机会
func (a *CrossExchangeFundingArbitrage) FindCrossExchangeOpportunities(ctx context.Context, market string) (*CrossFundingOpportunity, error) {
	// 获取所有交易所的资金费率
	rates := make(map[string]float64)

	for name, ex := range a.exchanges {
		fundingRate, err := ex.GetFundingRate(ctx, market)
		if err != nil {
			continue
		}

		rate, _ := new(big.Float).SetInt(fundingRate.Rate).Float64()
		annualized := rate / 1e18 * 365 * 24 * 3 // 假设 8 小时一次
		rates[name] = annualized
	}

	if len(rates) < 2 {
		return nil, nil
	}

	// 找出最高和最低费率
	var maxExchange, minExchange string
	var maxRate, minRate float64 = -1e10, 1e10

	for ex, rate := range rates {
		if rate > maxRate {
			maxRate = rate
			maxExchange = ex
		}
		if rate < minRate {
			minRate = rate
			minExchange = ex
		}
	}

	// 计算净收益
	// 在高费率交易所做空（收取），在低费率交易所做多（收取负费率或支付较少）
	netAPR := maxRate - minRate

	if netAPR < a.config.MinRateDiffAPR {
		return nil, nil
	}

	return &CrossFundingOpportunity{
		ShortExchange:    maxExchange, // 高费率做空
		LongExchange:     minExchange, // 低费率做多
		Market:           market,
		ShortFundingRate: maxRate * 100,
		LongFundingRate:  minRate * 100,
		NetAPR:           netAPR * 100,
	}, nil
}

// Execute 执行跨交易所套利
func (a *CrossExchangeFundingArbitrage) Execute(ctx context.Context, opp *CrossFundingOpportunity, amount *big.Int) error {
	shortEx := a.exchanges[opp.ShortExchange]
	longEx := a.exchanges[opp.LongExchange]

	// 同时开仓
	errChan := make(chan error, 2)

	// 在高费率交易所做空
	go func() {
		_, err := shortEx.OpenPosition(ctx, &OpenPositionParams{
			Market:   opp.Market,
			Side:     Short,
			Size:     amount,
			Leverage: 1,
		})
		errChan <- err
	}()

	// 在低费率交易所做多
	go func() {
		_, err := longEx.OpenPosition(ctx, &OpenPositionParams{
			Market:   opp.Market,
			Side:     Long,
			Size:     amount,
			Leverage: 1,
		})
		errChan <- err
	}()

	// 检查结果
	for i := 0; i < 2; i++ {
		if err := <-errChan; err != nil {
			// 需要处理部分失败的情况
			return err
		}
	}

	return nil
}
```

## 4. 期现套利

### 4.1 期货-现货基差套利

```go
package arbitrage

import (
	"context"
	"math/big"
	"time"
)

// BasisArbitrage 期现基差套利
type BasisArbitrage struct {
	perpExchange PerpetualExchange
	spotExchange SpotExchange
	priceOracle  PriceOracle

	config BasisArbConfig
}

type BasisArbConfig struct {
	MinBasisPct      float64 // 最小基差百分比
	MaxPositionUSD   float64
	EntryThreshold   float64 // 入场阈值
	ExitThreshold    float64 // 出场阈值
	MaxHoldingPeriod time.Duration
}

// BasisOpportunity 基差套利机会
type BasisOpportunity struct {
	Market          string
	SpotPrice       *big.Int
	FuturesPrice    *big.Int
	BasisPct        float64 // 基差百分比
	Direction       string  // "cash_and_carry" or "reverse"
	AnnualizedYield float64
	FundingRate     float64
}

// FindBasisOpportunities 寻找基差套利机会
func (a *BasisArbitrage) FindBasisOpportunities(ctx context.Context) ([]*BasisOpportunity, error) {
	var opportunities []*BasisOpportunity

	markets, _ := a.perpExchange.GetMarkets(ctx)

	for _, market := range markets {
		opp := a.evaluateBasis(ctx, market.Symbol)
		if opp != nil && abs(opp.BasisPct) >= a.config.MinBasisPct {
			opportunities = append(opportunities, opp)
		}
	}

	return opportunities, nil
}

// evaluateBasis 评估基差
func (a *BasisArbitrage) evaluateBasis(ctx context.Context, market string) *BasisOpportunity {
	// 获取期货价格
	futuresPrice, err := a.perpExchange.GetMarkPrice(ctx, market)
	if err != nil {
		return nil
	}

	// 获取现货价格
	spotPrice, err := a.spotExchange.GetPrice(ctx, a.marketToToken(market))
	if err != nil {
		return nil
	}

	// 计算基差
	basis := new(big.Float).Sub(
		new(big.Float).SetInt(futuresPrice),
		new(big.Float).SetInt(spotPrice),
	)
	basisPct, _ := new(big.Float).Quo(basis, new(big.Float).SetInt(spotPrice)).Float64()

	// 获取资金费率
	fundingRate, _ := a.perpExchange.GetFundingRate(ctx, market)
	fundingRateFloat, _ := new(big.Float).SetInt(fundingRate.Rate).Float64()

	var direction string
	var yield float64

	if basisPct > a.config.EntryThreshold {
		// 期货溢价：Cash and Carry
		// 买入现货，做空期货
		direction = "cash_and_carry"
		yield = basisPct + fundingRateFloat/1e18 // 收取资金费率
	} else if basisPct < -a.config.EntryThreshold {
		// 期货折价：Reverse Cash and Carry
		// 卖出现货（或借入），做多期货
		direction = "reverse"
		yield = -basisPct - fundingRateFloat/1e18 // 支付资金费率
	} else {
		return nil
	}

	// 年化收益
	annualizedYield := yield * 365 * 24 / a.config.MaxHoldingPeriod.Hours()

	return &BasisOpportunity{
		Market:          market,
		SpotPrice:       spotPrice,
		FuturesPrice:    futuresPrice,
		BasisPct:        basisPct * 100,
		Direction:       direction,
		AnnualizedYield: annualizedYield * 100,
		FundingRate:     fundingRateFloat / 1e18 * 100,
	}
}

// ExecuteCashAndCarry 执行正向套利
func (a *BasisArbitrage) ExecuteCashAndCarry(ctx context.Context, opp *BasisOpportunity, amount *big.Int) error {
	// 1. 买入现货
	token := a.marketToToken(opp.Market)
	err := a.spotExchange.Buy(ctx, token, amount)
	if err != nil {
		return fmt.Errorf("spot buy failed: %w", err)
	}

	// 2. 做空等量期货
	_, err = a.perpExchange.OpenPosition(ctx, &OpenPositionParams{
		Market:   opp.Market,
		Side:     Short,
		Size:     amount,
		Leverage: 1,
	})
	if err != nil {
		// 卖出现货回滚
		a.spotExchange.Sell(ctx, token, amount)
		return fmt.Errorf("perp short failed: %w", err)
	}

	return nil
}

// CloseCashAndCarry 平仓正向套利
func (a *BasisArbitrage) CloseCashAndCarry(ctx context.Context, market string, amount *big.Int) error {
	// 1. 平掉期货空头
	err := a.perpExchange.ClosePosition(ctx, &ClosePositionParams{
		Market: market,
		Size:   amount,
	})
	if err != nil {
		return err
	}

	// 2. 卖出现货
	token := a.marketToToken(market)
	return a.spotExchange.Sell(ctx, token, amount)
}
```

## 5. 完整套利机器人

```go
package main

import (
	"context"
	"log"
	"time"
)

func main() {
	ctx := context.Background()

	// 初始化客户端
	gmx, _ := gmx.NewGMXClient(arbitrumClient)

	// 创建资金费率套利器
	fundingArb := &FundingRateArbitrage{
		exchanges: map[string]PerpetualExchange{
			"GMX": gmx,
		},
		spotExchange: spotAggregator,
		config: FundingArbConfig{
			MinFundingRateAPR:  10.0,  // 最小 10% APR
			MaxPositionUSD:     50000, // 最大 5 万美元
			MinHoldingPeriod:   24 * time.Hour,
			RebalanceThreshold: 0.05, // 5% 再平衡
		},
	}

	// 运行
	go fundingArb.Monitor(ctx)

	// 定期检查机会
	ticker := time.NewTicker(5 * time.Minute)
	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			opportunities, err := fundingArb.FindOpportunities(ctx)
			if err != nil {
				log.Printf("Error finding opportunities: %v", err)
				continue
			}

			for _, opp := range opportunities {
				log.Printf("Found opportunity: %s %s, APR: %.2f%%",
					opp.Exchange, opp.Market, opp.AnnualizedRate)

				if opp.AnnualizedRate > 20 { // 年化超过 20% 则执行
					amount := big.NewInt(1e18) // 1 ETH 等值
					fundingArb.ExecuteHedgedPosition(ctx, opp, amount)
				}
			}
		}
	}
}
```

## 6. 总结

### 6.1 永续合约协议对比

| 协议            | 链                   | 特点           | 资金费率   |
|---------------|---------------------|--------------|--------|
| **GMX**       | Arbitrum, Avalanche | 零滑点、GLP 池    | 借贷费用   |
| **dYdX**      | dYdX Chain (Cosmos) | 订单簿、高性能      | 每 8 小时 |
| **Perpetual** | Optimism            | vAMM 模型      | 每 1 小时 |
| **Kwenta**    | Optimism            | Synthetix 衍生 | 实时     |

### 6.2 套利策略总结

| 策略     | 风险 | 收益来源 | 复杂度 |
|--------|----|------|-----|
| 资金费率对冲 | 低  | 资金费率 | 中   |
| 跨交易所费率 | 中  | 费率差  | 高   |
| 期现基差   | 低  | 基差收敛 | 中   |
| 清算套利   | 高  | 清算折扣 | 高   |

### 6.3 风险管理

```
┌─────────────────────────────────────────────────────────────┐
│                     风险管理要点                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Delta 中性                                              │
│     └─ 保持对冲仓位 delta = 0                               │
│                                                             │
│  2. 资金管理                                                 │
│     └─ 单个策略不超过总资金 20%                             │
│                                                             │
│  3. 清算风险                                                 │
│     └─ 使用低杠杆 (1-2x)                                    │
│     └─ 监控维持保证金率                                     │
│                                                             │
│  4. 协议风险                                                 │
│     └─ 分散在多个协议                                       │
│     └─ 监控协议 TVL 和健康状况                              │
│                                                             │
│  5. 滑点风险                                                 │
│     └─ 大仓位分批建立                                       │
│     └─ 避免低流动性市场                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

永续合约为套利者提供了独特的机会，特别是资金费率套利。通过合理的对冲策略和风险管理，可以获得相对稳定的收益。

---

## 附录 A：dYdX V4 集成

dYdX V4 是基于 Cosmos SDK 构建的独立应用链，提供去中心化订单簿和高性能交易。

### A.1 dYdX V4 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    dYdX V4 架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  dYdX Chain (Cosmos)                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  Validators │  │ Order Book  │  │  Indexer    │  │   │
│  │  │  (PoS)      │  │  (Off-chain)│  │  (REST/WS)  │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Cross-Chain Bridging                    │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  IBC        │  │  Noble USDC │  │  Squid      │  │   │
│  │  │  (Cosmos)   │  │  Bridge     │  │  Router     │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  特点:                                                      │
│  • 完全去中心化订单簿                                        │
│  • 无 Gas 交易（Validator 代付）                            │
│  • 每秒处理数千笔订单                                        │
│  • 跨链资产桥接（IBC、CCTP）                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### A.2 dYdX V4 客户端

```go
package dydxv4

import (
	"context"
	"crypto/ed25519"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"time"

	"github.com/gorilla/websocket"
)

// dYdX V4 网络配置
const (
	MainnetIndexerREST = "https://indexer.dydx.trade/v4"
	MainnetIndexerWS   = "wss://indexer.dydx.trade/v4/ws"
	MainnetChainID     = "dydx-mainnet-1"
	TestnetIndexerREST = "https://indexer.v4testnet.dydx.exchange/v4"
	TestnetIndexerWS   = "wss://indexer.v4testnet.dydx.exchange/v4/ws"
)

// DYDXClient dYdX V4 客户端
type DYDXClient struct {
	indexerURL string
	wsURL      string
	chainID    string
	httpClient *http.Client
	wsConn     *websocket.Conn

	// 签名相关
	privateKey ed25519.PrivateKey
	subaccount *Subaccount
}

// Subaccount dYdX 子账户
type Subaccount struct {
	Address string `json:"address"`
	Number  int    `json:"subaccountNumber"`
}

// Market 市场信息
type Market struct {
	Ticker                    string `json:"ticker"`
	Status                    string `json:"status"`
	BaseAsset                 string `json:"baseAsset"`
	QuoteAsset                string `json:"quoteAsset"`
	InitialMarginFraction     string `json:"initialMarginFraction"`
	MaintenanceMarginFraction string `json:"maintenanceMarginFraction"`
	StepSize                  string `json:"stepSize"`
	TickSize                  string `json:"tickSize"`
	StepBaseQuantums          int64  `json:"stepBaseQuantums"`
	SubticksPerTick           int64  `json:"subticksPerTick"`
}

// Position 仓位信息
type Position struct {
	Market        string `json:"market"`
	Status        string `json:"status"`
	Side          string `json:"side"`
	Size          string `json:"size"`
	MaxSize       string `json:"maxSize"`
	EntryPrice    string `json:"entryPrice"`
	UnrealizedPnl string `json:"unrealizedPnl"`
	RealizedPnl   string `json:"realizedPnl"`
	CreatedAt     string `json:"createdAt"`
	SumOpen       string `json:"sumOpen"`
	SumClose      string `json:"sumClose"`
	NetFunding    string `json:"netFunding"`
}

// FundingPayment 资金费率
type FundingPayment struct {
	Market       string `json:"market"`
	Payment      string `json:"payment"`
	Rate         string `json:"rate"`
	PositionSize string `json:"positionSize"`
	EffectiveAt  string `json:"effectiveAt"`
}

// Order 订单
type Order struct {
	ID              string `json:"id"`
	SubaccountID    string `json:"subaccountId"`
	ClientID        string `json:"clientId"`
	Market          string `json:"ticker"`
	Side            string `json:"side"`
	Size            string `json:"size"`
	Price           string `json:"price"`
	Type            string `json:"type"`
	Status          string `json:"status"`
	TimeInForce     string `json:"timeInForce"`
	PostOnly        bool   `json:"postOnly"`
	ReduceOnly      bool   `json:"reduceOnly"`
	GoodTilBlock    int64  `json:"goodTilBlock"`
	CreatedAtHeight int64  `json:"createdAtHeight"`
}

// NewDYDXClient 创建客户端
func NewDYDXClient(isMainnet bool, privateKey ed25519.PrivateKey) *DYDXClient {
	indexerURL := TestnetIndexerREST
	wsURL := TestnetIndexerWS
	chainID := "dydx-testnet-4"

	if isMainnet {
		indexerURL = MainnetIndexerREST
		wsURL = MainnetIndexerWS
		chainID = MainnetChainID
	}

	return &DYDXClient{
		indexerURL: indexerURL,
		wsURL:      wsURL,
		chainID:    chainID,
		httpClient: &http.Client{Timeout: 30 * time.Second},
		privateKey: privateKey,
	}
}

// GetMarkets 获取所有市场
func (c *DYDXClient) GetMarkets(ctx context.Context) (map[string]*Market, error) {
	resp, err := c.httpClient.Get(c.indexerURL + "/perpetualMarkets")
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result struct {
		Markets map[string]*Market `json:"markets"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	return result.Markets, nil
}

// GetPositions 获取仓位
func (c *DYDXClient) GetPositions(ctx context.Context, address string, subaccountNumber int) ([]*Position, error) {
	url := fmt.Sprintf("%s/addresses/%s/subaccountNumber/%d/perpetualPositions",
		c.indexerURL, address, subaccountNumber)

	resp, err := c.httpClient.Get(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result struct {
		Positions []*Position `json:"positions"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	return result.Positions, nil
}

// GetFundingPayments 获取资金费率支付记录
func (c *DYDXClient) GetFundingPayments(ctx context.Context, address string, subaccountNumber int) ([]*FundingPayment, error) {
	url := fmt.Sprintf("%s/addresses/%s/subaccountNumber/%d/historicalFunding",
		c.indexerURL, address, subaccountNumber)

	resp, err := c.httpClient.Get(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result struct {
		HistoricalFunding []*FundingPayment `json:"historicalFunding"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	return result.HistoricalFunding, nil
}

// GetCurrentFundingRates 获取当前资金费率
func (c *DYDXClient) GetCurrentFundingRates(ctx context.Context) (map[string]string, error) {
	markets, err := c.GetMarkets(ctx)
	if err != nil {
		return nil, err
	}

	rates := make(map[string]string)
	for ticker, market := range markets {
		// 从市场数据中提取资金费率
		// 实际实现需要调用额外的 API
		_ = market
		rates[ticker] = "0" // 占位
	}

	return rates, nil
}

// GetOrderbook 获取订单簿
func (c *DYDXClient) GetOrderbook(ctx context.Context, market string) (*Orderbook, error) {
	url := fmt.Sprintf("%s/orderbooks/perpetualMarket/%s", c.indexerURL, market)

	resp, err := c.httpClient.Get(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var orderbook Orderbook
	if err := json.NewDecoder(resp.Body).Decode(&orderbook); err != nil {
		return nil, err
	}

	return &orderbook, nil
}

// Orderbook 订单簿
type Orderbook struct {
	Bids []OrderbookEntry `json:"bids"`
	Asks []OrderbookEntry `json:"asks"`
}

type OrderbookEntry struct {
	Price string `json:"price"`
	Size  string `json:"size"`
}

// PlaceOrder 下单（Short-Term Order）
func (c *DYDXClient) PlaceOrder(ctx context.Context, params *PlaceOrderParams) (*Order, error) {
	// dYdX V4 使用 Cosmos 交易格式
	// 需要构建并签名 MsgPlaceOrder 交易

	order := &OrderToPlace{
		SubaccountID:   params.SubaccountID,
		ClientID:       params.ClientID,
		MarketID:       c.getMarketID(params.Market),
		Side:           params.Side,
		Quantums:       params.Quantums,
		Subticks:       params.Subticks,
		TimeInForce:    params.TimeInForce,
		ReduceOnly:     params.ReduceOnly,
		GoodTilBlock:   params.GoodTilBlock,
		ClientMetadata: 0,
	}

	// 签名并广播
	txHash, err := c.signAndBroadcast(ctx, order)
	if err != nil {
		return nil, err
	}

	return &Order{
		ID:       txHash,
		ClientID: fmt.Sprintf("%d", params.ClientID),
		Market:   params.Market,
		Side:     params.Side.String(),
		Status:   "PENDING",
	}, nil
}

// PlaceOrderParams 下单参数
type PlaceOrderParams struct {
	SubaccountID SubaccountID
	ClientID     uint32
	Market       string
	Side         OrderSide
	Quantums     uint64 // 基础数量（最小单位）
	Subticks     uint64 // 价格 subticks
	TimeInForce  TimeInForce
	ReduceOnly   bool
	GoodTilBlock uint32 // 订单有效区块
}

type SubaccountID struct {
	Owner  string
	Number uint32
}

type OrderSide int

const (
	OrderSideBuy OrderSide = iota + 1
	OrderSideSell
)

func (s OrderSide) String() string {
	if s == OrderSideBuy {
		return "BUY"
	}
	return "SELL"
}

type TimeInForce int

const (
	TimeInForceUnspecified TimeInForce = iota
	TimeInForceIOC                     // Immediate or Cancel
	TimeInForcePostOnly                // Post Only
	TimeInForceFOK                     // Fill or Kill
)

type OrderToPlace struct {
	SubaccountID   SubaccountID
	ClientID       uint32
	MarketID       uint32
	Side           OrderSide
	Quantums       uint64
	Subticks       uint64
	TimeInForce    TimeInForce
	ReduceOnly     bool
	GoodTilBlock   uint32
	ClientMetadata uint32
}

func (c *DYDXClient) getMarketID(market string) uint32 {
	// 市场 ID 映射
	marketIDs := map[string]uint32{
		"BTC-USD":  0,
		"ETH-USD":  1,
		"SOL-USD":  2,
		"AVAX-USD": 3,
		"DOGE-USD": 4,
		// ... 更多市场
	}
	return marketIDs[market]
}

func (c *DYDXClient) signAndBroadcast(ctx context.Context, order *OrderToPlace) (string, error) {
	// 实际实现需要:
	// 1. 构建 Cosmos SDK MsgPlaceOrder
	// 2. 使用 ed25519 私钥签名
	// 3. 广播到 dYdX 链
	return "", fmt.Errorf("not implemented")
}

// CancelOrder 取消订单
func (c *DYDXClient) CancelOrder(ctx context.Context, params *CancelOrderParams) error {
	// 构建取消订单消息
	// MsgCancelOrder
	return nil
}

type CancelOrderParams struct {
	SubaccountID SubaccountID
	ClientID     uint32
	MarketID     uint32
	GoodTilBlock uint32
}
```

### A.3 dYdX V4 WebSocket 订阅

```go
package dydxv4

import (
	"context"
	"encoding/json"
	"fmt"
	"sync"

	"github.com/gorilla/websocket"
)

// WSClient WebSocket 客户端
type WSClient struct {
	conn     *websocket.Conn
	url      string
	mu       sync.RWMutex
	handlers map[string][]chan json.RawMessage
	done     chan struct{}
}

// NewWSClient 创建 WebSocket 客户端
func NewWSClient(url string) *WSClient {
	return &WSClient{
		url:      url,
		handlers: make(map[string][]chan json.RawMessage),
		done:     make(chan struct{}),
	}
}

// Connect 连接
func (c *WSClient) Connect(ctx context.Context) error {
	conn, _, err := websocket.DefaultDialer.DialContext(ctx, c.url, nil)
	if err != nil {
		return err
	}
	c.conn = conn

	// 启动消息处理
	go c.readMessages()

	return nil
}

// SubscribeOrderbook 订阅订单簿
func (c *WSClient) SubscribeOrderbook(market string) (<-chan *OrderbookUpdate, error) {
	msg := map[string]interface{}{
		"type":    "subscribe",
		"channel": "v4_orderbook",
		"id":      market,
	}

	if err := c.conn.WriteJSON(msg); err != nil {
		return nil, err
	}

	ch := make(chan *OrderbookUpdate, 100)
	c.addHandler("v4_orderbook/"+market, ch)

	return c.wrapOrderbookChannel(ch), nil
}

// SubscribeTrades 订阅成交
func (c *WSClient) SubscribeTrades(market string) (<-chan *Trade, error) {
	msg := map[string]interface{}{
		"type":    "subscribe",
		"channel": "v4_trades",
		"id":      market,
	}

	if err := c.conn.WriteJSON(msg); err != nil {
		return nil, err
	}

	ch := make(chan *Trade, 100)
	rawCh := make(chan json.RawMessage, 100)
	c.addHandler("v4_trades/"+market, rawCh)

	go func() {
		for raw := range rawCh {
			var trade Trade
			if err := json.Unmarshal(raw, &trade); err == nil {
				ch <- &trade
			}
		}
		close(ch)
	}()

	return ch, nil
}

// SubscribeSubaccount 订阅子账户更新
func (c *WSClient) SubscribeSubaccount(address string, subaccountNumber int) (<-chan *SubaccountUpdate, error) {
	msg := map[string]interface{}{
		"type":    "subscribe",
		"channel": "v4_subaccounts",
		"id":      fmt.Sprintf("%s/%d", address, subaccountNumber),
	}

	if err := c.conn.WriteJSON(msg); err != nil {
		return nil, err
	}

	ch := make(chan *SubaccountUpdate, 100)
	rawCh := make(chan json.RawMessage, 100)
	c.addHandler(fmt.Sprintf("v4_subaccounts/%s/%d", address, subaccountNumber), rawCh)

	go func() {
		for raw := range rawCh {
			var update SubaccountUpdate
			if err := json.Unmarshal(raw, &update); err == nil {
				ch <- &update
			}
		}
		close(ch)
	}()

	return ch, nil
}

func (c *WSClient) addHandler(key string, ch interface{}) {
	c.mu.Lock()
	defer c.mu.Unlock()

	rawCh, ok := ch.(chan json.RawMessage)
	if !ok {
		return
	}
	c.handlers[key] = append(c.handlers[key], rawCh)
}

func (c *WSClient) readMessages() {
	for {
		select {
		case <-c.done:
			return
		default:
			_, message, err := c.conn.ReadMessage()
			if err != nil {
				return
			}

			var msg WSMessage
			if err := json.Unmarshal(message, &msg); err != nil {
				continue
			}

			// 路由消息
			key := msg.Channel + "/" + msg.ID
			c.mu.RLock()
			handlers := c.handlers[key]
			c.mu.RUnlock()

			for _, h := range handlers {
				select {
				case h <- msg.Contents:
				default:
				}
			}
		}
	}
}

func (c *WSClient) wrapOrderbookChannel(rawCh chan json.RawMessage) <-chan *OrderbookUpdate {
	ch := make(chan *OrderbookUpdate, 100)
	go func() {
		for raw := range rawCh {
			var update OrderbookUpdate
			if err := json.Unmarshal(raw, &update); err == nil {
				ch <- &update
			}
		}
		close(ch)
	}()
	return ch
}

// Close 关闭连接
func (c *WSClient) Close() error {
	close(c.done)
	if c.conn != nil {
		return c.conn.Close()
	}
	return nil
}

type WSMessage struct {
	Type     string          `json:"type"`
	Channel  string          `json:"channel"`
	ID       string          `json:"id"`
	Contents json.RawMessage `json:"contents"`
}

type OrderbookUpdate struct {
	Bids []OrderbookEntry `json:"bids"`
	Asks []OrderbookEntry `json:"asks"`
}

type Trade struct {
	ID        string `json:"id"`
	Side      string `json:"side"`
	Size      string `json:"size"`
	Price     string `json:"price"`
	CreatedAt string `json:"createdAt"`
}

type SubaccountUpdate struct {
	Equity                 string               `json:"equity"`
	FreeCollateral         string               `json:"freeCollateral"`
	OpenPerpetualPositions map[string]*Position `json:"openPerpetualPositions"`
	Orders                 []*Order             `json:"orders"`
}
```

---

## 附录 B：Synthetix Perps V3 集成

Synthetix Perps V3 是 Synthetix 生态的第三代永续合约系统，部署在 Optimism 和 Base 上。

### B.1 Synthetix V3 架构

```
┌─────────────────────────────────────────────────────────────┐
│                 Synthetix Perps V3 架构                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Core System                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  snxUSD     │  │  Account    │  │  Perps      │  │   │
│  │  │  (Stablecoin)│  │  Module    │  │  Market     │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Price Feeds                        │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  Pyth       │  │  Chainlink  │  │  Internal   │  │   │
│  │  │  Network    │  │  CCIP       │  │  Oracle     │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  特点:                                                      │
│  • 模块化架构（Synthetix V3 Core）                          │
│  • 跨保证金（Cross-margin）支持                             │
│  • Pyth Network 低延迟价格                                  │
│  • 多抵押品类型（snxUSD, sUSD）                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### B.2 Synthetix Perps V3 客户端

```go
package synthetixv3

import (
	"context"
	"math/big"
	"strings"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Synthetix Perps V3 合约地址 (Base)
var (
	BasePerpsMarketProxy = common.HexToAddress("0x0A2AF931eFFd34b81ebcc57E3d3c9B1E1dE1C9Ce")
	BaseAccountProxy     = common.HexToAddress("0xcb68b813210aFa0373F076239Ad4803f8809e8cf")
	BaseSpotMarketProxy  = common.HexToAddress("0x18141523403e2595D31b22604AcB8Fc06a4CaA61")
	BaseSNXUSD           = common.HexToAddress("0x09d51516F38980035153a554c26Df3C6f51a23C3")

	// Optimism
	OptimismPerpsMarketProxy = common.HexToAddress("0xd8aA8f3be2fbd0a5C6a1f2a3C7f6E71AC82D0d22")
)

// SynthetixPerpsClient Synthetix V3 永续客户端
type SynthetixPerpsClient struct {
	ethClient       *ethclient.Client
	perpsMarketABI  abi.ABI
	accountABI      abi.ABI
	perpsMarketAddr common.Address
	accountAddr     common.Address
	chainID         *big.Int
}

// NewSynthetixPerpsClient 创建客户端
func NewSynthetixPerpsClient(ethClient *ethclient.Client, isBase bool) (*SynthetixPerpsClient, error) {
	perpsMarketABI, err := abi.JSON(strings.NewReader(PerpsMarketProxyABI))
	if err != nil {
		return nil, err
	}

	accountABI, err := abi.JSON(strings.NewReader(AccountProxyABI))
	if err != nil {
		return nil, err
	}

	perpsAddr := OptimismPerpsMarketProxy
	accountAddr := common.Address{}
	if isBase {
		perpsAddr = BasePerpsMarketProxy
		accountAddr = BaseAccountProxy
	}

	chainID, _ := ethClient.ChainID(context.Background())

	return &SynthetixPerpsClient{
		ethClient:       ethClient,
		perpsMarketABI:  perpsMarketABI,
		accountABI:      accountABI,
		perpsMarketAddr: perpsAddr,
		accountAddr:     accountAddr,
		chainID:         chainID,
	}, nil
}

// CreateAccount 创建交易账户
func (c *SynthetixPerpsClient) CreateAccount(ctx context.Context, auth *bind.TransactOpts) (uint128 *big.Int, error) {
	// createAccount() returns accountId
	data, err := c.perpsMarketABI.Pack("createAccount")
	if err != nil {
		return nil, err
	}

	// 发送交易
	tx, err := c.sendTransaction(ctx, auth, c.perpsMarketAddr, data, nil)
	if err != nil {
		return nil, err
	}

	// 等待确认并解析事件获取 accountId
	receipt, err := bind.WaitMined(ctx, c.ethClient, tx)
	if err != nil {
		return nil, err
	}

	// 解析 AccountCreated 事件
	for _, log := range receipt.Logs {
		if log.Topics[0].Hex() == c.perpsMarketABI.Events["AccountCreated"].ID.Hex() {
			accountId := new(big.Int).SetBytes(log.Topics[1].Bytes())
			return accountId, nil
		}
	}

	return nil, fmt.Errorf("account creation event not found")
}

// GetAvailableMargin 获取可用保证金
func (c *SynthetixPerpsClient) GetAvailableMargin(ctx context.Context, accountId *big.Int) (*big.Int, error) {
	data, err := c.perpsMarketABI.Pack("getAvailableMargin", accountId)
	if err != nil {
		return nil, err
	}

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &c.perpsMarketAddr,
		Data: data,
	}, nil)
	if err != nil {
		return nil, err
	}

	var margin *big.Int
	if err := c.perpsMarketABI.UnpackIntoInterface(&margin, "getAvailableMargin", result); err != nil {
		return nil, err
	}

	return margin, nil
}

// GetOpenPosition 获取持仓
func (c *SynthetixPerpsClient) GetOpenPosition(ctx context.Context, accountId *big.Int, marketId *big.Int) (*OpenPosition, error) {
	data, err := c.perpsMarketABI.Pack("getOpenPosition", accountId, marketId)
	if err != nil {
		return nil, err
	}

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &c.perpsMarketAddr,
		Data: data,
	}, nil)
	if err != nil {
		return nil, err
	}

	// 返回值: (totalPnl, accruedFunding, positionSize, owedInterest)
	values, err := c.perpsMarketABI.Unpack("getOpenPosition", result)
	if err != nil {
		return nil, err
	}

	return &OpenPosition{
		TotalPnl:       values[0].(*big.Int),
		AccruedFunding: values[1].(*big.Int),
		PositionSize:   values[2].(*big.Int),
		OwedInterest:   values[3].(*big.Int),
	}, nil
}

// OpenPosition 持仓信息
type OpenPosition struct {
	TotalPnl       *big.Int
	AccruedFunding *big.Int
	PositionSize   *big.Int
	OwedInterest   *big.Int
}

// ModifyCollateral 修改保证金
func (c *SynthetixPerpsClient) ModifyCollateral(
	ctx context.Context,
	auth *bind.TransactOpts,
	accountId *big.Int,
	collateralId *big.Int, // snxUSD 的 synthMarketId
	amount *big.Int,       // 正数存入，负数取出
) error {
	data, err := c.perpsMarketABI.Pack("modifyCollateral", accountId, collateralId, amount)
	if err != nil {
		return err
	}

	_, err = c.sendTransaction(ctx, auth, c.perpsMarketAddr, data, nil)
	return err
}

// CommitOrder 提交订单（异步执行）
func (c *SynthetixPerpsClient) CommitOrder(
	ctx context.Context,
	auth *bind.TransactOpts,
	params *CommitOrderParams,
) (*AsyncOrder, error) {
	// Synthetix V3 使用异步订单系统
	// 1. commitOrder - 提交订单意图
	// 2. settleOrder - 在下一个区块由 keeper 执行

	orderCommitment := OrderCommitmentRequest{
		MarketId:             params.MarketId,
		AccountId:            params.AccountId,
		SizeDelta:            params.SizeDelta,
		SettlementStrategyId: params.SettlementStrategyId,
		AcceptablePrice:      params.AcceptablePrice,
		TrackingCode:         params.TrackingCode,
		Referrer:             params.Referrer,
	}

	data, err := c.perpsMarketABI.Pack("commitOrder", orderCommitment)
	if err != nil {
		return nil, err
	}

	tx, err := c.sendTransaction(ctx, auth, c.perpsMarketAddr, data, nil)
	if err != nil {
		return nil, err
	}

	receipt, err := bind.WaitMined(ctx, c.ethClient, tx)
	if err != nil {
		return nil, err
	}

	// 解析 OrderCommitted 事件
	for _, log := range receipt.Logs {
		if log.Topics[0].Hex() == c.perpsMarketABI.Events["OrderCommitted"].ID.Hex() {
			// 解析订单信息
			return &AsyncOrder{
				MarketId:        params.MarketId,
				AccountId:       params.AccountId,
				SizeDelta:       params.SizeDelta,
				CommitmentTime:  new(big.Int).SetUint64(receipt.BlockNumber),
				AcceptablePrice: params.AcceptablePrice,
			}, nil
		}
	}

	return nil, fmt.Errorf("order commitment event not found")
}

// CommitOrderParams 提交订单参数
type CommitOrderParams struct {
	MarketId             *big.Int       // 市场 ID
	AccountId            *big.Int       // 账户 ID
	SizeDelta            *big.Int       // 仓位变化（正=做多，负=做空）
	SettlementStrategyId *big.Int       // 结算策略 ID（通常为 0）
	AcceptablePrice      *big.Int       // 可接受价格
	TrackingCode         [32]byte       // 追踪码（用于推荐）
	Referrer             common.Address // 推荐人
}

type OrderCommitmentRequest struct {
	MarketId             *big.Int
	AccountId            *big.Int
	SizeDelta            *big.Int
	SettlementStrategyId *big.Int
	AcceptablePrice      *big.Int
	TrackingCode         [32]byte
	Referrer             common.Address
}

// AsyncOrder 异步订单
type AsyncOrder struct {
	MarketId        *big.Int
	AccountId       *big.Int
	SizeDelta       *big.Int
	CommitmentTime  *big.Int
	AcceptablePrice *big.Int
}

// SettleOrder 结算订单（需要 Pyth 价格更新）
func (c *SynthetixPerpsClient) SettleOrder(
	ctx context.Context,
	auth *bind.TransactOpts,
	accountId *big.Int,
	priceUpdateData [][]byte, // Pyth 价格更新数据
) error {
	// 需要传入 Pyth 价格更新
	data, err := c.perpsMarketABI.Pack("settleOrder", accountId, priceUpdateData)
	if err != nil {
		return err
	}

	// 需要支付 Pyth 更新费用
	value := big.NewInt(int64(len(priceUpdateData))) // 每个更新约 1 wei

	_, err = c.sendTransaction(ctx, auth, c.perpsMarketAddr, data, value)
	return err
}

// GetFundingParameters 获取资金费率参数
func (c *SynthetixPerpsClient) GetFundingParameters(ctx context.Context, marketId *big.Int) (*FundingParameters, error) {
	data, err := c.perpsMarketABI.Pack("getFundingParameters", marketId)
	if err != nil {
		return nil, err
	}

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &c.perpsMarketAddr,
		Data: data,
	}, nil)
	if err != nil {
		return nil, err
	}

	// 返回值: (skewScale, maxFundingVelocity)
	values, err := c.perpsMarketABI.Unpack("getFundingParameters", result)
	if err != nil {
		return nil, err
	}

	return &FundingParameters{
		SkewScale:          values[0].(*big.Int),
		MaxFundingVelocity: values[1].(*big.Int),
	}, nil
}

// FundingParameters 资金费率参数
type FundingParameters struct {
	SkewScale          *big.Int // 偏斜规模
	MaxFundingVelocity *big.Int // 最大资金费率速度
}

// CurrentFundingRate 获取当前资金费率
func (c *SynthetixPerpsClient) CurrentFundingRate(ctx context.Context, marketId *big.Int) (*big.Int, error) {
	data, err := c.perpsMarketABI.Pack("currentFundingRate", marketId)
	if err != nil {
		return nil, err
	}

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &c.perpsMarketAddr,
		Data: data,
	}, nil)
	if err != nil {
		return nil, err
	}

	var rate *big.Int
	if err := c.perpsMarketABI.UnpackIntoInterface(&rate, "currentFundingRate", result); err != nil {
		return nil, err
	}

	return rate, nil
}

// CurrentFundingVelocity 获取资金费率速度
func (c *SynthetixPerpsClient) CurrentFundingVelocity(ctx context.Context, marketId *big.Int) (*big.Int, error) {
	data, err := c.perpsMarketABI.Pack("currentFundingVelocity", marketId)
	if err != nil {
		return nil, err
	}

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &c.perpsMarketAddr,
		Data: data,
	}, nil)
	if err != nil {
		return nil, err
	}

	var velocity *big.Int
	if err := c.perpsMarketABI.UnpackIntoInterface(&velocity, "currentFundingVelocity", result); err != nil {
		return nil, err
	}

	return velocity, nil
}

func (c *SynthetixPerpsClient) sendTransaction(
	ctx context.Context,
	auth *bind.TransactOpts,
	to common.Address,
	data []byte,
	value *big.Int,
) (*types.Transaction, error) {
	nonce, err := c.ethClient.PendingNonceAt(ctx, auth.From)
	if err != nil {
		return nil, err
	}

	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	msg := ethereum.CallMsg{
		From:  auth.From,
		To:    &to,
		Data:  data,
		Value: value,
	}

	gasLimit, err := c.ethClient.EstimateGas(ctx, msg)
	if err != nil {
		return nil, err
	}

	tx := types.NewTransaction(nonce, to, value, gasLimit, gasPrice, data)

	signedTx, err := auth.Signer(auth.From, tx)
	if err != nil {
		return nil, err
	}

	if err := c.ethClient.SendTransaction(ctx, signedTx); err != nil {
		return nil, err
	}

	return signedTx, nil
}

// Synthetix V3 市场 ID
var SynthetixMarketIDs = map[string]*big.Int{
	"ETH":  big.NewInt(100),
	"BTC":  big.NewInt(200),
	"SOL":  big.NewInt(300),
	"WIF":  big.NewInt(400),
	"PEPE": big.NewInt(500),
	"DOGE": big.NewInt(600),
	// ... 更多市场
}
```

### B.3 Pyth 价格集成

```go
package synthetixv3

import (
	"context"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"time"
)

// Pyth Network 配置
const (
	PythHermesURL = "https://hermes.pyth.network"
)

// PythClient Pyth 价格客户端
type PythClient struct {
	hermesURL  string
	httpClient *http.Client
}

// NewPythClient 创建 Pyth 客户端
func NewPythClient() *PythClient {
	return &PythClient{
		hermesURL:  PythHermesURL,
		httpClient: &http.Client{Timeout: 10 * time.Second},
	}
}

// Pyth Price Feed IDs (Crypto)
var PythPriceFeedIDs = map[string]string{
	"ETH/USD":  "0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace",
	"BTC/USD":  "0xe62df6c8b4a85fe1a67db44dc12de5db330f7ac66b72dc658afedf0f4a415b43",
	"SOL/USD":  "0xef0d8b6fda2ceba41da15d4095d1da392a0d2f8ed0c6c7bc0f4cfac8c280b56d",
	"AVAX/USD": "0x93da3352f9f1d105fdfe4971cfa80e9dd777bfc5d0f683ebb6e1294b92137bb7",
	"DOGE/USD": "0xdcef817388c8ff40ff73fb25c73faab14fd84bfc4d6f5eb0fdb050a2fdc8d01d",
}

// GetPriceUpdateData 获取价格更新数据（用于链上验证）
func (c *PythClient) GetPriceUpdateData(ctx context.Context, feedIDs []string) ([][]byte, error) {
	// 构建请求 URL
	url := fmt.Sprintf("%s/api/latest_vaas?", c.hermesURL)
	for i, id := range feedIDs {
		if i > 0 {
			url += "&"
		}
		url += "ids[]=" + id
	}

	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		return nil, err
	}

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var vaas []string
	if err := json.NewDecoder(resp.Body).Decode(&vaas); err != nil {
		return nil, err
	}

	// 解码 VAA 数据
	updateData := make([][]byte, len(vaas))
	for i, vaa := range vaas {
		data, err := hex.DecodeString(vaa)
		if err != nil {
			return nil, err
		}
		updateData[i] = data
	}

	return updateData, nil
}

// GetLatestPrice 获取最新价格
func (c *PythClient) GetLatestPrice(ctx context.Context, feedID string) (*PythPrice, error) {
	url := fmt.Sprintf("%s/api/latest_price_feeds?ids[]=%s", c.hermesURL, feedID)

	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		return nil, err
	}

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var prices []PythPriceFeed
	if err := json.NewDecoder(resp.Body).Decode(&prices); err != nil {
		return nil, err
	}

	if len(prices) == 0 {
		return nil, fmt.Errorf("no price found")
	}

	return &prices[0].Price, nil
}

// PythPriceFeed Pyth 价格 Feed
type PythPriceFeed struct {
	ID    string    `json:"id"`
	Price PythPrice `json:"price"`
	EMA   PythPrice `json:"ema_price"`
}

// PythPrice Pyth 价格
type PythPrice struct {
	Price       string `json:"price"`
	Conf        string `json:"conf"`
	Expo        int    `json:"expo"`
	PublishTime int64  `json:"publish_time"`
}

// ToBigInt 转换为 big.Int（考虑指数）
func (p *PythPrice) ToBigInt() *big.Int {
	price, _ := new(big.Int).SetString(p.Price, 10)

	// 调整指数到 18 位小数
	targetDecimals := 18
	currentDecimals := -p.Expo

	if targetDecimals > currentDecimals {
		// 需要乘以 10^diff
		diff := targetDecimals - currentDecimals
		multiplier := new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(diff)), nil)
		price.Mul(price, multiplier)
	} else if targetDecimals < currentDecimals {
		// 需要除以 10^diff
		diff := currentDecimals - targetDecimals
		divisor := new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(diff)), nil)
		price.Div(price, divisor)
	}

	return price
}

// SynthetixPythIntegration Synthetix + Pyth 集成
type SynthetixPythIntegration struct {
	synthetix *SynthetixPerpsClient
	pyth      *PythClient
}

// NewSynthetixPythIntegration 创建集成客户端
func NewSynthetixPythIntegration(synthetix *SynthetixPerpsClient) *SynthetixPythIntegration {
	return &SynthetixPythIntegration{
		synthetix: synthetix,
		pyth:      NewPythClient(),
	}
}

// CommitAndSettle 提交并结算订单
func (s *SynthetixPythIntegration) CommitAndSettle(
	ctx context.Context,
	auth *bind.TransactOpts,
	params *CommitOrderParams,
	marketSymbol string, // 如 "ETH/USD"
) error {
	// 1. 提交订单
	_, err := s.synthetix.CommitOrder(ctx, auth, params)
	if err != nil {
		return fmt.Errorf("commit order failed: %w", err)
	}

	// 2. 等待下一个区块
	time.Sleep(2 * time.Second)

	// 3. 获取 Pyth 价格更新
	feedID := PythPriceFeedIDs[marketSymbol]
	priceUpdateData, err := s.pyth.GetPriceUpdateData(ctx, []string{feedID})
	if err != nil {
		return fmt.Errorf("get price update failed: %w", err)
	}

	// 4. 结算订单
	if err := s.synthetix.SettleOrder(ctx, auth, params.AccountId, priceUpdateData); err != nil {
		return fmt.Errorf("settle order failed: %w", err)
	}

	return nil
}
```

---

## 附录 C：Hyperliquid 集成

Hyperliquid 是高性能 L1 永续合约交易所，采用订单簿模式和自定义共识。

### C.1 Hyperliquid 架构

```
┌─────────────────────────────────────────────────────────────┐
│                   Hyperliquid 架构                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Hyperliquid L1                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  HyperBFT   │  │  Order Book │  │  Clearance  │  │   │
│  │  │  Consensus  │  │  Engine     │  │  System     │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    API Layer                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  REST API   │  │  WebSocket  │  │  EVM Bridge │  │   │
│  │  │  (Info/Exch)│  │  (Realtime) │  │  (Arbitrum) │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  特点:                                                      │
│  • 200ms 区块时间                                           │
│  • 100k+ TPS 吞吐量                                         │
│  • 完全链上订单簿                                            │
│  • 原生跨保证金                                              │
│  • 无 Gas 费交易（通过 L1 代币质押）                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### C.2 Hyperliquid 客户端

```go
package hyperliquid

import (
	"bytes"
	"context"
	"crypto/ecdsa"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// Hyperliquid API 配置
const (
	MainnetInfoURL     = "https://api.hyperliquid.xyz/info"
	MainnetExchangeURL = "https://api.hyperliquid.xyz/exchange"
	TestnetInfoURL     = "https://api.hyperliquid-testnet.xyz/info"
	TestnetExchangeURL = "https://api.hyperliquid-testnet.xyz/exchange"
)

// HyperliquidClient Hyperliquid 客户端
type HyperliquidClient struct {
	infoURL     string
	exchangeURL string
	httpClient  *http.Client
	privateKey  *ecdsa.PrivateKey
	address     common.Address
	isMainnet   bool
}

// NewHyperliquidClient 创建客户端
func NewHyperliquidClient(privateKey *ecdsa.PrivateKey, isMainnet bool) *HyperliquidClient {
	infoURL := TestnetInfoURL
	exchangeURL := TestnetExchangeURL

	if isMainnet {
		infoURL = MainnetInfoURL
		exchangeURL = MainnetExchangeURL
	}

	address := crypto.PubkeyToAddress(privateKey.PublicKey)

	return &HyperliquidClient{
		infoURL:     infoURL,
		exchangeURL: exchangeURL,
		httpClient:  &http.Client{Timeout: 30 * time.Second},
		privateKey:  privateKey,
		address:     address,
		isMainnet:   isMainnet,
	}
}

// Meta 获取所有市场元数据
func (c *HyperliquidClient) Meta(ctx context.Context) (*MetaResponse, error) {
	req := InfoRequest{Type: "meta"}

	var result MetaResponse
	if err := c.postInfo(ctx, req, &result); err != nil {
		return nil, err
	}

	return &result, nil
}

// MetaResponse 元数据响应
type MetaResponse struct {
	Universe []AssetInfo `json:"universe"`
}

// AssetInfo 资产信息
type AssetInfo struct {
	Name       string `json:"name"`
	SzDecimals int    `json:"szDecimals"`
	MaxLeverage int   `json:"maxLeverage"`
	OnlyIsolated bool `json:"onlyIsolated"`
}

// AllMids 获取所有市场中间价
func (c *HyperliquidClient) AllMids(ctx context.Context) (map[string]string, error) {
	req := InfoRequest{Type: "allMids"}

	var result map[string]string
	if err := c.postInfo(ctx, req, &result); err != nil {
		return nil, err
	}

	return result, nil
}

// UserState 获取用户状态
func (c *HyperliquidClient) UserState(ctx context.Context, address common.Address) (*UserStateResponse, error) {
	req := InfoRequest{
		Type: "clearinghouseState",
		User: address.Hex(),
	}

	var result UserStateResponse
	if err := c.postInfo(ctx, req, &result); err != nil {
		return nil, err
	}

	return &result, nil
}

// UserStateResponse 用户状态响应
type UserStateResponse struct {
	MarginSummary  MarginSummary      `json:"marginSummary"`
	CrossMarginSummary MarginSummary  `json:"crossMarginSummary"`
	AssetPositions []AssetPosition    `json:"assetPositions"`
}

// MarginSummary 保证金摘要
type MarginSummary struct {
	AccountValue    string `json:"accountValue"`
	TotalNtlPos     string `json:"totalNtlPos"`
	TotalRawUsd     string `json:"totalRawUsd"`
	TotalMarginUsed string `json:"totalMarginUsed"`
}

// AssetPosition 资产仓位
type AssetPosition struct {
	Position    PositionData `json:"position"`
	Type        string       `json:"type"`
}

// PositionData 仓位数据
type PositionData struct {
	Coin           string `json:"coin"`
	Szi            string `json:"szi"`           // 仓位大小（带符号）
	EntryPx        string `json:"entryPx"`       // 入场价格
	PositionValue  string `json:"positionValue"` // 仓位价值
	UnrealizedPnl  string `json:"unrealizedPnl"` // 未实现盈亏
	ReturnOnEquity string `json:"returnOnEquity"`
	Liquidation    *LiquidationInfo `json:"liquidationPx"`
	Leverage       LeverageInfo `json:"leverage"`
	CumFunding     CumFunding   `json:"cumFunding"`
}

type LiquidationInfo struct {
	Px string `json:"px"`
}

type LeverageInfo struct {
	Type   string `json:"type"`
	Value  int    `json:"value"`
	RawUsd string `json:"rawUsd"`
}

type CumFunding struct {
	AllTime  string `json:"allTime"`
	SinceOpen string `json:"sinceOpen"`
	SinceChange string `json:"sinceChange"`
}

// FundingHistory 获取资金费率历史
func (c *HyperliquidClient) FundingHistory(ctx context.Context, coin string, startTime, endTime int64) ([]FundingEntry, error) {
	req := InfoRequest{
		Type:      "fundingHistory",
		Coin:      coin,
		StartTime: startTime,
		EndTime:   endTime,
	}

	var result []FundingEntry
	if err := c.postInfo(ctx, req, &result); err != nil {
		return nil, err
	}

	return result, nil
}

// FundingEntry 资金费率条目
type FundingEntry struct {
	Coin        string `json:"coin"`
	FundingRate string `json:"fundingRate"`
	Premium     string `json:"premium"`
	Time        int64  `json:"time"`
}

// PlaceOrder 下单
func (c *HyperliquidClient) PlaceOrder(ctx context.Context, order *OrderRequest) (*OrderResponse, error) {
	// 构建订单操作
	action := ExchangeAction{
		Type: "order",
		Orders: []OrderWire{{
			Asset:      order.Asset,
			IsBuy:      order.IsBuy,
			LimitPx:    order.Price,
			Sz:         order.Size,
			ReduceOnly: order.ReduceOnly,
			OrderType:  order.OrderType,
			Cloid:      order.ClientOrderId,
		}},
		Grouping: "na",
	}

	return c.sendAction(ctx, action)
}

// OrderRequest 下单请求
type OrderRequest struct {
	Asset         int     // 资产索引
	IsBuy         bool    // 买入/卖出
	Price         string  // 限价
	Size          string  // 数量
	ReduceOnly    bool    // 只减仓
	OrderType     OrderType
	ClientOrderId string
}

// OrderType 订单类型
type OrderType struct {
	Limit *LimitOrderType `json:"limit,omitempty"`
	Trigger *TriggerOrderType `json:"trigger,omitempty"`
}

type LimitOrderType struct {
	Tif string `json:"tif"` // "Gtc", "Ioc", "Alo"
}

type TriggerOrderType struct {
	IsMarket  bool   `json:"isMarket"`
	TriggerPx string `json:"triggerPx"`
	TpSl      string `json:"tpsl"` // "tp" or "sl"
}

// CancelOrder 取消订单
func (c *HyperliquidClient) CancelOrder(ctx context.Context, asset int, oid int64) (*OrderResponse, error) {
	action := ExchangeAction{
		Type: "cancel",
		Cancels: []CancelWire{{
			Asset: asset,
			Oid:   oid,
		}},
	}

	return c.sendAction(ctx, action)
}

// CancelWire 取消订单请求
type CancelWire struct {
	Asset int   `json:"a"`
	Oid   int64 `json:"o"`
}

// UpdateLeverage 更新杠杆
func (c *HyperliquidClient) UpdateLeverage(ctx context.Context, asset int, isCross bool, leverage int) (*OrderResponse, error) {
	action := ExchangeAction{
		Type:     "updateLeverage",
		Asset:    asset,
		IsCross:  isCross,
		Leverage: leverage,
	}

	return c.sendAction(ctx, action)
}

// UpdateIsolatedMargin 更新隔离保证金
func (c *HyperliquidClient) UpdateIsolatedMargin(ctx context.Context, asset int, isBuy bool, ntli int64) (*OrderResponse, error) {
	action := ExchangeAction{
		Type:  "updateIsolatedMargin",
		Asset: asset,
		IsBuy: isBuy,
		Ntli:  ntli,
	}

	return c.sendAction(ctx, action)
}

// ExchangeAction 交易所操作
type ExchangeAction struct {
	Type     string       `json:"type"`
	Orders   []OrderWire  `json:"orders,omitempty"`
	Grouping string       `json:"grouping,omitempty"`
	Cancels  []CancelWire `json:"cancels,omitempty"`
	Asset    int          `json:"asset,omitempty"`
	IsCross  bool         `json:"isCross,omitempty"`
	Leverage int          `json:"leverage,omitempty"`
	IsBuy    bool         `json:"isBuy,omitempty"`
	Ntli     int64        `json:"ntli,omitempty"`
}

// OrderWire 订单线格式
type OrderWire struct {
	Asset      int       `json:"a"`
	IsBuy      bool      `json:"b"`
	LimitPx    string    `json:"p"`
	Sz         string    `json:"s"`
	ReduceOnly bool      `json:"r"`
	OrderType  OrderType `json:"t"`
	Cloid      string    `json:"c,omitempty"`
}

// OrderResponse 订单响应
type OrderResponse struct {
	Status   string         `json:"status"`
	Response *OrderResult   `json:"response,omitempty"`
}

type OrderResult struct {
	Type string `json:"type"`
	Data struct {
		Statuses []OrderStatus `json:"statuses"`
	} `json:"data"`
}

type OrderStatus struct {
	Resting *RestingOrder `json:"resting,omitempty"`
	Filled  *FilledOrder  `json:"filled,omitempty"`
	Error   string        `json:"error,omitempty"`
}

type RestingOrder struct {
	Oid int64 `json:"oid"`
}

type FilledOrder struct {
	TotalSz  string `json:"totalSz"`
	AvgPx    string `json:"avgPx"`
	Oid      int64  `json:"oid"`
}

// InfoRequest 信息请求
type InfoRequest struct {
	Type      string `json:"type"`
	User      string `json:"user,omitempty"`
	Coin      string `json:"coin,omitempty"`
	StartTime int64  `json:"startTime,omitempty"`
	EndTime   int64  `json:"endTime,omitempty"`
}

func (c *HyperliquidClient) postInfo(ctx context.Context, req interface{}, result interface{}) error {
	body, err := json.Marshal(req)
	if err != nil {
		return err
	}

	httpReq, err := http.NewRequestWithContext(ctx, "POST", c.infoURL, bytes.NewReader(body))
	if err != nil {
		return err
	}
	httpReq.Header.Set("Content-Type", "application/json")

	resp, err := c.httpClient.Do(httpReq)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	return json.NewDecoder(resp.Body).Decode(result)
}

func (c *HyperliquidClient) sendAction(ctx context.Context, action ExchangeAction) (*OrderResponse, error) {
	// 获取 nonce
	nonce := time.Now().UnixMilli()

	// 构建签名消息
	actionBytes, _ := json.Marshal(action)

	// Hyperliquid 使用 EIP-712 类型的签名
	signaturePayload := SignaturePayload{
		Action:         action,
		Nonce:          nonce,
		VaultAddress:   nil,
	}

	// 签名
	signature, err := c.signPayload(signaturePayload)
	if err != nil {
		return nil, err
	}

	// 构建请求
	request := ExchangeRequest{
		Action:       action,
		Nonce:        nonce,
		Signature:    signature,
		VaultAddress: nil,
	}

	body, _ := json.Marshal(request)

	httpReq, err := http.NewRequestWithContext(ctx, "POST", c.exchangeURL, bytes.NewReader(body))
	if err != nil {
		return nil, err
	}
	httpReq.Header.Set("Content-Type", "application/json")

	resp, err := c.httpClient.Do(httpReq)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result OrderResponse
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	return &result, nil
}

type SignaturePayload struct {
	Action       ExchangeAction `json:"action"`
	Nonce        int64          `json:"nonce"`
	VaultAddress *string        `json:"vaultAddress"`
}

type ExchangeRequest struct {
	Action       ExchangeAction `json:"action"`
	Nonce        int64          `json:"nonce"`
	Signature    Signature      `json:"signature"`
	VaultAddress *string        `json:"vaultAddress"`
}

type Signature struct {
	R string `json:"r"`
	S string `json:"s"`
	V int    `json:"v"`
}

func (c *HyperliquidClient) signPayload(payload SignaturePayload) (Signature, error) {
	// 构建 EIP-712 类型哈希
	// Hyperliquid 使用自定义的签名方案

	payloadBytes, _ := json.Marshal(payload)
	hash := crypto.Keccak256(payloadBytes)

	sig, err := crypto.Sign(hash, c.privateKey)
	if err != nil {
		return Signature{}, err
	}

	return Signature{
		R: "0x" + hex.EncodeToString(sig[:32]),
		S: "0x" + hex.EncodeToString(sig[32:64]),
		V: int(sig[64]) + 27,
	}, nil
}
```

### C.3 Hyperliquid WebSocket

```go
package hyperliquid

import (
	"context"
	"encoding/json"
	"fmt"
	"sync"

	"github.com/gorilla/websocket"
)

const (
	MainnetWSURL = "wss://api.hyperliquid.xyz/ws"
	TestnetWSURL = "wss://api.hyperliquid-testnet.xyz/ws"
)

// HyperliquidWS WebSocket 客户端
type HyperliquidWS struct {
	url      string
	conn     *websocket.Conn
	mu       sync.RWMutex
	handlers map[string]chan json.RawMessage
	done     chan struct{}
}

// NewHyperliquidWS 创建 WebSocket 客户端
func NewHyperliquidWS(isMainnet bool) *HyperliquidWS {
	url := TestnetWSURL
	if isMainnet {
		url = MainnetWSURL
	}

	return &HyperliquidWS{
		url:      url,
		handlers: make(map[string]chan json.RawMessage),
		done:     make(chan struct{}),
	}
}

// Connect 连接
func (ws *HyperliquidWS) Connect(ctx context.Context) error {
	conn, _, err := websocket.DefaultDialer.DialContext(ctx, ws.url, nil)
	if err != nil {
		return err
	}
	ws.conn = conn

	go ws.readMessages()

	return nil
}

// SubscribeAllMids 订阅所有中间价
func (ws *HyperliquidWS) SubscribeAllMids() (<-chan AllMidsUpdate, error) {
	msg := map[string]interface{}{
		"method":       "subscribe",
		"subscription": map[string]string{"type": "allMids"},
	}

	if err := ws.conn.WriteJSON(msg); err != nil {
		return nil, err
	}

	ch := make(chan AllMidsUpdate, 100)
	rawCh := make(chan json.RawMessage, 100)
	ws.addHandler("allMids", rawCh)

	go func() {
		for raw := range rawCh {
			var update AllMidsUpdate
			if err := json.Unmarshal(raw, &update); err == nil {
				ch <- update
			}
		}
		close(ch)
	}()

	return ch, nil
}

// AllMidsUpdate 中间价更新
type AllMidsUpdate struct {
	Mids map[string]string `json:"mids"`
}

// SubscribeL2Book 订阅 L2 订单簿
func (ws *HyperliquidWS) SubscribeL2Book(coin string) (<-chan L2BookUpdate, error) {
	msg := map[string]interface{}{
		"method": "subscribe",
		"subscription": map[string]string{
			"type": "l2Book",
			"coin": coin,
		},
	}

	if err := ws.conn.WriteJSON(msg); err != nil {
		return nil, err
	}

	ch := make(chan L2BookUpdate, 100)
	rawCh := make(chan json.RawMessage, 100)
	ws.addHandler("l2Book/"+coin, rawCh)

	go func() {
		for raw := range rawCh {
			var update L2BookUpdate
			if err := json.Unmarshal(raw, &update); err == nil {
				ch <- update
			}
		}
		close(ch)
	}()

	return ch, nil
}

// L2BookUpdate L2 订单簿更新
type L2BookUpdate struct {
	Coin   string      `json:"coin"`
	Time   int64       `json:"time"`
	Levels [][]L2Level `json:"levels"` // [bids, asks]
}

// L2Level L2 价格档位
type L2Level struct {
	Px string `json:"px"`
	Sz string `json:"sz"`
	N  int    `json:"n"` // 订单数量
}

// SubscribeTrades 订阅成交
func (ws *HyperliquidWS) SubscribeTrades(coin string) (<-chan TradesUpdate, error) {
	msg := map[string]interface{}{
		"method": "subscribe",
		"subscription": map[string]string{
			"type": "trades",
			"coin": coin,
		},
	}

	if err := ws.conn.WriteJSON(msg); err != nil {
		return nil, err
	}

	ch := make(chan TradesUpdate, 100)
	rawCh := make(chan json.RawMessage, 100)
	ws.addHandler("trades/"+coin, rawCh)

	go func() {
		for raw := range rawCh {
			var update TradesUpdate
			if err := json.Unmarshal(raw, &update); err == nil {
				ch <- update
			}
		}
		close(ch)
	}()

	return ch, nil
}

// TradesUpdate 成交更新
type TradesUpdate struct {
	Trades []TradeData `json:"data"`
}

// TradeData 成交数据
type TradeData struct {
	Coin string `json:"coin"`
	Side string `json:"side"`
	Px   string `json:"px"`
	Sz   string `json:"sz"`
	Time int64  `json:"time"`
	Hash string `json:"hash"`
}

// SubscribeUserEvents 订阅用户事件
func (ws *HyperliquidWS) SubscribeUserEvents(address string) (<-chan UserEventUpdate, error) {
	msg := map[string]interface{}{
		"method": "subscribe",
		"subscription": map[string]string{
			"type": "userEvents",
			"user": address,
		},
	}

	if err := ws.conn.WriteJSON(msg); err != nil {
		return nil, err
	}

	ch := make(chan UserEventUpdate, 100)
	rawCh := make(chan json.RawMessage, 100)
	ws.addHandler("userEvents/"+address, rawCh)

	go func() {
		for raw := range rawCh {
			var update UserEventUpdate
			if err := json.Unmarshal(raw, &update); err == nil {
				ch <- update
			}
		}
		close(ch)
	}()

	return ch, nil
}

// UserEventUpdate 用户事件更新
type UserEventUpdate struct {
	Fills         []FillData       `json:"fills,omitempty"`
	Funding       *FundingData     `json:"funding,omitempty"`
	Liquidation   *LiquidationData `json:"liquidation,omitempty"`
	NonUserCancel []CancelData     `json:"nonUserCancel,omitempty"`
}

// FillData 成交数据
type FillData struct {
	Coin      string `json:"coin"`
	Px        string `json:"px"`
	Sz        string `json:"sz"`
	Side      string `json:"side"`
	Time      int64  `json:"time"`
	StartPos  string `json:"startPosition"`
	Dir       string `json:"dir"`
	ClosedPnl string `json:"closedPnl"`
	Hash      string `json:"hash"`
	Oid       int64  `json:"oid"`
	Crossed   bool   `json:"crossed"`
	Fee       string `json:"fee"`
}

// FundingData 资金费率数据
type FundingData struct {
	Time        int64  `json:"time"`
	Coin        string `json:"coin"`
	Usdc        string `json:"usdc"`
	Szi         string `json:"szi"`
	FundingRate string `json:"fundingRate"`
}

// LiquidationData 清算数据
type LiquidationData struct {
	Lid            int64  `json:"lid"`
	Liquidator     string `json:"liquidator"`
	LiquidatedUser string `json:"liquidatedUser"`
	NtlPos         string `json:"ntlPos"`
	AccountValue   string `json:"accountValue"`
}

// CancelData 取消数据
type CancelData struct {
	Coin string `json:"coin"`
	Oid  int64  `json:"oid"`
}

func (ws *HyperliquidWS) addHandler(key string, ch chan json.RawMessage) {
	ws.mu.Lock()
	defer ws.mu.Unlock()
	ws.handlers[key] = ch
}

func (ws *HyperliquidWS) readMessages() {
	for {
		select {
		case <-ws.done:
			return
		default:
			_, message, err := ws.conn.ReadMessage()
			if err != nil {
				return
			}

			var msg WSMessage
			if err := json.Unmarshal(message, &msg); err != nil {
				continue
			}

			// 路由消息
			key := msg.Channel
			if msg.Data.Coin != "" {
				key += "/" + msg.Data.Coin
			} else if msg.Data.User != "" {
				key += "/" + msg.Data.User
			}

			ws.mu.RLock()
			handler, ok := ws.handlers[key]
			ws.mu.RUnlock()

			if ok {
				select {
				case handler <- msg.Data.Raw:
				default:
				}
			}
		}
	}
}

// Close 关闭连接
func (ws *HyperliquidWS) Close() error {
	close(ws.done)
	if ws.conn != nil {
		return ws.conn.Close()
	}
	return nil
}

type WSMessage struct {
	Channel string `json:"channel"`
	Data    struct {
		Coin string          `json:"coin,omitempty"`
		User string          `json:"user,omitempty"`
		Raw  json.RawMessage `json:"data"`
	} `json:"data"`
}
```

### C.4 跨平台永续合约套利器

```go
package arbitrage

import (
	"context"
	"fmt"
	"math/big"
	"sync"
	"time"
)

// MultiPlatformPerpsArbitrage 多平台永续合约套利
type MultiPlatformPerpsArbitrage struct {
	// 各平台客户端
	gmx         *gmx.GMXClient
	dydx        *dydxv4.DYDXClient
	synthetix   *synthetixv3.SynthetixPerpsClient
	hyperliquid *hyperliquid.HyperliquidClient

	config    MultiPlatformConfig
	positions map[string]*HedgedPosition
	mu        sync.RWMutex
}

// MultiPlatformConfig 配置
type MultiPlatformConfig struct {
	MinFundingRateDiff float64       // 最小资金费率差（APR）
	MaxPositionUSD     float64       // 最大仓位
	RebalanceInterval  time.Duration // 再平衡间隔
	Markets            []string      // 支持的市场
}

// HedgedPosition 对冲仓位
type HedgedPosition struct {
	Market        string
	LongPlatform  string
	ShortPlatform string
	Size          *big.Int
	OpenTime      time.Time
	LongEntry     *big.Int
	ShortEntry    *big.Int
	TotalFunding  *big.Int
}

// CrossPlatformOpportunity 跨平台机会
type CrossPlatformOpportunity struct {
	Market          string
	LongPlatform    string
	ShortPlatform   string
	LongFundingAPR  float64
	ShortFundingAPR float64
	NetAPR          float64
	LongPrice       *big.Int
	ShortPrice      *big.Int
	PriceDiffPct    float64
}

// NewMultiPlatformPerpsArbitrage 创建套利器
func NewMultiPlatformPerpsArbitrage(
	gmx *gmx.GMXClient,
	dydx *dydxv4.DYDXClient,
	synthetix *synthetixv3.SynthetixPerpsClient,
	hyperliquid *hyperliquid.HyperliquidClient,
	config MultiPlatformConfig,
) *MultiPlatformPerpsArbitrage {
	return &MultiPlatformPerpsArbitrage{
		gmx:         gmx,
		dydx:        dydx,
		synthetix:   synthetix,
		hyperliquid: hyperliquid,
		config:      config,
		positions:   make(map[string]*HedgedPosition),
	}
}

// ScanOpportunities 扫描套利机会
func (a *MultiPlatformPerpsArbitrage) ScanOpportunities(ctx context.Context) ([]*CrossPlatformOpportunity, error) {
	var opportunities []*CrossPlatformOpportunity

	for _, market := range a.config.Markets {
		// 获取各平台资金费率
		rates := a.getFundingRates(ctx, market)

		if len(rates) < 2 {
			continue
		}

		// 找最高和最低费率
		var maxPlatform, minPlatform string
		var maxRate, minRate float64 = -1e10, 1e10

		for platform, rate := range rates {
			if rate > maxRate {
				maxRate = rate
				maxPlatform = platform
			}
			if rate < minRate {
				minRate = rate
				minPlatform = platform
			}
		}

		netAPR := maxRate - minRate

		if netAPR >= a.config.MinFundingRateDiff {
			// 获取价格
			longPrice := a.getPrice(ctx, minPlatform, market)
			shortPrice := a.getPrice(ctx, maxPlatform, market)

			priceDiff := float64(0)
			if longPrice != nil && shortPrice != nil {
				diff := new(big.Float).Sub(
					new(big.Float).SetInt(shortPrice),
					new(big.Float).SetInt(longPrice),
				)
				priceDiff, _ = new(big.Float).Quo(
					diff,
					new(big.Float).SetInt(longPrice),
				).Float64()
			}

			opportunities = append(opportunities, &CrossPlatformOpportunity{
				Market:          market,
				LongPlatform:    minPlatform, // 低费率做多
				ShortPlatform:   maxPlatform, // 高费率做空
				LongFundingAPR:  minRate,
				ShortFundingAPR: maxRate,
				NetAPR:          netAPR,
				LongPrice:       longPrice,
				ShortPrice:      shortPrice,
				PriceDiffPct:    priceDiff * 100,
			})
		}
	}

	return opportunities, nil
}

// getFundingRates 获取各平台资金费率
func (a *MultiPlatformPerpsArbitrage) getFundingRates(ctx context.Context, market string) map[string]float64 {
	rates := make(map[string]float64)
	var mu sync.Mutex
	var wg sync.WaitGroup

	// GMX
	wg.Add(1)
	go func() {
		defer wg.Done()
		// GMX 使用借贷费而不是资金费率
		// 简化处理
		mu.Lock()
		rates["GMX"] = 0 // 需要根据实际计算
		mu.Unlock()
	}()

	// dYdX
	wg.Add(1)
	go func() {
		defer wg.Done()
		dydxRates, err := a.dydx.GetCurrentFundingRates(ctx)
		if err == nil {
			if rate, ok := dydxRates[market]; ok {
				rateFloat := parseRate(rate)
				mu.Lock()
				rates["dYdX"] = rateFloat * 365 * 24 // 年化
				mu.Unlock()
			}
		}
	}()

	// Synthetix
	wg.Add(1)
	go func() {
		defer wg.Done()
		marketId := synthetixv3.SynthetixMarketIDs[market]
		if marketId != nil {
			rate, err := a.synthetix.CurrentFundingRate(ctx, marketId)
			if err == nil && rate != nil {
				rateFloat, _ := new(big.Float).SetInt(rate).Float64()
				mu.Lock()
				rates["Synthetix"] = rateFloat / 1e18 * 365 * 24 // 年化
				mu.Unlock()
			}
		}
	}()

	// Hyperliquid
	wg.Add(1)
	go func() {
		defer wg.Done()
		history, err := a.hyperliquid.FundingHistory(ctx, market,
			time.Now().Add(-1*time.Hour).UnixMilli(),
			time.Now().UnixMilli())
		if err == nil && len(history) > 0 {
			rateFloat := parseRate(history[len(history)-1].FundingRate)
			mu.Lock()
			rates["Hyperliquid"] = rateFloat * 365 * 24 // 年化
			mu.Unlock()
		}
	}()

	wg.Wait()
	return rates
}

// getPrice 获取平台价格
func (a *MultiPlatformPerpsArbitrage) getPrice(ctx context.Context, platform, market string) *big.Int {
	switch platform {
	case "GMX":
		// 从 GMX 获取价格
		return nil
	case "dYdX":
		// 从 dYdX 获取价格
		return nil
	case "Synthetix":
		// 从 Synthetix 获取价格
		return nil
	case "Hyperliquid":
		mids, _ := a.hyperliquid.AllMids(ctx)
		if price, ok := mids[market]; ok {
			p, _ := new(big.Float).SetString(price)
			// 转换为 18 位小数
			p.Mul(p, big.NewFloat(1e18))
			result, _ := p.Int(nil)
			return result
		}
	}
	return nil
}

// ExecuteArbitrage 执行套利
func (a *MultiPlatformPerpsArbitrage) ExecuteArbitrage(ctx context.Context, opp *CrossPlatformOpportunity, sizeUSD float64) error {
	// 1. 在低费率平台做多
	if err := a.openLong(ctx, opp.LongPlatform, opp.Market, sizeUSD); err != nil {
		return fmt.Errorf("open long failed: %w", err)
	}

	// 2. 在高费率平台做空
	if err := a.openShort(ctx, opp.ShortPlatform, opp.Market, sizeUSD); err != nil {
		// 平掉多头
		a.closeLong(ctx, opp.LongPlatform, opp.Market)
		return fmt.Errorf("open short failed: %w", err)
	}

	// 3. 记录仓位
	a.mu.Lock()
	a.positions[opp.Market] = &HedgedPosition{
		Market:        opp.Market,
		LongPlatform:  opp.LongPlatform,
		ShortPlatform: opp.ShortPlatform,
		Size:          big.NewInt(int64(sizeUSD * 1e18)),
		OpenTime:      time.Now(),
		LongEntry:     opp.LongPrice,
		ShortEntry:    opp.ShortPrice,
		TotalFunding:  big.NewInt(0),
	}
	a.mu.Unlock()

	return nil
}

func (a *MultiPlatformPerpsArbitrage) openLong(ctx context.Context, platform, market string, sizeUSD float64) error {
	// 根据平台执行做多
	switch platform {
	case "dYdX":
		// dYdX 做多
	case "Synthetix":
		// Synthetix 做多
	case "Hyperliquid":
		// Hyperliquid 做多
	}
	return nil
}

func (a *MultiPlatformPerpsArbitrage) openShort(ctx context.Context, platform, market string, sizeUSD float64) error {
	// 根据平台执行做空
	return nil
}

func (a *MultiPlatformPerpsArbitrage) closeLong(ctx context.Context, platform, market string) error {
	return nil
}

func parseRate(s string) float64 {
	f, _ := new(big.Float).SetString(s)
	result, _ := f.Float64()
	return result
}

// MonitorPositions 监控仓位
func (a *MultiPlatformPerpsArbitrage) MonitorPositions(ctx context.Context) {
	ticker := time.NewTicker(a.config.RebalanceInterval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			a.mu.RLock()
			for market, pos := range a.positions {
				// 检查资金费率是否反转
				rates := a.getFundingRates(ctx, market)

				longRate := rates[pos.LongPlatform]
				shortRate := rates[pos.ShortPlatform]

				// 如果费率反转，关闭仓位
				if longRate > shortRate {
					fmt.Printf("Funding rate reversed for %s, closing position\n", market)
					a.closePosition(ctx, pos)
				}
			}
			a.mu.RUnlock()
		}
	}
}

func (a *MultiPlatformPerpsArbitrage) closePosition(ctx context.Context, pos *HedgedPosition) error {
	// 关闭多头
	a.closeLong(ctx, pos.LongPlatform, pos.Market)

	// 关闭空头
	// ...

	// 移除仓位记录
	a.mu.Lock()
	delete(a.positions, pos.Market)
	a.mu.Unlock()

	return nil
}
```

---

## 附录 D：永续合约协议对比

### D.1 协议特性对比

| 特性        | GMX V2         | dYdX V4         | Synthetix V3  | Hyperliquid |
|-----------|----------------|-----------------|---------------|-------------|
| **架构**    | EVM (Arbitrum) | Cosmos AppChain | EVM (Base/OP) | Custom L1   |
| **订单类型**  | AMM + Keeper   | 订单簿             | 异步订单          | 订单簿         |
| **资金费率**  | 借贷费            | 每8小时            | 连续            | 每小时         |
| **最大杠杆**  | 100x           | 20x             | 50x           | 50x         |
| **抵押品**   | ETH, USDC 等    | USDC            | snxUSD        | USDC        |
| **Gas 费** | 需要             | 免费              | 需要            | 免费          |
| **延迟**    | ~1-2s          | ~500ms          | ~2-4s         | ~200ms      |
| **去中心化**  | 高              | 高               | 高             | 中           |

### D.2 资金费率机制对比

```
┌─────────────────────────────────────────────────────────────┐
│                   资金费率机制对比                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  GMX V2:                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 不使用传统资金费率                                 │   │
│  │  • 使用借贷费 (Borrowing Fee)                        │   │
│  │  • 费率 = 借贷因子 × 开放利息 / 池子大小             │   │
│  │  • 多空双方都支付借贷费                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  dYdX V4:                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 每 8 小时结算                                     │   │
│  │  • 基于预言机价格和指数价格差                         │   │
│  │  • Rate = (Oracle - Index) / Index × Factor          │   │
│  │  • 上限: ±0.75% per 8h                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Synthetix V3:                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 连续累积（Velocity Based）                        │   │
│  │  • fundingVelocity = skew / skewScale × maxVelocity  │   │
│  │  • 费率随时间积累，无固定结算周期                     │   │
│  │  • 偏斜越大，费率变化越快                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Hyperliquid:                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 每小时结算                                        │   │
│  │  • 基于 premium + interest rate                      │   │
│  │  • Premium = (Mark - Index) / Index                  │   │
│  │  • Interest = 0.01% per 8h (fixed)                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### D.3 套利策略适用性

| 策略         | GMX V2 | dYdX V4 | Synthetix V3 | Hyperliquid |
|------------|--------|---------|--------------|-------------|
| **资金费率对冲** | ⚠️ 特殊  | ✅ 适合    | ✅ 适合         | ✅ 适合        |
| **跨平台费率**  | ⚠️ 困难  | ✅ 推荐    | ✅ 可行         | ✅ 推荐        |
| **期现套利**   | ✅ 可行   | ✅ 推荐    | ✅ 可行         | ✅ 推荐        |
| **清算套利**   | ✅ 可行   | ❌ 链上    | ⚠️ 有限        | ❌ 链上        |
| **价差套利**   | ⚠️ 滑点  | ✅ 推荐    | ⚠️ 延迟        | ✅ 推荐        |

### D.4 风险提示

```
┌─────────────────────────────────────────────────────────────┐
│                     风险管理要点                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 协议风险                                                 │
│     • 智能合约漏洞                                          │
│     • 预言机故障或延迟                                       │
│     • 清算机制失效                                          │
│                                                             │
│  2. 市场风险                                                 │
│     • 资金费率突然反转                                       │
│     • 极端行情导致滑点                                       │
│     • 流动性枯竭                                            │
│                                                             │
│  3. 操作风险                                                 │
│     • 跨平台对冲时机不同步                                   │
│     • Gas 费波动影响收益                                     │
│     • 网络拥堵导致交易延迟                                   │
│                                                             │
│  4. 合规风险                                                 │
│     • 不同司法管辖区监管差异                                 │
│     • 税务合规                                              │
│                                                             │
│  建议:                                                       │
│     • 使用低杠杆 (1-3x)                                     │
│     • 分散在多个协议                                        │
│     • 设置严格止损                                          │
│     • 监控所有仓位实时状态                                   │
│     • 保持充足的可用保证金                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
