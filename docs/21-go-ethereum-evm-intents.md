# Go-Ethereum EVM 深入解析：意图协议 (Intent-Based Protocols)

## 概述

意图协议 (Intent-Based Protocols) 代表了 DeFi 交易范式的重大转变。用户不再指定"如何"执行交易，而是表达"想要什么"
结果，由专业的求解者 (Solvers) 竞争提供最优执行。

## 1. 意图协议基础

### 1.1 传统交易 vs 意图交易

```
传统交易流程:
┌──────────┐    ┌──────────┐    ┌──────────┐
│  用户    │───▶│  DEX     │───▶│  链上    │
│  指定路由 │    │  Router  │    │  执行    │
└──────────┘    └──────────┘    └──────────┘
     │
     └─ 用户承担：路由选择、滑点、MEV 风险

意图交易流程:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  用户    │───▶│  签名    │───▶│  Solver  │───▶│  链上    │
│  表达意图 │    │  意图    │    │  竞争    │    │  结算    │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │
     └─ Solver 承担：寻找最优路径、承担执行风险
```

### 1.2 核心概念

```go
package intents

import (
	"context"
	"crypto/ecdsa"
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// Intent 表示用户意图
type Intent struct {
	// 基本信息
	ID        string         `json:"id"`
	Creator   common.Address `json:"creator"`
	CreatedAt time.Time      `json:"createdAt"`
	ExpiresAt time.Time      `json:"expiresAt"`

	// 交易意图
	TokenIn      common.Address `json:"tokenIn"`
	TokenOut     common.Address `json:"tokenOut"`
	AmountIn     *big.Int       `json:"amountIn"`
	MinAmountOut *big.Int       `json:"minAmountOut"`

	// 约束条件
	Constraints IntentConstraints `json:"constraints"`

	// 签名
	Signature []byte `json:"signature"`
}

// IntentConstraints 意图约束
type IntentConstraints struct {
	// 价格约束
	MaxSlippage float64  `json:"maxSlippage"` // 最大滑点 (basis points)
	LimitPrice  *big.Int `json:"limitPrice"`  // 限价 (可选)

	// 时间约束
	ValidFrom  time.Time `json:"validFrom"`
	ValidUntil time.Time `json:"validUntil"`

	// 执行约束
	PartialFill bool     `json:"partialFill"` // 是否允许部分成交
	ExcludeDEXs []string `json:"excludeDEXs"` // 排除的 DEX

	// Gas 约束
	MaxGasPrice *big.Int `json:"maxGasPrice"`
}

// IntentSolution 求解方案
type IntentSolution struct {
	IntentID string         `json:"intentId"`
	Solver   common.Address `json:"solver"`

	// 执行方案
	AmountOut     *big.Int   `json:"amountOut"`
	ExecutionPlan []SwapStep `json:"executionPlan"`

	// 成本
	GasEstimate uint64   `json:"gasEstimate"`
	SolverFee   *big.Int `json:"solverFee"`

	// 签名
	Signature []byte `json:"signature"`
}

// SwapStep 交换步骤
type SwapStep struct {
	Protocol  string         `json:"protocol"`
	Pool      common.Address `json:"pool"`
	TokenIn   common.Address `json:"tokenIn"`
	TokenOut  common.Address `json:"tokenOut"`
	AmountIn  *big.Int       `json:"amountIn"`
	AmountOut *big.Int       `json:"amountOut"`
}

// IntentProtocol 意图协议接口
type IntentProtocol interface {
	// 创建意图
	CreateIntent(ctx context.Context, intent *Intent) error

	// 提交解决方案
	SubmitSolution(ctx context.Context, solution *IntentSolution) error

	// 查询意图
	GetIntent(ctx context.Context, intentID string) (*Intent, error)

	// 监听意图
	WatchIntents(ctx context.Context) (<-chan *Intent, error)
}
```

## 2. UniswapX 集成

### 2.1 UniswapX 架构

```go
package uniswapx

import (
	"context"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"time"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

// UniswapX 合约地址
var (
	// Mainnet
	ExclusiveDutchOrderReactor = common.HexToAddress("0x6000da47483062A0D734Ba3dc7576Ce6A0B645C4")
	Permit2Address             = common.HexToAddress("0x000000000022D473030F116dDEE9F6B43aC78BA3")

	// API
	UniswapXAPIEndpoint = "https://api.uniswap.org/v2"
)

// DutchOrder UniswapX 荷兰拍卖订单
type DutchOrder struct {
	// 订单信息
	OrderInfo OrderInfo `json:"orderInfo"`

	// 衰减参数
	DecayStartTime uint64 `json:"decayStartTime"`
	DecayEndTime   uint64 `json:"decayEndTime"`

	// 输入
	Input DutchInput `json:"input"`

	// 输出 (可多个)
	Outputs []DutchOutput `json:"outputs"`
}

type OrderInfo struct {
	Reactor                      common.Address `json:"reactor"`
	Swapper                      common.Address `json:"swapper"`
	Nonce                        *big.Int       `json:"nonce"`
	Deadline                     uint64         `json:"deadline"`
	AdditionalValidationContract common.Address `json:"additionalValidationContract"`
	AdditionalValidationData     []byte         `json:"additionalValidationData"`
}

type DutchInput struct {
	Token       common.Address `json:"token"`
	StartAmount *big.Int       `json:"startAmount"`
	EndAmount   *big.Int       `json:"endAmount"`
}

type DutchOutput struct {
	Token       common.Address `json:"token"`
	StartAmount *big.Int       `json:"startAmount"`
	EndAmount   *big.Int       `json:"endAmount"`
	Recipient   common.Address `json:"recipient"`
}

// UniswapXClient UniswapX 客户端
type UniswapXClient struct {
	httpClient *http.Client
	ethClient  *ethclient.Client
	chainID    *big.Int
	privateKey *ecdsa.PrivateKey
	reactorABI abi.ABI
}

// NewUniswapXClient 创建客户端
func NewUniswapXClient(ethClient *ethclient.Client, privateKey *ecdsa.PrivateKey) (*UniswapXClient, error) {
	chainID, err := ethClient.ChainID(context.Background())
	if err != nil {
		return nil, err
	}

	reactorABI, err := abi.JSON(strings.NewReader(ExclusiveDutchOrderReactorABI))
	if err != nil {
		return nil, err
	}

	return &UniswapXClient{
		httpClient: &http.Client{Timeout: 30 * time.Second},
		ethClient:  ethClient,
		chainID:    chainID,
		privateKey: privateKey,
		reactorABI: reactorABI,
	}, nil
}

// GetQuote 获取报价
func (c *UniswapXClient) GetQuote(ctx context.Context, params *QuoteParams) (*Quote, error) {
	url := fmt.Sprintf("%s/quote", UniswapXAPIEndpoint)

	reqBody := map[string]interface{}{
		"tokenInChainId":    c.chainID.Uint64(),
		"tokenOutChainId":   c.chainID.Uint64(),
		"tokenIn":           params.TokenIn.Hex(),
		"tokenOut":          params.TokenOut.Hex(),
		"amount":            params.Amount.String(),
		"type":              "EXACT_INPUT",
		"swapper":           crypto.PubkeyToAddress(c.privateKey.PublicKey).Hex(),
		"slippageTolerance": params.SlippageBPS / 10000.0,
	}

	body, _ := json.Marshal(reqBody)
	req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var quote Quote
	if err := json.NewDecoder(resp.Body).Decode(&quote); err != nil {
		return nil, err
	}

	return &quote, nil
}

// CreateOrder 创建订单
func (c *UniswapXClient) CreateOrder(ctx context.Context, quote *Quote) (*DutchOrder, error) {
	swapper := crypto.PubkeyToAddress(c.privateKey.PublicKey)
	now := uint64(time.Now().Unix())

	order := &DutchOrder{
		OrderInfo: OrderInfo{
			Reactor:  ExclusiveDutchOrderReactor,
			Swapper:  swapper,
			Nonce:    big.NewInt(time.Now().UnixNano()),
			Deadline: now + 300, // 5 分钟有效期
		},
		DecayStartTime: now,
		DecayEndTime:   now + 60, // 1 分钟衰减
		Input: DutchInput{
			Token:       quote.TokenIn,
			StartAmount: quote.AmountIn,
			EndAmount:   quote.AmountIn,
		},
		Outputs: []DutchOutput{
			{
				Token:       quote.TokenOut,
				StartAmount: quote.AmountOut,                                                                         // 起始价格（最优）
				EndAmount:   new(big.Int).Mul(quote.AmountOut, big.NewInt(95)).Div(quote.AmountOut, big.NewInt(100)), // 衰减到 95%
				Recipient:   swapper,
			},
		},
	}

	return order, nil
}

// SignOrder 签名订单
func (c *UniswapXClient) SignOrder(order *DutchOrder) ([]byte, error) {
	// 构建 EIP-712 签名
	domainSeparator := c.getDomainSeparator()
	orderHash := c.hashOrder(order)

	// 构建签名数据
	data := []byte{0x19, 0x01}
	data = append(data, domainSeparator[:]...)
	data = append(data, orderHash[:]...)

	hash := crypto.Keccak256Hash(data)
	signature, err := crypto.Sign(hash.Bytes(), c.privateKey)
	if err != nil {
		return nil, err
	}

	// 调整 v 值
	signature[64] += 27

	return signature, nil
}

// SubmitOrder 提交订单
func (c *UniswapXClient) SubmitOrder(ctx context.Context, order *DutchOrder, signature []byte) error {
	url := fmt.Sprintf("%s/orders", UniswapXAPIEndpoint)

	reqBody := map[string]interface{}{
		"encodedOrder": c.encodeOrder(order),
		"signature":    hexutil.Encode(signature),
		"chainId":      c.chainID.Uint64(),
	}

	body, _ := json.Marshal(reqBody)
	req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("submit order failed: %d", resp.StatusCode)
	}

	return nil
}

func (c *UniswapXClient) getDomainSeparator() common.Hash {
	// EIP-712 domain
	return crypto.Keccak256Hash(
		crypto.Keccak256([]byte("EIP712Domain(string name,uint256 chainId,address verifyingContract)")),
		crypto.Keccak256([]byte("Permit2")),
		common.LeftPadBytes(c.chainID.Bytes(), 32),
		Permit2Address.Bytes(),
	)
}

func (c *UniswapXClient) hashOrder(order *DutchOrder) common.Hash {
	// 简化的订单哈希
	return crypto.Keccak256Hash(
		order.OrderInfo.Reactor.Bytes(),
		order.OrderInfo.Swapper.Bytes(),
		common.LeftPadBytes(order.OrderInfo.Nonce.Bytes(), 32),
		order.Input.Token.Bytes(),
		common.LeftPadBytes(order.Input.StartAmount.Bytes(), 32),
	)
}
```

### 2.2 UniswapX Solver 实现

```go
package solver

import (
	"context"
	"math/big"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// UniswapXSolver UniswapX 求解器
type UniswapXSolver struct {
	ethClient     *ethclient.Client
	dexAggregator DEXAggregator
	priceOracle   PriceOracle
	gasEstimator  GasEstimator

	// 配置
	config SolverConfig

	// 状态
	mu           sync.RWMutex
	activeOrders map[string]*DutchOrder
}

type SolverConfig struct {
	MinProfitUSD    float64 // 最小利润
	MaxGasPrice     *big.Int
	SupportedTokens []common.Address
	ExcludedPools   []common.Address
}

// WatchOrders 监听订单
func (s *UniswapXSolver) WatchOrders(ctx context.Context) error {
	// 轮询 UniswapX API 获取订单
	ticker := time.NewTicker(500 * time.Millisecond)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
			orders, err := s.fetchOpenOrders(ctx)
			if err != nil {
				continue
			}

			for _, order := range orders {
				go s.evaluateAndFill(ctx, order)
			}
		}
	}
}

// fetchOpenOrders 获取开放订单
func (s *UniswapXSolver) fetchOpenOrders(ctx context.Context) ([]*DutchOrder, error) {
	url := fmt.Sprintf("%s/orders?status=open&chainId=1", UniswapXAPIEndpoint)

	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var response struct {
		Orders []*DutchOrder `json:"orders"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&response); err != nil {
		return nil, err
	}

	return response.Orders, nil
}

// evaluateAndFill 评估并填充订单
func (s *UniswapXSolver) evaluateAndFill(ctx context.Context, order *DutchOrder) {
	// 检查是否已在处理
	s.mu.RLock()
	if _, exists := s.activeOrders[order.OrderInfo.Nonce.String()]; exists {
		s.mu.RUnlock()
		return
	}
	s.mu.RUnlock()

	// 标记为处理中
	s.mu.Lock()
	s.activeOrders[order.OrderInfo.Nonce.String()] = order
	s.mu.Unlock()
	defer func() {
		s.mu.Lock()
		delete(s.activeOrders, order.OrderInfo.Nonce.String())
		s.mu.Unlock()
	}()

	// 计算当前价格
	currentOutput := s.calculateCurrentOutput(order)

	// 寻找最优执行路径
	solution, err := s.findOptimalSolution(ctx, order, currentOutput)
	if err != nil {
		return
	}

	// 检查利润
	profit := s.calculateProfit(solution, currentOutput)
	if profit < s.config.MinProfitUSD {
		return
	}

	// 执行填充
	s.executeFill(ctx, order, solution)
}

// calculateCurrentOutput 计算当前荷兰拍卖价格
func (s *UniswapXSolver) calculateCurrentOutput(order *DutchOrder) *big.Int {
	now := uint64(time.Now().Unix())

	if now <= order.DecayStartTime {
		return order.Outputs[0].StartAmount
	}
	if now >= order.DecayEndTime {
		return order.Outputs[0].EndAmount
	}

	// 线性衰减
	totalDecay := order.DecayEndTime - order.DecayStartTime
	elapsed := now - order.DecayStartTime

	startAmount := order.Outputs[0].StartAmount
	endAmount := order.Outputs[0].EndAmount
	decay := new(big.Int).Sub(startAmount, endAmount)

	// currentOutput = startAmount - (decay * elapsed / totalDecay)
	decayAmount := new(big.Int).Mul(decay, big.NewInt(int64(elapsed)))
	decayAmount.Div(decayAmount, big.NewInt(int64(totalDecay)))

	return new(big.Int).Sub(startAmount, decayAmount)
}

// findOptimalSolution 寻找最优解决方案
func (s *UniswapXSolver) findOptimalSolution(ctx context.Context, order *DutchOrder, minOutput *big.Int) (*FillSolution, error) {
	// 获取所有可能的路径
	paths, err := s.dexAggregator.FindPaths(
		order.Input.Token,
		order.Outputs[0].Token,
		order.Input.StartAmount,
	)
	if err != nil {
		return nil, err
	}

	var bestSolution *FillSolution
	var bestOutput *big.Int

	for _, path := range paths {
		// 模拟执行
		output, err := s.simulatePath(ctx, path, order.Input.StartAmount)
		if err != nil {
			continue
		}

		// 检查是否满足最小输出
		if output.Cmp(minOutput) < 0 {
			continue
		}

		// 选择最优
		if bestOutput == nil || output.Cmp(bestOutput) > 0 {
			bestOutput = output
			bestSolution = &FillSolution{
				Path:      path,
				AmountOut: output,
			}
		}
	}

	if bestSolution == nil {
		return nil, fmt.Errorf("no profitable solution found")
	}

	return bestSolution, nil
}

// executeFill 执行填充
func (s *UniswapXSolver) executeFill(ctx context.Context, order *DutchOrder, solution *FillSolution) error {
	// 构建填充交易
	fillData, err := s.buildFillCalldata(order, solution)
	if err != nil {
		return err
	}

	// 估算 gas
	gasLimit, err := s.gasEstimator.Estimate(ctx, fillData)
	if err != nil {
		return err
	}

	// 获取当前 gas 价格
	gasPrice, err := s.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return err
	}

	// 发送交易
	tx := types.NewTransaction(
		0, // nonce 由签名者填充
		ExclusiveDutchOrderReactor,
		big.NewInt(0),
		gasLimit,
		gasPrice,
		fillData,
	)

	// 签名并发送
	signedTx, err := s.signTransaction(tx)
	if err != nil {
		return err
	}

	return s.ethClient.SendTransaction(ctx, signedTx)
}

// buildFillCalldata 构建填充 calldata
func (s *UniswapXSolver) buildFillCalldata(order *DutchOrder, solution *FillSolution) ([]byte, error) {
	// execute(SignedOrder calldata order, bytes calldata fillData)
	return s.reactorABI.Pack(
		"execute",
		order,
		solution.FillData,
	)
}

type FillSolution struct {
	Path      []SwapStep
	AmountOut *big.Int
	FillData  []byte
}
```

## 3. CoW Protocol 集成

### 3.1 CoW Protocol 概述

```go
package cow

import (
	"context"
	"crypto/ecdsa"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// CoW Protocol 地址
var (
	GPv2Settlement   = common.HexToAddress("0x9008D19f58AAbD9eD0D60971565AA8510560ab41")
	GPv2VaultRelayer = common.HexToAddress("0xC92E8bdf79f0507f65a392b0ab4667716BFE0110")

	CoWAPIEndpoint = "https://api.cow.fi/mainnet/api/v1"
)

// CoWOrder CoW Protocol 订单
type CoWOrder struct {
	SellToken         common.Address `json:"sellToken"`
	BuyToken          common.Address `json:"buyToken"`
	Receiver          common.Address `json:"receiver"`
	SellAmount        *big.Int       `json:"sellAmount"`
	BuyAmount         *big.Int       `json:"buyAmount"`
	ValidTo           uint32         `json:"validTo"`
	AppData           common.Hash    `json:"appData"`
	FeeAmount         *big.Int       `json:"feeAmount"`
	Kind              string         `json:"kind"` // "sell" or "buy"
	PartiallyFillable bool           `json:"partiallyFillable"`
	SellTokenBalance  string         `json:"sellTokenBalance"` // "erc20", "external", "internal"
	BuyTokenBalance   string         `json:"buyTokenBalance"`
}

// CoWClient CoW Protocol 客户端
type CoWClient struct {
	httpClient *http.Client
	privateKey *ecdsa.PrivateKey
	chainID    *big.Int
}

// NewCoWClient 创建客户端
func NewCoWClient(privateKey *ecdsa.PrivateKey, chainID *big.Int) *CoWClient {
	return &CoWClient{
		httpClient: &http.Client{Timeout: 30 * time.Second},
		privateKey: privateKey,
		chainID:    chainID,
	}
}

// GetQuote 获取报价
func (c *CoWClient) GetQuote(ctx context.Context, params *QuoteParams) (*CoWQuote, error) {
	url := fmt.Sprintf("%s/quote", CoWAPIEndpoint)

	from := crypto.PubkeyToAddress(c.privateKey.PublicKey)

	reqBody := map[string]interface{}{
		"sellToken":           params.SellToken.Hex(),
		"buyToken":            params.BuyToken.Hex(),
		"receiver":            from.Hex(),
		"validTo":             time.Now().Unix() + 1800, // 30 分钟
		"appData":             "0x0000000000000000000000000000000000000000000000000000000000000000",
		"partiallyFillable":   false,
		"sellTokenBalance":    "erc20",
		"buyTokenBalance":     "erc20",
		"from":                from.Hex(),
		"kind":                "sell",
		"sellAmountBeforeFee": params.Amount.String(),
	}

	body, _ := json.Marshal(reqBody)
	req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var quote CoWQuote
	if err := json.NewDecoder(resp.Body).Decode(&quote); err != nil {
		return nil, err
	}

	return &quote, nil
}

// CreateOrder 创建订单
func (c *CoWClient) CreateOrder(ctx context.Context, quote *CoWQuote) (*CoWOrder, error) {
	from := crypto.PubkeyToAddress(c.privateKey.PublicKey)

	order := &CoWOrder{
		SellToken:         quote.SellToken,
		BuyToken:          quote.BuyToken,
		Receiver:          from,
		SellAmount:        quote.SellAmount,
		BuyAmount:         quote.BuyAmount,
		ValidTo:           uint32(time.Now().Unix() + 1800),
		AppData:           common.Hash{},
		FeeAmount:         quote.FeeAmount,
		Kind:              "sell",
		PartiallyFillable: false,
		SellTokenBalance:  "erc20",
		BuyTokenBalance:   "erc20",
	}

	return order, nil
}

// SignOrder 签名订单 (EIP-712)
func (c *CoWClient) SignOrder(order *CoWOrder) ([]byte, error) {
	// 构建 domain separator
	domainSeparator := c.buildDomainSeparator()

	// 构建 struct hash
	structHash := c.hashOrder(order)

	// 构建签名数据
	data := []byte{0x19, 0x01}
	data = append(data, domainSeparator[:]...)
	data = append(data, structHash[:]...)

	hash := crypto.Keccak256Hash(data)
	signature, err := crypto.Sign(hash.Bytes(), c.privateKey)
	if err != nil {
		return nil, err
	}

	// 调整 v 值并设置签名方案
	signature[64] += 27

	// CoW 使用 EIP-712 签名方案 (0x00 前缀)
	return append(signature, 0x00), nil
}

// SubmitOrder 提交订单
func (c *CoWClient) SubmitOrder(ctx context.Context, order *CoWOrder, signature []byte) (string, error) {
	url := fmt.Sprintf("%s/orders", CoWAPIEndpoint)

	from := crypto.PubkeyToAddress(c.privateKey.PublicKey)

	reqBody := map[string]interface{}{
		"sellToken":         order.SellToken.Hex(),
		"buyToken":          order.BuyToken.Hex(),
		"receiver":          order.Receiver.Hex(),
		"sellAmount":        order.SellAmount.String(),
		"buyAmount":         order.BuyAmount.String(),
		"validTo":           order.ValidTo,
		"appData":           order.AppData.Hex(),
		"feeAmount":         order.FeeAmount.String(),
		"kind":              order.Kind,
		"partiallyFillable": order.PartiallyFillable,
		"sellTokenBalance":  order.SellTokenBalance,
		"buyTokenBalance":   order.BuyTokenBalance,
		"signingScheme":     "eip712",
		"signature":         hexutil.Encode(signature),
		"from":              from.Hex(),
	}

	body, _ := json.Marshal(reqBody)
	req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusCreated {
		var errResp map[string]interface{}
		json.NewDecoder(resp.Body).Decode(&errResp)
		return "", fmt.Errorf("submit failed: %v", errResp)
	}

	// 返回订单 UID
	var uid string
	json.NewDecoder(resp.Body).Decode(&uid)

	return uid, nil
}

func (c *CoWClient) buildDomainSeparator() common.Hash {
	return crypto.Keccak256Hash(
		crypto.Keccak256([]byte("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)")),
		crypto.Keccak256([]byte("Gnosis Protocol")),
		crypto.Keccak256([]byte("v2")),
		common.LeftPadBytes(c.chainID.Bytes(), 32),
		GPv2Settlement.Bytes(),
	)
}

func (c *CoWClient) hashOrder(order *CoWOrder) common.Hash {
	orderTypeHash := crypto.Keccak256Hash([]byte(
		"Order(address sellToken,address buyToken,address receiver,uint256 sellAmount,uint256 buyAmount,uint32 validTo,bytes32 appData,uint256 feeAmount,string kind,bool partiallyFillable,string sellTokenBalance,string buyTokenBalance)",
	))

	return crypto.Keccak256Hash(
		orderTypeHash.Bytes(),
		common.LeftPadBytes(order.SellToken.Bytes(), 32),
		common.LeftPadBytes(order.BuyToken.Bytes(), 32),
		common.LeftPadBytes(order.Receiver.Bytes(), 32),
		common.LeftPadBytes(order.SellAmount.Bytes(), 32),
		common.LeftPadBytes(order.BuyAmount.Bytes(), 32),
		common.LeftPadBytes(big.NewInt(int64(order.ValidTo)).Bytes(), 32),
		order.AppData.Bytes(),
		common.LeftPadBytes(order.FeeAmount.Bytes(), 32),
		crypto.Keccak256([]byte(order.Kind)),
		boolToBytes32(order.PartiallyFillable),
		crypto.Keccak256([]byte(order.SellTokenBalance)),
		crypto.Keccak256([]byte(order.BuyTokenBalance)),
	)
}

func boolToBytes32(b bool) []byte {
	result := make([]byte, 32)
	if b {
		result[31] = 1
	}
	return result
}
```

### 3.2 CoW Solver 实现

```go
package cow

import (
	"context"
	"math/big"
	"sync"
)

// CoWBatch 批量订单
type CoWBatch struct {
	Orders    []*CoWOrder
	Solutions []*CoWSolution
}

// CoWSolution 解决方案
type CoWSolution struct {
	OrderUID       string
	ExecutedAmount *big.Int
	Trades         []Trade
}

type Trade struct {
	Order              *CoWOrder
	ExecutedSellAmount *big.Int
	ExecutedBuyAmount  *big.Int
}

// CoWSolver CoW 求解器
type CoWSolver struct {
	dexAggregator DEXAggregator
	config        SolverConfig

	mu           sync.RWMutex
	currentBatch *CoWBatch
}

// SolveBatch 求解批量订单
func (s *CoWSolver) SolveBatch(ctx context.Context, orders []*CoWOrder) (*BatchSolution, error) {
	// 1. 找出匹配订单 (CoW - Coincidence of Wants)
	cowMatches := s.findCoWMatches(orders)

	// 2. 对剩余订单通过 AMM 执行
	ammSolutions := s.solveViaAMM(ctx, orders, cowMatches)

	// 3. 组合解决方案
	solution := &BatchSolution{
		CoWTrades:    cowMatches,
		AMMTrades:    ammSolutions,
		TotalSurplus: s.calculateSurplus(cowMatches, ammSolutions),
	}

	return solution, nil
}

// findCoWMatches 寻找订单匹配 (Coincidence of Wants)
func (s *CoWSolver) findCoWMatches(orders []*CoWOrder) []CoWMatch {
	var matches []CoWMatch

	// 按代币对分组
	buyOrders := make(map[string][]*CoWOrder)  // sellToken -> orders
	sellOrders := make(map[string][]*CoWOrder) // buyToken -> orders

	for _, order := range orders {
		key := order.SellToken.Hex() + "-" + order.BuyToken.Hex()
		sellOrders[key] = append(sellOrders[key], order)

		reverseKey := order.BuyToken.Hex() + "-" + order.SellToken.Hex()
		buyOrders[reverseKey] = append(buyOrders[reverseKey], order)
	}

	// 寻找匹配
	for key, sells := range sellOrders {
		if buys, ok := buyOrders[key]; ok {
			// 有匹配的订单对
			match := s.matchOrders(sells, buys)
			if match != nil {
				matches = append(matches, *match)
			}
		}
	}

	return matches
}

// matchOrders 匹配订单
func (s *CoWSolver) matchOrders(sells, buys []*CoWOrder) *CoWMatch {
	// 简化：取第一对匹配的订单
	for _, sell := range sells {
		for _, buy := range buys {
			// 检查价格是否匹配
			// sell 要卖 A 买 B，buy 要卖 B 买 A
			// sell 的价格: sellAmount / buyAmount (A per B)
			// buy 的价格: buyAmount / sellAmount (A per B，对于 buy 是反向)

			sellPrice := new(big.Float).Quo(
				new(big.Float).SetInt(sell.SellAmount),
				new(big.Float).SetInt(sell.BuyAmount),
			)

			buyPrice := new(big.Float).Quo(
				new(big.Float).SetInt(buy.BuyAmount),
				new(big.Float).SetInt(buy.SellAmount),
			)

			// 如果 sellPrice <= buyPrice，可以匹配
			if sellPrice.Cmp(buyPrice) <= 0 {
				// 计算匹配数量
				matchAmount := minBigInt(sell.SellAmount, buy.BuyAmount)

				return &CoWMatch{
					SellOrder:     sell,
					BuyOrder:      buy,
					MatchAmount:   matchAmount,
					ClearingPrice: sellPrice,
				}
			}
		}
	}

	return nil
}

// solveViaAMM 通过 AMM 求解剩余订单
func (s *CoWSolver) solveViaAMM(ctx context.Context, orders []*CoWOrder, cowMatches []CoWMatch) []AMMSolution {
	var solutions []AMMSolution

	// 标记已匹配的订单
	matched := make(map[*CoWOrder]bool)
	for _, match := range cowMatches {
		matched[match.SellOrder] = true
		matched[match.BuyOrder] = true
	}

	// 对未匹配订单寻找 AMM 路径
	for _, order := range orders {
		if matched[order] {
			continue
		}

		path, err := s.dexAggregator.FindBestPath(
			order.SellToken,
			order.BuyToken,
			order.SellAmount,
		)
		if err != nil {
			continue
		}

		// 检查输出是否满足要求
		if path.AmountOut.Cmp(order.BuyAmount) < 0 {
			continue
		}

		solutions = append(solutions, AMMSolution{
			Order: order,
			Path:  path,
		})
	}

	return solutions
}

type CoWMatch struct {
	SellOrder     *CoWOrder
	BuyOrder      *CoWOrder
	MatchAmount   *big.Int
	ClearingPrice *big.Float
}

type AMMSolution struct {
	Order *CoWOrder
	Path  *SwapPath
}

type BatchSolution struct {
	CoWTrades    []CoWMatch
	AMMTrades    []AMMSolution
	TotalSurplus *big.Int
}

func minBigInt(a, b *big.Int) *big.Int {
	if a.Cmp(b) < 0 {
		return new(big.Int).Set(a)
	}
	return new(big.Int).Set(b)
}
```

## 4. 1inch Fusion 集成

### 4.1 1inch Fusion API

```go
package fusion

import (
	"context"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

var (
	FusionAPIEndpoint = "https://fusion.1inch.io/v1.0/1" // Mainnet
	FusionSettlement  = common.HexToAddress("0xa88800cd213da5ae406ce248380802bd16b31c0b")
)

// FusionOrder 1inch Fusion 订单
type FusionOrder struct {
	Salt          *big.Int       `json:"salt"`
	MakerAsset    common.Address `json:"makerAsset"`
	TakerAsset    common.Address `json:"takerAsset"`
	Maker         common.Address `json:"maker"`
	Receiver      common.Address `json:"receiver"`
	AllowedSender common.Address `json:"allowedSender"`
	MakingAmount  *big.Int       `json:"makingAmount"`
	TakingAmount  *big.Int       `json:"takingAmount"`
	Offsets       *big.Int       `json:"offsets"`
	Interactions  []byte         `json:"interactions"`
}

// AuctionDetails 拍卖详情
type AuctionDetails struct {
	StartTime       uint64  `json:"startTime"`
	Duration        uint64  `json:"duration"`
	InitialRateBump uint64  `json:"initialRateBump"`
	Points          []Point `json:"points"`
}

type Point struct {
	Delay       uint64 `json:"delay"`
	Coefficient uint64 `json:"coefficient"`
}

// FusionClient 1inch Fusion 客户端
type FusionClient struct {
	httpClient *http.Client
	apiKey     string
}

// NewFusionClient 创建客户端
func NewFusionClient(apiKey string) *FusionClient {
	return &FusionClient{
		httpClient: &http.Client{Timeout: 30 * time.Second},
		apiKey:     apiKey,
	}
}

// GetQuote 获取 Fusion 报价
func (c *FusionClient) GetQuote(ctx context.Context, params *QuoteParams) (*FusionQuote, error) {
	url := fmt.Sprintf("%s/quote/receive?fromTokenAddress=%s&toTokenAddress=%s&amount=%s&walletAddress=%s",
		FusionAPIEndpoint,
		params.FromToken.Hex(),
		params.ToToken.Hex(),
		params.Amount.String(),
		params.WalletAddress.Hex(),
	)

	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	req.Header.Set("Authorization", "Bearer "+c.apiKey)

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var quote FusionQuote
	if err := json.NewDecoder(resp.Body).Decode(&quote); err != nil {
		return nil, err
	}

	return &quote, nil
}

// CreateOrder 创建 Fusion 订单
func (c *FusionClient) CreateOrder(ctx context.Context, quote *FusionQuote, privateKey *ecdsa.PrivateKey) (*FusionOrder, error) {
	maker := crypto.PubkeyToAddress(privateKey.PublicKey)

	order := &FusionOrder{
		Salt:         big.NewInt(time.Now().UnixNano()),
		MakerAsset:   quote.FromToken,
		TakerAsset:   quote.ToToken,
		Maker:        maker,
		Receiver:     maker,
		MakingAmount: quote.FromAmount,
		TakingAmount: quote.ToAmount,
	}

	return order, nil
}

// SubmitOrder 提交订单
func (c *FusionClient) SubmitOrder(ctx context.Context, order *FusionOrder, signature []byte) error {
	url := fmt.Sprintf("%s/order/submit", FusionAPIEndpoint)

	reqBody := map[string]interface{}{
		"order":     order,
		"signature": hexutil.Encode(signature),
	}

	body, _ := json.Marshal(reqBody)
	req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", "Bearer "+c.apiKey)

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("submit failed: %d", resp.StatusCode)
	}

	return nil
}

// WatchOrders 监听 Fusion 订单（作为 Resolver）
func (c *FusionClient) WatchOrders(ctx context.Context) (<-chan *FusionOrder, error) {
	orders := make(chan *FusionOrder, 100)

	go func() {
		defer close(orders)
		ticker := time.NewTicker(500 * time.Millisecond)
		defer ticker.Stop()

		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				activeOrders, err := c.getActiveOrders(ctx)
				if err != nil {
					continue
				}

				for _, order := range activeOrders {
					select {
					case orders <- order:
					default:
						// 通道满，跳过
					}
				}
			}
		}
	}()

	return orders, nil
}

func (c *FusionClient) getActiveOrders(ctx context.Context) ([]*FusionOrder, error) {
	url := fmt.Sprintf("%s/order/active", FusionAPIEndpoint)

	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	req.Header.Set("Authorization", "Bearer "+c.apiKey)

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result struct {
		Orders []*FusionOrder `json:"orders"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	return result.Orders, nil
}

type FusionQuote struct {
	FromToken  common.Address `json:"fromToken"`
	ToToken    common.Address `json:"toToken"`
	FromAmount *big.Int       `json:"fromAmount"`
	ToAmount   *big.Int       `json:"toAmount"`
	Preset     string         `json:"preset"` // "fast", "medium", "slow"
}

type QuoteParams struct {
	FromToken     common.Address
	ToToken       common.Address
	Amount        *big.Int
	WalletAddress common.Address
}
```

## 5. 统一意图聚合器

### 5.1 多协议聚合

```go
package aggregator

import (
	"context"
	"math/big"
	"sort"
	"sync"

	"github.com/ethereum/go-ethereum/common"
)

// IntentAggregator 意图聚合器
type IntentAggregator struct {
	uniswapX *UniswapXClient
	cow      *CoWClient
	fusion   *FusionClient

	priceOracle PriceOracle
}

// NewIntentAggregator 创建聚合器
func NewIntentAggregator(
	uniswapX *UniswapXClient,
	cow *CoWClient,
	fusion *FusionClient,
	priceOracle PriceOracle,
) *IntentAggregator {
	return &IntentAggregator{
		uniswapX:    uniswapX,
		cow:         cow,
		fusion:      fusion,
		priceOracle: priceOracle,
	}
}

// Quote 报价结果
type AggregatedQuote struct {
	Protocol     string        `json:"protocol"`
	AmountOut    *big.Int      `json:"amountOut"`
	GasEstimate  uint64        `json:"gasEstimate"`
	Fee          *big.Int      `json:"fee"`
	ExpectedTime time.Duration `json:"expectedTime"`
	Score        float64       `json:"score"`
}

// GetBestQuote 获取最优报价
func (a *IntentAggregator) GetBestQuote(ctx context.Context, params *SwapParams) (*AggregatedQuote, error) {
	// 并发获取所有协议报价
	var wg sync.WaitGroup
	quotes := make(chan *AggregatedQuote, 3)

	// UniswapX
	wg.Add(1)
	go func() {
		defer wg.Done()
		quote, err := a.getUniswapXQuote(ctx, params)
		if err == nil {
			quotes <- quote
		}
	}()

	// CoW Protocol
	wg.Add(1)
	go func() {
		defer wg.Done()
		quote, err := a.getCoWQuote(ctx, params)
		if err == nil {
			quotes <- quote
		}
	}()

	// 1inch Fusion
	wg.Add(1)
	go func() {
		defer wg.Done()
		quote, err := a.getFusionQuote(ctx, params)
		if err == nil {
			quotes <- quote
		}
	}()

	// 等待所有报价
	go func() {
		wg.Wait()
		close(quotes)
	}()

	// 收集并排序
	var allQuotes []*AggregatedQuote
	for quote := range quotes {
		allQuotes = append(allQuotes, quote)
	}

	if len(allQuotes) == 0 {
		return nil, fmt.Errorf("no quotes available")
	}

	// 计算评分并排序
	for _, quote := range allQuotes {
		quote.Score = a.calculateScore(quote, params)
	}

	sort.Slice(allQuotes, func(i, j int) bool {
		return allQuotes[i].Score > allQuotes[j].Score
	})

	return allQuotes[0], nil
}

// calculateScore 计算综合评分
func (a *IntentAggregator) calculateScore(quote *AggregatedQuote, params *SwapParams) float64 {
	// 获取 ETH 价格用于计算 gas 成本
	ethPrice, _ := a.priceOracle.GetPriceUSD(common.Address{}) // ETH

	// 获取输出代币价格
	outPrice, _ := a.priceOracle.GetPriceUSD(params.TokenOut)

	// 输出价值 (USD)
	outValue := new(big.Float).Mul(
		new(big.Float).SetInt(quote.AmountOut),
		big.NewFloat(outPrice),
	)
	outValueFloat, _ := outValue.Float64()

	// Gas 成本 (USD)
	gasCost := float64(quote.GasEstimate) * 30e9 / 1e18 * ethPrice // 假设 30 gwei

	// 费用 (USD)
	feeValue := new(big.Float).Mul(
		new(big.Float).SetInt(quote.Fee),
		big.NewFloat(outPrice),
	)
	feeValueFloat, _ := feeValue.Float64()

	// 时间惩罚（越快越好）
	timePenalty := float64(quote.ExpectedTime.Seconds()) * 0.001

	// 综合评分 = 输出价值 - 成本 - 时间惩罚
	return outValueFloat - gasCost - feeValueFloat - timePenalty
}

func (a *IntentAggregator) getUniswapXQuote(ctx context.Context, params *SwapParams) (*AggregatedQuote, error) {
	quote, err := a.uniswapX.GetQuote(ctx, &QuoteParams{
		TokenIn:     params.TokenIn,
		TokenOut:    params.TokenOut,
		Amount:      params.AmountIn,
		SlippageBPS: params.SlippageBPS,
	})
	if err != nil {
		return nil, err
	}

	return &AggregatedQuote{
		Protocol:     "UniswapX",
		AmountOut:    quote.AmountOut,
		GasEstimate:  0, // 用户不付 gas
		Fee:          quote.Fee,
		ExpectedTime: 30 * time.Second,
	}, nil
}

func (a *IntentAggregator) getCoWQuote(ctx context.Context, params *SwapParams) (*AggregatedQuote, error) {
	quote, err := a.cow.GetQuote(ctx, &QuoteParams{
		SellToken: params.TokenIn,
		BuyToken:  params.TokenOut,
		Amount:    params.AmountIn,
	})
	if err != nil {
		return nil, err
	}

	return &AggregatedQuote{
		Protocol:     "CoW Protocol",
		AmountOut:    quote.BuyAmount,
		GasEstimate:  0,
		Fee:          quote.FeeAmount,
		ExpectedTime: 60 * time.Second, // CoW 批量结算较慢
	}, nil
}

func (a *IntentAggregator) getFusionQuote(ctx context.Context, params *SwapParams) (*AggregatedQuote, error) {
	quote, err := a.fusion.GetQuote(ctx, &QuoteParams{
		FromToken:     params.TokenIn,
		ToToken:       params.TokenOut,
		Amount:        params.AmountIn,
		WalletAddress: params.Sender,
	})
	if err != nil {
		return nil, err
	}

	return &AggregatedQuote{
		Protocol:     "1inch Fusion",
		AmountOut:    quote.ToAmount,
		GasEstimate:  0,
		Fee:          big.NewInt(0), // Fusion 通过拍卖收费
		ExpectedTime: 45 * time.Second,
	}, nil
}

type SwapParams struct {
	TokenIn     common.Address
	TokenOut    common.Address
	AmountIn    *big.Int
	SlippageBPS uint64
	Sender      common.Address
}
```

### 5.2 意图套利策略

```go
package strategy

import (
	"context"
	"math/big"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

// IntentArbitrageStrategy 意图套利策略
type IntentArbitrageStrategy struct {
	aggregator    *IntentAggregator
	dexAggregator DEXAggregator
	priceOracle   PriceOracle

	config StrategyConfig
}

type StrategyConfig struct {
	MinProfitUSD    float64
	MaxPositionUSD  float64
	MonitorInterval time.Duration
}

// Monitor 监控套利机会
func (s *IntentArbitrageStrategy) Monitor(ctx context.Context) error {
	ticker := time.NewTicker(s.config.MonitorInterval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
			opportunities := s.findOpportunities(ctx)
			for _, opp := range opportunities {
				go s.executeOpportunity(ctx, opp)
			}
		}
	}
}

// findOpportunities 寻找套利机会
func (s *IntentArbitrageStrategy) findOpportunities(ctx context.Context) []*IntentOpportunity {
	var opportunities []*IntentOpportunity

	// 获取热门代币对
	pairs := s.getHotPairs()

	var wg sync.WaitGroup
	var mu sync.Mutex

	for _, pair := range pairs {
		wg.Add(1)
		go func(p TokenPair) {
			defer wg.Done()

			opp := s.checkPairArbitrage(ctx, p)
			if opp != nil {
				mu.Lock()
				opportunities = append(opportunities, opp)
				mu.Unlock()
			}
		}(pair)
	}

	wg.Wait()
	return opportunities
}

// checkPairArbitrage 检查代币对套利
func (s *IntentArbitrageStrategy) checkPairArbitrage(ctx context.Context, pair TokenPair) *IntentOpportunity {
	// 获取意图协议报价
	intentQuote, err := s.aggregator.GetBestQuote(ctx, &SwapParams{
		TokenIn:     pair.TokenA,
		TokenOut:    pair.TokenB,
		AmountIn:    pair.TestAmount,
		SlippageBPS: 50, // 0.5%
	})
	if err != nil {
		return nil
	}

	// 获取 DEX 直接交易报价
	dexQuote, err := s.dexAggregator.GetQuote(ctx, pair.TokenA, pair.TokenB, pair.TestAmount)
	if err != nil {
		return nil
	}

	// 比较价格差异
	intentPrice := new(big.Float).Quo(
		new(big.Float).SetInt(intentQuote.AmountOut),
		new(big.Float).SetInt(pair.TestAmount),
	)
	dexPrice := new(big.Float).Quo(
		new(big.Float).SetInt(dexQuote.AmountOut),
		new(big.Float).SetInt(pair.TestAmount),
	)

	// 计算价差
	var priceDiff *big.Float
	var direction string

	if intentPrice.Cmp(dexPrice) > 0 {
		// Intent 更便宜，通过 DEX 买，通过 Intent 卖
		priceDiff = new(big.Float).Sub(intentPrice, dexPrice)
		direction = "DEX->Intent"
	} else {
		// DEX 更便宜，通过 Intent 买，通过 DEX 卖
		priceDiff = new(big.Float).Sub(dexPrice, intentPrice)
		direction = "Intent->DEX"
	}

	// 计算利润
	profitRatio, _ := new(big.Float).Quo(priceDiff, dexPrice).Float64()

	if profitRatio < 0.001 { // 小于 0.1% 不值得
		return nil
	}

	return &IntentOpportunity{
		Pair:        pair,
		Direction:   direction,
		IntentQuote: intentQuote,
		DEXQuote:    dexQuote,
		ProfitRatio: profitRatio,
	}
}

// executeOpportunity 执行套利
func (s *IntentArbitrageStrategy) executeOpportunity(ctx context.Context, opp *IntentOpportunity) error {
	// 计算最优输入金额
	optimalInput := s.calculateOptimalInput(opp)

	if opp.Direction == "DEX->Intent" {
		// 1. 通过 DEX 买入 TokenB
		// 2. 通过 Intent 协议卖出 TokenB 换回 TokenA

		// DEX 买入
		dexTx, err := s.dexAggregator.Swap(ctx, opp.Pair.TokenA, opp.Pair.TokenB, optimalInput)
		if err != nil {
			return err
		}

		// 等待确认
		receipt, err := s.waitForReceipt(ctx, dexTx)
		if err != nil {
			return err
		}

		// 获取实际买入数量
		boughtAmount := s.extractAmountFromReceipt(receipt)

		// 通过 Intent 卖出
		return s.sellViaIntent(ctx, opp.Pair.TokenB, opp.Pair.TokenA, boughtAmount)

	} else {
		// 1. 通过 Intent 协议买入 TokenB
		// 2. 通过 DEX 卖出 TokenB 换回 TokenA

		// Intent 买入
		err := s.buyViaIntent(ctx, opp.Pair.TokenA, opp.Pair.TokenB, optimalInput)
		if err != nil {
			return err
		}

		// 等待 Intent 完成（可能需要轮询）
		boughtAmount, err := s.waitForIntentFill(ctx, opp.Pair.TokenB)
		if err != nil {
			return err
		}

		// DEX 卖出
		_, err = s.dexAggregator.Swap(ctx, opp.Pair.TokenB, opp.Pair.TokenA, boughtAmount)
		return err
	}
}

type TokenPair struct {
	TokenA     common.Address
	TokenB     common.Address
	TestAmount *big.Int
}

type IntentOpportunity struct {
	Pair        TokenPair
	Direction   string
	IntentQuote *AggregatedQuote
	DEXQuote    *DEXQuote
	ProfitRatio float64
}

func (s *IntentArbitrageStrategy) getHotPairs() []TokenPair {
	// 返回热门代币对
	return []TokenPair{
		{
			TokenA:     common.HexToAddress("0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"), // WETH
			TokenB:     common.HexToAddress("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"), // USDC
			TestAmount: big.NewInt(1e18),                                                  // 1 ETH
		},
		{
			TokenA:     common.HexToAddress("0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"), // WETH
			TokenB:     common.HexToAddress("0xdAC17F958D2ee523a2206206994597C13D831ec7"), // USDT
			TestAmount: big.NewInt(1e18),
		},
		// 更多热门对...
	}
}
```

## 6. 总结

### 6.1 意图协议对比

| 协议               | 特点                    | 结算时间       | 适用场景        |
|------------------|-----------------------|------------|-------------|
| **UniswapX**     | 荷兰拍卖、Exclusive Filler | 快 (< 30s)  | 大额交易、低滑点需求  |
| **CoW Protocol** | 批量拍卖、CoW 匹配           | 中 (30-60s) | 减少 MEV、批量交易 |
| **1inch Fusion** | 荷兰拍卖、Resolver 网络      | 中 (30-45s) | 通用交易        |

### 6.2 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     意图协议生态系统                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  用户层                                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  签名意图 (不上链) → 指定输入/输出/约束                │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│  协议层                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ UniswapX │  │   CoW    │  │  Fusion  │                 │
│  │  API     │  │   API    │  │   API    │                 │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                 │
│       │             │             │                        │
│       └─────────────┼─────────────┘                        │
│                     ▼                                       │
│  Solver/Resolver 层                                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  竞争求解 → 路由优化 → 提交链上                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│  执行层                                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Settlement Contract → 验证 → 转账                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 最佳实践

1. **作为用户**: 使用聚合器比较多个协议报价
2. **作为 Solver**: 优化路由算法、降低延迟
3. **作为套利者**: 监控 Intent 与 DEX 价差

意图协议代表了 DeFi 交易的未来方向，将复杂的执行逻辑从用户转移到专业的求解者，实现更好的价格发现和 MEV 保护。
