# Go-Ethereum EVM 深入解析：账户抽象 (ERC-4337)

## 概述

账户抽象 (Account Abstraction, AA) 通过 ERC-4337 实现了无需协议层更改的智能合约钱包。它引入了
UserOperation、Bundler、Paymaster 等新概念，为套利机器人提供了新的机会和挑战。

## 1. ERC-4337 架构

### 1.1 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                     ERC-4337 架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐                                           │
│  │    用户      │                                           │
│  │ Smart Wallet │                                           │
│  └──────┬───────┘                                           │
│         │ 签名 UserOp                                        │
│         ▼                                                   │
│  ┌──────────────┐    ┌──────────────┐                       │
│  │   Bundler    │───▶│  EntryPoint  │                       │
│  │  (打包者)    │    │   (入口点)   │                         │
│  └──────────────┘    └──────┬───────┘                       │
│                             │                               │
│         ┌───────────────────┼───────────────────┐           │
│         ▼                   ▼                   ▼           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Account    │    │  Paymaster   │    │  Aggregator  │   │
│  │  (智能账户)   │    │  (支付者)     │    │  (聚合器)     │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心数据结构

```go
package aa

import (
	"math/big"

	"github.com/ethereum/go-ethereum/common"
)

// UserOperation ERC-4337 用户操作
type UserOperation struct {
	Sender               common.Address `json:"sender"`
	Nonce                *big.Int       `json:"nonce"`
	InitCode             []byte         `json:"initCode"`
	CallData             []byte         `json:"callData"`
	CallGasLimit         *big.Int       `json:"callGasLimit"`
	VerificationGasLimit *big.Int       `json:"verificationGasLimit"`
	PreVerificationGas   *big.Int       `json:"preVerificationGas"`
	MaxFeePerGas         *big.Int       `json:"maxFeePerGas"`
	MaxPriorityFeePerGas *big.Int       `json:"maxPriorityFeePerGas"`
	PaymasterAndData     []byte         `json:"paymasterAndData"`
	Signature            []byte         `json:"signature"`
}

// PackedUserOperation ERC-4337 v0.7 打包格式
type PackedUserOperation struct {
	Sender             common.Address `json:"sender"`
	Nonce              *big.Int       `json:"nonce"`
	InitCode           []byte         `json:"initCode"`
	CallData           []byte         `json:"callData"`
	AccountGasLimits   [32]byte       `json:"accountGasLimits"` // verificationGasLimit (16) + callGasLimit (16)
	PreVerificationGas *big.Int       `json:"preVerificationGas"`
	GasFees            [32]byte       `json:"gasFees"` // maxPriorityFeePerGas (16) + maxFeePerGas (16)
	PaymasterAndData   []byte         `json:"paymasterAndData"`
	Signature          []byte         `json:"signature"`
}

// EntryPoint 合约地址 (v0.6 和 v0.7)
var (
	EntryPointV06 = common.HexToAddress("0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789")
	EntryPointV07 = common.HexToAddress("0x0000000071727De22E5E9d8BAf0edAc6f37da032")
)

// UserOpHash 计算 UserOperation 哈希
func (op *UserOperation) Hash(entryPoint common.Address, chainID *big.Int) common.Hash {
	// 打包 UserOp 数据
	packed := op.Pack()

	// keccak256(packed)
	opHash := crypto.Keccak256Hash(packed)

	// keccak256(abi.encode(opHash, entryPoint, chainId))
	encoded := make([]byte, 96)
	copy(encoded[0:32], opHash.Bytes())
	copy(encoded[32:64], common.LeftPadBytes(entryPoint.Bytes(), 32))
	copy(encoded[64:96], common.LeftPadBytes(chainID.Bytes(), 32))

	return crypto.Keccak256Hash(encoded)
}

// Pack 打包 UserOperation
func (op *UserOperation) Pack() []byte {
	return crypto.Keccak256(
		common.LeftPadBytes(op.Sender.Bytes(), 32),
		common.LeftPadBytes(op.Nonce.Bytes(), 32),
		crypto.Keccak256(op.InitCode),
		crypto.Keccak256(op.CallData),
		common.LeftPadBytes(op.CallGasLimit.Bytes(), 32),
		common.LeftPadBytes(op.VerificationGasLimit.Bytes(), 32),
		common.LeftPadBytes(op.PreVerificationGas.Bytes(), 32),
		common.LeftPadBytes(op.MaxFeePerGas.Bytes(), 32),
		common.LeftPadBytes(op.MaxPriorityFeePerGas.Bytes(), 32),
		crypto.Keccak256(op.PaymasterAndData),
	)
}
```

## 2. Bundler 实现

### 2.1 Bundler 服务

```go
package bundler

import (
	"context"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Bundler ERC-4337 Bundler 实现
type Bundler struct {
	ethClient     *ethclient.Client
	entryPoint    common.Address
	entryPointABI abi.ABI

	// mempool
	mempool   *UserOpMempool
	mempoolMu sync.RWMutex

	// 配置
	config BundlerConfig

	// 私钥
	privateKey *ecdsa.PrivateKey
}

type BundlerConfig struct {
	ChainID            *big.Int
	MinStake           *big.Int
	MinUnstakeDelay    uint64
	MaxBundleSize      int
	BundleInterval     time.Duration
	MaxGasPerBundle    uint64
	MinProfitPerBundle *big.Int
}

// UserOpMempool UserOperation 内存池
type UserOpMempool struct {
	ops     map[common.Hash]*UserOperation
	byNonce map[common.Address]map[uint64]*UserOperation
	mu      sync.RWMutex
}

func NewUserOpMempool() *UserOpMempool {
	return &UserOpMempool{
		ops:     make(map[common.Hash]*UserOperation),
		byNonce: make(map[common.Address]map[uint64]*UserOperation),
	}
}

// Add 添加 UserOp
func (m *UserOpMempool) Add(op *UserOperation, hash common.Hash) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	if _, exists := m.ops[hash]; exists {
		return fmt.Errorf("userOp already exists")
	}

	m.ops[hash] = op

	if m.byNonce[op.Sender] == nil {
		m.byNonce[op.Sender] = make(map[uint64]*UserOperation)
	}
	m.byNonce[op.Sender][op.Nonce.Uint64()] = op

	return nil
}

// GetPending 获取待处理的 UserOps
func (m *UserOpMempool) GetPending(maxCount int) []*UserOperation {
	m.mu.RLock()
	defer m.mu.RUnlock()

	var ops []*UserOperation
	for _, op := range m.ops {
		ops = append(ops, op)
		if len(ops) >= maxCount {
			break
		}
	}
	return ops
}

// NewBundler 创建 Bundler
func NewBundler(ethClient *ethclient.Client, privateKey *ecdsa.PrivateKey, config BundlerConfig) (*Bundler, error) {
	entryPointABI, err := abi.JSON(strings.NewReader(EntryPointABI))
	if err != nil {
		return nil, err
	}

	return &Bundler{
		ethClient:     ethClient,
		entryPoint:    EntryPointV06,
		entryPointABI: entryPointABI,
		mempool:       NewUserOpMempool(),
		config:        config,
		privateKey:    privateKey,
	}, nil
}

// HandleUserOp 处理 eth_sendUserOperation
func (b *Bundler) HandleUserOp(ctx context.Context, op *UserOperation, entryPoint common.Address) (common.Hash, error) {
	// 1. 验证 UserOp
	if err := b.validateUserOp(ctx, op); err != nil {
		return common.Hash{}, fmt.Errorf("validation failed: %w", err)
	}

	// 2. 模拟执行
	if err := b.simulateUserOp(ctx, op); err != nil {
		return common.Hash{}, fmt.Errorf("simulation failed: %w", err)
	}

	// 3. 计算哈希
	opHash := op.Hash(entryPoint, b.config.ChainID)

	// 4. 添加到 mempool
	if err := b.mempool.Add(op, opHash); err != nil {
		return common.Hash{}, err
	}

	return opHash, nil
}

// validateUserOp 验证 UserOp
func (b *Bundler) validateUserOp(ctx context.Context, op *UserOperation) error {
	// 检查 sender
	if op.Sender == (common.Address{}) {
		return fmt.Errorf("invalid sender")
	}

	// 检查 gas 参数
	if op.CallGasLimit.Sign() <= 0 {
		return fmt.Errorf("invalid callGasLimit")
	}
	if op.VerificationGasLimit.Sign() <= 0 {
		return fmt.Errorf("invalid verificationGasLimit")
	}

	// 检查 nonce
	currentNonce, err := b.getAccountNonce(ctx, op.Sender)
	if err != nil {
		return err
	}
	if op.Nonce.Cmp(currentNonce) < 0 {
		return fmt.Errorf("nonce too low")
	}

	// 检查账户是否存在或有 initCode
	code, err := b.ethClient.CodeAt(ctx, op.Sender, nil)
	if err != nil {
		return err
	}
	if len(code) == 0 && len(op.InitCode) == 0 {
		return fmt.Errorf("account not deployed and no initCode")
	}

	return nil
}

// simulateUserOp 模拟 UserOp 执行
func (b *Bundler) simulateUserOp(ctx context.Context, op *UserOperation) error {
	// 调用 EntryPoint.simulateValidation
	callData, err := b.entryPointABI.Pack("simulateValidation", op)
	if err != nil {
		return err
	}

	msg := ethereum.CallMsg{
		To:   &b.entryPoint,
		Data: callData,
	}

	// simulateValidation 总是 revert，我们需要解析 revert 数据
	_, err = b.ethClient.CallContract(ctx, msg, nil)
	if err != nil {
		// 解析 ValidationResult
		return b.parseSimulationResult(err)
	}

	return nil
}

// parseSimulationResult 解析模拟结果
func (b *Bundler) parseSimulationResult(err error) error {
	// 从错误中提取 revert data
	// ValidationResult(returnInfo, senderInfo, factoryInfo, paymasterInfo)
	// 如果验证成功，会 revert ValidationResult
	// 如果验证失败，会 revert FailedOp
	return nil // 简化处理
}

// BundleAndSubmit 打包并提交
func (b *Bundler) BundleAndSubmit(ctx context.Context) error {
	// 获取待处理的 UserOps
	ops := b.mempool.GetPending(b.config.MaxBundleSize)
	if len(ops) == 0 {
		return nil
	}

	// 按利润排序
	sortedOps := b.sortByProfit(ops)

	// 构建 bundle
	bundle, err := b.buildBundle(ctx, sortedOps)
	if err != nil {
		return err
	}

	// 提交到链上
	return b.submitBundle(ctx, bundle)
}

// buildBundle 构建 bundle
func (b *Bundler) buildBundle(ctx context.Context, ops []*UserOperation) (*Bundle, error) {
	var bundleOps []*UserOperation
	var totalGas uint64

	for _, op := range ops {
		// 估算 gas
		gas := op.CallGasLimit.Uint64() + op.VerificationGasLimit.Uint64() + op.PreVerificationGas.Uint64()

		if totalGas+gas > b.config.MaxGasPerBundle {
			break
		}

		// 二次验证
		if err := b.simulateUserOp(ctx, op); err != nil {
			continue
		}

		bundleOps = append(bundleOps, op)
		totalGas += gas
	}

	return &Bundle{
		Ops:      bundleOps,
		TotalGas: totalGas,
	}, nil
}

// submitBundle 提交 bundle
func (b *Bundler) submitBundle(ctx context.Context, bundle *Bundle) error {
	// 构建 handleOps calldata
	callData, err := b.entryPointABI.Pack("handleOps", bundle.Ops, b.beneficiary())
	if err != nil {
		return err
	}

	// 估算 gas
	gasLimit, err := b.ethClient.EstimateGas(ctx, ethereum.CallMsg{
		From: b.beneficiary(),
		To:   &b.entryPoint,
		Data: callData,
	})
	if err != nil {
		return err
	}

	// 获取 gas 价格
	gasPrice, err := b.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return err
	}

	// 构建交易
	nonce, err := b.ethClient.PendingNonceAt(ctx, b.beneficiary())
	if err != nil {
		return err
	}

	tx := types.NewTransaction(
		nonce,
		b.entryPoint,
		big.NewInt(0),
		gasLimit,
		gasPrice,
		callData,
	)

	// 签名
	signedTx, err := types.SignTx(tx, types.NewEIP155Signer(b.config.ChainID), b.privateKey)
	if err != nil {
		return err
	}

	// 发送
	return b.ethClient.SendTransaction(ctx, signedTx)
}

func (b *Bundler) beneficiary() common.Address {
	return crypto.PubkeyToAddress(b.privateKey.PublicKey)
}

func (b *Bundler) getAccountNonce(ctx context.Context, account common.Address) (*big.Int, error) {
	callData, _ := b.entryPointABI.Pack("getNonce", account, big.NewInt(0))

	result, err := b.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &b.entryPoint,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	return new(big.Int).SetBytes(result), nil
}

func (b *Bundler) sortByProfit(ops []*UserOperation) []*UserOperation {
	// 按 maxPriorityFeePerGas 排序
	sort.Slice(ops, func(i, j int) bool {
		return ops[i].MaxPriorityFeePerGas.Cmp(ops[j].MaxPriorityFeePerGas) > 0
	})
	return ops
}

type Bundle struct {
	Ops      []*UserOperation
	TotalGas uint64
}
```

### 2.2 Bundler RPC 服务

```go
package bundler

import (
	"context"
	"encoding/json"
	"net/http"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/common/hexutil"
)

// RPCServer Bundler RPC 服务
type RPCServer struct {
	bundler *Bundler
}

// NewRPCServer 创建 RPC 服务
func NewRPCServer(bundler *Bundler) *RPCServer {
	return &RPCServer{bundler: bundler}
}

// ServeHTTP 处理 HTTP 请求
func (s *RPCServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	var req JSONRPCRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		s.writeError(w, req.ID, -32700, "Parse error")
		return
	}

	var result interface{}
	var rpcErr *RPCError

	switch req.Method {
	case "eth_sendUserOperation":
		result, rpcErr = s.handleSendUserOp(r.Context(), req.Params)
	case "eth_estimateUserOperationGas":
		result, rpcErr = s.handleEstimateGas(r.Context(), req.Params)
	case "eth_getUserOperationByHash":
		result, rpcErr = s.handleGetUserOp(r.Context(), req.Params)
	case "eth_getUserOperationReceipt":
		result, rpcErr = s.handleGetReceipt(r.Context(), req.Params)
	case "eth_supportedEntryPoints":
		result = []string{EntryPointV06.Hex(), EntryPointV07.Hex()}
	case "eth_chainId":
		result = hexutil.EncodeBig(s.bundler.config.ChainID)
	default:
		rpcErr = &RPCError{Code: -32601, Message: "Method not found"}
	}

	if rpcErr != nil {
		s.writeError(w, req.ID, rpcErr.Code, rpcErr.Message)
		return
	}

	s.writeResult(w, req.ID, result)
}

// handleSendUserOp 处理 eth_sendUserOperation
func (s *RPCServer) handleSendUserOp(ctx context.Context, params json.RawMessage) (interface{}, *RPCError) {
	var args []json.RawMessage
	if err := json.Unmarshal(params, &args); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid params"}
	}

	if len(args) < 2 {
		return nil, &RPCError{Code: -32602, Message: "Missing params"}
	}

	var op UserOperation
	if err := json.Unmarshal(args[0], &op); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid userOp"}
	}

	var entryPoint common.Address
	if err := json.Unmarshal(args[1], &entryPoint); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid entryPoint"}
	}

	hash, err := s.bundler.HandleUserOp(ctx, &op, entryPoint)
	if err != nil {
		return nil, &RPCError{Code: -32500, Message: err.Error()}
	}

	return hash.Hex(), nil
}

// handleEstimateGas 处理 eth_estimateUserOperationGas
func (s *RPCServer) handleEstimateGas(ctx context.Context, params json.RawMessage) (interface{}, *RPCError) {
	var args []json.RawMessage
	if err := json.Unmarshal(params, &args); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid params"}
	}

	var op UserOperation
	if err := json.Unmarshal(args[0], &op); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid userOp"}
	}

	// 估算 gas
	estimate, err := s.bundler.EstimateGas(ctx, &op)
	if err != nil {
		return nil, &RPCError{Code: -32500, Message: err.Error()}
	}

	return map[string]string{
		"preVerificationGas":   hexutil.EncodeBig(estimate.PreVerificationGas),
		"verificationGasLimit": hexutil.EncodeBig(estimate.VerificationGasLimit),
		"callGasLimit":         hexutil.EncodeBig(estimate.CallGasLimit),
	}, nil
}

type JSONRPCRequest struct {
	JSONRPC string          `json:"jsonrpc"`
	ID      interface{}     `json:"id"`
	Method  string          `json:"method"`
	Params  json.RawMessage `json:"params"`
}

type RPCError struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
}

func (s *RPCServer) writeError(w http.ResponseWriter, id interface{}, code int, message string) {
	resp := map[string]interface{}{
		"jsonrpc": "2.0",
		"id":      id,
		"error": map[string]interface{}{
			"code":    code,
			"message": message,
		},
	}
	json.NewEncoder(w).Encode(resp)
}

func (s *RPCServer) writeResult(w http.ResponseWriter, id interface{}, result interface{}) {
	resp := map[string]interface{}{
		"jsonrpc": "2.0",
		"id":      id,
		"result":  result,
	}
	json.NewEncoder(w).Encode(resp)
}
```

## 3. Paymaster 实现

### 3.1 Paymaster 接口

```go
package paymaster

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// IPaymaster Paymaster 接口
type IPaymaster interface {
	// ValidatePaymasterUserOp 验证并决定是否为 UserOp 付款
	ValidatePaymasterUserOp(op *UserOperation, maxCost *big.Int) (*PaymasterValidation, error)

	// PostOp 执行后回调
	PostOp(mode PostOpMode, context []byte, actualGasCost *big.Int) error
}

type PaymasterValidation struct {
	Context    []byte
	ValidUntil uint64
	ValidAfter uint64
}

type PostOpMode uint8

const (
	OpSucceeded PostOpMode = iota
	OpReverted
	PostOpReverted
)

// VerifyingPaymaster 验证型 Paymaster
type VerifyingPaymaster struct {
	ethClient  *ethclient.Client
	address    common.Address
	signer     *ecdsa.PrivateKey
	entryPoint common.Address
}

// NewVerifyingPaymaster 创建验证型 Paymaster
func NewVerifyingPaymaster(
	ethClient *ethclient.Client,
	address common.Address,
	signer *ecdsa.PrivateKey,
) *VerifyingPaymaster {
	return &VerifyingPaymaster{
		ethClient:  ethClient,
		address:    address,
		signer:     signer,
		entryPoint: EntryPointV06,
	}
}

// SignUserOp 为 UserOp 签名
func (p *VerifyingPaymaster) SignUserOp(op *UserOperation, validUntil, validAfter uint64) ([]byte, error) {
	// 构建要签名的数据
	hash := p.getHash(op, validUntil, validAfter)

	// 签名
	signature, err := crypto.Sign(hash.Bytes(), p.signer)
	if err != nil {
		return nil, err
	}

	// 调整 v 值
	signature[64] += 27

	// 构建 paymasterAndData
	// paymaster (20) + validUntil (6) + validAfter (6) + signature (65)
	paymasterAndData := make([]byte, 0, 97)
	paymasterAndData = append(paymasterAndData, p.address.Bytes()...)
	paymasterAndData = append(paymasterAndData, encodeUint48(validUntil)...)
	paymasterAndData = append(paymasterAndData, encodeUint48(validAfter)...)
	paymasterAndData = append(paymasterAndData, signature...)

	return paymasterAndData, nil
}

func (p *VerifyingPaymaster) getHash(op *UserOperation, validUntil, validAfter uint64) common.Hash {
	// 类似 UserOp hash，但包含时间戳
	return crypto.Keccak256Hash(
		op.Pack(),
		p.address.Bytes(),
		encodeUint48(validUntil),
		encodeUint48(validAfter),
	)
}

func encodeUint48(v uint64) []byte {
	b := make([]byte, 6)
	for i := 5; i >= 0; i-- {
		b[i] = byte(v)
		v >>= 8
	}
	return b
}

// TokenPaymaster ERC20 代币支付 Paymaster
type TokenPaymaster struct {
	ethClient *ethclient.Client
	address   common.Address
	token     common.Address
	oracle    PriceOracle
	markup    *big.Int // basis points (10000 = 100%)
}

// GetTokenCost 计算代币成本
func (p *TokenPaymaster) GetTokenCost(op *UserOperation) (*big.Int, error) {
	// 计算 ETH 成本
	gasPrice := op.MaxFeePerGas
	totalGas := new(big.Int).Add(op.CallGasLimit, op.VerificationGasLimit)
	totalGas.Add(totalGas, op.PreVerificationGas)

	ethCost := new(big.Int).Mul(totalGas, gasPrice)

	// 获取代币/ETH 价格
	tokenPrice, err := p.oracle.GetPrice(p.token, common.Address{}) // token/ETH
	if err != nil {
		return nil, err
	}

	// 转换为代币数量
	tokenCost := new(big.Int).Mul(ethCost, tokenPrice)
	tokenCost.Div(tokenCost, big.NewInt(1e18))

	// 添加加价
	tokenCost.Mul(tokenCost, p.markup)
	tokenCost.Div(tokenCost, big.NewInt(10000))

	return tokenCost, nil
}

// SponsoringPaymaster 赞助型 Paymaster
type SponsoringPaymaster struct {
	ethClient *ethclient.Client
	address   common.Address
	signer    *ecdsa.PrivateKey

	// 赞助规则
	sponsoredApps map[common.Address]bool
	dailyLimit    *big.Int
	spent         map[common.Address]*big.Int
}

// ShouldSponsor 判断是否应该赞助
func (p *SponsoringPaymaster) ShouldSponsor(op *UserOperation) bool {
	// 检查是否是被赞助的应用
	// 从 callData 解析目标合约
	target := p.extractTarget(op.CallData)
	if !p.sponsoredApps[target] {
		return false
	}

	// 检查每日限额
	spent := p.spent[op.Sender]
	if spent == nil {
		spent = big.NewInt(0)
	}

	cost := p.estimateCost(op)
	if new(big.Int).Add(spent, cost).Cmp(p.dailyLimit) > 0 {
		return false
	}

	return true
}

func (p *SponsoringPaymaster) extractTarget(callData []byte) common.Address {
	// 解析 execute(target, value, data) 调用
	if len(callData) < 36 {
		return common.Address{}
	}
	return common.BytesToAddress(callData[16:36])
}

func (p *SponsoringPaymaster) estimateCost(op *UserOperation) *big.Int {
	totalGas := new(big.Int).Add(op.CallGasLimit, op.VerificationGasLimit)
	totalGas.Add(totalGas, op.PreVerificationGas)
	return new(big.Int).Mul(totalGas, op.MaxFeePerGas)
}
```

## 4. Smart Account 实现

### 4.1 简单账户

```go
package account

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// SimpleAccount 简单智能账户
type SimpleAccount struct {
	address    common.Address
	owner      common.Address
	entryPoint common.Address
	factory    common.Address
}

// SimpleAccountFactory 账户工厂
type SimpleAccountFactory struct {
	address common.Address
	abi     abi.ABI
}

// GetAddress 预计算账户地址
func (f *SimpleAccountFactory) GetAddress(owner common.Address, salt *big.Int) common.Address {
	// CREATE2 地址计算
	// address = keccak256(0xff + factory + salt + keccak256(initCode))[12:]

	initCodeHash := f.getInitCodeHash(owner)

	data := make([]byte, 85)
	data[0] = 0xff
	copy(data[1:21], f.address.Bytes())
	copy(data[21:53], common.LeftPadBytes(salt.Bytes(), 32))
	copy(data[53:85], initCodeHash.Bytes())

	return common.BytesToAddress(crypto.Keccak256(data)[12:])
}

func (f *SimpleAccountFactory) getInitCodeHash(owner common.Address) common.Hash {
	// 简化：实际需要包含完整的 initCode
	return crypto.Keccak256Hash(owner.Bytes())
}

// GetInitCode 获取部署初始化代码
func (f *SimpleAccountFactory) GetInitCode(owner common.Address, salt *big.Int) []byte {
	// factory address (20) + createAccount(owner, salt) calldata
	callData, _ := f.abi.Pack("createAccount", owner, salt)

	initCode := make([]byte, 20+len(callData))
	copy(initCode[:20], f.address.Bytes())
	copy(initCode[20:], callData)

	return initCode
}

// SmartAccountClient 智能账户客户端
type SmartAccountClient struct {
	ethClient  *ethclient.Client
	bundlerURL string
	account    *SimpleAccount
	privateKey *ecdsa.PrivateKey
	chainID    *big.Int
}

// NewSmartAccountClient 创建客户端
func NewSmartAccountClient(
	ethClient *ethclient.Client,
	bundlerURL string,
	privateKey *ecdsa.PrivateKey,
	chainID *big.Int,
) (*SmartAccountClient, error) {
	owner := crypto.PubkeyToAddress(privateKey.PublicKey)

	// 计算账户地址
	factory := &SimpleAccountFactory{
		address: common.HexToAddress("0x9406Cc6185a346906296840746125a0E44976454"), // 示例
	}
	accountAddr := factory.GetAddress(owner, big.NewInt(0))

	return &SmartAccountClient{
		ethClient:  ethClient,
		bundlerURL: bundlerURL,
		account: &SimpleAccount{
			address:    accountAddr,
			owner:      owner,
			entryPoint: EntryPointV06,
			factory:    factory.address,
		},
		privateKey: privateKey,
		chainID:    chainID,
	}, nil
}

// BuildUserOp 构建 UserOperation
func (c *SmartAccountClient) BuildUserOp(ctx context.Context, calls []Call) (*UserOperation, error) {
	// 检查账户是否已部署
	code, err := c.ethClient.CodeAt(ctx, c.account.address, nil)
	if err != nil {
		return nil, err
	}

	var initCode []byte
	if len(code) == 0 {
		// 需要部署
		factory := &SimpleAccountFactory{address: c.account.factory}
		initCode = factory.GetInitCode(c.account.owner, big.NewInt(0))
	}

	// 构建 callData
	callData, err := c.encodeExecute(calls)
	if err != nil {
		return nil, err
	}

	// 获取 nonce
	nonce, err := c.getNonce(ctx)
	if err != nil {
		return nil, err
	}

	// 获取 gas 价格
	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	op := &UserOperation{
		Sender:               c.account.address,
		Nonce:                nonce,
		InitCode:             initCode,
		CallData:             callData,
		CallGasLimit:         big.NewInt(200000),
		VerificationGasLimit: big.NewInt(100000),
		PreVerificationGas:   big.NewInt(50000),
		MaxFeePerGas:         gasPrice,
		MaxPriorityFeePerGas: big.NewInt(1e9), // 1 gwei
		PaymasterAndData:     nil,
		Signature:            nil,
	}

	return op, nil
}

// SignUserOp 签名 UserOperation
func (c *SmartAccountClient) SignUserOp(op *UserOperation) error {
	// 计算 UserOp hash
	opHash := op.Hash(c.account.entryPoint, c.chainID)

	// 签名
	signature, err := crypto.Sign(opHash.Bytes(), c.privateKey)
	if err != nil {
		return err
	}

	// 调整 v 值
	signature[64] += 27

	op.Signature = signature
	return nil
}

// SendUserOp 发送 UserOperation
func (c *SmartAccountClient) SendUserOp(ctx context.Context, op *UserOperation) (common.Hash, error) {
	// 签名
	if err := c.SignUserOp(op); err != nil {
		return common.Hash{}, err
	}

	// 发送到 Bundler
	client := &http.Client{}
	reqBody := map[string]interface{}{
		"jsonrpc": "2.0",
		"id":      1,
		"method":  "eth_sendUserOperation",
		"params":  []interface{}{op, c.account.entryPoint.Hex()},
	}

	body, _ := json.Marshal(reqBody)
	resp, err := client.Post(c.bundlerURL, "application/json", bytes.NewReader(body))
	if err != nil {
		return common.Hash{}, err
	}
	defer resp.Body.Close()

	var result struct {
		Result string `json:"result"`
		Error  *struct {
			Message string `json:"message"`
		} `json:"error"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return common.Hash{}, err
	}

	if result.Error != nil {
		return common.Hash{}, fmt.Errorf(result.Error.Message)
	}

	return common.HexToHash(result.Result), nil
}

func (c *SmartAccountClient) encodeExecute(calls []Call) ([]byte, error) {
	if len(calls) == 1 {
		// execute(target, value, data)
		return c.accountABI.Pack("execute", calls[0].Target, calls[0].Value, calls[0].Data)
	}

	// executeBatch(targets[], values[], datas[])
	targets := make([]common.Address, len(calls))
	values := make([]*big.Int, len(calls))
	datas := make([][]byte, len(calls))

	for i, call := range calls {
		targets[i] = call.Target
		values[i] = call.Value
		datas[i] = call.Data
	}

	return c.accountABI.Pack("executeBatch", targets, values, datas)
}

func (c *SmartAccountClient) getNonce(ctx context.Context) (*big.Int, error) {
	// 调用 EntryPoint.getNonce
	entryPointABI, _ := abi.JSON(strings.NewReader(EntryPointABI))
	callData, _ := entryPointABI.Pack("getNonce", c.account.address, big.NewInt(0))

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &c.account.entryPoint,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	return new(big.Int).SetBytes(result), nil
}

type Call struct {
	Target common.Address
	Value  *big.Int
	Data   []byte
}
```

## 5. AA 套利机会

### 5.1 Bundler MEV 提取

```go
package mev

import (
	"context"
	"math/big"
	"sort"

	"github.com/ethereum/go-ethereum/common"
)

// BundlerMEV Bundler MEV 提取
type BundlerMEV struct {
	bundler         *Bundler
	dexAggregator   DEXAggregator
	flashbotsClient *FlashbotsClient
}

// ExtractMEV 从 UserOp bundle 中提取 MEV
func (m *BundlerMEV) ExtractMEV(ctx context.Context, ops []*UserOperation) (*MEVBundle, error) {
	// 分析每个 UserOp 的交易意图
	intents := make([]*TxIntent, 0, len(ops))
	for _, op := range ops {
		intent := m.analyzeUserOp(op)
		if intent != nil {
			intents = append(intents, intent)
		}
	}

	// 寻找 MEV 机会
	opportunities := m.findMEVOpportunities(ctx, intents)

	if len(opportunities) == 0 {
		return nil, nil
	}

	// 构建 MEV bundle
	return m.buildMEVBundle(ctx, ops, opportunities)
}

// analyzeUserOp 分析 UserOp 交易意图
func (m *BundlerMEV) analyzeUserOp(op *UserOperation) *TxIntent {
	// 解析 callData
	if len(op.CallData) < 4 {
		return nil
	}

	selector := op.CallData[:4]

	// 常见的交易模式
	switch {
	case bytes.Equal(selector, swapExactTokensSelector):
		return m.parseSwapIntent(op)
	case bytes.Equal(selector, executeSelector):
		return m.parseExecuteIntent(op)
	default:
		return nil
	}
}

// parseSwapIntent 解析 swap 意图
func (m *BundlerMEV) parseSwapIntent(op *UserOperation) *TxIntent {
	// 解析 swap 参数
	// swapExactTokensForTokens(amountIn, amountOutMin, path, to, deadline)

	return &TxIntent{
		Type:         IntentSwap,
		Sender:       op.Sender,
		TokenIn:      common.Address{}, // 从 path 解析
		TokenOut:     common.Address{},
		AmountIn:     big.NewInt(0),
		MinAmountOut: big.NewInt(0),
	}
}

// findMEVOpportunities 寻找 MEV 机会
func (m *BundlerMEV) findMEVOpportunities(ctx context.Context, intents []*TxIntent) []*MEVOpportunity {
	var opportunities []*MEVOpportunity

	// 1. 寻找三明治攻击机会
	for _, intent := range intents {
		if intent.Type == IntentSwap {
			sandwich := m.findSandwichOpportunity(ctx, intent)
			if sandwich != nil {
				opportunities = append(opportunities, sandwich)
			}
		}
	}

	// 2. 寻找套利机会（基于 UserOps 的影响）
	arbitrage := m.findArbitrageAfterUserOps(ctx, intents)
	if arbitrage != nil {
		opportunities = append(opportunities, arbitrage)
	}

	// 3. 寻找清算机会
	liquidations := m.findLiquidationOpportunities(ctx, intents)
	opportunities = append(opportunities, liquidations...)

	return opportunities
}

// buildMEVBundle 构建 MEV bundle
func (m *BundlerMEV) buildMEVBundle(ctx context.Context, userOps []*UserOperation, mevOps []*MEVOpportunity) (*MEVBundle, error) {
	bundle := &MEVBundle{
		UserOps: userOps,
	}

	// 对每个 MEV 机会，插入前置/后置交易
	for _, mev := range mevOps {
		switch mev.Type {
		case MEVSandwich:
			// 前置交易
			bundle.PreTxs = append(bundle.PreTxs, mev.FrontrunTx)
			// 后置交易
			bundle.PostTxs = append(bundle.PostTxs, mev.BackrunTx)

		case MEVArbitrage:
			// 在 UserOps 之后执行
			bundle.PostTxs = append(bundle.PostTxs, mev.ArbitrageTx)

		case MEVLiquidation:
			// 在 UserOps 之后执行
			bundle.PostTxs = append(bundle.PostTxs, mev.LiquidationTx)
		}
	}

	return bundle, nil
}

type TxIntent struct {
	Type         IntentType
	Sender       common.Address
	TokenIn      common.Address
	TokenOut     common.Address
	AmountIn     *big.Int
	MinAmountOut *big.Int
}

type IntentType int

const (
	IntentSwap IntentType = iota
	IntentTransfer
	IntentApprove
	IntentOther
)

type MEVOpportunity struct {
	Type           MEVType
	ExpectedProfit *big.Int

	// Sandwich
	FrontrunTx *types.Transaction
	BackrunTx  *types.Transaction

	// Arbitrage
	ArbitrageTx *types.Transaction

	// Liquidation
	LiquidationTx *types.Transaction
}

type MEVType int

const (
	MEVSandwich MEVType = iota
	MEVArbitrage
	MEVLiquidation
)

type MEVBundle struct {
	PreTxs  []*types.Transaction
	UserOps []*UserOperation
	PostTxs []*types.Transaction
}
```

### 5.2 UserOp 抢跑保护

```go
package protection

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
)

// FrontrunProtector 抢跑保护
type FrontrunProtector struct {
	flashbotsClient *FlashbotsClient
	privatePool     PrivatePoolProvider
}

// ProtectedSubmit 保护性提交
func (p *FrontrunProtector) ProtectedSubmit(ctx context.Context, op *UserOperation) (common.Hash, error) {
	// 1. 分析 UserOp 是否容易被抢跑
	risk := p.analyzeRisk(op)

	if risk.Level == RiskLow {
		// 低风险，可以通过公共 mempool
		return p.submitPublic(ctx, op)
	}

	// 2. 高风险，使用私有提交
	if risk.Level == RiskHigh {
		// 计算需要的贿赂金额
		bribe := p.calculateBribe(risk)

		// 通过 Flashbots 提交
		return p.submitViaFlashbots(ctx, op, bribe)
	}

	// 3. 中等风险，使用 MEV Blocker
	return p.submitViaMEVBlocker(ctx, op)
}

// analyzeRisk 分析抢跑风险
func (p *FrontrunProtector) analyzeRisk(op *UserOperation) *RiskAnalysis {
	// 解析交易意图
	intent := p.parseIntent(op)

	risk := &RiskAnalysis{
		Level: RiskLow,
	}

	switch intent.Type {
	case IntentSwap:
		// 大额 swap 风险高
		if intent.AmountIn.Cmp(big.NewInt(10e18)) > 0 { // > 10 ETH
			risk.Level = RiskHigh
			risk.PotentialLoss = p.estimateSandwichLoss(intent)
		} else if intent.AmountIn.Cmp(big.NewInt(1e18)) > 0 { // > 1 ETH
			risk.Level = RiskMedium
		}

		// 检查滑点设置
		slippage := p.calculateSlippage(intent)
		if slippage > 0.05 { // > 5%
			risk.Level = RiskHigh
		}

	case IntentApprove:
		// approve 通常安全
		risk.Level = RiskLow
	}

	return risk
}

// estimateSandwichLoss 估算三明治攻击损失
func (p *FrontrunProtector) estimateSandwichLoss(intent *TxIntent) *big.Int {
	// 基于交易规模和池深度估算
	// 简化计算
	return new(big.Int).Div(intent.AmountIn, big.NewInt(100)) // ~1%
}

// submitViaFlashbots 通过 Flashbots 提交
func (p *FrontrunProtector) submitViaFlashbots(ctx context.Context, op *UserOperation, bribe *big.Int) (common.Hash, error) {
	// 构建包含贿赂的 bundle
	bundle := &FlashbotsBundle{
		Transactions: []interface{}{
			&UserOpWrapper{UserOp: op},
		},
		BribeAmount: bribe,
	}

	return p.flashbotsClient.SendBundle(ctx, bundle)
}

type RiskLevel int

const (
	RiskLow RiskLevel = iota
	RiskMedium
	RiskHigh
)

type RiskAnalysis struct {
	Level         RiskLevel
	PotentialLoss *big.Int
	Reasons       []string
}
```

## 6. 完整示例：AA 套利机器人

```go
package main

import (
	"context"
	"log"
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

func main() {
	// 连接以太坊
	ethClient, err := ethclient.Dial("https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY")
	if err != nil {
		log.Fatal(err)
	}

	// 加载私钥
	privateKey, err := crypto.HexToECDSA("YOUR_PRIVATE_KEY")
	if err != nil {
		log.Fatal(err)
	}

	// 创建智能账户客户端
	aaClient, err := NewSmartAccountClient(
		ethClient,
		"https://bundler.example.com/rpc",
		privateKey,
		big.NewInt(1), // mainnet
	)
	if err != nil {
		log.Fatal(err)
	}

	// 套利机器人
	bot := &AAArbitrageBot{
		aaClient:      aaClient,
		dexAggregator: NewDEXAggregator(ethClient),
		priceOracle:   NewPriceOracle(ethClient),
	}

	// 运行
	ctx := context.Background()
	bot.Run(ctx)
}

// AAArbitrageBot AA 套利机器人
type AAArbitrageBot struct {
	aaClient      *SmartAccountClient
	dexAggregator *DEXAggregator
	priceOracle   *PriceOracle
}

// Run 运行机器人
func (b *AAArbitrageBot) Run(ctx context.Context) {
	ticker := time.NewTicker(12 * time.Second) // 每个区块
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			if err := b.checkAndExecute(ctx); err != nil {
				log.Printf("Error: %v", err)
			}
		}
	}
}

// checkAndExecute 检查并执行套利
func (b *AAArbitrageBot) checkAndExecute(ctx context.Context) error {
	// 寻找套利机会
	opportunity, err := b.findOpportunity(ctx)
	if err != nil {
		return err
	}

	if opportunity == nil {
		return nil
	}

	log.Printf("Found opportunity: profit=%s", opportunity.Profit.String())

	// 构建套利调用
	calls := b.buildArbitrageCalls(opportunity)

	// 构建 UserOp
	op, err := b.aaClient.BuildUserOp(ctx, calls)
	if err != nil {
		return err
	}

	// 使用 Paymaster（如果有）
	if b.hasPaymaster() {
		paymasterData, err := b.getPaymasterData(ctx, op)
		if err == nil {
			op.PaymasterAndData = paymasterData
		}
	}

	// 发送 UserOp
	hash, err := b.aaClient.SendUserOp(ctx, op)
	if err != nil {
		return err
	}

	log.Printf("UserOp submitted: %s", hash.Hex())

	// 等待确认
	return b.waitForReceipt(ctx, hash)
}

// findOpportunity 寻找套利机会
func (b *AAArbitrageBot) findOpportunity(ctx context.Context) (*ArbitrageOpportunity, error) {
	// 获取热门代币对价格
	pairs := []TokenPair{
		{WETH, USDC},
		{WETH, USDT},
		{WETH, DAI},
	}

	for _, pair := range pairs {
		// 检查跨 DEX 套利
		opp := b.checkCrossDEXArbitrage(ctx, pair)
		if opp != nil && opp.Profit.Cmp(b.minProfit()) > 0 {
			return opp, nil
		}
	}

	return nil, nil
}

// buildArbitrageCalls 构建套利调用
func (b *AAArbitrageBot) buildArbitrageCalls(opp *ArbitrageOpportunity) []Call {
	var calls []Call

	// 1. 第一个 DEX swap
	calls = append(calls, Call{
		Target: opp.DEX1Router,
		Value:  big.NewInt(0),
		Data:   b.encodeSwap(opp.DEX1, opp.TokenIn, opp.TokenMid, opp.AmountIn),
	})

	// 2. 第二个 DEX swap
	calls = append(calls, Call{
		Target: opp.DEX2Router,
		Value:  big.NewInt(0),
		Data:   b.encodeSwap(opp.DEX2, opp.TokenMid, opp.TokenOut, opp.ExpectedMidAmount),
	})

	return calls
}

func (b *AAArbitrageBot) minProfit() *big.Int {
	return big.NewInt(1e16) // 0.01 ETH
}

type ArbitrageOpportunity struct {
	DEX1Router        common.Address
	DEX2Router        common.Address
	DEX1              string
	DEX2              string
	TokenIn           common.Address
	TokenMid          common.Address
	TokenOut          common.Address
	AmountIn          *big.Int
	ExpectedMidAmount *big.Int
	ExpectedOutput    *big.Int
	Profit            *big.Int
}
```

## 7. 总结

### 7.1 AA 生态对比

| 组件                | 功能         | 主要项目                      |
|-------------------|------------|---------------------------|
| **EntryPoint**    | 统一入口       | ERC-4337 标准               |
| **Bundler**       | 打包 UserOps | Stackup, Pimlico, Alchemy |
| **Paymaster**     | 代付 Gas     | Pimlico, Stackup          |
| **Smart Account** | 智能钱包       | Safe, Biconomy, ZeroDev   |

### 7.2 AA 套利优势

1. **批量操作**: 单个 UserOp 包含多个调用
2. **Gas 抽象**: 可用代币支付或赞助
3. **灵活验证**: 支持多签、社交恢复等
4. **MEV 保护**: 通过 Bundler 私有提交

### 7.3 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     AA 套利架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │  价格监控     │───▶│  机会检测      │───▶│  策略选择     │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                │            │
│                                                ▼            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  Smart Account                       │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐      │   │
│  │  │  Swap A    │─▶│  Swap B    │─▶│  Swap C    │      │   │
│  │  └────────────┘  └────────────┘  └────────────┘      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                │            │
│                                                ▼            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Bundler    │───▶│  EntryPoint  │───▶│   链上执行    │   │
│  │  (可选私有)   │    │              │    │              │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

账户抽象为套利机器人提供了更灵活的执行环境，通过批量操作和 Gas 抽象可以优化成本，同时 Bundler 层的私有提交也提供了一定的
MEV 保护。

---

## 8. EIP-7702：协议层账户抽象

### 8.1 背景与动机

EIP-7702 是 Vitalik Buterin 等人在 2024 年提出的协议层账户抽象方案，已被纳入 **Prague/Electra (Pectra)** 升级。它允许
EOA（外部拥有账户）在单笔交易执行期间临时"委托"给智能合约代码，结合了 EOA 的简洁性和智能合约钱包的灵活性。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    账户抽象方案演进                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐        │
│  │   EIP-3074    │    │   ERC-4337    │    │   EIP-7702    │        │
│  │   (已废弃)     │    │   (应用层)     │    │   (协议层)     │        │
│  └───────────────┘    └───────────────┘    └───────────────┘        │
│         │                    │                    │                 │
│         ▼                    ▼                    ▼                 │
│  AUTH + AUTHCALL       UserOperation         SET_CODE_TX            │
│  安全风险高            需要新基础设施         EOA 临时升级               │
│                        (Bundler/Paymaster)   复用现有基础设施          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```# Go-Ethereum EVM 深入解析：账户抽象 (ERC-4337)

## 概述

账户抽象 (Account Abstraction, AA) 通过 ERC-4337 实现了无需协议层更改的智能合约钱包。它引入了
UserOperation、Bundler、Paymaster 等新概念，为套利机器人提供了新的机会和挑战。

## 1. ERC-4337 架构

### 1.1 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                     ERC-4337 架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐                                           │
│  │    用户      │                                           │
│  │ Smart Wallet │                                           │
│  └──────┬───────┘                                           │
│         │ 签名 UserOp                                        │
│         ▼                                                   │
│  ┌──────────────┐    ┌──────────────┐                       │
│  │   Bundler    │───▶│  EntryPoint  │                       │
│  │  (打包者)    │    │   (入口点)   │                         │
│  └──────────────┘    └──────┬───────┘                       │
│                             │                               │
│         ┌───────────────────┼───────────────────┐           │
│         ▼                   ▼                   ▼           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Account    │    │  Paymaster   │    │  Aggregator  │   │
│  │  (智能账户)   │    │  (支付者)     │    │  (聚合器)     │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心数据结构

```go
package aa

import (
	"math/big"

	"github.com/ethereum/go-ethereum/common"
)

// UserOperation ERC-4337 用户操作
type UserOperation struct {
	Sender               common.Address `json:"sender"`
	Nonce                *big.Int       `json:"nonce"`
	InitCode             []byte         `json:"initCode"`
	CallData             []byte         `json:"callData"`
	CallGasLimit         *big.Int       `json:"callGasLimit"`
	VerificationGasLimit *big.Int       `json:"verificationGasLimit"`
	PreVerificationGas   *big.Int       `json:"preVerificationGas"`
	MaxFeePerGas         *big.Int       `json:"maxFeePerGas"`
	MaxPriorityFeePerGas *big.Int       `json:"maxPriorityFeePerGas"`
	PaymasterAndData     []byte         `json:"paymasterAndData"`
	Signature            []byte         `json:"signature"`
}

// PackedUserOperation ERC-4337 v0.7 打包格式
type PackedUserOperation struct {
	Sender             common.Address `json:"sender"`
	Nonce              *big.Int       `json:"nonce"`
	InitCode           []byte         `json:"initCode"`
	CallData           []byte         `json:"callData"`
	AccountGasLimits   [32]byte       `json:"accountGasLimits"` // verificationGasLimit (16) + callGasLimit (16)
	PreVerificationGas *big.Int       `json:"preVerificationGas"`
	GasFees            [32]byte       `json:"gasFees"` // maxPriorityFeePerGas (16) + maxFeePerGas (16)
	PaymasterAndData   []byte         `json:"paymasterAndData"`
	Signature          []byte         `json:"signature"`
}

// EntryPoint 合约地址 (v0.6 和 v0.7)
var (
	EntryPointV06 = common.HexToAddress("0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789")
	EntryPointV07 = common.HexToAddress("0x0000000071727De22E5E9d8BAf0edAc6f37da032")
)

// UserOpHash 计算 UserOperation 哈希
func (op *UserOperation) Hash(entryPoint common.Address, chainID *big.Int) common.Hash {
	// 打包 UserOp 数据
	packed := op.Pack()

	// keccak256(packed)
	opHash := crypto.Keccak256Hash(packed)

	// keccak256(abi.encode(opHash, entryPoint, chainId))
	encoded := make([]byte, 96)
	copy(encoded[0:32], opHash.Bytes())
	copy(encoded[32:64], common.LeftPadBytes(entryPoint.Bytes(), 32))
	copy(encoded[64:96], common.LeftPadBytes(chainID.Bytes(), 32))

	return crypto.Keccak256Hash(encoded)
}

// Pack 打包 UserOperation
func (op *UserOperation) Pack() []byte {
	return crypto.Keccak256(
		common.LeftPadBytes(op.Sender.Bytes(), 32),
		common.LeftPadBytes(op.Nonce.Bytes(), 32),
		crypto.Keccak256(op.InitCode),
		crypto.Keccak256(op.CallData),
		common.LeftPadBytes(op.CallGasLimit.Bytes(), 32),
		common.LeftPadBytes(op.VerificationGasLimit.Bytes(), 32),
		common.LeftPadBytes(op.PreVerificationGas.Bytes(), 32),
		common.LeftPadBytes(op.MaxFeePerGas.Bytes(), 32),
		common.LeftPadBytes(op.MaxPriorityFeePerGas.Bytes(), 32),
		crypto.Keccak256(op.PaymasterAndData),
	)
}
```

## 2. Bundler 实现

### 2.1 Bundler 服务

```go
package bundler

import (
	"context"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Bundler ERC-4337 Bundler 实现
type Bundler struct {
	ethClient     *ethclient.Client
	entryPoint    common.Address
	entryPointABI abi.ABI

	// mempool
	mempool   *UserOpMempool
	mempoolMu sync.RWMutex

	// 配置
	config BundlerConfig

	// 私钥
	privateKey *ecdsa.PrivateKey
}

type BundlerConfig struct {
	ChainID            *big.Int
	MinStake           *big.Int
	MinUnstakeDelay    uint64
	MaxBundleSize      int
	BundleInterval     time.Duration
	MaxGasPerBundle    uint64
	MinProfitPerBundle *big.Int
}

// UserOpMempool UserOperation 内存池
type UserOpMempool struct {
	ops     map[common.Hash]*UserOperation
	byNonce map[common.Address]map[uint64]*UserOperation
	mu      sync.RWMutex
}

func NewUserOpMempool() *UserOpMempool {
	return &UserOpMempool{
		ops:     make(map[common.Hash]*UserOperation),
		byNonce: make(map[common.Address]map[uint64]*UserOperation),
	}
}

// Add 添加 UserOp
func (m *UserOpMempool) Add(op *UserOperation, hash common.Hash) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	if _, exists := m.ops[hash]; exists {
		return fmt.Errorf("userOp already exists")
	}

	m.ops[hash] = op

	if m.byNonce[op.Sender] == nil {
		m.byNonce[op.Sender] = make(map[uint64]*UserOperation)
	}
	m.byNonce[op.Sender][op.Nonce.Uint64()] = op

	return nil
}

// GetPending 获取待处理的 UserOps
func (m *UserOpMempool) GetPending(maxCount int) []*UserOperation {
	m.mu.RLock()
	defer m.mu.RUnlock()

	var ops []*UserOperation
	for _, op := range m.ops {
		ops = append(ops, op)
		if len(ops) >= maxCount {
			break
		}
	}
	return ops
}

// NewBundler 创建 Bundler
func NewBundler(ethClient *ethclient.Client, privateKey *ecdsa.PrivateKey, config BundlerConfig) (*Bundler, error) {
	entryPointABI, err := abi.JSON(strings.NewReader(EntryPointABI))
	if err != nil {
		return nil, err
	}

	return &Bundler{
		ethClient:     ethClient,
		entryPoint:    EntryPointV06,
		entryPointABI: entryPointABI,
		mempool:       NewUserOpMempool(),
		config:        config,
		privateKey:    privateKey,
	}, nil
}

// HandleUserOp 处理 eth_sendUserOperation
func (b *Bundler) HandleUserOp(ctx context.Context, op *UserOperation, entryPoint common.Address) (common.Hash, error) {
	// 1. 验证 UserOp
	if err := b.validateUserOp(ctx, op); err != nil {
		return common.Hash{}, fmt.Errorf("validation failed: %w", err)
	}

	// 2. 模拟执行
	if err := b.simulateUserOp(ctx, op); err != nil {
		return common.Hash{}, fmt.Errorf("simulation failed: %w", err)
	}

	// 3. 计算哈希
	opHash := op.Hash(entryPoint, b.config.ChainID)

	// 4. 添加到 mempool
	if err := b.mempool.Add(op, opHash); err != nil {
		return common.Hash{}, err
	}

	return opHash, nil
}

// validateUserOp 验证 UserOp
func (b *Bundler) validateUserOp(ctx context.Context, op *UserOperation) error {
	// 检查 sender
	if op.Sender == (common.Address{}) {
		return fmt.Errorf("invalid sender")
	}

	// 检查 gas 参数
	if op.CallGasLimit.Sign() <= 0 {
		return fmt.Errorf("invalid callGasLimit")
	}
	if op.VerificationGasLimit.Sign() <= 0 {
		return fmt.Errorf("invalid verificationGasLimit")
	}

	// 检查 nonce
	currentNonce, err := b.getAccountNonce(ctx, op.Sender)
	if err != nil {
		return err
	}
	if op.Nonce.Cmp(currentNonce) < 0 {
		return fmt.Errorf("nonce too low")
	}

	// 检查账户是否存在或有 initCode
	code, err := b.ethClient.CodeAt(ctx, op.Sender, nil)
	if err != nil {
		return err
	}
	if len(code) == 0 && len(op.InitCode) == 0 {
		return fmt.Errorf("account not deployed and no initCode")
	}

	return nil
}

// simulateUserOp 模拟 UserOp 执行
func (b *Bundler) simulateUserOp(ctx context.Context, op *UserOperation) error {
	// 调用 EntryPoint.simulateValidation
	callData, err := b.entryPointABI.Pack("simulateValidation", op)
	if err != nil {
		return err
	}

	msg := ethereum.CallMsg{
		To:   &b.entryPoint,
		Data: callData,
	}

	// simulateValidation 总是 revert，我们需要解析 revert 数据
	_, err = b.ethClient.CallContract(ctx, msg, nil)
	if err != nil {
		// 解析 ValidationResult
		return b.parseSimulationResult(err)
	}

	return nil
}

// parseSimulationResult 解析模拟结果
func (b *Bundler) parseSimulationResult(err error) error {
	// 从错误中提取 revert data
	// ValidationResult(returnInfo, senderInfo, factoryInfo, paymasterInfo)
	// 如果验证成功，会 revert ValidationResult
	// 如果验证失败，会 revert FailedOp
	return nil // 简化处理
}

// BundleAndSubmit 打包并提交
func (b *Bundler) BundleAndSubmit(ctx context.Context) error {
	// 获取待处理的 UserOps
	ops := b.mempool.GetPending(b.config.MaxBundleSize)
	if len(ops) == 0 {
		return nil
	}

	// 按利润排序
	sortedOps := b.sortByProfit(ops)

	// 构建 bundle
	bundle, err := b.buildBundle(ctx, sortedOps)
	if err != nil {
		return err
	}

	// 提交到链上
	return b.submitBundle(ctx, bundle)
}

// buildBundle 构建 bundle
func (b *Bundler) buildBundle(ctx context.Context, ops []*UserOperation) (*Bundle, error) {
	var bundleOps []*UserOperation
	var totalGas uint64

	for _, op := range ops {
		// 估算 gas
		gas := op.CallGasLimit.Uint64() + op.VerificationGasLimit.Uint64() + op.PreVerificationGas.Uint64()

		if totalGas+gas > b.config.MaxGasPerBundle {
			break
		}

		// 二次验证
		if err := b.simulateUserOp(ctx, op); err != nil {
			continue
		}

		bundleOps = append(bundleOps, op)
		totalGas += gas
	}

	return &Bundle{
		Ops:      bundleOps,
		TotalGas: totalGas,
	}, nil
}

// submitBundle 提交 bundle
func (b *Bundler) submitBundle(ctx context.Context, bundle *Bundle) error {
	// 构建 handleOps calldata
	callData, err := b.entryPointABI.Pack("handleOps", bundle.Ops, b.beneficiary())
	if err != nil {
		return err
	}

	// 估算 gas
	gasLimit, err := b.ethClient.EstimateGas(ctx, ethereum.CallMsg{
		From: b.beneficiary(),
		To:   &b.entryPoint,
		Data: callData,
	})
	if err != nil {
		return err
	}

	// 获取 gas 价格
	gasPrice, err := b.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return err
	}

	// 构建交易
	nonce, err := b.ethClient.PendingNonceAt(ctx, b.beneficiary())
	if err != nil {
		return err
	}

	tx := types.NewTransaction(
		nonce,
		b.entryPoint,
		big.NewInt(0),
		gasLimit,
		gasPrice,
		callData,
	)

	// 签名
	signedTx, err := types.SignTx(tx, types.NewEIP155Signer(b.config.ChainID), b.privateKey)
	if err != nil {
		return err
	}

	// 发送
	return b.ethClient.SendTransaction(ctx, signedTx)
}

func (b *Bundler) beneficiary() common.Address {
	return crypto.PubkeyToAddress(b.privateKey.PublicKey)
}

func (b *Bundler) getAccountNonce(ctx context.Context, account common.Address) (*big.Int, error) {
	callData, _ := b.entryPointABI.Pack("getNonce", account, big.NewInt(0))

	result, err := b.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &b.entryPoint,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	return new(big.Int).SetBytes(result), nil
}

func (b *Bundler) sortByProfit(ops []*UserOperation) []*UserOperation {
	// 按 maxPriorityFeePerGas 排序
	sort.Slice(ops, func(i, j int) bool {
		return ops[i].MaxPriorityFeePerGas.Cmp(ops[j].MaxPriorityFeePerGas) > 0
	})
	return ops
}

type Bundle struct {
	Ops      []*UserOperation
	TotalGas uint64
}
```

### 2.2 Bundler RPC 服务

```go
package bundler

import (
	"context"
	"encoding/json"
	"net/http"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/common/hexutil"
)

// RPCServer Bundler RPC 服务
type RPCServer struct {
	bundler *Bundler
}

// NewRPCServer 创建 RPC 服务
func NewRPCServer(bundler *Bundler) *RPCServer {
	return &RPCServer{bundler: bundler}
}

// ServeHTTP 处理 HTTP 请求
func (s *RPCServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	var req JSONRPCRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		s.writeError(w, req.ID, -32700, "Parse error")
		return
	}

	var result interface{}
	var rpcErr *RPCError

	switch req.Method {
	case "eth_sendUserOperation":
		result, rpcErr = s.handleSendUserOp(r.Context(), req.Params)
	case "eth_estimateUserOperationGas":
		result, rpcErr = s.handleEstimateGas(r.Context(), req.Params)
	case "eth_getUserOperationByHash":
		result, rpcErr = s.handleGetUserOp(r.Context(), req.Params)
	case "eth_getUserOperationReceipt":
		result, rpcErr = s.handleGetReceipt(r.Context(), req.Params)
	case "eth_supportedEntryPoints":
		result = []string{EntryPointV06.Hex(), EntryPointV07.Hex()}
	case "eth_chainId":
		result = hexutil.EncodeBig(s.bundler.config.ChainID)
	default:
		rpcErr = &RPCError{Code: -32601, Message: "Method not found"}
	}

	if rpcErr != nil {
		s.writeError(w, req.ID, rpcErr.Code, rpcErr.Message)
		return
	}

	s.writeResult(w, req.ID, result)
}

// handleSendUserOp 处理 eth_sendUserOperation
func (s *RPCServer) handleSendUserOp(ctx context.Context, params json.RawMessage) (interface{}, *RPCError) {
	var args []json.RawMessage
	if err := json.Unmarshal(params, &args); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid params"}
	}

	if len(args) < 2 {
		return nil, &RPCError{Code: -32602, Message: "Missing params"}
	}

	var op UserOperation
	if err := json.Unmarshal(args[0], &op); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid userOp"}
	}

	var entryPoint common.Address
	if err := json.Unmarshal(args[1], &entryPoint); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid entryPoint"}
	}

	hash, err := s.bundler.HandleUserOp(ctx, &op, entryPoint)
	if err != nil {
		return nil, &RPCError{Code: -32500, Message: err.Error()}
	}

	return hash.Hex(), nil
}

// handleEstimateGas 处理 eth_estimateUserOperationGas
func (s *RPCServer) handleEstimateGas(ctx context.Context, params json.RawMessage) (interface{}, *RPCError) {
	var args []json.RawMessage
	if err := json.Unmarshal(params, &args); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid params"}
	}

	var op UserOperation
	if err := json.Unmarshal(args[0], &op); err != nil {
		return nil, &RPCError{Code: -32602, Message: "Invalid userOp"}
	}

	// 估算 gas
	estimate, err := s.bundler.EstimateGas(ctx, &op)
	if err != nil {
		return nil, &RPCError{Code: -32500, Message: err.Error()}
	}

	return map[string]string{
		"preVerificationGas":   hexutil.EncodeBig(estimate.PreVerificationGas),
		"verificationGasLimit": hexutil.EncodeBig(estimate.VerificationGasLimit),
		"callGasLimit":         hexutil.EncodeBig(estimate.CallGasLimit),
	}, nil
}

type JSONRPCRequest struct {
	JSONRPC string          `json:"jsonrpc"`
	ID      interface{}     `json:"id"`
	Method  string          `json:"method"`
	Params  json.RawMessage `json:"params"`
}

type RPCError struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
}

func (s *RPCServer) writeError(w http.ResponseWriter, id interface{}, code int, message string) {
	resp := map[string]interface{}{
		"jsonrpc": "2.0",
		"id":      id,
		"error": map[string]interface{}{
			"code":    code,
			"message": message,
		},
	}
	json.NewEncoder(w).Encode(resp)
}

func (s *RPCServer) writeResult(w http.ResponseWriter, id interface{}, result interface{}) {
	resp := map[string]interface{}{
		"jsonrpc": "2.0",
		"id":      id,
		"result":  result,
	}
	json.NewEncoder(w).Encode(resp)
}
```

## 3. Paymaster 实现

### 3.1 Paymaster 接口

```go
package paymaster

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// IPaymaster Paymaster 接口
type IPaymaster interface {
	// ValidatePaymasterUserOp 验证并决定是否为 UserOp 付款
	ValidatePaymasterUserOp(op *UserOperation, maxCost *big.Int) (*PaymasterValidation, error)

	// PostOp 执行后回调
	PostOp(mode PostOpMode, context []byte, actualGasCost *big.Int) error
}

type PaymasterValidation struct {
	Context    []byte
	ValidUntil uint64
	ValidAfter uint64
}

type PostOpMode uint8

const (
	OpSucceeded PostOpMode = iota
	OpReverted
	PostOpReverted
)

// VerifyingPaymaster 验证型 Paymaster
type VerifyingPaymaster struct {
	ethClient  *ethclient.Client
	address    common.Address
	signer     *ecdsa.PrivateKey
	entryPoint common.Address
}

// NewVerifyingPaymaster 创建验证型 Paymaster
func NewVerifyingPaymaster(
	ethClient *ethclient.Client,
	address common.Address,
	signer *ecdsa.PrivateKey,
) *VerifyingPaymaster {
	return &VerifyingPaymaster{
		ethClient:  ethClient,
		address:    address,
		signer:     signer,
		entryPoint: EntryPointV06,
	}
}

// SignUserOp 为 UserOp 签名
func (p *VerifyingPaymaster) SignUserOp(op *UserOperation, validUntil, validAfter uint64) ([]byte, error) {
	// 构建要签名的数据
	hash := p.getHash(op, validUntil, validAfter)

	// 签名
	signature, err := crypto.Sign(hash.Bytes(), p.signer)
	if err != nil {
		return nil, err
	}

	// 调整 v 值
	signature[64] += 27

	// 构建 paymasterAndData
	// paymaster (20) + validUntil (6) + validAfter (6) + signature (65)
	paymasterAndData := make([]byte, 0, 97)
	paymasterAndData = append(paymasterAndData, p.address.Bytes()...)
	paymasterAndData = append(paymasterAndData, encodeUint48(validUntil)...)
	paymasterAndData = append(paymasterAndData, encodeUint48(validAfter)...)
	paymasterAndData = append(paymasterAndData, signature...)

	return paymasterAndData, nil
}

func (p *VerifyingPaymaster) getHash(op *UserOperation, validUntil, validAfter uint64) common.Hash {
	// 类似 UserOp hash，但包含时间戳
	return crypto.Keccak256Hash(
		op.Pack(),
		p.address.Bytes(),
		encodeUint48(validUntil),
		encodeUint48(validAfter),
	)
}

func encodeUint48(v uint64) []byte {
	b := make([]byte, 6)
	for i := 5; i >= 0; i-- {
		b[i] = byte(v)
		v >>= 8
	}
	return b
}

// TokenPaymaster ERC20 代币支付 Paymaster
type TokenPaymaster struct {
	ethClient *ethclient.Client
	address   common.Address
	token     common.Address
	oracle    PriceOracle
	markup    *big.Int // basis points (10000 = 100%)
}

// GetTokenCost 计算代币成本
func (p *TokenPaymaster) GetTokenCost(op *UserOperation) (*big.Int, error) {
	// 计算 ETH 成本
	gasPrice := op.MaxFeePerGas
	totalGas := new(big.Int).Add(op.CallGasLimit, op.VerificationGasLimit)
	totalGas.Add(totalGas, op.PreVerificationGas)

	ethCost := new(big.Int).Mul(totalGas, gasPrice)

	// 获取代币/ETH 价格
	tokenPrice, err := p.oracle.GetPrice(p.token, common.Address{}) // token/ETH
	if err != nil {
		return nil, err
	}

	// 转换为代币数量
	tokenCost := new(big.Int).Mul(ethCost, tokenPrice)
	tokenCost.Div(tokenCost, big.NewInt(1e18))

	// 添加加价
	tokenCost.Mul(tokenCost, p.markup)
	tokenCost.Div(tokenCost, big.NewInt(10000))

	return tokenCost, nil
}

// SponsoringPaymaster 赞助型 Paymaster
type SponsoringPaymaster struct {
	ethClient *ethclient.Client
	address   common.Address
	signer    *ecdsa.PrivateKey

	// 赞助规则
	sponsoredApps map[common.Address]bool
	dailyLimit    *big.Int
	spent         map[common.Address]*big.Int
}

// ShouldSponsor 判断是否应该赞助
func (p *SponsoringPaymaster) ShouldSponsor(op *UserOperation) bool {
	// 检查是否是被赞助的应用
	// 从 callData 解析目标合约
	target := p.extractTarget(op.CallData)
	if !p.sponsoredApps[target] {
		return false
	}

	// 检查每日限额
	spent := p.spent[op.Sender]
	if spent == nil {
		spent = big.NewInt(0)
	}

	cost := p.estimateCost(op)
	if new(big.Int).Add(spent, cost).Cmp(p.dailyLimit) > 0 {
		return false
	}

	return true
}

func (p *SponsoringPaymaster) extractTarget(callData []byte) common.Address {
	// 解析 execute(target, value, data) 调用
	if len(callData) < 36 {
		return common.Address{}
	}
	return common.BytesToAddress(callData[16:36])
}

func (p *SponsoringPaymaster) estimateCost(op *UserOperation) *big.Int {
	totalGas := new(big.Int).Add(op.CallGasLimit, op.VerificationGasLimit)
	totalGas.Add(totalGas, op.PreVerificationGas)
	return new(big.Int).Mul(totalGas, op.MaxFeePerGas)
}
```

## 4. Smart Account 实现

### 4.1 简单账户

```go
package account

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// SimpleAccount 简单智能账户
type SimpleAccount struct {
	address    common.Address
	owner      common.Address
	entryPoint common.Address
	factory    common.Address
}

// SimpleAccountFactory 账户工厂
type SimpleAccountFactory struct {
	address common.Address
	abi     abi.ABI
}

// GetAddress 预计算账户地址
func (f *SimpleAccountFactory) GetAddress(owner common.Address, salt *big.Int) common.Address {
	// CREATE2 地址计算
	// address = keccak256(0xff + factory + salt + keccak256(initCode))[12:]

	initCodeHash := f.getInitCodeHash(owner)

	data := make([]byte, 85)
	data[0] = 0xff
	copy(data[1:21], f.address.Bytes())
	copy(data[21:53], common.LeftPadBytes(salt.Bytes(), 32))
	copy(data[53:85], initCodeHash.Bytes())

	return common.BytesToAddress(crypto.Keccak256(data)[12:])
}

func (f *SimpleAccountFactory) getInitCodeHash(owner common.Address) common.Hash {
	// 简化：实际需要包含完整的 initCode
	return crypto.Keccak256Hash(owner.Bytes())
}

// GetInitCode 获取部署初始化代码
func (f *SimpleAccountFactory) GetInitCode(owner common.Address, salt *big.Int) []byte {
	// factory address (20) + createAccount(owner, salt) calldata
	callData, _ := f.abi.Pack("createAccount", owner, salt)

	initCode := make([]byte, 20+len(callData))
	copy(initCode[:20], f.address.Bytes())
	copy(initCode[20:], callData)

	return initCode
}

// SmartAccountClient 智能账户客户端
type SmartAccountClient struct {
	ethClient  *ethclient.Client
	bundlerURL string
	account    *SimpleAccount
	privateKey *ecdsa.PrivateKey
	chainID    *big.Int
}

// NewSmartAccountClient 创建客户端
func NewSmartAccountClient(
	ethClient *ethclient.Client,
	bundlerURL string,
	privateKey *ecdsa.PrivateKey,
	chainID *big.Int,
) (*SmartAccountClient, error) {
	owner := crypto.PubkeyToAddress(privateKey.PublicKey)

	// 计算账户地址
	factory := &SimpleAccountFactory{
		address: common.HexToAddress("0x9406Cc6185a346906296840746125a0E44976454"), // 示例
	}
	accountAddr := factory.GetAddress(owner, big.NewInt(0))

	return &SmartAccountClient{
		ethClient:  ethClient,
		bundlerURL: bundlerURL,
		account: &SimpleAccount{
			address:    accountAddr,
			owner:      owner,
			entryPoint: EntryPointV06,
			factory:    factory.address,
		},
		privateKey: privateKey,
		chainID:    chainID,
	}, nil
}

// BuildUserOp 构建 UserOperation
func (c *SmartAccountClient) BuildUserOp(ctx context.Context, calls []Call) (*UserOperation, error) {
	// 检查账户是否已部署
	code, err := c.ethClient.CodeAt(ctx, c.account.address, nil)
	if err != nil {
		return nil, err
	}

	var initCode []byte
	if len(code) == 0 {
		// 需要部署
		factory := &SimpleAccountFactory{address: c.account.factory}
		initCode = factory.GetInitCode(c.account.owner, big.NewInt(0))
	}

	// 构建 callData
	callData, err := c.encodeExecute(calls)
	if err != nil {
		return nil, err
	}

	// 获取 nonce
	nonce, err := c.getNonce(ctx)
	if err != nil {
		return nil, err
	}

	// 获取 gas 价格
	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	op := &UserOperation{
		Sender:               c.account.address,
		Nonce:                nonce,
		InitCode:             initCode,
		CallData:             callData,
		CallGasLimit:         big.NewInt(200000),
		VerificationGasLimit: big.NewInt(100000),
		PreVerificationGas:   big.NewInt(50000),
		MaxFeePerGas:         gasPrice,
		MaxPriorityFeePerGas: big.NewInt(1e9), // 1 gwei
		PaymasterAndData:     nil,
		Signature:            nil,
	}

	return op, nil
}

// SignUserOp 签名 UserOperation
func (c *SmartAccountClient) SignUserOp(op *UserOperation) error {
	// 计算 UserOp hash
	opHash := op.Hash(c.account.entryPoint, c.chainID)

	// 签名
	signature, err := crypto.Sign(opHash.Bytes(), c.privateKey)
	if err != nil {
		return err
	}

	// 调整 v 值
	signature[64] += 27

	op.Signature = signature
	return nil
}

// SendUserOp 发送 UserOperation
func (c *SmartAccountClient) SendUserOp(ctx context.Context, op *UserOperation) (common.Hash, error) {
	// 签名
	if err := c.SignUserOp(op); err != nil {
		return common.Hash{}, err
	}

	// 发送到 Bundler
	client := &http.Client{}
	reqBody := map[string]interface{}{
		"jsonrpc": "2.0",
		"id":      1,
		"method":  "eth_sendUserOperation",
		"params":  []interface{}{op, c.account.entryPoint.Hex()},
	}

	body, _ := json.Marshal(reqBody)
	resp, err := client.Post(c.bundlerURL, "application/json", bytes.NewReader(body))
	if err != nil {
		return common.Hash{}, err
	}
	defer resp.Body.Close()

	var result struct {
		Result string `json:"result"`
		Error  *struct {
			Message string `json:"message"`
		} `json:"error"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return common.Hash{}, err
	}

	if result.Error != nil {
		return common.Hash{}, fmt.Errorf(result.Error.Message)
	}

	return common.HexToHash(result.Result), nil
}

func (c *SmartAccountClient) encodeExecute(calls []Call) ([]byte, error) {
	if len(calls) == 1 {
		// execute(target, value, data)
		return c.accountABI.Pack("execute", calls[0].Target, calls[0].Value, calls[0].Data)
	}

	// executeBatch(targets[], values[], datas[])
	targets := make([]common.Address, len(calls))
	values := make([]*big.Int, len(calls))
	datas := make([][]byte, len(calls))

	for i, call := range calls {
		targets[i] = call.Target
		values[i] = call.Value
		datas[i] = call.Data
	}

	return c.accountABI.Pack("executeBatch", targets, values, datas)
}

func (c *SmartAccountClient) getNonce(ctx context.Context) (*big.Int, error) {
	// 调用 EntryPoint.getNonce
	entryPointABI, _ := abi.JSON(strings.NewReader(EntryPointABI))
	callData, _ := entryPointABI.Pack("getNonce", c.account.address, big.NewInt(0))

	result, err := c.ethClient.CallContract(ctx, ethereum.CallMsg{
		To:   &c.account.entryPoint,
		Data: callData,
	}, nil)
	if err != nil {
		return nil, err
	}

	return new(big.Int).SetBytes(result), nil
}

type Call struct {
	Target common.Address
	Value  *big.Int
	Data   []byte
}
```

## 5. AA 套利机会

### 5.1 Bundler MEV 提取

```go
package mev

import (
	"context"
	"math/big"
	"sort"

	"github.com/ethereum/go-ethereum/common"
)

// BundlerMEV Bundler MEV 提取
type BundlerMEV struct {
	bundler         *Bundler
	dexAggregator   DEXAggregator
	flashbotsClient *FlashbotsClient
}

// ExtractMEV 从 UserOp bundle 中提取 MEV
func (m *BundlerMEV) ExtractMEV(ctx context.Context, ops []*UserOperation) (*MEVBundle, error) {
	// 分析每个 UserOp 的交易意图
	intents := make([]*TxIntent, 0, len(ops))
	for _, op := range ops {
		intent := m.analyzeUserOp(op)
		if intent != nil {
			intents = append(intents, intent)
		}
	}

	// 寻找 MEV 机会
	opportunities := m.findMEVOpportunities(ctx, intents)

	if len(opportunities) == 0 {
		return nil, nil
	}

	// 构建 MEV bundle
	return m.buildMEVBundle(ctx, ops, opportunities)
}

// analyzeUserOp 分析 UserOp 交易意图
func (m *BundlerMEV) analyzeUserOp(op *UserOperation) *TxIntent {
	// 解析 callData
	if len(op.CallData) < 4 {
		return nil
	}

	selector := op.CallData[:4]

	// 常见的交易模式
	switch {
	case bytes.Equal(selector, swapExactTokensSelector):
		return m.parseSwapIntent(op)
	case bytes.Equal(selector, executeSelector):
		return m.parseExecuteIntent(op)
	default:
		return nil
	}
}

// parseSwapIntent 解析 swap 意图
func (m *BundlerMEV) parseSwapIntent(op *UserOperation) *TxIntent {
	// 解析 swap 参数
	// swapExactTokensForTokens(amountIn, amountOutMin, path, to, deadline)

	return &TxIntent{
		Type:         IntentSwap,
		Sender:       op.Sender,
		TokenIn:      common.Address{}, // 从 path 解析
		TokenOut:     common.Address{},
		AmountIn:     big.NewInt(0),
		MinAmountOut: big.NewInt(0),
	}
}

// findMEVOpportunities 寻找 MEV 机会
func (m *BundlerMEV) findMEVOpportunities(ctx context.Context, intents []*TxIntent) []*MEVOpportunity {
	var opportunities []*MEVOpportunity

	// 1. 寻找三明治攻击机会
	for _, intent := range intents {
		if intent.Type == IntentSwap {
			sandwich := m.findSandwichOpportunity(ctx, intent)
			if sandwich != nil {
				opportunities = append(opportunities, sandwich)
			}
		}
	}

	// 2. 寻找套利机会（基于 UserOps 的影响）
	arbitrage := m.findArbitrageAfterUserOps(ctx, intents)
	if arbitrage != nil {
		opportunities = append(opportunities, arbitrage)
	}

	// 3. 寻找清算机会
	liquidations := m.findLiquidationOpportunities(ctx, intents)
	opportunities = append(opportunities, liquidations...)

	return opportunities
}

// buildMEVBundle 构建 MEV bundle
func (m *BundlerMEV) buildMEVBundle(ctx context.Context, userOps []*UserOperation, mevOps []*MEVOpportunity) (*MEVBundle, error) {
	bundle := &MEVBundle{
		UserOps: userOps,
	}

	// 对每个 MEV 机会，插入前置/后置交易
	for _, mev := range mevOps {
		switch mev.Type {
		case MEVSandwich:
			// 前置交易
			bundle.PreTxs = append(bundle.PreTxs, mev.FrontrunTx)
			// 后置交易
			bundle.PostTxs = append(bundle.PostTxs, mev.BackrunTx)

		case MEVArbitrage:
			// 在 UserOps 之后执行
			bundle.PostTxs = append(bundle.PostTxs, mev.ArbitrageTx)

		case MEVLiquidation:
			// 在 UserOps 之后执行
			bundle.PostTxs = append(bundle.PostTxs, mev.LiquidationTx)
		}
	}

	return bundle, nil
}

type TxIntent struct {
	Type         IntentType
	Sender       common.Address
	TokenIn      common.Address
	TokenOut     common.Address
	AmountIn     *big.Int
	MinAmountOut *big.Int
}

type IntentType int

const (
	IntentSwap IntentType = iota
	IntentTransfer
	IntentApprove
	IntentOther
)

type MEVOpportunity struct {
	Type           MEVType
	ExpectedProfit *big.Int

	// Sandwich
	FrontrunTx *types.Transaction
	BackrunTx  *types.Transaction

	// Arbitrage
	ArbitrageTx *types.Transaction

	// Liquidation
	LiquidationTx *types.Transaction
}

type MEVType int

const (
	MEVSandwich MEVType = iota
	MEVArbitrage
	MEVLiquidation
)

type MEVBundle struct {
	PreTxs  []*types.Transaction
	UserOps []*UserOperation
	PostTxs []*types.Transaction
}
```

### 5.2 UserOp 抢跑保护

```go
package protection

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
)

// FrontrunProtector 抢跑保护
type FrontrunProtector struct {
	flashbotsClient *FlashbotsClient
	privatePool     PrivatePoolProvider
}

// ProtectedSubmit 保护性提交
func (p *FrontrunProtector) ProtectedSubmit(ctx context.Context, op *UserOperation) (common.Hash, error) {
	// 1. 分析 UserOp 是否容易被抢跑
	risk := p.analyzeRisk(op)

	if risk.Level == RiskLow {
		// 低风险，可以通过公共 mempool
		return p.submitPublic(ctx, op)
	}

	// 2. 高风险，使用私有提交
	if risk.Level == RiskHigh {
		// 计算需要的贿赂金额
		bribe := p.calculateBribe(risk)

		// 通过 Flashbots 提交
		return p.submitViaFlashbots(ctx, op, bribe)
	}

	// 3. 中等风险，使用 MEV Blocker
	return p.submitViaMEVBlocker(ctx, op)
}

// analyzeRisk 分析抢跑风险
func (p *FrontrunProtector) analyzeRisk(op *UserOperation) *RiskAnalysis {
	// 解析交易意图
	intent := p.parseIntent(op)

	risk := &RiskAnalysis{
		Level: RiskLow,
	}

	switch intent.Type {
	case IntentSwap:
		// 大额 swap 风险高
		if intent.AmountIn.Cmp(big.NewInt(10e18)) > 0 { // > 10 ETH
			risk.Level = RiskHigh
			risk.PotentialLoss = p.estimateSandwichLoss(intent)
		} else if intent.AmountIn.Cmp(big.NewInt(1e18)) > 0 { // > 1 ETH
			risk.Level = RiskMedium
		}

		// 检查滑点设置
		slippage := p.calculateSlippage(intent)
		if slippage > 0.05 { // > 5%
			risk.Level = RiskHigh
		}

	case IntentApprove:
		// approve 通常安全
		risk.Level = RiskLow
	}

	return risk
}

// estimateSandwichLoss 估算三明治攻击损失
func (p *FrontrunProtector) estimateSandwichLoss(intent *TxIntent) *big.Int {
	// 基于交易规模和池深度估算
	// 简化计算
	return new(big.Int).Div(intent.AmountIn, big.NewInt(100)) // ~1%
}

// submitViaFlashbots 通过 Flashbots 提交
func (p *FrontrunProtector) submitViaFlashbots(ctx context.Context, op *UserOperation, bribe *big.Int) (common.Hash, error) {
	// 构建包含贿赂的 bundle
	bundle := &FlashbotsBundle{
		Transactions: []interface{}{
			&UserOpWrapper{UserOp: op},
		},
		BribeAmount: bribe,
	}

	return p.flashbotsClient.SendBundle(ctx, bundle)
}

type RiskLevel int

const (
	RiskLow RiskLevel = iota
	RiskMedium
	RiskHigh
)

type RiskAnalysis struct {
	Level         RiskLevel
	PotentialLoss *big.Int
	Reasons       []string
}
```

## 6. 完整示例：AA 套利机器人

```go
package main

import (
	"context"
	"log"
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

func main() {
	// 连接以太坊
	ethClient, err := ethclient.Dial("https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY")
	if err != nil {
		log.Fatal(err)
	}

	// 加载私钥
	privateKey, err := crypto.HexToECDSA("YOUR_PRIVATE_KEY")
	if err != nil {
		log.Fatal(err)
	}

	// 创建智能账户客户端
	aaClient, err := NewSmartAccountClient(
		ethClient,
		"https://bundler.example.com/rpc",
		privateKey,
		big.NewInt(1), // mainnet
	)
	if err != nil {
		log.Fatal(err)
	}

	// 套利机器人
	bot := &AAArbitrageBot{
		aaClient:      aaClient,
		dexAggregator: NewDEXAggregator(ethClient),
		priceOracle:   NewPriceOracle(ethClient),
	}

	// 运行
	ctx := context.Background()
	bot.Run(ctx)
}

// AAArbitrageBot AA 套利机器人
type AAArbitrageBot struct {
	aaClient      *SmartAccountClient
	dexAggregator *DEXAggregator
	priceOracle   *PriceOracle
}

// Run 运行机器人
func (b *AAArbitrageBot) Run(ctx context.Context) {
	ticker := time.NewTicker(12 * time.Second) // 每个区块
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			if err := b.checkAndExecute(ctx); err != nil {
				log.Printf("Error: %v", err)
			}
		}
	}
}

// checkAndExecute 检查并执行套利
func (b *AAArbitrageBot) checkAndExecute(ctx context.Context) error {
	// 寻找套利机会
	opportunity, err := b.findOpportunity(ctx)
	if err != nil {
		return err
	}

	if opportunity == nil {
		return nil
	}

	log.Printf("Found opportunity: profit=%s", opportunity.Profit.String())

	// 构建套利调用
	calls := b.buildArbitrageCalls(opportunity)

	// 构建 UserOp
	op, err := b.aaClient.BuildUserOp(ctx, calls)
	if err != nil {
		return err
	}

	// 使用 Paymaster（如果有）
	if b.hasPaymaster() {
		paymasterData, err := b.getPaymasterData(ctx, op)
		if err == nil {
			op.PaymasterAndData = paymasterData
		}
	}

	// 发送 UserOp
	hash, err := b.aaClient.SendUserOp(ctx, op)
	if err != nil {
		return err
	}

	log.Printf("UserOp submitted: %s", hash.Hex())

	// 等待确认
	return b.waitForReceipt(ctx, hash)
}

// findOpportunity 寻找套利机会
func (b *AAArbitrageBot) findOpportunity(ctx context.Context) (*ArbitrageOpportunity, error) {
	// 获取热门代币对价格
	pairs := []TokenPair{
		{WETH, USDC},
		{WETH, USDT},
		{WETH, DAI},
	}

	for _, pair := range pairs {
		// 检查跨 DEX 套利
		opp := b.checkCrossDEXArbitrage(ctx, pair)
		if opp != nil && opp.Profit.Cmp(b.minProfit()) > 0 {
			return opp, nil
		}
	}

	return nil, nil
}

// buildArbitrageCalls 构建套利调用
func (b *AAArbitrageBot) buildArbitrageCalls(opp *ArbitrageOpportunity) []Call {
	var calls []Call

	// 1. 第一个 DEX swap
	calls = append(calls, Call{
		Target: opp.DEX1Router,
		Value:  big.NewInt(0),
		Data:   b.encodeSwap(opp.DEX1, opp.TokenIn, opp.TokenMid, opp.AmountIn),
	})

	// 2. 第二个 DEX swap
	calls = append(calls, Call{
		Target: opp.DEX2Router,
		Value:  big.NewInt(0),
		Data:   b.encodeSwap(opp.DEX2, opp.TokenMid, opp.TokenOut, opp.ExpectedMidAmount),
	})

	return calls
}

func (b *AAArbitrageBot) minProfit() *big.Int {
	return big.NewInt(1e16) // 0.01 ETH
}

type ArbitrageOpportunity struct {
	DEX1Router        common.Address
	DEX2Router        common.Address
	DEX1              string
	DEX2              string
	TokenIn           common.Address
	TokenMid          common.Address
	TokenOut          common.Address
	AmountIn          *big.Int
	ExpectedMidAmount *big.Int
	ExpectedOutput    *big.Int
	Profit            *big.Int
}
```

## 7. 总结

### 7.1 AA 生态对比

| 组件                | 功能         | 主要项目                      |
|-------------------|------------|---------------------------|
| **EntryPoint**    | 统一入口       | ERC-4337 标准               |
| **Bundler**       | 打包 UserOps | Stackup, Pimlico, Alchemy |
| **Paymaster**     | 代付 Gas     | Pimlico, Stackup          |
| **Smart Account** | 智能钱包       | Safe, Biconomy, ZeroDev   |

### 7.2 AA 套利优势

1. **批量操作**: 单个 UserOp 包含多个调用
2. **Gas 抽象**: 可用代币支付或赞助
3. **灵活验证**: 支持多签、社交恢复等
4. **MEV 保护**: 通过 Bundler 私有提交

### 7.3 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     AA 套利架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │  价格监控     │───▶│  机会检测      │───▶│  策略选择     │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                │            │
│                                                ▼            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  Smart Account                       │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐      │   │
│  │  │  Swap A    │─▶│  Swap B    │─▶│  Swap C    │      │   │
│  │  └────────────┘  └────────────┘  └────────────┘      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                │            │
│                                                ▼            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Bundler    │───▶│  EntryPoint  │───▶│   链上执行    │   │
│  │  (可选私有)   │    │              │    │              │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

账户抽象为套利机器人提供了更灵活的执行环境，通过批量操作和 Gas 抽象可以优化成本，同时 Bundler 层的私有提交也提供了一定的
MEV 保护。

---

## 8. EIP-7702：协议层账户抽象

### 8.1 背景与动机

EIP-7702 是 Vitalik Buterin 等人在 2024 年提出的协议层账户抽象方案，已被纳入 **Prague/Electra (Pectra)** 升级。它允许
EOA（外部拥有账户）在单笔交易执行期间临时"委托"给智能合约代码，结合了 EOA 的简洁性和智能合约钱包的灵活性。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    账户抽象方案演进                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐        │
│  │   EIP-3074    │    │   ERC-4337    │    │   EIP-7702    │        │
│  │   (已废弃)     │    │   (应用层)     │    │   (协议层)     │        │
│  └───────────────┘    └───────────────┘    └───────────────┘        │
│         │                    │                    │                 │
│         ▼                    ▼                    ▼                 │
│  AUTH + AUTHCALL       UserOperation         SET_CODE_TX            │
│  安全风险高            需要新基础设施         EOA 临时升级               │
│                        (Bundler/Paymaster)   复用现有基础设施          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**为什么需要 EIP-7702？**

| 痛点         | ERC-4337 方案  | EIP-7702 方案  |
|------------|--------------|--------------|
| EOA 无法批量操作 | 需迁移到智能账户     | EOA 直接获得能力   |
| 新用户需要 ETH  | Paymaster 代付 | Paymaster 代付 |
| 资产迁移成本     | 需转移所有资产      | 无需迁移         |
| 协议兼容性      | 需要适配         | 完全兼容         |

### 8.2 EIP-7702 vs ERC-4337 对比

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 vs ERC-4337                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ERC-4337 (应用层)                                                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   User ──▶ UserOp ──▶ Bundler ──▶ EntryPoint ──▶ Account    │   │
│  │                                        │                     │   │
│  │                              validateUserOp()                │   │
│  │                              execute()                       │   │
│  │                                                              │   │
│  │   特点：                                                      │   │
│  │   • 需要智能合约钱包                                           │   │
│  │   • 需要 Bundler 基础设施                                      │   │
│  │   • 完全去中心化                                              │   │
│  │   • Gas 开销较高 (~42k 额外 gas)                              │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  EIP-7702 (协议层)                                                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   User ──▶ SetCodeTx ──▶ Mempool ──▶ 直接执行                 │   │
│  │                │                                             │   │
│  │        authorization_list                                    │   │
│  │        [chain_id, address, nonce, y_parity, r, s]           │   │
│  │                                                              │   │
│  │   特点：                                                      │   │
│  │   • EOA 临时获得智能合约能力                                    │   │
│  │   • 复用现有 mempool                                          │   │
│  │   • 交易完成后代码自动清除                                      │   │
│  │   • Gas 开销低 (~25k per authorization)                       │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**详细对比表**

| 特性         | ERC-4337  | EIP-7702          |
|------------|-----------|-------------------|
| 实现层级       | 应用层       | 协议层               |
| 账户类型       | 智能合约账户    | EOA + 临时代码        |
| 需要迁移       | 是         | 否                 |
| Bundler    | 必需        | 可选                |
| EntryPoint | 必需        | 不需要               |
| Gas 开销     | 较高        | 较低                |
| 代码持久性      | 永久        | 临时/可撤销            |
| 批量操作       | 支持        | 支持                |
| Gas 代付     | Paymaster | Paymaster/Sponsor |
| 兼容性        | 需要适配      | 完全兼容              |
| 上线时间       | 已上线       | Pectra 升级         |

### 8.3 工作原理

#### 8.3.1 核心概念

EIP-7702 引入了新的交易类型 `0x04`（SET_CODE_TX），允许 EOA 通过签名授权，将自己的代码槽临时指向一个智能合约地址。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 执行流程                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 用户签名授权                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   EOA Owner 签名：                                            │   │
│  │   sign(keccak256(MAGIC || chain_id || nonce || address))    │   │
│  │                                                              │   │
│  │   MAGIC = 0x05 (EIP-7702 专用前缀)                            │   │
│  │   address = 委托目标合约地址                                    │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  2. 构建 SetCode 交易                                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   {                                                          │   │
│  │     type: 0x04,                                              │   │
│  │     chain_id: 1,                                             │   │
│  │     nonce: 5,                                                │   │
│  │     to: <EOA_address>,  // 调用自己                           │   │
│  │     data: <execute_calldata>,                                │   │
│  │     authorization_list: [                                    │   │
│  │       { chain_id, address, nonce, y_parity, r, s }           │   │
│  │     ]                                                        │   │
│  │   }                                                          │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  3. 协议层处理                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   对每个 authorization:                                       │   │
│  │   ├─ 验证签名                                                 │   │
│  │   ├─ 验证 nonce                                              │   │
│  │   ├─ 设置 EOA.code = 0xef0100 || address (delegation)        │   │
│  │   └─ 递增 EOA.nonce                                          │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  4. 执行交易                                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   CALL to EOA:                                               │   │
│  │   ├─ 检测 delegation designator (0xef0100)                    │   │
│  │   ├─ 加载目标合约代码                                          │   │
│  │   └─ 在 EOA 上下文中执行 (storage, balance 都是 EOA 的)         │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 8.3.2 Delegation Designator

EIP-7702 使用特殊的 **delegation designator** 来标记委托代码：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Delegation Designator                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   格式: 0xef0100 || address (23 bytes)                              │
│                                                                     │
│   ┌────────┬────────────────────────────────────────┐               │
│   │ ef0100 │     20-byte contract address          │               │
│   │ 3 bytes│              20 bytes                 │               │
│   └────────┴────────────────────────────────────────┘               │
│                                                                     │
│   • 0xef 是 EOF (EVM Object Format) 保留前缀                         │
│   • 0x01 表示 delegation 类型                                        │
│   • 0x00 是版本号                                                    │
│   • 后面的 20 bytes 是委托目标地址                                     │
│                                                                     │
│   当 EVM 检测到这个前缀时：                                            │
│   1. 不执行这个 "code"                                               │
│   2. 而是加载目标地址的代码                                            │
│   3. 在当前账户上下文中执行                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.4 数据结构与实现

#### 8.4.1 Go 语言实现

```go
package eip7702

import (
	"crypto/ecdsa"
	"errors"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/rlp"
)

// Authorization EIP-7702 授权结构
type Authorization struct {
	ChainID uint64         `json:"chainId"`
	Address common.Address `json:"address"` // 委托目标合约
	Nonce   uint64         `json:"nonce"`   // 授权者的 nonce
	YParity uint8          `json:"yParity"` // 签名 v 值 (0 或 1)
	R       *big.Int       `json:"r"`
	S       *big.Int       `json:"s"`
}

// SetCodeTx EIP-7702 交易类型
type SetCodeTx struct {
	ChainID           *big.Int
	Nonce             uint64
	GasTipCap         *big.Int // maxPriorityFeePerGas
	GasFeeCap         *big.Int // maxFeePerGas
	Gas               uint64
	To                *common.Address
	Value             *big.Int
	Data              []byte
	AccessList        AccessList
	AuthorizationList []Authorization
	V, R, S           *big.Int
}

const (
	// SetCodeTxType EIP-7702 交易类型标识
	SetCodeTxType = 0x04

	// DelegationPrefix delegation designator 前缀
	DelegationPrefix = 0xef0100

	// AuthMagic 授权签名 magic 字节
	AuthMagic = 0x05

	// PER_EMPTY_ACCOUNT_COST 设置空账户代码的 gas 成本
	PER_EMPTY_ACCOUNT_COST = 25000

	// PER_AUTH_BASE_COST 每个授权的基础 gas 成本
	PER_AUTH_BASE_COST = 2500
)

// SignAuthorization 签名授权
func SignAuthorization(
	privateKey *ecdsa.PrivateKey,
	chainID uint64,
	delegateAddress common.Address,
	nonce uint64,
) (*Authorization, error) {
	// 构建签名数据
	// MAGIC || rlp([chain_id, address, nonce])
	authData := []interface{}{chainID, delegateAddress, nonce}
	encoded, err := rlp.EncodeToBytes(authData)
	if err != nil {
		return nil, err
	}

	// 添加 magic 前缀
	toSign := append([]byte{AuthMagic}, encoded...)
	hash := crypto.Keccak256(toSign)

	// 签名
	sig, err := crypto.Sign(hash, privateKey)
	if err != nil {
		return nil, err
	}

	return &Authorization{
		ChainID: chainID,
		Address: delegateAddress,
		Nonce:   nonce,
		YParity: sig[64], // v - 27, 已经是 0 或 1
		R:       new(big.Int).SetBytes(sig[:32]),
		S:       new(big.Int).SetBytes(sig[32:64]),
	}, nil
}

// RecoverAuthority 从授权中恢复签名者地址
func (auth *Authorization) RecoverAuthority() (common.Address, error) {
	// 重建签名数据
	authData := []interface{}{auth.ChainID, auth.Address, auth.Nonce}
	encoded, err := rlp.EncodeToBytes(authData)
	if err != nil {
		return common.Address{}, err
	}

	toSign := append([]byte{AuthMagic}, encoded...)
	hash := crypto.Keccak256(toSign)

	// 重建签名
	sig := make([]byte, 65)
	auth.R.FillBytes(sig[:32])
	auth.S.FillBytes(sig[32:64])
	sig[64] = auth.YParity

	// 恢复公钥
	pubKey, err := crypto.SigToPub(hash, sig)
	if err != nil {
		return common.Address{}, err
	}

	return crypto.PubkeyToAddress(*pubKey), nil
}

// DelegationDesignator 生成 delegation designator
func DelegationDesignator(target common.Address) []byte {
	// 0xef0100 || address
	code := make([]byte, 23)
	code[0] = 0xef
	code[1] = 0x01
	code[2] = 0x00
	copy(code[3:], target.Bytes())
	return code
}

// IsDelegation 检查代码是否是 delegation
func IsDelegation(code []byte) bool {
	return len(code) == 23 &&
		code[0] == 0xef &&
		code[1] == 0x01 &&
		code[2] == 0x00
}

// GetDelegationTarget 获取委托目标地址
func GetDelegationTarget(code []byte) (common.Address, bool) {
	if !IsDelegation(code) {
		return common.Address{}, false
	}
	return common.BytesToAddress(code[3:23]), true
}
```

#### 8.4.2 交易处理器实现

```go
package eip7702

import (
	"errors"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/state"
)

var (
	ErrInvalidChainID     = errors.New("invalid chain id")
	ErrInvalidNonce       = errors.New("invalid nonce")
	ErrInvalidSignature   = errors.New("invalid signature")
	ErrAccountHasCode     = errors.New("account already has code")
	ErrAuthorizationEmpty = errors.New("authorization list empty")
)

// AuthorizationProcessor 授权处理器
type AuthorizationProcessor struct {
	chainID *big.Int
	state   *state.StateDB
}

// NewAuthorizationProcessor 创建授权处理器
func NewAuthorizationProcessor(chainID *big.Int, state *state.StateDB) *AuthorizationProcessor {
	return &AuthorizationProcessor{
		chainID: chainID,
		state:   state,
	}
}

// ProcessAuthorizationList 处理授权列表
func (p *AuthorizationProcessor) ProcessAuthorizationList(
	authList []Authorization,
) (uint64, error) {
	if len(authList) == 0 {
		return 0, ErrAuthorizationEmpty
	}

	var totalGas uint64

	for _, auth := range authList {
		gas, err := p.processAuthorization(&auth)
		if err != nil {
			// 单个授权失败不影响其他授权
			// 但不计入 gas 退款
			continue
		}
		totalGas += gas
	}

	return totalGas, nil
}

// processAuthorization 处理单个授权
func (p *AuthorizationProcessor) processAuthorization(auth *Authorization) (uint64, error) {
	// 1. 验证 chain_id
	// chain_id 必须是 0 (任意链) 或当前链 ID
	if auth.ChainID != 0 && auth.ChainID != p.chainID.Uint64() {
		return 0, ErrInvalidChainID
	}

	// 2. 恢复授权者地址
	authority, err := auth.RecoverAuthority()
	if err != nil {
		return 0, ErrInvalidSignature
	}

	// 3. 验证 nonce
	currentNonce := p.state.GetNonce(authority)
	if auth.Nonce != currentNonce {
		return 0, ErrInvalidNonce
	}

	// 4. 计算 gas
	var gas uint64
	if p.state.Empty(authority) {
		gas = PER_EMPTY_ACCOUNT_COST
	} else {
		gas = PER_AUTH_BASE_COST
	}

	// 5. 增加 nonce
	p.state.SetNonce(authority, currentNonce+1)

	// 6. 设置 delegation
	if auth.Address == (common.Address{}) {
		// 清除 delegation
		p.state.SetCode(authority, nil)
	} else {
		// 设置 delegation designator
		designator := DelegationDesignator(auth.Address)
		p.state.SetCode(authority, designator)
	}

	return gas, nil
}

// ExecuteWithDelegation 在委托上下文中执行
func ExecuteWithDelegation(
	state *state.StateDB,
	caller common.Address,
	target common.Address,
	input []byte,
	gas uint64,
	value *big.Int,
) ([]byte, uint64, error) {
	// 获取目标账户代码
	code := state.GetCode(target)

	// 检查是否是 delegation
	if delegateAddr, isDelegation := GetDelegationTarget(code); isDelegation {
		// 加载委托目标的代码
		delegateCode := state.GetCode(delegateAddr)

		// 在 target 的上下文中执行 delegateCode
		// storage 和 balance 操作都作用于 target
		return executeCode(state, target, caller, delegateCode, input, gas, value)
	}

	// 普通执行
	return executeCode(state, target, caller, code, input, gas, value)
}

// executeCode 执行代码 (简化版)
func executeCode(
	state *state.StateDB,
	context common.Address,
	caller common.Address,
	code []byte,
	input []byte,
	gas uint64,
	value *big.Int,
) ([]byte, uint64, error) {
	// 实际实现需要调用 EVM 解释器
	// 这里简化处理
	return nil, gas, nil
}
```

#### 8.4.3 智能账户委托合约示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SimpleDelegate
 * @notice EIP-7702 委托合约示例
 * @dev EOA 可以通过 EIP-7702 委托到此合约，获得批量执行等能力
 */
contract SimpleDelegate {
    // 执行单个调用
    function execute(
        address target,
        uint256 value,
        bytes calldata data
    ) external payable returns (bytes memory) {
        // msg.sender 是交易发送者
        // address(this) 是委托的 EOA 地址
        require(msg.sender == address(this), "only self");

        (bool success, bytes memory result) = target.call{value: value}(data);
        require(success, "call failed");
        return result;
    }

    // 批量执行
    function executeBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata datas
    ) external payable returns (bytes[] memory results) {
        require(msg.sender == address(this), "only self");
        require(
            targets.length == values.length && values.length == datas.length,
            "length mismatch"
        );

        results = new bytes[](targets.length);

        for (uint256 i = 0; i < targets.length; i++) {
            (bool success, bytes memory result) = targets[i].call{value: values[i]}(datas[i]);
            require(success, "call failed");
            results[i] = result;
        }
    }

    // 支持接收 ETH
    receive() external payable {}
}

/**
 * @title SponsoredDelegate
 * @notice 支持 Gas 代付的委托合约
 */
contract SponsoredDelegate {
    mapping(address => bool) public sponsors;
    mapping(address => uint256) public nonces;

    event Sponsored(address indexed sponsor, address indexed user, uint256 gasCost);

    /**
     * @notice 代付执行
     * @param user 用户 EOA 地址 (委托到此合约的)
     * @param target 目标合约
     * @param data 调用数据
     * @param userSig 用户签名
     */
    function sponsoredExecute(
        address user,
        address target,
        bytes calldata data,
        bytes calldata userSig
    ) external {
        require(sponsors[msg.sender], "not sponsor");

        // 验证用户签名
        bytes32 hash = keccak256(abi.encodePacked(
            user,
            target,
            data,
            nonces[user]++
        ));
        bytes32 ethHash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            hash
        ));

        address signer = ecrecover(ethHash,
            uint8(userSig[64]) + 27,
            bytes32(userSig[0 : 32]),
            bytes32(userSig[32 : 64])
        );
        require(signer == user, "invalid signature");

        // 作为用户执行 (需要用户已委托到此合约)
        uint256 gasBefore = gasleft();

        // 这里的 call 会在用户账户上下文中执行
        (bool success,) = target.call(data);
        require(success, "execution failed");

        uint256 gasUsed = gasBefore - gasleft();
        emit Sponsored(msg.sender, user, gasUsed);
    }

    function addSponsor(address sponsor) external {
        // 简化：实际应有权限控制
        sponsors[sponsor] = true;
    }
}

/**
 * @title MultiSigDelegate
 * @notice 多签委托合约
 */
contract MultiSigDelegate {
    uint256 public threshold;
    mapping(address => bool) public signers;
    uint256 public signerCount;

    function initialize(address[] calldata _signers, uint256 _threshold) external {
        require(threshold == 0, "already initialized");
        require(_threshold > 0 && _threshold <= _signers.length, "invalid threshold");

        for (uint256 i = 0; i < _signers.length; i++) {
            signers[_signers[i]] = true;
        }
        signerCount = _signers.length;
        threshold = _threshold;
    }

    function executeMultiSig(
        address target,
        uint256 value,
        bytes calldata data,
        bytes[] calldata signatures
    ) external payable returns (bytes memory) {
        require(signatures.length >= threshold, "not enough signatures");

        bytes32 hash = keccak256(abi.encodePacked(
            address(this), // EOA 地址
            target,
            value,
            data
        ));

        address lastSigner;
        for (uint256 i = 0; i < threshold; i++) {
            address signer = recoverSigner(hash, signatures[i]);
            require(signers[signer], "invalid signer");
            require(signer > lastSigner, "signatures not sorted");
            lastSigner = signer;
        }

        (bool success, bytes memory result) = target.call{value: value}(data);
        require(success, "call failed");
        return result;
    }

    function recoverSigner(bytes32 hash, bytes calldata sig) internal pure returns (address) {
        bytes32 ethHash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            hash
        ));
        return ecrecover(ethHash,
            uint8(sig[64]) + 27,
            bytes32(sig[0 : 32]),
            bytes32(sig[32 : 64])
        );
    }
}
```

### 8.5 完整使用示例

#### 8.5.1 Go 客户端实现

```go
package main

import (
	"context"
	"crypto/ecdsa"
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

// EIP7702Client EIP-7702 客户端
type EIP7702Client struct {
	ethClient    *ethclient.Client
	privateKey   *ecdsa.PrivateKey
	address      common.Address
	chainID      *big.Int
	delegateAddr common.Address // 委托目标合约
	delegateABI  abi.ABI
}

// NewEIP7702Client 创建客户端
func NewEIP7702Client(
	ethClient *ethclient.Client,
	privateKey *ecdsa.PrivateKey,
	chainID *big.Int,
	delegateAddr common.Address,
) *EIP7702Client {
	address := crypto.PubkeyToAddress(privateKey.PublicKey)

	// SimpleDelegate ABI
	delegateABI, _ := abi.JSON(strings.NewReader(`[
		{
			"name": "execute",
			"type": "function",
			"inputs": [
				{"name": "target", "type": "address"},
				{"name": "value", "type": "uint256"},
				{"name": "data", "type": "bytes"}
			],
			"outputs": [{"name": "", "type": "bytes"}]
		},
		{
			"name": "executeBatch",
			"type": "function",
			"inputs": [
				{"name": "targets", "type": "address[]"},
				{"name": "values", "type": "uint256[]"},
				{"name": "datas", "type": "bytes[]"}
			],
			"outputs": [{"name": "", "type": "bytes[]"}]
		}
	]`))

	return &EIP7702Client{
		ethClient:    ethClient,
		privateKey:   privateKey,
		address:      address,
		chainID:      chainID,
		delegateAddr: delegateAddr,
		delegateABI:  delegateABI,
	}
}

// Execute 执行单个调用
func (c *EIP7702Client) Execute(
	ctx context.Context,
	target common.Address,
	value *big.Int,
	data []byte,
) (*types.Transaction, error) {
	// 获取当前 nonce
	nonce, err := c.ethClient.PendingNonceAt(ctx, c.address)
	if err != nil {
		return nil, err
	}

	// 创建授权
	auth, err := SignAuthorization(
		c.privateKey,
		c.chainID.Uint64(),
		c.delegateAddr,
		nonce,
	)
	if err != nil {
		return nil, err
	}

	// 构建调用数据
	callData, err := c.delegateABI.Pack("execute", target, value, data)
	if err != nil {
		return nil, err
	}

	// 获取 gas 价格
	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	// 构建 SetCode 交易
	tx := &SetCodeTx{
		ChainID:           c.chainID,
		Nonce:             nonce + 1, // 授权会消耗一个 nonce
		GasTipCap:         big.NewInt(1e9),
		GasFeeCap:         new(big.Int).Add(gasPrice, big.NewInt(1e9)),
		Gas:               200000,
		To:                &c.address, // 调用自己
		Value:             value,
		Data:              callData,
		AuthorizationList: []Authorization{*auth},
	}

	// 签名交易
	signedTx, err := c.signSetCodeTx(tx)
	if err != nil {
		return nil, err
	}

	// 发送交易
	err = c.ethClient.SendTransaction(ctx, signedTx)
	if err != nil {
		return nil, err
	}

	return signedTx, nil
}

// ExecuteBatch 批量执行
func (c *EIP7702Client) ExecuteBatch(
	ctx context.Context,
	calls []Call,
) (*types.Transaction, error) {
	nonce, err := c.ethClient.PendingNonceAt(ctx, c.address)
	if err != nil {
		return nil, err
	}

	// 创建授权
	auth, err := SignAuthorization(
		c.privateKey,
		c.chainID.Uint64(),
		c.delegateAddr,
		nonce,
	)
	if err != nil {
		return nil, err
	}

	// 构建批量调用数据
	targets := make([]common.Address, len(calls))
	values := make([]*big.Int, len(calls))
	datas := make([][]byte, len(calls))

	var totalValue = big.NewInt(0)
	for i, call := range calls {
		targets[i] = call.Target
		values[i] = call.Value
		datas[i] = call.Data
		totalValue.Add(totalValue, call.Value)
	}

	callData, err := c.delegateABI.Pack("executeBatch", targets, values, datas)
	if err != nil {
		return nil, err
	}

	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	tx := &SetCodeTx{
		ChainID:           c.chainID,
		Nonce:             nonce + 1,
		GasTipCap:         big.NewInt(1e9),
		GasFeeCap:         new(big.Int).Add(gasPrice, big.NewInt(1e9)),
		Gas:               uint64(100000 + 50000*len(calls)),
		To:                &c.address,
		Value:             totalValue,
		Data:              callData,
		AuthorizationList: []Authorization{*auth},
	}

	signedTx, err := c.signSetCodeTx(tx)
	if err != nil {
		return nil, err
	}

	return signedTx, c.ethClient.SendTransaction(ctx, signedTx)
}

// RevokeAuthorization 撤销授权
func (c *EIP7702Client) RevokeAuthorization(ctx context.Context) (*types.Transaction, error) {
	nonce, err := c.ethClient.PendingNonceAt(ctx, c.address)
	if err != nil {
		return nil, err
	}

	// 授权到零地址等于清除代码
	auth, err := SignAuthorization(
		c.privateKey,
		c.chainID.Uint64(),
		common.Address{}, // 零地址
		nonce,
	)
	if err != nil {
		return nil, err
	}

	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	tx := &SetCodeTx{
		ChainID:           c.chainID,
		Nonce:             nonce + 1,
		GasTipCap:         big.NewInt(1e9),
		GasFeeCap:         new(big.Int).Add(gasPrice, big.NewInt(1e9)),
		Gas:               30000,
		To:                nil, // 可以为空
		Value:             big.NewInt(0),
		Data:              nil,
		AuthorizationList: []Authorization{*auth},
	}

	signedTx, err := c.signSetCodeTx(tx)
	if err != nil {
		return nil, err
	}

	return signedTx, c.ethClient.SendTransaction(ctx, signedTx)
}

func (c *EIP7702Client) signSetCodeTx(tx *SetCodeTx) (*types.Transaction, error) {
	// 实际实现需要使用 types.NewTx 和适当的签名方法
	// 这里简化处理
	return nil, nil
}

type Call struct {
	Target common.Address
	Value  *big.Int
	Data   []byte
}
```

#### 8.5.2 套利示例

```go
package main

import (
	"context"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

// EIP7702Arbitrage 使用 EIP-7702 的套利机器人
type EIP7702Arbitrage struct {
	client        *EIP7702Client
	dexAggregator *DEXAggregator
}

func main() {
	ctx := context.Background()

	// 连接节点
	ethClient, err := ethclient.Dial("https://eth.llamarpc.com")
	if err != nil {
		log.Fatal(err)
	}

	// 加载私钥
	privateKey, err := crypto.HexToECDSA("YOUR_PRIVATE_KEY")
	if err != nil {
		log.Fatal(err)
	}

	// 委托合约地址 (需要预先部署)
	delegateAddr := common.HexToAddress("0x...")

	// 创建 EIP-7702 客户端
	client := NewEIP7702Client(
		ethClient,
		privateKey,
		big.NewInt(1), // mainnet
		delegateAddr,
	)

	// 套利机器人
	bot := &EIP7702Arbitrage{
		client:        client,
		dexAggregator: NewDEXAggregator(ethClient),
	}

	// 执行套利
	if err := bot.ExecuteArbitrage(ctx); err != nil {
		log.Fatal(err)
	}
}

// ExecuteArbitrage 执行套利
func (b *EIP7702Arbitrage) ExecuteArbitrage(ctx context.Context) error {
	// 1. 发现套利机会
	opp, err := b.dexAggregator.FindArbitrageOpportunity(ctx)
	if err != nil || opp == nil {
		return err
	}

	log.Printf("Found arbitrage: %s -> %s, profit: %s",
		opp.Path[0].Hex(), opp.Path[len(opp.Path)-1].Hex(),
		opp.Profit.String())

	// 2. 构建批量调用
	// 在 EIP-7702 之前，每个 swap 需要单独交易
	// 现在可以在一个交易中完成所有操作
	calls := []Call{
		// Approve DEX1
		{
			Target: opp.TokenIn,
			Value:  big.NewInt(0),
			Data:   encodeApprove(opp.DEX1Router, opp.AmountIn),
		},
		// Swap on DEX1
		{
			Target: opp.DEX1Router,
			Value:  big.NewInt(0),
			Data:   encodeSwap(opp.TokenIn, opp.TokenMid, opp.AmountIn),
		},
		// Approve DEX2
		{
			Target: opp.TokenMid,
			Value:  big.NewInt(0),
			Data:   encodeApprove(opp.DEX2Router, opp.ExpectedMid),
		},
		// Swap on DEX2
		{
			Target: opp.DEX2Router,
			Value:  big.NewInt(0),
			Data:   encodeSwap(opp.TokenMid, opp.TokenOut, opp.ExpectedMid),
		},
	}

	// 3. 使用 EIP-7702 批量执行
	tx, err := b.client.ExecuteBatch(ctx, calls)
	if err != nil {
		return err
	}

	log.Printf("Arbitrage tx submitted: %s", tx.Hash().Hex())
	return nil
}

// 传统方式 vs EIP-7702
//
// 传统方式 (4 个交易):
// 1. Approve DEX1   - 46k gas
// 2. Swap DEX1      - 150k gas
// 3. Approve DEX2   - 46k gas
// 4. Swap DEX2      - 150k gas
// 总计: ~392k gas + 4 次交易费
//
// EIP-7702 (1 个交易):
// 1. BatchExecute   - 350k gas + 25k auth
// 总计: ~375k gas, 1 次交易
// 节省: ~17k gas + 时间优势（原子执行）
```

### 8.6 Gas 成本分析

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 Gas 成本                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  授权成本:                                                          │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ 场景                        │ Gas 成本                      │     │
│  ├────────────────────────────┼──────────────────────────────┤     │
│  │ 新 EOA 首次授权             │ 25,000 (PER_EMPTY_ACCOUNT)    │     │
│  │ 已有授权的 EOA 更新          │ 2,500 (PER_AUTH_BASE)         │     │
│  │ 清除授权                    │ 2,500                        │     │
│  └────────────────────────────┴──────────────────────────────┘     │
│                                                                     │
│  与 ERC-4337 对比:                                                  │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ 操作                  │ ERC-4337  │ EIP-7702              │     │
│  ├──────────────────────┼──────────┼───────────────────────┤     │
│  │ 首次使用             │ ~300k    │ ~50k                  │     │
│  │ 单次调用             │ ~42k额外  │ ~25k额外               │     │
│  │ 批量 N 次调用        │ ~42k+N*X │ ~25k+N*X              │     │
│  │ Gas 代付验证         │ ~20k     │ ~5k                   │     │
│  └──────────────────────┴──────────┴───────────────────────┘     │
│                                                                     │
│  实际场景成本预估:                                                   │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ 场景                        │ 预估 Gas                     │     │
│  ├────────────────────────────┼──────────────────────────────┤     │
│  │ 简单转账 + 授权             │ ~46,000                      │     │
│  │ 单次 DEX Swap + 授权        │ ~175,000                     │     │
│  │ 批量 3 次 Swap + 授权       │ ~425,000                     │     │
│  │ 多签执行 (2/3) + 授权       │ ~100,000                     │     │
│  └────────────────────────────┴──────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.7 安全考量

#### 8.7.1 安全风险与缓解

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 安全考量                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  风险 1: 恶意委托合约                                                │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  攻击: 用户被诱导签名授权到恶意合约                           │     │
│  │                                                            │     │
│  │  缓解:                                                      │     │
│  │  • 钱包应显示完整的授权详情                                  │     │
│  │  • 只授权到已审计的合约                                      │     │
│  │  • 使用 chain_id = 0 需谨慎 (跨链重放)                       │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  风险 2: 授权签名重放                                                │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  攻击: 在其他链或账户状态变化后重放授权                        │     │
│  │                                                            │     │
│  │  缓解:                                                      │     │
│  │  • 授权绑定 chain_id                                        │     │
│  │  • 授权绑定 nonce，每次使用后失效                             │     │
│  │  • 0x05 magic 前缀防止与其他签名混淆                         │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  风险 3: 存储冲突                                                    │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  攻击: 委托合约使用的 storage slot 与 EOA 状态冲突             │     │
│  │                                                            │     │
│  │  缓解:                                                      │     │
│  │  • EOA 本身没有 storage                                     │     │
│  │  • 委托合约应使用 ERC-7201 namespaced storage               │     │
│  │  • 或使用 ERC-1967 风格的随机 slot                           │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  风险 4: 权限提升                                                    │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  场景: 委托合约有后门或升级功能                               │     │
│  │                                                            │     │
│  │  缓解:                                                      │     │
│  │  • 只委托到不可升级的合约                                    │     │
│  │  • 检查合约是否有 selfdestruct                              │     │
│  │  • 验证合约所有外部调用都有适当权限检查                        │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 8.7.2 安全检查代码

```go
package security

import (
	"errors"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

var (
	ErrUnverifiedContract  = errors.New("contract not verified")
	ErrUpgradeableContract = errors.New("contract is upgradeable")
	ErrDangerousFunction   = errors.New("contract has dangerous functions")
)

// DelegateSecurityChecker 委托安全检查器
type DelegateSecurityChecker struct {
	ethClient *ethclient.Client

	// 已知安全的委托合约白名单
	trustedDelegates map[common.Address]bool

	// 危险函数签名
	dangerousSelectors [][]byte
}

// NewDelegateSecurityChecker 创建检查器
func NewDelegateSecurityChecker(ethClient *ethclient.Client) *DelegateSecurityChecker {
	return &DelegateSecurityChecker{
		ethClient:        ethClient,
		trustedDelegates: make(map[common.Address]bool),
		dangerousSelectors: [][]byte{
			{0xff, 0x00, 0x00, 0x00}, // selfdestruct 相关
			{0x3c, 0xcf, 0xd6, 0x0b}, // delegatecall
		},
	}
}

// CheckDelegateContract 检查委托合约安全性
func (c *DelegateSecurityChecker) CheckDelegateContract(
	ctx context.Context,
	delegateAddr common.Address,
) error {
	// 1. 检查是否在白名单
	if c.trustedDelegates[delegateAddr] {
		return nil
	}

	// 2. 获取合约代码
	code, err := c.ethClient.CodeAt(ctx, delegateAddr, nil)
	if err != nil {
		return err
	}

	if len(code) == 0 {
		return errors.New("no code at delegate address")
	}

	// 3. 检查危险操作码
	if containsDangerousOpcodes(code) {
		return ErrDangerousFunction
	}

	// 4. 检查是否是代理合约 (可升级)
	if isProxyContract(code) {
		return ErrUpgradeableContract
	}

	return nil
}

// containsDangerousOpcodes 检查危险操作码
func containsDangerousOpcodes(code []byte) bool {
	// SELFDESTRUCT (0xff)
	for i := 0; i < len(code); i++ {
		if code[i] == 0xff {
			// 需要更精确的分析，这里简化
			return true
		}
	}
	return false
}

// isProxyContract 检查是否是代理合约
func isProxyContract(code []byte) bool {
	// 检查 ERC-1967 代理模式
	// 简化检查：查找 delegatecall 模式
	for i := 0; i < len(code)-1; i++ {
		if code[i] == 0xf4 { // DELEGATECALL
			return true
		}
	}
	return false
}

// ValidateAuthorization 验证授权安全性
func (c *DelegateSecurityChecker) ValidateAuthorization(
	auth *Authorization,
	expectedChainID uint64,
) error {
	// 1. 检查 chain_id
	if auth.ChainID != 0 && auth.ChainID != expectedChainID {
		return errors.New("chain id mismatch")
	}

	// 2. 警告 chain_id = 0 (跨链授权)
	if auth.ChainID == 0 {
		// 记录警告，但不阻止
		log.Warn("Authorization with chainId=0 can be replayed on other chains")
	}

	// 3. 检查委托合约
	return c.CheckDelegateContract(context.Background(), auth.Address)
}

// AddTrustedDelegate 添加受信任的委托合约
func (c *DelegateSecurityChecker) AddTrustedDelegate(addr common.Address) {
	c.trustedDelegates[addr] = true
}
```

### 8.8 与 ERC-4337 组合使用

EIP-7702 和 ERC-4337 不是互斥的，可以组合使用：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 + ERC-4337 组合                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  模式 1: EOA 作为 4337 账户                                          │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │   EOA ──(7702授权)──▶ 4337-Compatible Delegate             │     │
│  │         │                     │                            │     │
│  │         │              validateUserOp()                    │     │
│  │         │              execute()                           │     │
│  │         │                     │                            │     │
│  │         └─────────────────────┼────────▶ Bundler           │     │
│  │                               │              │             │     │
│  │                               └──────────────┼──▶ EntryPoint│     │
│  │                                                            │     │
│  │   优势: EOA 无需迁移即可使用 4337 生态                        │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  模式 2: 混合签名验证                                                │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │   小额交易: EIP-7702 直接执行 (低 gas)                       │     │
│  │   大额交易: ERC-4337 + 多签 (高安全性)                       │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
// EIP7702Compatible4337Account 兼容 4337 的委托合约
contract EIP7702Compatible4337Account {
address public immutable entryPoint;

constructor(address _entryPoint) {
entryPoint = _entryPoint;
}

// ERC-4337 接口
function validateUserOp(
UserOperation calldata userOp,
bytes32 userOpHash,
uint256 missingAccountFunds
) external returns (uint256 validationData) {
require(msg.sender == entryPoint, "not from EntryPoint");

// 验证签名 - 使用 EOA 的私钥
// address(this) 就是 EOA 地址
bytes32 hash = keccak256(abi.encodePacked(
"\x19Ethereum Signed Message:\n32",
userOpHash
));

address signer = ECDSA.recover(hash, userOp.signature);
if (signer != address(this)) {
return SIG_VALIDATION_FAILED;
}

// 支付 gas
if (missingAccountFunds > 0) {
(bool success, ) = payable(msg.sender).call{value: missingAccountFunds}("");
require(success);
}

return 0;
}

// 通用执行接口
function execute(
address target,
uint256 value,
bytes calldata data
) external {
require(
msg.sender == entryPoint || msg.sender == address(this),
"not authorized"
);
(bool success, ) = target.call{value: value}(data);
require(success);
}
}
```

### 8.9 总结

#### 8.9.1 EIP-7702 优势

| 优势        | 说明                     |
|-----------|------------------------|
| **无需迁移**  | 现有 EOA 直接获得智能合约能力      |
| **低 Gas** | 比 ERC-4337 节省约 40% gas |
| **协议原生**  | 不需要额外基础设施              |
| **可撤销**   | 随时可以清除委托               |
| **兼容性好**  | 与现有协议完全兼容              |

#### 8.9.2 适用场景

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 最佳使用场景                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✅ 推荐使用:                                                        │
│  • 批量代币操作 (approve + swap 原子执行)                            │
│  • 简单的 Gas 赞助                                                  │
│  • 临时多签需求                                                      │
│  • 套利机器人 (原子多跳交易)                                          │
│  • Session key 实现                                                 │
│                                                                     │
│  ⚠️ 谨慎使用:                                                        │
│  • 长期资产托管                                                      │
│  • 复杂的权限管理                                                    │
│  • 跨链操作 (chain_id = 0)                                          │
│                                                                     │
│  ❌ 不适合:                                                          │
│  • 需要永久智能合约功能                                              │
│  • 复杂的社交恢复逻辑                                                │
│  • 需要合约内部状态的场景                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 8.9.3 未来展望

EIP-7702 是以太坊账户抽象的重要里程碑，预计 Pectra 升级后：

1. **钱包升级**：MetaMask 等主流钱包将支持 EIP-7702
2. **DApp 适配**：DeFi 协议可以提供更好的 UX
3. **与 4337 融合**：两种方案互补，形成完整的 AA 生态
4. **新应用涌现**：基于 EOA 的 session key、临时权限等新模式

```
选择指南:

需要永久智能账户功能？
├─ 是 → ERC-4337 智能合约钱包
└─ 否 →
    需要复杂权限管理？
    ├─ 是 → ERC-4337
    └─ 否 →
        需要最低 Gas 成本？
        ├─ 是 → EIP-7702
        └─ 否 → 根据具体需求选择
```


**为什么需要 EIP-7702？**

| 痛点         | ERC-4337 方案  | EIP-7702 方案  |
|------------|--------------|--------------|
| EOA 无法批量操作 | 需迁移到智能账户     | EOA 直接获得能力   |
| 新用户需要 ETH  | Paymaster 代付 | Paymaster 代付 |
| 资产迁移成本     | 需转移所有资产      | 无需迁移         |
| 协议兼容性      | 需要适配         | 完全兼容         |

### 8.2 EIP-7702 vs ERC-4337 对比

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 vs ERC-4337                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ERC-4337 (应用层)                                                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   User ──▶ UserOp ──▶ Bundler ──▶ EntryPoint ──▶ Account    │   │
│  │                                        │                     │   │
│  │                              validateUserOp()                │   │
│  │                              execute()                       │   │
│  │                                                              │   │
│  │   特点：                                                      │   │
│  │   • 需要智能合约钱包                                           │   │
│  │   • 需要 Bundler 基础设施                                      │   │
│  │   • 完全去中心化                                              │   │
│  │   • Gas 开销较高 (~42k 额外 gas)                              │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  EIP-7702 (协议层)                                                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   User ──▶ SetCodeTx ──▶ Mempool ──▶ 直接执行                 │   │
│  │                │                                             │   │
│  │        authorization_list                                    │   │
│  │        [chain_id, address, nonce, y_parity, r, s]           │   │
│  │                                                              │   │
│  │   特点：                                                      │   │
│  │   • EOA 临时获得智能合约能力                                    │   │
│  │   • 复用现有 mempool                                          │   │
│  │   • 交易完成后代码自动清除                                      │   │
│  │   • Gas 开销低 (~25k per authorization)                       │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**详细对比表**

| 特性         | ERC-4337  | EIP-7702          |
|------------|-----------|-------------------|
| 实现层级       | 应用层       | 协议层               |
| 账户类型       | 智能合约账户    | EOA + 临时代码        |
| 需要迁移       | 是         | 否                 |
| Bundler    | 必需        | 可选                |
| EntryPoint | 必需        | 不需要               |
| Gas 开销     | 较高        | 较低                |
| 代码持久性      | 永久        | 临时/可撤销            |
| 批量操作       | 支持        | 支持                |
| Gas 代付     | Paymaster | Paymaster/Sponsor |
| 兼容性        | 需要适配      | 完全兼容              |
| 上线时间       | 已上线       | Pectra 升级         |

### 8.3 工作原理

#### 8.3.1 核心概念

EIP-7702 引入了新的交易类型 `0x04`（SET_CODE_TX），允许 EOA 通过签名授权，将自己的代码槽临时指向一个智能合约地址。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 执行流程                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 用户签名授权                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   EOA Owner 签名：                                            │   │
│  │   sign(keccak256(MAGIC || chain_id || nonce || address))    │   │
│  │                                                              │   │
│  │   MAGIC = 0x05 (EIP-7702 专用前缀)                            │   │
│  │   address = 委托目标合约地址                                    │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  2. 构建 SetCode 交易                                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   {                                                          │   │
│  │     type: 0x04,                                              │   │
│  │     chain_id: 1,                                             │   │
│  │     nonce: 5,                                                │   │
│  │     to: <EOA_address>,  // 调用自己                           │   │
│  │     data: <execute_calldata>,                                │   │
│  │     authorization_list: [                                    │   │
│  │       { chain_id, address, nonce, y_parity, r, s }           │   │
│  │     ]                                                        │   │
│  │   }                                                          │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  3. 协议层处理                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   对每个 authorization:                                       │   │
│  │   ├─ 验证签名                                                 │   │
│  │   ├─ 验证 nonce                                              │   │
│  │   ├─ 设置 EOA.code = 0xef0100 || address (delegation)        │   │
│  │   └─ 递增 EOA.nonce                                          │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  4. 执行交易                                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   CALL to EOA:                                               │   │
│  │   ├─ 检测 delegation designator (0xef0100)                    │   │
│  │   ├─ 加载目标合约代码                                          │   │
│  │   └─ 在 EOA 上下文中执行 (storage, balance 都是 EOA 的)         │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 8.3.2 Delegation Designator

EIP-7702 使用特殊的 **delegation designator** 来标记委托代码：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Delegation Designator                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   格式: 0xef0100 || address (23 bytes)                              │
│                                                                     │
│   ┌────────┬────────────────────────────────────────┐               │
│   │ ef0100 │     20-byte contract address          │               │
│   │ 3 bytes│              20 bytes                 │               │
│   └────────┴────────────────────────────────────────┘               │
│                                                                     │
│   • 0xef 是 EOF (EVM Object Format) 保留前缀                         │
│   • 0x01 表示 delegation 类型                                        │
│   • 0x00 是版本号                                                    │
│   • 后面的 20 bytes 是委托目标地址                                     │
│                                                                     │
│   当 EVM 检测到这个前缀时：                                            │
│   1. 不执行这个 "code"                                               │
│   2. 而是加载目标地址的代码                                            │
│   3. 在当前账户上下文中执行                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.4 数据结构与实现

#### 8.4.1 Go 语言实现

```go
package eip7702

import (
	"crypto/ecdsa"
	"errors"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/rlp"
)

// Authorization EIP-7702 授权结构
type Authorization struct {
	ChainID uint64         `json:"chainId"`
	Address common.Address `json:"address"` // 委托目标合约
	Nonce   uint64         `json:"nonce"`   // 授权者的 nonce
	YParity uint8          `json:"yParity"` // 签名 v 值 (0 或 1)
	R       *big.Int       `json:"r"`
	S       *big.Int       `json:"s"`
}

// SetCodeTx EIP-7702 交易类型
type SetCodeTx struct {
	ChainID           *big.Int
	Nonce             uint64
	GasTipCap         *big.Int // maxPriorityFeePerGas
	GasFeeCap         *big.Int // maxFeePerGas
	Gas               uint64
	To                *common.Address
	Value             *big.Int
	Data              []byte
	AccessList        AccessList
	AuthorizationList []Authorization
	V, R, S           *big.Int
}

const (
	// SetCodeTxType EIP-7702 交易类型标识
	SetCodeTxType = 0x04

	// DelegationPrefix delegation designator 前缀
	DelegationPrefix = 0xef0100

	// AuthMagic 授权签名 magic 字节
	AuthMagic = 0x05

	// PER_EMPTY_ACCOUNT_COST 设置空账户代码的 gas 成本
	PER_EMPTY_ACCOUNT_COST = 25000

	// PER_AUTH_BASE_COST 每个授权的基础 gas 成本
	PER_AUTH_BASE_COST = 2500
)

// SignAuthorization 签名授权
func SignAuthorization(
	privateKey *ecdsa.PrivateKey,
	chainID uint64,
	delegateAddress common.Address,
	nonce uint64,
) (*Authorization, error) {
	// 构建签名数据
	// MAGIC || rlp([chain_id, address, nonce])
	authData := []interface{}{chainID, delegateAddress, nonce}
	encoded, err := rlp.EncodeToBytes(authData)
	if err != nil {
		return nil, err
	}

	// 添加 magic 前缀
	toSign := append([]byte{AuthMagic}, encoded...)
	hash := crypto.Keccak256(toSign)

	// 签名
	sig, err := crypto.Sign(hash, privateKey)
	if err != nil {
		return nil, err
	}

	return &Authorization{
		ChainID: chainID,
		Address: delegateAddress,
		Nonce:   nonce,
		YParity: sig[64], // v - 27, 已经是 0 或 1
		R:       new(big.Int).SetBytes(sig[:32]),
		S:       new(big.Int).SetBytes(sig[32:64]),
	}, nil
}

// RecoverAuthority 从授权中恢复签名者地址
func (auth *Authorization) RecoverAuthority() (common.Address, error) {
	// 重建签名数据
	authData := []interface{}{auth.ChainID, auth.Address, auth.Nonce}
	encoded, err := rlp.EncodeToBytes(authData)
	if err != nil {
		return common.Address{}, err
	}

	toSign := append([]byte{AuthMagic}, encoded...)
	hash := crypto.Keccak256(toSign)

	// 重建签名
	sig := make([]byte, 65)
	auth.R.FillBytes(sig[:32])
	auth.S.FillBytes(sig[32:64])
	sig[64] = auth.YParity

	// 恢复公钥
	pubKey, err := crypto.SigToPub(hash, sig)
	if err != nil {
		return common.Address{}, err
	}

	return crypto.PubkeyToAddress(*pubKey), nil
}

// DelegationDesignator 生成 delegation designator
func DelegationDesignator(target common.Address) []byte {
	// 0xef0100 || address
	code := make([]byte, 23)
	code[0] = 0xef
	code[1] = 0x01
	code[2] = 0x00
	copy(code[3:], target.Bytes())
	return code
}

// IsDelegation 检查代码是否是 delegation
func IsDelegation(code []byte) bool {
	return len(code) == 23 &&
		code[0] == 0xef &&
		code[1] == 0x01 &&
		code[2] == 0x00
}

// GetDelegationTarget 获取委托目标地址
func GetDelegationTarget(code []byte) (common.Address, bool) {
	if !IsDelegation(code) {
		return common.Address{}, false
	}
	return common.BytesToAddress(code[3:23]), true
}
```

#### 8.4.2 交易处理器实现

```go
package eip7702

import (
	"errors"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/state"
)

var (
	ErrInvalidChainID     = errors.New("invalid chain id")
	ErrInvalidNonce       = errors.New("invalid nonce")
	ErrInvalidSignature   = errors.New("invalid signature")
	ErrAccountHasCode     = errors.New("account already has code")
	ErrAuthorizationEmpty = errors.New("authorization list empty")
)

// AuthorizationProcessor 授权处理器
type AuthorizationProcessor struct {
	chainID *big.Int
	state   *state.StateDB
}

// NewAuthorizationProcessor 创建授权处理器
func NewAuthorizationProcessor(chainID *big.Int, state *state.StateDB) *AuthorizationProcessor {
	return &AuthorizationProcessor{
		chainID: chainID,
		state:   state,
	}
}

// ProcessAuthorizationList 处理授权列表
func (p *AuthorizationProcessor) ProcessAuthorizationList(
	authList []Authorization,
) (uint64, error) {
	if len(authList) == 0 {
		return 0, ErrAuthorizationEmpty
	}

	var totalGas uint64

	for _, auth := range authList {
		gas, err := p.processAuthorization(&auth)
		if err != nil {
			// 单个授权失败不影响其他授权
			// 但不计入 gas 退款
			continue
		}
		totalGas += gas
	}

	return totalGas, nil
}

// processAuthorization 处理单个授权
func (p *AuthorizationProcessor) processAuthorization(auth *Authorization) (uint64, error) {
	// 1. 验证 chain_id
	// chain_id 必须是 0 (任意链) 或当前链 ID
	if auth.ChainID != 0 && auth.ChainID != p.chainID.Uint64() {
		return 0, ErrInvalidChainID
	}

	// 2. 恢复授权者地址
	authority, err := auth.RecoverAuthority()
	if err != nil {
		return 0, ErrInvalidSignature
	}

	// 3. 验证 nonce
	currentNonce := p.state.GetNonce(authority)
	if auth.Nonce != currentNonce {
		return 0, ErrInvalidNonce
	}

	// 4. 计算 gas
	var gas uint64
	if p.state.Empty(authority) {
		gas = PER_EMPTY_ACCOUNT_COST
	} else {
		gas = PER_AUTH_BASE_COST
	}

	// 5. 增加 nonce
	p.state.SetNonce(authority, currentNonce+1)

	// 6. 设置 delegation
	if auth.Address == (common.Address{}) {
		// 清除 delegation
		p.state.SetCode(authority, nil)
	} else {
		// 设置 delegation designator
		designator := DelegationDesignator(auth.Address)
		p.state.SetCode(authority, designator)
	}

	return gas, nil
}

// ExecuteWithDelegation 在委托上下文中执行
func ExecuteWithDelegation(
	state *state.StateDB,
	caller common.Address,
	target common.Address,
	input []byte,
	gas uint64,
	value *big.Int,
) ([]byte, uint64, error) {
	// 获取目标账户代码
	code := state.GetCode(target)

	// 检查是否是 delegation
	if delegateAddr, isDelegation := GetDelegationTarget(code); isDelegation {
		// 加载委托目标的代码
		delegateCode := state.GetCode(delegateAddr)

		// 在 target 的上下文中执行 delegateCode
		// storage 和 balance 操作都作用于 target
		return executeCode(state, target, caller, delegateCode, input, gas, value)
	}

	// 普通执行
	return executeCode(state, target, caller, code, input, gas, value)
}

// executeCode 执行代码 (简化版)
func executeCode(
	state *state.StateDB,
	context common.Address,
	caller common.Address,
	code []byte,
	input []byte,
	gas uint64,
	value *big.Int,
) ([]byte, uint64, error) {
	// 实际实现需要调用 EVM 解释器
	// 这里简化处理
	return nil, gas, nil
}
```

#### 8.4.3 智能账户委托合约示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SimpleDelegate
 * @notice EIP-7702 委托合约示例
 * @dev EOA 可以通过 EIP-7702 委托到此合约，获得批量执行等能力
 */
contract SimpleDelegate {
    // 执行单个调用
    function execute(
        address target,
        uint256 value,
        bytes calldata data
    ) external payable returns (bytes memory) {
        // msg.sender 是交易发送者
        // address(this) 是委托的 EOA 地址
        require(msg.sender == address(this), "only self");

        (bool success, bytes memory result) = target.call{value: value}(data);
        require(success, "call failed");
        return result;
    }

    // 批量执行
    function executeBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata datas
    ) external payable returns (bytes[] memory results) {
        require(msg.sender == address(this), "only self");
        require(
            targets.length == values.length && values.length == datas.length,
            "length mismatch"
        );

        results = new bytes[](targets.length);

        for (uint256 i = 0; i < targets.length; i++) {
            (bool success, bytes memory result) = targets[i].call{value: values[i]}(datas[i]);
            require(success, "call failed");
            results[i] = result;
        }
    }

    // 支持接收 ETH
    receive() external payable {}
}

/**
 * @title SponsoredDelegate
 * @notice 支持 Gas 代付的委托合约
 */
contract SponsoredDelegate {
    mapping(address => bool) public sponsors;
    mapping(address => uint256) public nonces;

    event Sponsored(address indexed sponsor, address indexed user, uint256 gasCost);

    /**
     * @notice 代付执行
     * @param user 用户 EOA 地址 (委托到此合约的)
     * @param target 目标合约
     * @param data 调用数据
     * @param userSig 用户签名
     */
    function sponsoredExecute(
        address user,
        address target,
        bytes calldata data,
        bytes calldata userSig
    ) external {
        require(sponsors[msg.sender], "not sponsor");

        // 验证用户签名
        bytes32 hash = keccak256(abi.encodePacked(
            user,
            target,
            data,
            nonces[user]++
        ));
        bytes32 ethHash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            hash
        ));

        address signer = ecrecover(ethHash,
            uint8(userSig[64]) + 27,
            bytes32(userSig[0 : 32]),
            bytes32(userSig[32 : 64])
        );
        require(signer == user, "invalid signature");

        // 作为用户执行 (需要用户已委托到此合约)
        uint256 gasBefore = gasleft();

        // 这里的 call 会在用户账户上下文中执行
        (bool success,) = target.call(data);
        require(success, "execution failed");

        uint256 gasUsed = gasBefore - gasleft();
        emit Sponsored(msg.sender, user, gasUsed);
    }

    function addSponsor(address sponsor) external {
        // 简化：实际应有权限控制
        sponsors[sponsor] = true;
    }
}

/**
 * @title MultiSigDelegate
 * @notice 多签委托合约
 */
contract MultiSigDelegate {
    uint256 public threshold;
    mapping(address => bool) public signers;
    uint256 public signerCount;

    function initialize(address[] calldata _signers, uint256 _threshold) external {
        require(threshold == 0, "already initialized");
        require(_threshold > 0 && _threshold <= _signers.length, "invalid threshold");

        for (uint256 i = 0; i < _signers.length; i++) {
            signers[_signers[i]] = true;
        }
        signerCount = _signers.length;
        threshold = _threshold;
    }

    function executeMultiSig(
        address target,
        uint256 value,
        bytes calldata data,
        bytes[] calldata signatures
    ) external payable returns (bytes memory) {
        require(signatures.length >= threshold, "not enough signatures");

        bytes32 hash = keccak256(abi.encodePacked(
            address(this), // EOA 地址
            target,
            value,
            data
        ));

        address lastSigner;
        for (uint256 i = 0; i < threshold; i++) {
            address signer = recoverSigner(hash, signatures[i]);
            require(signers[signer], "invalid signer");
            require(signer > lastSigner, "signatures not sorted");
            lastSigner = signer;
        }

        (bool success, bytes memory result) = target.call{value: value}(data);
        require(success, "call failed");
        return result;
    }

    function recoverSigner(bytes32 hash, bytes calldata sig) internal pure returns (address) {
        bytes32 ethHash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            hash
        ));
        return ecrecover(ethHash,
            uint8(sig[64]) + 27,
            bytes32(sig[0 : 32]),
            bytes32(sig[32 : 64])
        );
    }
}
```

### 8.5 完整使用示例

#### 8.5.1 Go 客户端实现

```go
package main

import (
	"context"
	"crypto/ecdsa"
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

// EIP7702Client EIP-7702 客户端
type EIP7702Client struct {
	ethClient    *ethclient.Client
	privateKey   *ecdsa.PrivateKey
	address      common.Address
	chainID      *big.Int
	delegateAddr common.Address // 委托目标合约
	delegateABI  abi.ABI
}

// NewEIP7702Client 创建客户端
func NewEIP7702Client(
	ethClient *ethclient.Client,
	privateKey *ecdsa.PrivateKey,
	chainID *big.Int,
	delegateAddr common.Address,
) *EIP7702Client {
	address := crypto.PubkeyToAddress(privateKey.PublicKey)

	// SimpleDelegate ABI
	delegateABI, _ := abi.JSON(strings.NewReader(`[
		{
			"name": "execute",
			"type": "function",
			"inputs": [
				{"name": "target", "type": "address"},
				{"name": "value", "type": "uint256"},
				{"name": "data", "type": "bytes"}
			],
			"outputs": [{"name": "", "type": "bytes"}]
		},
		{
			"name": "executeBatch",
			"type": "function",
			"inputs": [
				{"name": "targets", "type": "address[]"},
				{"name": "values", "type": "uint256[]"},
				{"name": "datas", "type": "bytes[]"}
			],
			"outputs": [{"name": "", "type": "bytes[]"}]
		}
	]`))

	return &EIP7702Client{
		ethClient:    ethClient,
		privateKey:   privateKey,
		address:      address,
		chainID:      chainID,
		delegateAddr: delegateAddr,
		delegateABI:  delegateABI,
	}
}

// Execute 执行单个调用
func (c *EIP7702Client) Execute(
	ctx context.Context,
	target common.Address,
	value *big.Int,
	data []byte,
) (*types.Transaction, error) {
	// 获取当前 nonce
	nonce, err := c.ethClient.PendingNonceAt(ctx, c.address)
	if err != nil {
		return nil, err
	}

	// 创建授权
	auth, err := SignAuthorization(
		c.privateKey,
		c.chainID.Uint64(),
		c.delegateAddr,
		nonce,
	)
	if err != nil {
		return nil, err
	}

	// 构建调用数据
	callData, err := c.delegateABI.Pack("execute", target, value, data)
	if err != nil {
		return nil, err
	}

	// 获取 gas 价格
	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	// 构建 SetCode 交易
	tx := &SetCodeTx{
		ChainID:           c.chainID,
		Nonce:             nonce + 1, // 授权会消耗一个 nonce
		GasTipCap:         big.NewInt(1e9),
		GasFeeCap:         new(big.Int).Add(gasPrice, big.NewInt(1e9)),
		Gas:               200000,
		To:                &c.address, // 调用自己
		Value:             value,
		Data:              callData,
		AuthorizationList: []Authorization{*auth},
	}

	// 签名交易
	signedTx, err := c.signSetCodeTx(tx)
	if err != nil {
		return nil, err
	}

	// 发送交易
	err = c.ethClient.SendTransaction(ctx, signedTx)
	if err != nil {
		return nil, err
	}

	return signedTx, nil
}

// ExecuteBatch 批量执行
func (c *EIP7702Client) ExecuteBatch(
	ctx context.Context,
	calls []Call,
) (*types.Transaction, error) {
	nonce, err := c.ethClient.PendingNonceAt(ctx, c.address)
	if err != nil {
		return nil, err
	}

	// 创建授权
	auth, err := SignAuthorization(
		c.privateKey,
		c.chainID.Uint64(),
		c.delegateAddr,
		nonce,
	)
	if err != nil {
		return nil, err
	}

	// 构建批量调用数据
	targets := make([]common.Address, len(calls))
	values := make([]*big.Int, len(calls))
	datas := make([][]byte, len(calls))

	var totalValue = big.NewInt(0)
	for i, call := range calls {
		targets[i] = call.Target
		values[i] = call.Value
		datas[i] = call.Data
		totalValue.Add(totalValue, call.Value)
	}

	callData, err := c.delegateABI.Pack("executeBatch", targets, values, datas)
	if err != nil {
		return nil, err
	}

	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	tx := &SetCodeTx{
		ChainID:           c.chainID,
		Nonce:             nonce + 1,
		GasTipCap:         big.NewInt(1e9),
		GasFeeCap:         new(big.Int).Add(gasPrice, big.NewInt(1e9)),
		Gas:               uint64(100000 + 50000*len(calls)),
		To:                &c.address,
		Value:             totalValue,
		Data:              callData,
		AuthorizationList: []Authorization{*auth},
	}

	signedTx, err := c.signSetCodeTx(tx)
	if err != nil {
		return nil, err
	}

	return signedTx, c.ethClient.SendTransaction(ctx, signedTx)
}

// RevokeAuthorization 撤销授权
func (c *EIP7702Client) RevokeAuthorization(ctx context.Context) (*types.Transaction, error) {
	nonce, err := c.ethClient.PendingNonceAt(ctx, c.address)
	if err != nil {
		return nil, err
	}

	// 授权到零地址等于清除代码
	auth, err := SignAuthorization(
		c.privateKey,
		c.chainID.Uint64(),
		common.Address{}, // 零地址
		nonce,
	)
	if err != nil {
		return nil, err
	}

	gasPrice, err := c.ethClient.SuggestGasPrice(ctx)
	if err != nil {
		return nil, err
	}

	tx := &SetCodeTx{
		ChainID:           c.chainID,
		Nonce:             nonce + 1,
		GasTipCap:         big.NewInt(1e9),
		GasFeeCap:         new(big.Int).Add(gasPrice, big.NewInt(1e9)),
		Gas:               30000,
		To:                nil, // 可以为空
		Value:             big.NewInt(0),
		Data:              nil,
		AuthorizationList: []Authorization{*auth},
	}

	signedTx, err := c.signSetCodeTx(tx)
	if err != nil {
		return nil, err
	}

	return signedTx, c.ethClient.SendTransaction(ctx, signedTx)
}

func (c *EIP7702Client) signSetCodeTx(tx *SetCodeTx) (*types.Transaction, error) {
	// 实际实现需要使用 types.NewTx 和适当的签名方法
	// 这里简化处理
	return nil, nil
}

type Call struct {
	Target common.Address
	Value  *big.Int
	Data   []byte
}
```

#### 8.5.2 套利示例

```go
package main

import (
	"context"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

// EIP7702Arbitrage 使用 EIP-7702 的套利机器人
type EIP7702Arbitrage struct {
	client        *EIP7702Client
	dexAggregator *DEXAggregator
}

func main() {
	ctx := context.Background()

	// 连接节点
	ethClient, err := ethclient.Dial("https://eth.llamarpc.com")
	if err != nil {
		log.Fatal(err)
	}

	// 加载私钥
	privateKey, err := crypto.HexToECDSA("YOUR_PRIVATE_KEY")
	if err != nil {
		log.Fatal(err)
	}

	// 委托合约地址 (需要预先部署)
	delegateAddr := common.HexToAddress("0x...")

	// 创建 EIP-7702 客户端
	client := NewEIP7702Client(
		ethClient,
		privateKey,
		big.NewInt(1), // mainnet
		delegateAddr,
	)

	// 套利机器人
	bot := &EIP7702Arbitrage{
		client:        client,
		dexAggregator: NewDEXAggregator(ethClient),
	}

	// 执行套利
	if err := bot.ExecuteArbitrage(ctx); err != nil {
		log.Fatal(err)
	}
}

// ExecuteArbitrage 执行套利
func (b *EIP7702Arbitrage) ExecuteArbitrage(ctx context.Context) error {
	// 1. 发现套利机会
	opp, err := b.dexAggregator.FindArbitrageOpportunity(ctx)
	if err != nil || opp == nil {
		return err
	}

	log.Printf("Found arbitrage: %s -> %s, profit: %s",
		opp.Path[0].Hex(), opp.Path[len(opp.Path)-1].Hex(),
		opp.Profit.String())

	// 2. 构建批量调用
	// 在 EIP-7702 之前，每个 swap 需要单独交易
	// 现在可以在一个交易中完成所有操作
	calls := []Call{
		// Approve DEX1
		{
			Target: opp.TokenIn,
			Value:  big.NewInt(0),
			Data:   encodeApprove(opp.DEX1Router, opp.AmountIn),
		},
		// Swap on DEX1
		{
			Target: opp.DEX1Router,
			Value:  big.NewInt(0),
			Data:   encodeSwap(opp.TokenIn, opp.TokenMid, opp.AmountIn),
		},
		// Approve DEX2
		{
			Target: opp.TokenMid,
			Value:  big.NewInt(0),
			Data:   encodeApprove(opp.DEX2Router, opp.ExpectedMid),
		},
		// Swap on DEX2
		{
			Target: opp.DEX2Router,
			Value:  big.NewInt(0),
			Data:   encodeSwap(opp.TokenMid, opp.TokenOut, opp.ExpectedMid),
		},
	}

	// 3. 使用 EIP-7702 批量执行
	tx, err := b.client.ExecuteBatch(ctx, calls)
	if err != nil {
		return err
	}

	log.Printf("Arbitrage tx submitted: %s", tx.Hash().Hex())
	return nil
}

// 传统方式 vs EIP-7702
//
// 传统方式 (4 个交易):
// 1. Approve DEX1   - 46k gas
// 2. Swap DEX1      - 150k gas
// 3. Approve DEX2   - 46k gas
// 4. Swap DEX2      - 150k gas
// 总计: ~392k gas + 4 次交易费
//
// EIP-7702 (1 个交易):
// 1. BatchExecute   - 350k gas + 25k auth
// 总计: ~375k gas, 1 次交易
// 节省: ~17k gas + 时间优势（原子执行）
```

### 8.6 Gas 成本分析

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 Gas 成本                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  授权成本:                                                          │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ 场景                        │ Gas 成本                      │     │
│  ├────────────────────────────┼──────────────────────────────┤     │
│  │ 新 EOA 首次授权             │ 25,000 (PER_EMPTY_ACCOUNT)    │     │
│  │ 已有授权的 EOA 更新          │ 2,500 (PER_AUTH_BASE)         │     │
│  │ 清除授权                    │ 2,500                        │     │
│  └────────────────────────────┴──────────────────────────────┘     │
│                                                                     │
│  与 ERC-4337 对比:                                                  │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ 操作                  │ ERC-4337  │ EIP-7702              │     │
│  ├──────────────────────┼──────────┼───────────────────────┤     │
│  │ 首次使用             │ ~300k    │ ~50k                  │     │
│  │ 单次调用             │ ~42k额外  │ ~25k额外               │     │
│  │ 批量 N 次调用        │ ~42k+N*X │ ~25k+N*X              │     │
│  │ Gas 代付验证         │ ~20k     │ ~5k                   │     │
│  └──────────────────────┴──────────┴───────────────────────┘     │
│                                                                     │
│  实际场景成本预估:                                                   │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ 场景                        │ 预估 Gas                     │     │
│  ├────────────────────────────┼──────────────────────────────┤     │
│  │ 简单转账 + 授权             │ ~46,000                      │     │
│  │ 单次 DEX Swap + 授权        │ ~175,000                     │     │
│  │ 批量 3 次 Swap + 授权       │ ~425,000                     │     │
│  │ 多签执行 (2/3) + 授权       │ ~100,000                     │     │
│  └────────────────────────────┴──────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.7 安全考量

#### 8.7.1 安全风险与缓解

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 安全考量                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  风险 1: 恶意委托合约                                                │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  攻击: 用户被诱导签名授权到恶意合约                           │     │
│  │                                                            │     │
│  │  缓解:                                                      │     │
│  │  • 钱包应显示完整的授权详情                                  │     │
│  │  • 只授权到已审计的合约                                      │     │
│  │  • 使用 chain_id = 0 需谨慎 (跨链重放)                       │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  风险 2: 授权签名重放                                                │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  攻击: 在其他链或账户状态变化后重放授权                        │     │
│  │                                                            │     │
│  │  缓解:                                                      │     │
│  │  • 授权绑定 chain_id                                        │     │
│  │  • 授权绑定 nonce，每次使用后失效                             │     │
│  │  • 0x05 magic 前缀防止与其他签名混淆                         │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  风险 3: 存储冲突                                                    │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  攻击: 委托合约使用的 storage slot 与 EOA 状态冲突             │     │
│  │                                                            │     │
│  │  缓解:                                                      │     │
│  │  • EOA 本身没有 storage                                     │     │
│  │  • 委托合约应使用 ERC-7201 namespaced storage               │     │
│  │  • 或使用 ERC-1967 风格的随机 slot                           │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  风险 4: 权限提升                                                    │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  场景: 委托合约有后门或升级功能                               │     │
│  │                                                            │     │
│  │  缓解:                                                      │     │
│  │  • 只委托到不可升级的合约                                    │     │
│  │  • 检查合约是否有 selfdestruct                              │     │
│  │  • 验证合约所有外部调用都有适当权限检查                        │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 8.7.2 安全检查代码

```go
package security

import (
	"errors"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

var (
	ErrUnverifiedContract  = errors.New("contract not verified")
	ErrUpgradeableContract = errors.New("contract is upgradeable")
	ErrDangerousFunction   = errors.New("contract has dangerous functions")
)

// DelegateSecurityChecker 委托安全检查器
type DelegateSecurityChecker struct {
	ethClient *ethclient.Client

	// 已知安全的委托合约白名单
	trustedDelegates map[common.Address]bool

	// 危险函数签名
	dangerousSelectors [][]byte
}

// NewDelegateSecurityChecker 创建检查器
func NewDelegateSecurityChecker(ethClient *ethclient.Client) *DelegateSecurityChecker {
	return &DelegateSecurityChecker{
		ethClient:        ethClient,
		trustedDelegates: make(map[common.Address]bool),
		dangerousSelectors: [][]byte{
			{0xff, 0x00, 0x00, 0x00}, // selfdestruct 相关
			{0x3c, 0xcf, 0xd6, 0x0b}, // delegatecall
		},
	}
}

// CheckDelegateContract 检查委托合约安全性
func (c *DelegateSecurityChecker) CheckDelegateContract(
	ctx context.Context,
	delegateAddr common.Address,
) error {
	// 1. 检查是否在白名单
	if c.trustedDelegates[delegateAddr] {
		return nil
	}

	// 2. 获取合约代码
	code, err := c.ethClient.CodeAt(ctx, delegateAddr, nil)
	if err != nil {
		return err
	}

	if len(code) == 0 {
		return errors.New("no code at delegate address")
	}

	// 3. 检查危险操作码
	if containsDangerousOpcodes(code) {
		return ErrDangerousFunction
	}

	// 4. 检查是否是代理合约 (可升级)
	if isProxyContract(code) {
		return ErrUpgradeableContract
	}

	return nil
}

// containsDangerousOpcodes 检查危险操作码
func containsDangerousOpcodes(code []byte) bool {
	// SELFDESTRUCT (0xff)
	for i := 0; i < len(code); i++ {
		if code[i] == 0xff {
			// 需要更精确的分析，这里简化
			return true
		}
	}
	return false
}

// isProxyContract 检查是否是代理合约
func isProxyContract(code []byte) bool {
	// 检查 ERC-1967 代理模式
	// 简化检查：查找 delegatecall 模式
	for i := 0; i < len(code)-1; i++ {
		if code[i] == 0xf4 { // DELEGATECALL
			return true
		}
	}
	return false
}

// ValidateAuthorization 验证授权安全性
func (c *DelegateSecurityChecker) ValidateAuthorization(
	auth *Authorization,
	expectedChainID uint64,
) error {
	// 1. 检查 chain_id
	if auth.ChainID != 0 && auth.ChainID != expectedChainID {
		return errors.New("chain id mismatch")
	}

	// 2. 警告 chain_id = 0 (跨链授权)
	if auth.ChainID == 0 {
		// 记录警告，但不阻止
		log.Warn("Authorization with chainId=0 can be replayed on other chains")
	}

	// 3. 检查委托合约
	return c.CheckDelegateContract(context.Background(), auth.Address)
}

// AddTrustedDelegate 添加受信任的委托合约
func (c *DelegateSecurityChecker) AddTrustedDelegate(addr common.Address) {
	c.trustedDelegates[addr] = true
}
```

### 8.8 与 ERC-4337 组合使用

EIP-7702 和 ERC-4337 不是互斥的，可以组合使用：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 + ERC-4337 组合                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  模式 1: EOA 作为 4337 账户                                          │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │   EOA ──(7702授权)──▶ 4337-Compatible Delegate             │     │
│  │         │                     │                            │     │
│  │         │              validateUserOp()                    │     │
│  │         │              execute()                           │     │
│  │         │                     │                            │     │
│  │         └─────────────────────┼────────▶ Bundler           │     │
│  │                               │              │             │     │
│  │                               └──────────────┼──▶ EntryPoint│     │
│  │                                                            │     │
│  │   优势: EOA 无需迁移即可使用 4337 生态                        │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  模式 2: 混合签名验证                                                │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │   小额交易: EIP-7702 直接执行 (低 gas)                       │     │
│  │   大额交易: ERC-4337 + 多签 (高安全性)                       │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
// EIP7702Compatible4337Account 兼容 4337 的委托合约
contract EIP7702Compatible4337Account {
address public immutable entryPoint;

constructor(address _entryPoint) {
entryPoint = _entryPoint;
}

// ERC-4337 接口
function validateUserOp(
UserOperation calldata userOp,
bytes32 userOpHash,
uint256 missingAccountFunds
) external returns (uint256 validationData) {
require(msg.sender == entryPoint, "not from EntryPoint");

// 验证签名 - 使用 EOA 的私钥
// address(this) 就是 EOA 地址
bytes32 hash = keccak256(abi.encodePacked(
"\x19Ethereum Signed Message:\n32",
userOpHash
));

address signer = ECDSA.recover(hash, userOp.signature);
if (signer != address(this)) {
return SIG_VALIDATION_FAILED;
}

// 支付 gas
if (missingAccountFunds > 0) {
(bool success, ) = payable(msg.sender).call{value: missingAccountFunds}("");
require(success);
}

return 0;
}

// 通用执行接口
function execute(
address target,
uint256 value,
bytes calldata data
) external {
require(
msg.sender == entryPoint || msg.sender == address(this),
"not authorized"
);
(bool success, ) = target.call{value: value}(data);
require(success);
}
}
```

### 8.9 总结

#### 8.9.1 EIP-7702 优势

| 优势        | 说明                     |
|-----------|------------------------|
| **无需迁移**  | 现有 EOA 直接获得智能合约能力      |
| **低 Gas** | 比 ERC-4337 节省约 40% gas |
| **协议原生**  | 不需要额外基础设施              |
| **可撤销**   | 随时可以清除委托               |
| **兼容性好**  | 与现有协议完全兼容              |

#### 8.9.2 适用场景

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIP-7702 最佳使用场景                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✅ 推荐使用:                                                        │
│  • 批量代币操作 (approve + swap 原子执行)                            │
│  • 简单的 Gas 赞助                                                  │
│  • 临时多签需求                                                      │
│  • 套利机器人 (原子多跳交易)                                          │
│  • Session key 实现                                                 │
│                                                                     │
│  ⚠️ 谨慎使用:                                                        │
│  • 长期资产托管                                                      │
│  • 复杂的权限管理                                                    │
│  • 跨链操作 (chain_id = 0)                                          │
│                                                                     │
│  ❌ 不适合:                                                          │
│  • 需要永久智能合约功能                                              │
│  • 复杂的社交恢复逻辑                                                │
│  • 需要合约内部状态的场景                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 8.9.3 未来展望

EIP-7702 是以太坊账户抽象的重要里程碑，预计 Pectra 升级后：

1. **钱包升级**：MetaMask 等主流钱包将支持 EIP-7702
2. **DApp 适配**：DeFi 协议可以提供更好的 UX
3. **与 4337 融合**：两种方案互补，形成完整的 AA 生态
4. **新应用涌现**：基于 EOA 的 session key、临时权限等新模式

```
选择指南:

需要永久智能账户功能？
├─ 是 → ERC-4337 智能合约钱包
└─ 否 →
    需要复杂权限管理？
    ├─ 是 → ERC-4337
    └─ 否 →
        需要最低 Gas 成本？
        ├─ 是 → EIP-7702
        └─ 否 → 根据具体需求选择
```
