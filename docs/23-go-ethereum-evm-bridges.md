# Go-Ethereum EVM 深入解析：跨链桥机制

## 概述

跨链桥 (Cross-Chain Bridges) 是连接不同区块链网络的基础设施，实现资产和消息的跨链传递。理解桥的工作原理对于跨链套利和风险管理至关重要。

## 1. 跨链桥分类

### 1.1 桥的类型

```
┌─────────────────────────────────────────────────────────────┐
│                      跨链桥分类                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  信任模型                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Trusted (中心化)                                    │   │
│  │  └─ 多签桥：Multichain, Portal                       │   │
│  │                                                      │   │
│  │  Trust-minimized (最小化信任)                        │   │
│  │  └─ 轻客户端：IBC, Succinct                         │   │
│  │                                                      │   │
│  │  Optimistic (乐观)                                   │   │
│  │  └─ 欺诈证明：Across, Connext                       │   │
│  │                                                      │   │
│  │  ZK-based (零知识)                                   │   │
│  │  └─ 有效性证明：zkBridge, Polymer                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  资产模型                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Lock & Mint: 锁定原生资产，铸造包装代币             │   │
│  │  Burn & Mint: 销毁一边，铸造另一边                   │   │
│  │  Liquidity Pool: 流动性池原子交换                    │   │
│  │  Native Swap: 原生资产直接交换                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心数据结构

```go
package bridge

import (
	"context"
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

// BridgeType 桥类型
type BridgeType int

const (
	BridgeLockMint BridgeType = iota
	BridgeBurnMint
	BridgeLiquidity
	BridgeNative
)

// CrossChainMessage 跨链消息
type CrossChainMessage struct {
	ID          string         `json:"id"`
	SourceChain uint64         `json:"sourceChain"`
	DestChain   uint64         `json:"destChain"`
	Sender      common.Address `json:"sender"`
	Receiver    common.Address `json:"receiver"`
	Token       common.Address `json:"token"`
	Amount      *big.Int       `json:"amount"`
	Data        []byte         `json:"data"`
	Nonce       uint64         `json:"nonce"`
	Timestamp   time.Time      `json:"timestamp"`
	Status      MessageStatus  `json:"status"`
}

type MessageStatus int

const (
	MessagePending MessageStatus = iota
	MessageSourceConfirmed
	MessageRelayed
	MessageDestConfirmed
	MessageCompleted
	MessageFailed
)

// BridgeProvider 桥接口
type BridgeProvider interface {
	// 基本信息
	Name() string
	SupportedChains() []uint64
	SupportedTokens(chainID uint64) []common.Address

	// 报价
	GetQuote(ctx context.Context, params *BridgeQuoteParams) (*BridgeQuote, error)

	// 执行
	Bridge(ctx context.Context, params *BridgeParams) (*CrossChainMessage, error)

	// 查询
	GetMessageStatus(ctx context.Context, messageID string) (MessageStatus, error)

	// 预估时间
	EstimatedTime(sourceChain, destChain uint64) time.Duration
}

type BridgeQuoteParams struct {
	SourceChain uint64
	DestChain   uint64
	Token       common.Address
	Amount      *big.Int
	Sender      common.Address
	Receiver    common.Address
}

type BridgeQuote struct {
	Provider      string
	SourceChain   uint64
	DestChain     uint64
	TokenIn       common.Address
	TokenOut      common.Address
	AmountIn      *big.Int
	AmountOut     *big.Int
	Fee           *big.Int
	EstimatedTime time.Duration
	Route         []BridgeHop
}

type BridgeHop struct {
	Chain    uint64
	Protocol string
	TokenIn  common.Address
	TokenOut common.Address
}

type BridgeParams struct {
	Quote    *BridgeQuote
	Sender   common.Address
	Receiver common.Address
	Deadline time.Time
}
```

## 2. 主流桥实现

### 2.1 Across Protocol

```go
package across

import (
	"context"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"time"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Across 合约地址
var (
	SpokePoolMainnet  = common.HexToAddress("0x5c7BCd6E7De5423a257D81B442095A1a6ced35C5")
	SpokePoolArbitrum = common.HexToAddress("0xe35e9842fceaCA96570B734083f4a58e8F7C5f2A")
	SpokePoolOptimism = common.HexToAddress("0x6f26Bf09B1C792e3228e5467807a900A503c0281")
	SpokePoolBase     = common.HexToAddress("0x09aea4b2242abC8bb4BB78D537A67a245A7bEC64")

	AcrossAPI = "https://across.to/api"
)

// AcrossClient Across 协议客户端
type AcrossClient struct {
	clients    map[uint64]*ethclient.Client
	spokePools map[uint64]common.Address
	spokeABI   abi.ABI
	httpClient *http.Client
}

// NewAcrossClient 创建客户端
func NewAcrossClient(clients map[uint64]*ethclient.Client) (*AcrossClient, error) {
	spokeABI, err := abi.JSON(strings.NewReader(SpokePoolABI))
	if err != nil {
		return nil, err
	}

	return &AcrossClient{
		clients: clients,
		spokePools: map[uint64]common.Address{
			1:     SpokePoolMainnet,
			42161: SpokePoolArbitrum,
			10:    SpokePoolOptimism,
			8453:  SpokePoolBase,
		},
		spokeABI:   spokeABI,
		httpClient: &http.Client{Timeout: 30 * time.Second},
	}, nil
}

// GetQuote 获取报价
func (c *AcrossClient) GetQuote(ctx context.Context, params *BridgeQuoteParams) (*BridgeQuote, error) {
	url := fmt.Sprintf(
		"%s/suggested-fees?token=%s&destinationChainId=%d&originChainId=%d&amount=%s",
		AcrossAPI,
		params.Token.Hex(),
		params.DestChain,
		params.SourceChain,
		params.Amount.String(),
	)

	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result struct {
		TotalRelayFee struct {
			Total string `json:"total"`
		} `json:"totalRelayFee"`
		RelayerCapitalFee struct {
			Total string `json:"total"`
		} `json:"relayerCapitalFee"`
		RelayerGasFee struct {
			Total string `json:"total"`
		} `json:"relayerGasFee"`
		LpFee struct {
			Total string `json:"total"`
		} `json:"lpFee"`
		Timestamp      string `json:"timestamp"`
		IsAmountTooLow bool   `json:"isAmountTooLow"`
	}

	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	if result.IsAmountTooLow {
		return nil, fmt.Errorf("amount too low")
	}

	totalFee, _ := new(big.Int).SetString(result.TotalRelayFee.Total, 10)
	amountOut := new(big.Int).Sub(params.Amount, totalFee)

	return &BridgeQuote{
		Provider:      "Across",
		SourceChain:   params.SourceChain,
		DestChain:     params.DestChain,
		TokenIn:       params.Token,
		TokenOut:      params.Token, // Across 使用相同代币
		AmountIn:      params.Amount,
		AmountOut:     amountOut,
		Fee:           totalFee,
		EstimatedTime: 2 * time.Minute, // Across 通常很快
	}, nil
}

// Bridge 执行跨链
func (c *AcrossClient) Bridge(ctx context.Context, params *BridgeParams) (*CrossChainMessage, error) {
	client := c.clients[params.Quote.SourceChain]
	spokePool := c.spokePools[params.Quote.SourceChain]

	// 构建 deposit 调用
	// deposit(
	//   address recipient,
	//   address originToken,
	//   uint256 amount,
	//   uint256 destinationChainId,
	//   int64 relayerFeePct,
	//   uint32 quoteTimestamp,
	//   bytes message,
	//   uint256 maxCount
	// )

	timestamp := uint32(time.Now().Unix())
	relayerFeePct := c.calculateFeePct(params.Quote)

	callData, err := c.spokeABI.Pack(
		"deposit",
		params.Receiver,
		params.Quote.TokenIn,
		params.Quote.AmountIn,
		big.NewInt(int64(params.Quote.DestChain)),
		relayerFeePct,
		timestamp,
		[]byte{},
		big.NewInt(0), // maxCount = unlimited
	)
	if err != nil {
		return nil, err
	}

	// 构建交易
	tx, err := c.buildTransaction(ctx, client, spokePool, callData, params.Quote.AmountIn)
	if err != nil {
		return nil, err
	}

	// 发送交易
	if err := client.SendTransaction(ctx, tx); err != nil {
		return nil, err
	}

	return &CrossChainMessage{
		ID:          tx.Hash().Hex(),
		SourceChain: params.Quote.SourceChain,
		DestChain:   params.Quote.DestChain,
		Sender:      params.Sender,
		Receiver:    params.Receiver,
		Token:       params.Quote.TokenIn,
		Amount:      params.Quote.AmountIn,
		Timestamp:   time.Now(),
		Status:      MessagePending,
	}, nil
}

// GetFillStatus 获取填充状态
func (c *AcrossClient) GetFillStatus(ctx context.Context, depositTxHash string) (*FillStatus, error) {
	url := fmt.Sprintf("%s/deposit/status?depositTxHash=%s", AcrossAPI, depositTxHash)

	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var status FillStatus
	if err := json.NewDecoder(resp.Body).Decode(&status); err != nil {
		return nil, err
	}

	return &status, nil
}

type FillStatus struct {
	Status        string `json:"status"` // "pending", "filled", "expired"
	FillTxHash    string `json:"fillTxHash"`
	FillTimestamp int64  `json:"fillTimestamp"`
}

func (c *AcrossClient) calculateFeePct(quote *BridgeQuote) int64 {
	// 计算 relayer fee percentage (basis points * 1e18 / 1e4)
	feePct := new(big.Int).Mul(quote.Fee, big.NewInt(1e18))
	feePct.Div(feePct, quote.AmountIn)
	return feePct.Int64()
}
```

### 2.2 Stargate (LayerZero)

```go
package stargate

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Stargate Router 地址
var (
	RouterMainnet  = common.HexToAddress("0x8731d54E9D02c286767d56ac03e8037C07e01e98")
	RouterArbitrum = common.HexToAddress("0x53Bf833A5d6c4ddA888F69c22C88C9f356a41614")
	RouterOptimism = common.HexToAddress("0xB0D502E938ed5f4df2E681fE6E419ff29631d62b")
)

// Pool IDs
const (
	PoolUSDC = 1
	PoolUSDT = 2
	PoolDAI  = 3
	PoolETH  = 13
)

// Chain IDs (LayerZero)
const (
	LzChainEthereum = 101
	LzChainArbitrum = 110
	LzChainOptimism = 111
	LzChainBase     = 184
)

// StargateClient Stargate 客户端
type StargateClient struct {
	clients    map[uint64]*ethclient.Client
	routers    map[uint64]common.Address
	routerABI  abi.ABI
	lzChainIDs map[uint64]uint16
}

// NewStargateClient 创建客户端
func NewStargateClient(clients map[uint64]*ethclient.Client) (*StargateClient, error) {
	routerABI, err := abi.JSON(strings.NewReader(StargateRouterABI))
	if err != nil {
		return nil, err
	}

	return &StargateClient{
		clients: clients,
		routers: map[uint64]common.Address{
			1:     RouterMainnet,
			42161: RouterArbitrum,
			10:    RouterOptimism,
		},
		routerABI: routerABI,
		lzChainIDs: map[uint64]uint16{
			1:     LzChainEthereum,
			42161: LzChainArbitrum,
			10:    LzChainOptimism,
			8453:  LzChainBase,
		},
	}, nil
}

// GetQuote 获取报价
func (c *StargateClient) GetQuote(ctx context.Context, params *BridgeQuoteParams) (*BridgeQuote, error) {
	client := c.clients[params.SourceChain]
	router := c.routers[params.SourceChain]
	destLzChainID := c.lzChainIDs[params.DestChain]

	// 调用 quoteLayerZeroFee 获取跨链费用
	callData, _ := c.routerABI.Pack(
		"quoteLayerZeroFee",
		destLzChainID,
		uint8(1), // function type: swap
		params.Receiver.Bytes(),
		[]byte{},
		IStargateRouter.lzTxObj{
			DstGasForCall:   big.NewInt(0),
			DstNativeAmount: big.NewInt(0),
			DstNativeAddr:   []byte{},
		},
	)

	result, err := client.CallContract(ctx, ethereum.CallMsg{
		To:   &router,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	// 解析费用
	nativeFee := new(big.Int).SetBytes(result[:32])

	// 估算输出金额（扣除协议费）
	// Stargate 收取 0.06% 的协议费
	protocolFee := new(big.Int).Mul(params.Amount, big.NewInt(6))
	protocolFee.Div(protocolFee, big.NewInt(10000))

	amountOut := new(big.Int).Sub(params.Amount, protocolFee)

	return &BridgeQuote{
		Provider:      "Stargate",
		SourceChain:   params.SourceChain,
		DestChain:     params.DestChain,
		TokenIn:       params.Token,
		TokenOut:      params.Token,
		AmountIn:      params.Amount,
		AmountOut:     amountOut,
		Fee:           new(big.Int).Add(nativeFee, protocolFee),
		EstimatedTime: 5 * time.Minute,
	}, nil
}

// Swap 执行跨链交换
func (c *StargateClient) Swap(ctx context.Context, params *SwapParams) (*CrossChainMessage, error) {
	client := c.clients[params.SourceChain]
	router := c.routers[params.SourceChain]
	destLzChainID := c.lzChainIDs[params.DestChain]

	// swap(
	//   uint16 _dstChainId,
	//   uint256 _srcPoolId,
	//   uint256 _dstPoolId,
	//   address payable _refundAddress,
	//   uint256 _amountLD,
	//   uint256 _minAmountLD,
	//   lzTxObj memory _lzTxParams,
	//   bytes calldata _to,
	//   bytes calldata _payload
	// )

	minAmountOut := new(big.Int).Mul(params.AmountOut, big.NewInt(99))
	minAmountOut.Div(minAmountOut, big.NewInt(100)) // 1% slippage

	callData, err := c.routerABI.Pack(
		"swap",
		destLzChainID,
		params.SrcPoolID,
		params.DstPoolID,
		params.Sender, // refund address
		params.AmountIn,
		minAmountOut,
		IStargateRouter.lzTxObj{
			DstGasForCall:   big.NewInt(0),
			DstNativeAmount: big.NewInt(0),
			DstNativeAddr:   []byte{},
		},
		params.Receiver.Bytes(),
		[]byte{},
	)
	if err != nil {
		return nil, err
	}

	// 需要发送 native token 作为 LayerZero 费用
	tx, err := c.buildTransactionWithValue(ctx, client, router, callData, params.NativeFee)
	if err != nil {
		return nil, err
	}

	if err := client.SendTransaction(ctx, tx); err != nil {
		return nil, err
	}

	return &CrossChainMessage{
		ID:          tx.Hash().Hex(),
		SourceChain: params.SourceChain,
		DestChain:   params.DestChain,
		Sender:      params.Sender,
		Receiver:    params.Receiver,
		Amount:      params.AmountIn,
		Status:      MessagePending,
	}, nil
}

type SwapParams struct {
	SourceChain uint64
	DestChain   uint64
	SrcPoolID   *big.Int
	DstPoolID   *big.Int
	Sender      common.Address
	Receiver    common.Address
	AmountIn    *big.Int
	AmountOut   *big.Int
	NativeFee   *big.Int
}
```

### 2.3 Hyperlane

```go
package hyperlane

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Hyperlane 合约地址
var (
	MailboxMainnet  = common.HexToAddress("0xc005dc82818d67AF737725bD4bf75435d065D239")
	MailboxArbitrum = common.HexToAddress("0x979Ca5202784112f4738403dBec5D0F3B9daabB9")
)

// HyperlaneClient Hyperlane 客户端
type HyperlaneClient struct {
	clients    map[uint64]*ethclient.Client
	mailboxes  map[uint64]common.Address
	mailboxABI abi.ABI
}

// NewHyperlaneClient 创建客户端
func NewHyperlaneClient(clients map[uint64]*ethclient.Client) (*HyperlaneClient, error) {
	mailboxABI, err := abi.JSON(strings.NewReader(MailboxABI))
	if err != nil {
		return nil, err
	}

	return &HyperlaneClient{
		clients: clients,
		mailboxes: map[uint64]common.Address{
			1:     MailboxMainnet,
			42161: MailboxArbitrum,
		},
		mailboxABI: mailboxABI,
	}, nil
}

// DispatchMessage 发送跨链消息
func (c *HyperlaneClient) DispatchMessage(ctx context.Context, params *DispatchParams) (common.Hash, error) {
	client := c.clients[params.SourceChain]
	mailbox := c.mailboxes[params.SourceChain]

	// dispatch(
	//   uint32 _destinationDomain,
	//   bytes32 _recipientAddress,
	//   bytes calldata _messageBody
	// )

	recipientBytes32 := common.LeftPadBytes(params.Recipient.Bytes(), 32)

	callData, err := c.mailboxABI.Pack(
		"dispatch",
		uint32(params.DestChain),
		[32]byte(recipientBytes32),
		params.MessageBody,
	)
	if err != nil {
		return common.Hash{}, err
	}

	// 获取消息费用
	fee, err := c.quoteDispatch(ctx, params)
	if err != nil {
		return common.Hash{}, err
	}

	tx, err := c.buildTransactionWithValue(ctx, client, mailbox, callData, fee)
	if err != nil {
		return common.Hash{}, err
	}

	if err := client.SendTransaction(ctx, tx); err != nil {
		return common.Hash{}, err
	}

	return tx.Hash(), nil
}

// quoteDispatch 获取消息费用
func (c *HyperlaneClient) quoteDispatch(ctx context.Context, params *DispatchParams) (*big.Int, error) {
	client := c.clients[params.SourceChain]
	mailbox := c.mailboxes[params.SourceChain]

	recipientBytes32 := common.LeftPadBytes(params.Recipient.Bytes(), 32)

	callData, _ := c.mailboxABI.Pack(
		"quoteDispatch",
		uint32(params.DestChain),
		[32]byte(recipientBytes32),
		params.MessageBody,
	)

	result, err := client.CallContract(ctx, ethereum.CallMsg{
		To:   &mailbox,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	return new(big.Int).SetBytes(result), nil
}

type DispatchParams struct {
	SourceChain uint64
	DestChain   uint64
	Recipient   common.Address
	MessageBody []byte
}
```

## 3. 跨链桥聚合器

### 3.1 统一接口

```go
package aggregator

import (
	"context"
	"sort"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

// BridgeAggregator 桥聚合器
type BridgeAggregator struct {
	providers map[string]BridgeProvider
}

// NewBridgeAggregator 创建聚合器
func NewBridgeAggregator() *BridgeAggregator {
	return &BridgeAggregator{
		providers: make(map[string]BridgeProvider),
	}
}

// RegisterProvider 注册桥提供者
func (a *BridgeAggregator) RegisterProvider(name string, provider BridgeProvider) {
	a.providers[name] = provider
}

// GetBestRoute 获取最优路由
func (a *BridgeAggregator) GetBestRoute(ctx context.Context, params *BridgeQuoteParams) (*BridgeQuote, error) {
	// 并发获取所有报价
	var wg sync.WaitGroup
	quotes := make(chan *BridgeQuote, len(a.providers))
	errors := make(chan error, len(a.providers))

	for name, provider := range a.providers {
		// 检查是否支持该路由
		if !a.supportsRoute(provider, params.SourceChain, params.DestChain, params.Token) {
			continue
		}

		wg.Add(1)
		go func(n string, p BridgeProvider) {
			defer wg.Done()

			ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
			defer cancel()

			quote, err := p.GetQuote(ctx, params)
			if err != nil {
				errors <- err
				return
			}

			quote.Provider = n
			quotes <- quote
		}(name, provider)
	}

	// 等待所有报价
	go func() {
		wg.Wait()
		close(quotes)
		close(errors)
	}()

	// 收集报价
	var allQuotes []*BridgeQuote
	for quote := range quotes {
		allQuotes = append(allQuotes, quote)
	}

	if len(allQuotes) == 0 {
		return nil, fmt.Errorf("no routes available")
	}

	// 按输出金额排序（最高优先）
	sort.Slice(allQuotes, func(i, j int) bool {
		return allQuotes[i].AmountOut.Cmp(allQuotes[j].AmountOut) > 0
	})

	return allQuotes[0], nil
}

// GetAllQuotes 获取所有报价
func (a *BridgeAggregator) GetAllQuotes(ctx context.Context, params *BridgeQuoteParams) ([]*BridgeQuote, error) {
	var wg sync.WaitGroup
	quotes := make(chan *BridgeQuote, len(a.providers))

	for name, provider := range a.providers {
		if !a.supportsRoute(provider, params.SourceChain, params.DestChain, params.Token) {
			continue
		}

		wg.Add(1)
		go func(n string, p BridgeProvider) {
			defer wg.Done()

			quote, err := p.GetQuote(ctx, params)
			if err != nil {
				return
			}

			quote.Provider = n
			quotes <- quote
		}(name, provider)
	}

	go func() {
		wg.Wait()
		close(quotes)
	}()

	var allQuotes []*BridgeQuote
	for quote := range quotes {
		allQuotes = append(allQuotes, quote)
	}

	// 按多个指标排序
	a.sortQuotes(allQuotes)

	return allQuotes, nil
}

// sortQuotes 多维度排序
func (a *BridgeAggregator) sortQuotes(quotes []*BridgeQuote) {
	// 计算综合评分
	for _, q := range quotes {
		q.Score = a.calculateScore(q)
	}

	sort.Slice(quotes, func(i, j int) bool {
		return quotes[i].Score > quotes[j].Score
	})
}

func (a *BridgeAggregator) calculateScore(q *BridgeQuote) float64 {
	// 输出金额权重 (70%)
	amountScore := float64(q.AmountOut.Int64()) / 1e18 * 0.7

	// 时间权重 (20%) - 越快越好
	timeScore := (1.0 - float64(q.EstimatedTime.Minutes())/60.0) * 0.2

	// 费用权重 (10%)
	feeScore := (1.0 - float64(q.Fee.Int64())/float64(q.AmountIn.Int64())) * 0.1

	return amountScore + timeScore + feeScore
}

func (a *BridgeAggregator) supportsRoute(p BridgeProvider, srcChain, dstChain uint64, token common.Address) bool {
	chains := p.SupportedChains()

	var srcSupported, dstSupported bool
	for _, c := range chains {
		if c == srcChain {
			srcSupported = true
		}
		if c == dstChain {
			dstSupported = true
		}
	}

	if !srcSupported || !dstSupported {
		return false
	}

	// 检查代币支持
	tokens := p.SupportedTokens(srcChain)
	for _, t := range tokens {
		if t == token {
			return true
		}
	}

	return false
}
```

## 4. 跨链套利策略

### 4.1 跨链价差套利

```go
package arbitrage

import (
	"context"
	"math/big"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// CrossChainArbitrage 跨链套利
type CrossChainArbitrage struct {
	clients          map[uint64]*ethclient.Client
	bridgeAggregator *BridgeAggregator
	dexAggregators   map[uint64]DEXAggregator
	priceOracle      PriceOracle

	config ArbitrageConfig
}

type ArbitrageConfig struct {
	MinProfitUSD    float64
	MaxPositionUSD  float64
	SupportedChains []uint64
	SupportedTokens []common.Address
	BridgeTimeout   time.Duration
}

// CrossChainOpportunity 跨链套利机会
type CrossChainOpportunity struct {
	// 源链
	SourceChain uint64
	SourceDEX   string
	SourcePrice *big.Float

	// 目标链
	DestChain uint64
	DestDEX   string
	DestPrice *big.Float

	// 代币
	Token     common.Address
	BaseToken common.Address // 通常是稳定币

	// 套利参数
	Direction      string // "source_to_dest" or "dest_to_source"
	OptimalAmount  *big.Int
	ExpectedProfit *big.Int
	ProfitUSD      float64

	// 桥信息
	BridgeQuote *BridgeQuote
	BridgeTime  time.Duration
}

// FindOpportunities 寻找跨链套利机会
func (a *CrossChainArbitrage) FindOpportunities(ctx context.Context) ([]*CrossChainOpportunity, error) {
	var opportunities []*CrossChainOpportunity
	var mu sync.Mutex
	var wg sync.WaitGroup

	// 遍历所有链对
	for i, srcChain := range a.config.SupportedChains {
		for j, dstChain := range a.config.SupportedChains {
			if i >= j {
				continue // 避免重复
			}

			wg.Add(1)
			go func(src, dst uint64) {
				defer wg.Done()

				opps := a.findOpportunitiesForPair(ctx, src, dst)
				if len(opps) > 0 {
					mu.Lock()
					opportunities = append(opportunities, opps...)
					mu.Unlock()
				}
			}(srcChain, dstChain)
		}
	}

	wg.Wait()
	return opportunities, nil
}

// findOpportunitiesForPair 寻找特定链对的套利机会
func (a *CrossChainArbitrage) findOpportunitiesForPair(ctx context.Context, srcChain, dstChain uint64) []*CrossChainOpportunity {
	var opportunities []*CrossChainOpportunity

	srcDEX := a.dexAggregators[srcChain]
	dstDEX := a.dexAggregators[dstChain]

	for _, token := range a.config.SupportedTokens {
		// 获取两条链上的价格
		srcPrice, err := srcDEX.GetPrice(ctx, token, USDC)
		if err != nil {
			continue
		}

		dstPrice, err := dstDEX.GetPrice(ctx, token, USDC)
		if err != nil {
			continue
		}

		// 计算价差
		priceDiff := new(big.Float).Sub(srcPrice, dstPrice)
		priceDiffAbs := new(big.Float).Abs(priceDiff)

		// 计算相对价差
		avgPrice := new(big.Float).Add(srcPrice, dstPrice)
		avgPrice.Quo(avgPrice, big.NewFloat(2))

		relativeDiff, _ := new(big.Float).Quo(priceDiffAbs, avgPrice).Float64()

		// 如果价差超过阈值（考虑桥费用）
		if relativeDiff > 0.005 { // 0.5%
			opp := a.evaluateOpportunity(ctx, srcChain, dstChain, token, srcPrice, dstPrice)
			if opp != nil && opp.ProfitUSD >= a.config.MinProfitUSD {
				opportunities = append(opportunities, opp)
			}
		}
	}

	return opportunities
}

// evaluateOpportunity 评估套利机会
func (a *CrossChainArbitrage) evaluateOpportunity(
	ctx context.Context,
	srcChain, dstChain uint64,
	token common.Address,
	srcPrice, dstPrice *big.Float,
) *CrossChainOpportunity {
	var direction string
	var buyChain, sellChain uint64
	var buyPrice, sellPrice *big.Float

	if srcPrice.Cmp(dstPrice) < 0 {
		// 源链便宜，在源链买，桥接到目标链卖
		direction = "source_to_dest"
		buyChain = srcChain
		sellChain = dstChain
		buyPrice = srcPrice
		sellPrice = dstPrice
	} else {
		// 目标链便宜
		direction = "dest_to_source"
		buyChain = dstChain
		sellChain = srcChain
		buyPrice = dstPrice
		sellPrice = srcPrice
	}

	// 获取桥报价
	testAmount := big.NewInt(1e18) // 1 token
	bridgeQuote, err := a.bridgeAggregator.GetBestRoute(ctx, &BridgeQuoteParams{
		SourceChain: buyChain,
		DestChain:   sellChain,
		Token:       token,
		Amount:      testAmount,
	})
	if err != nil {
		return nil
	}

	// 计算利润
	// 买入成本
	buyCost := new(big.Float).Mul(buyPrice, new(big.Float).SetInt(testAmount))

	// 卖出收入（扣除桥费后的金额）
	sellIncome := new(big.Float).Mul(sellPrice, new(big.Float).SetInt(bridgeQuote.AmountOut))

	// 毛利润
	grossProfit := new(big.Float).Sub(sellIncome, buyCost)

	// 桥费用（转换为 USD）
	bridgeFeeUSD := a.convertToUSD(bridgeQuote.Fee, buyChain)

	// 净利润
	netProfit := new(big.Float).Sub(grossProfit, big.NewFloat(bridgeFeeUSD))
	netProfitFloat, _ := netProfit.Float64()

	if netProfitFloat <= 0 {
		return nil
	}

	// 计算最优金额
	optimalAmount := a.calculateOptimalAmount(ctx, buyChain, sellChain, token, buyPrice, sellPrice)

	return &CrossChainOpportunity{
		SourceChain:    srcChain,
		SourcePrice:    srcPrice,
		DestChain:      dstChain,
		DestPrice:      dstPrice,
		Token:          token,
		BaseToken:      USDC,
		Direction:      direction,
		OptimalAmount:  optimalAmount,
		ExpectedProfit: a.floatToBigInt(netProfit),
		ProfitUSD:      netProfitFloat,
		BridgeQuote:    bridgeQuote,
		BridgeTime:     bridgeQuote.EstimatedTime,
	}
}

// Execute 执行跨链套利
func (a *CrossChainArbitrage) Execute(ctx context.Context, opp *CrossChainOpportunity) error {
	var buyChain, sellChain uint64
	if opp.Direction == "source_to_dest" {
		buyChain = opp.SourceChain
		sellChain = opp.DestChain
	} else {
		buyChain = opp.DestChain
		sellChain = opp.SourceChain
	}

	// 1. 在买入链购买代币
	buyDEX := a.dexAggregators[buyChain]
	buyTx, err := buyDEX.Swap(ctx, opp.BaseToken, opp.Token, opp.OptimalAmount)
	if err != nil {
		return fmt.Errorf("buy failed: %w", err)
	}

	// 等待买入确认
	if err := a.waitForTx(ctx, buyChain, buyTx); err != nil {
		return err
	}

	// 2. 桥接到卖出链
	bridgeMsg, err := a.bridgeAggregator.providers[opp.BridgeQuote.Provider].Bridge(ctx, &BridgeParams{
		Quote: opp.BridgeQuote,
	})
	if err != nil {
		return fmt.Errorf("bridge failed: %w", err)
	}

	// 3. 等待桥接完成
	if err := a.waitForBridge(ctx, bridgeMsg); err != nil {
		return err
	}

	// 4. 在卖出链卖出代币
	sellDEX := a.dexAggregators[sellChain]
	_, err = sellDEX.Swap(ctx, opp.Token, opp.BaseToken, opp.BridgeQuote.AmountOut)
	if err != nil {
		return fmt.Errorf("sell failed: %w", err)
	}

	return nil
}

func (a *CrossChainArbitrage) calculateOptimalAmount(
	ctx context.Context,
	buyChain, sellChain uint64,
	token common.Address,
	buyPrice, sellPrice *big.Float,
) *big.Int {
	// 简化：使用最大仓位的一半
	maxPosition := new(big.Float).SetFloat64(a.config.MaxPositionUSD)
	optimal := new(big.Float).Quo(maxPosition, buyPrice)
	optimal.Mul(optimal, big.NewFloat(0.5))

	result, _ := optimal.Int(nil)
	return result
}
```

### 4.2 桥流动性套利

```go
package arbitrage

import (
	"context"
	"math/big"
)

// BridgeLiquidityArbitrage 桥流动性套利
type BridgeLiquidityArbitrage struct {
	bridges map[string]BridgeProvider
	dexes   map[uint64]DEXAggregator
}

// 利用不同桥之间的价差
func (a *BridgeLiquidityArbitrage) FindBridgeArbitrage(ctx context.Context, params *BridgeArbParams) (*BridgeArbOpportunity, error) {
	// 获取所有桥的报价
	quotes := make(map[string]*BridgeQuote)

	for name, bridge := range a.bridges {
		quote, err := bridge.GetQuote(ctx, &BridgeQuoteParams{
			SourceChain: params.SourceChain,
			DestChain:   params.DestChain,
			Token:       params.Token,
			Amount:      params.Amount,
		})
		if err != nil {
			continue
		}
		quotes[name] = quote
	}

	if len(quotes) < 2 {
		return nil, nil
	}

	// 找出最高和最低输出
	var bestBridge, worstBridge string
	var bestOutput, worstOutput *big.Int

	for name, quote := range quotes {
		if bestOutput == nil || quote.AmountOut.Cmp(bestOutput) > 0 {
			bestOutput = quote.AmountOut
			bestBridge = name
		}
		if worstOutput == nil || quote.AmountOut.Cmp(worstOutput) < 0 {
			worstOutput = quote.AmountOut
			worstBridge = name
		}
	}

	// 计算价差
	diff := new(big.Int).Sub(bestOutput, worstOutput)
	diffPct := new(big.Float).Quo(
		new(big.Float).SetInt(diff),
		new(big.Float).SetInt(params.Amount),
	)

	diffFloat, _ := diffPct.Float64()

	// 如果价差足够大（考虑执行成本）
	if diffFloat < 0.002 { // 0.2%
		return nil, nil
	}

	return &BridgeArbOpportunity{
		SourceChain: params.SourceChain,
		DestChain:   params.DestChain,
		Token:       params.Token,
		Amount:      params.Amount,
		BestBridge:  bestBridge,
		BestOutput:  bestOutput,
		WorstBridge: worstBridge,
		WorstOutput: worstOutput,
		ProfitPct:   diffFloat,
	}, nil
}

type BridgeArbParams struct {
	SourceChain uint64
	DestChain   uint64
	Token       common.Address
	Amount      *big.Int
}

type BridgeArbOpportunity struct {
	SourceChain uint64
	DestChain   uint64
	Token       common.Address
	Amount      *big.Int
	BestBridge  string
	BestOutput  *big.Int
	WorstBridge string
	WorstOutput *big.Int
	ProfitPct   float64
}
```

## 5. 跨链安全

### 5.1 桥安全监控

```go
package security

import (
	"context"
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

// BridgeSecurityMonitor 桥安全监控
type BridgeSecurityMonitor struct {
	bridges        map[string]BridgeProvider
	alerter        Alerter
	riskThresholds RiskThresholds
}

type RiskThresholds struct {
	MaxSingleTxUSD     float64  // 单笔交易最大金额
	MaxDailyVolumeUSD  float64  // 每日最大交易量
	MinBridgeLiquidity *big.Int // 最小桥流动性
	MaxDelayMinutes    int      // 最大延迟时间
	SuspiciousPatterns []string // 可疑模式
}

// MonitorBridgeHealth 监控桥健康状态
func (m *BridgeSecurityMonitor) MonitorBridgeHealth(ctx context.Context) {
	ticker := time.NewTicker(1 * time.Minute)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			for name, bridge := range m.bridges {
				health := m.checkBridgeHealth(ctx, name, bridge)
				if health.RiskLevel >= RiskHigh {
					m.alerter.Alert(AlertCritical, fmt.Sprintf("Bridge %s health critical: %s", name, health.Reason))
				}
			}
		}
	}
}

// checkBridgeHealth 检查桥健康状态
func (m *BridgeSecurityMonitor) checkBridgeHealth(ctx context.Context, name string, bridge BridgeProvider) *BridgeHealth {
	health := &BridgeHealth{
		Bridge:    name,
		Timestamp: time.Now(),
		RiskLevel: RiskLow,
	}

	// 1. 检查流动性
	liquidity := m.getBridgeLiquidity(ctx, bridge)
	if liquidity.Cmp(m.riskThresholds.MinBridgeLiquidity) < 0 {
		health.RiskLevel = RiskMedium
		health.Reason = "Low liquidity"
	}

	// 2. 检查延迟
	avgDelay := m.getAverageDelay(ctx, bridge)
	if avgDelay > time.Duration(m.riskThresholds.MaxDelayMinutes)*time.Minute {
		health.RiskLevel = RiskHigh
		health.Reason = "High delay"
	}

	// 3. 检查异常活动
	if m.detectAnomalousActivity(ctx, bridge) {
		health.RiskLevel = RiskCritical
		health.Reason = "Anomalous activity detected"
	}

	return health
}

// ValidateTransaction 验证跨链交易
func (m *BridgeSecurityMonitor) ValidateTransaction(ctx context.Context, tx *CrossChainTransaction) error {
	// 1. 金额检查
	txValueUSD := m.convertToUSD(tx.Amount, tx.Token)
	if txValueUSD > m.riskThresholds.MaxSingleTxUSD {
		return fmt.Errorf("transaction exceeds max single tx limit: %.2f USD", txValueUSD)
	}

	// 2. 检查每日限额
	dailyVolume := m.getDailyVolume(ctx, tx.Sender)
	if dailyVolume+txValueUSD > m.riskThresholds.MaxDailyVolumeUSD {
		return fmt.Errorf("daily volume limit exceeded")
	}

	// 3. 检查目标地址
	if m.isBlacklisted(tx.Receiver) {
		return fmt.Errorf("receiver address is blacklisted")
	}

	// 4. 检查可疑模式
	if m.matchesSuspiciousPattern(tx) {
		return fmt.Errorf("transaction matches suspicious pattern")
	}

	return nil
}

type BridgeHealth struct {
	Bridge    string
	Timestamp time.Time
	RiskLevel RiskLevel
	Reason    string
	Metrics   map[string]interface{}
}

type CrossChainTransaction struct {
	Bridge    string
	Sender    common.Address
	Receiver  common.Address
	Token     common.Address
	Amount    *big.Int
	Timestamp time.Time
}
```

## 6. 总结

### 6.1 主流桥对比

| 桥             | 类型         | 速度         | 费用 | 安全模型         |
|---------------|------------|------------|----|--------------|
| **Across**    | Optimistic | 快 (1-2min) | 低  | Relayer 竞争   |
| **Stargate**  | Liquidity  | 中 (5min)   | 中  | LayerZero 消息 |
| **Hyperlane** | Message    | 中 (5min)   | 低  | 模块化安全        |
| **Wormhole**  | Message    | 中 (10min)  | 低  | Guardian 网络  |
| **Connext**   | Liquidity  | 快 (2min)   | 中  | 流动性提供者       |

### 6.2 跨链套利最佳实践

1. **价格监控**: 实时监控多链价格
2. **桥选择**: 根据速度和费用选择最优桥
3. **风险管理**: 设置单笔和每日限额
4. **安全监控**: 监控桥健康状态

### 6.3 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     跨链套利架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  Chain A    │    │   Bridge    │    │  Chain B    │     │
│  │  DEX Price  │───▶│  Aggregator │───▶│  DEX Price  │     │
│  └─────────────┘    └──────┬──────┘    └─────────────┘     │
│                            │                                │
│                            ▼                                │
│                    ┌─────────────┐                          │
│                    │  Arbitrage  │                          │
│                    │  Detector   │                          │
│                    └──────┬──────┘                          │
│                           │                                 │
│         ┌─────────────────┼─────────────────┐              │
│         ▼                 ▼                 ▼              │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐          │
│  │  Across   │    │ Stargate  │    │ Hyperlane │          │
│  └───────────┘    └───────────┘    └───────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

跨链桥是实现多链套利的关键基础设施，理解不同桥的特性和风险对于制定有效的跨链策略至关重要。
