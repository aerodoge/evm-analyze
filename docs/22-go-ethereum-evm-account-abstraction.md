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
