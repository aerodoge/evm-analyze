# Go-Ethereum EVM 详解（五）：交易池与交易排序

## 目录

- [34. 交易池基础概念](#34-交易池基础概念)
- [35. TxPool 架构设计](#35-txpool-架构设计)
- [36. 交易验证机制](#36-交易验证机制)
- [37. 交易排序算法](#37-交易排序算法)
- [38. Nonce 管理](#38-nonce-管理)
- [39. Gas 价格机制](#39-gas-价格机制)
- [40. 交易替换与取消](#40-交易替换与取消)
- [41. 实战：监控与分析 TxPool](#41-实战监控与分析-txpool)

---

## 34. 交易池基础概念

### 34.1 什么是交易池（TxPool/Mempool）？

交易池是节点用来存储待打包交易的内存数据结构。

```
┌─────────────────────────────────────────────────────────────────┐
│                      交易生命周期                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户                                                           │
│    │                                                            │
│    │ 1. 签名交易                                                │
│    ▼                                                            │
│  ┌─────────────┐                                                │
│  │   节点 RPC   │  eth_sendRawTransaction                       │
│  └──────┬──────┘                                                │
│         │                                                       │
│         │ 2. 验证交易                                           │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │   TxPool    │  交易池（Mempool）                             │
│  │  ┌───────┐  │                                                │
│  │  │Pending│  │  可立即执行的交易                              │
│  │  └───────┘  │                                                │
│  │  ┌───────┐  │                                                │
│  │  │Queued │  │  等待前序交易的交易                            │
│  │  └───────┘  │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         │ 3. 广播到网络                                         │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │  其他节点    │  P2P 网络传播                                  │
│  └──────┬──────┘                                                │
│         │                                                       │
│         │ 4. 矿工/验证者打包                                    │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │    区块     │  交易被包含在区块中                            │
│  └─────────────┘                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 34.2 TxPool 的核心职责

```
┌─────────────────────────────────────────────────────────────────┐
│                    TxPool 核心职责                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 交易验证                                                     │
│     ├── 签名验证                                                 │
│     ├── Nonce 检查                                              │
│     ├── 余额检查                                                 │
│     ├── Gas 限制检查                                             │
│     └── 内在 Gas 检查                                            │
│                                                                 │
│  2. 交易存储                                                     │
│     ├── Pending 队列：可立即执行                                  │
│     └── Queued 队列：等待 nonce 连续                              │
│                                                                 │
│  3. 交易排序                                                     │
│     ├── 按 Gas 价格排序                                          │
│     ├── 按 nonce 排序（同一发送者）                                │
│     └── EIP-1559: 按 effective tip 排序                          │
│                                                                 │
│  4. 交易广播                                                     │
│     ├── 向对等节点广播新交易                                       │
│     └── 响应交易请求                                              │
│                                                                 │
│  5. 状态同步                                                     │
│     ├── 新区块到来时更新                                          │
│     ├── 移除已打包交易                                            │
│     └── 重组时恢复交易                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 34.3 交易池的重要性（对于 MEV）

```
┌─────────────────────────────────────────────────────────────────┐
│                TxPool 对 MEV 的重要性                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  为什么 MEV Searcher 关心 TxPool？                                │
│                                                                 │
│  1. 发现机会                                                    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  监控 pending 交易 ──▶ 发现大额 DEX 交易               │    │
│     │                    ──▶ 发现清算机会                   │    │
│     │                    ──▶ 发现套利机会                   │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  2. 抢跑（Front-running）                                       │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  看到目标交易 ──▶ 构建抢跑交易 ──▶ 提交更高 Gas    │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  3. 尾随（Back-running）                                        │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  看到价格变动交易 ──▶ 构建套利交易 ──▶ 紧随其后    │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  4. 三明治攻击                                                  │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  前置交易 + 目标交易 + 后置交易                      │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 35. TxPool 架构设计

### 35.1 go-ethereum TxPool 结构

```go
// core/txpool/legacypool/legacypool.go

// LegacyPool 传统交易池（EIP-1559 之前的设计）
type LegacyPool struct {
config      Config
chainconfig *params.ChainConfig
chain       BlockChain
gasTip      atomic.Pointer[uint256.Int] // 最低小费
txFeed      event.Feed                  // 交易事件订阅
signer      types.Signer

mu sync.RWMutex

currentHead   atomic.Pointer[types.Header] // 当前区块头
currentState  *state.StateDB // 当前状态
pendingNonces *noncer        // pending nonce 追踪

locals  *accountSet  // 本地账户（优先处理）
journal *journal     // 本地交易日志

reserve   int // 保留槽位
pending   map[common.Address]*list // 可执行交易
queue     map[common.Address]*list // 等待中的交易
beats     map[common.Address]time.Time // 最后心跳时间
all       *lookup     // 所有交易索引
priced    *pricedList // 按价格排序的列表

reqResetCh      chan *txpoolResetRequest
reqPromoteCh    chan *accountSet
queueTxEventCh  chan *types.Transaction
reorgDoneCh     chan chan struct{}
reorgShutdownCh chan struct{}
wg              sync.WaitGroup
initDoneCh      chan struct{}
}

// Config 交易池配置
type Config struct {
Locals    []common.Address // 本地账户地址
NoLocals  bool             // 是否禁用本地交易特权
Journal   string        // 日志文件路径
Rejournal time.Duration // 日志刷新间隔

PriceLimit uint64  // 最低 Gas 价格
PriceBump  uint64  // 替换交易的最低价格涨幅（%）

AccountSlots uint64 // 每个账户的 pending 槽位
GlobalSlots  uint64 // 全局 pending 槽位
AccountQueue uint64 // 每个账户的 queue 槽位
GlobalQueue  uint64 // 全局 queue 槽位

Lifetime time.Duration  // 非本地交易的最大存活时间
}

// 默认配置
var DefaultConfig = Config{
Journal:   "transactions.rlp",
Rejournal: time.Hour,

PriceLimit: 1,  // 1 wei
PriceBump:  10, // 10%

AccountSlots: 16,          // 每账户 16 个 pending 交易
GlobalSlots:  4096 + 1024, // 5120 个全局 pending
AccountQueue: 64,   // 每账户 64 个 queued 交易
GlobalQueue:  1024, // 1024 个全局 queued

Lifetime: 3 * time.Hour, // 3 小时过期
}
```

### 35.2 Pending vs Queue

```
┌─────────────────────────────────────────────────────────────────┐
│                   Pending vs Queue                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  账户 0x123... 当前链上 nonce = 5                               │
│                                                                 │
│  Pending（可执行）：                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  nonce=5 ──▶ nonce=6 ──▶ nonce=7                       │   │
│  │    │           │           │                            │   │
│  │   可以        需要5先      需要6先                       │   │
│  │  立即执行     执行完成     执行完成                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Queue（等待中）：                                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  nonce=10    nonce=12    nonce=15                       │   │
│  │    │           │           │                            │   │
│  │  等待8,9     等待8-11    等待8-14                       │   │
│  │  先到达      先到达      先到达                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  当 nonce=8 到达后：                                            │
│  • nonce=8 进入 pending                                        │
│  • 如果 nonce=9 已在 queue，也移入 pending                      │
│  • nonce=10 从 queue 移入 pending                               │
│  • 继续检查直到 nonce 不连续                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 35.3 交易索引结构

```go
// lookup 交易查找索引
type lookup struct {
slots   int // 当前槽位数
lock    sync.RWMutex
locals  map[common.Hash]*types.Transaction // 本地交易
remotes map[common.Hash]*types.Transaction // 远程交易
}

// list 账户的交易列表
type list struct {
strict bool       // nonce 是否严格连续
txs    *sortedMap // 按 nonce 排序的交易
costcap   *uint256.Int // 最高 cost 上限
gascap    uint64       // 最高 gas 上限
totalcost *uint256.Int // 总 cost
}

// sortedMap nonce -> 交易的有序映射
type sortedMap struct {
items map[uint64]*types.Transaction // nonce -> tx
index *nonceHeap         // nonce 堆（用于排序）
cache types.Transactions // 缓存的有序列表
}

// pricedList 按价格排序的交易列表
type pricedList struct {
all              *lookup
urgent, floating priceHeap // 紧急和浮动堆
stales           int64 // 过期计数
}
```

### 35.4 新架构：BlobPool（Cancun 升级后）

```go
// core/txpool/blobpool/blobpool.go
// EIP-4844 引入的 Blob 交易池

type BlobPool struct {
config   Config
reserve  uint64
gasTip   atomic.Pointer[uint256.Int]
state    *state.StateDB
head     *types.Header

index  map[common.Address][]*blobTxMeta // 账户索引
spent  map[common.Address]*uint256.Int // 账户已花费
evict  *evictHeap                      // 驱逐堆

lookup map[common.Hash]uint64 // hash -> 存储 ID
stored uint64                 // 已存储字节数
}

// Blob 交易的特点
// 1. 包含大量数据（用于 L2 数据可用性）
// 2. 有单独的 Gas 市场（blob gas）
// 3. 存储在磁盘而非内存
```

---

## 36. 交易验证机制

### 36.1 验证流程

```go
// validateTx 验证单个交易
func (pool *LegacyPool) validateTx(tx *types.Transaction, local bool) error {
// 1. 基础检查
if tx.Size() > txMaxSize {
return ErrOversizedData
}

// 2. 类型检查
if tx.Type() == types.BlobTxType {
return core.ErrTxTypeNotSupported
}

// 3. 签名验证
from, err := types.Sender(pool.signer, tx)
if err != nil {
return ErrInvalidSender
}

// 4. Gas 价格检查（EIP-1559）
if tx.GasTipCapIntCmp(pool.gasTip.Load()) < 0 {
return ErrUnderpriced
}

// 5. Gas 限制检查
if tx.Gas() > pool.currentHead.Load().GasLimit {
return ErrGasLimit
}

// 6. 余额检查
balance := pool.currentState.GetBalance(from)
cost := tx.Cost() // value + gas * gasPrice
if balance.Cmp(cost) < 0 {
return ErrInsufficientFunds
}

// 7. Nonce 检查
nonce := pool.currentState.GetNonce(from)
if tx.Nonce() < nonce {
return ErrNonceTooLow
}

// 8. 内在 Gas 检查
intrinsicGas, err := core.IntrinsicGas(tx.Data(), tx.AccessList(),
tx.To() == nil, true, true, false)
if err != nil {
return err
}
if tx.Gas() < intrinsicGas {
return ErrIntrinsicGas
}

return nil
}
```

### 36.2 验证检查详解

```
┌─────────────────────────────────────────────────────────────────┐
│                     交易验证检查                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 大小检查                                                    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  tx.Size() <= 128KB (txMaxSize)                     │    │
│     │  防止 DoS 攻击                                       │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  2. 签名验证                                                    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  • 验证 v, r, s 值有效                              │    │
│     │  • 恢复发送者地址                                   │    │
│     │  • 检查链 ID（EIP-155 重放保护）                    │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  3. Gas 价格                                                    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  Legacy: gasPrice >= pool.gasTip                    │    │
│     │  EIP-1559: maxPriorityFee >= pool.gasTip            │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  4. 余额检查                                                    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  balance >= value + gas * gasPrice                  │    │
│     │  对于 EIP-1559: balance >= value + gas * maxFeePerGas│    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  5. Nonce 检查                                                  │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  tx.nonce >= state.nonce (不能用已使用的 nonce)     │    │
│     │  tx.nonce <= state.nonce + AccountSlots + AccountQueue│  │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  6. 内在 Gas                                                    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  tx.gas >= intrinsicGas                             │    │
│     │  intrinsicGas = 21000 (基础)                        │    │
│     │              + 16/4 * len(data) (数据)              │    │
│     │              + accessListCost (访问列表)            │    │
│     │              + 32000 (如果是合约创建)               │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 36.3 验证代码实现

```go
// 内在 Gas 计算
func IntrinsicGas(data []byte, accessList types.AccessList,
isContractCreation, isHomestead, isEIP2028, isEIP3860 bool) (uint64, error) {

var gas uint64

// 基础 Gas
if isContractCreation && isHomestead {
gas = params.TxGasContractCreation // 53000
} else {
gas = params.TxGas // 21000
}

// 数据 Gas
if len(data) > 0 {
var nz uint64
for _, byt := range data {
if byt != 0 {
nz++
}
}
// 非零字节成本
nonZeroGas := params.TxDataNonZeroGasFrontier // 68
if isEIP2028 {
nonZeroGas = params.TxDataNonZeroGasEIP2028 // 16
}

gas += nz * nonZeroGas

// 零字节成本
z := uint64(len(data)) - nz
gas += z * params.TxDataZeroGas // 4
}

// 访问列表 Gas (EIP-2930)
if accessList != nil {
gas += uint64(len(accessList)) * params.TxAccessListAddressGas // 2400
for _, tuple := range accessList {
gas += uint64(len(tuple.StorageKeys)) * params.TxAccessListStorageKeyGas // 1900
}
}

// 初始化代码 Gas (EIP-3860)
if isContractCreation && isEIP3860 {
lenWords := toWordSize(uint64(len(data)))
gas += lenWords * params.InitCodeWordGas  // 2
}

return gas, nil
}
```

---

## 37. 交易排序算法

### 37.1 排序原则

```
┌─────────────────────────────────────────────────────────────────┐
│                    交易排序原则                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  目标：最大化区块价值（Gas 收入）                               │
│                                                                 │
│  排序规则：                                                     │
│                                                                 │
│  1. 同一账户内：按 nonce 升序（必须连续执行）                   │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  Account A: nonce 5 → nonce 6 → nonce 7            │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  2. 不同账户间：按 effective gas price 降序                     │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  Legacy: gasPrice                                   │    │
│     │  EIP-1559: min(maxFeePerGas, baseFee + maxPriorityFee)│  │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  3. 本地交易优先                                                │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  locals 账户的交易不受价格下限限制                   │    │
│     │  即使价格低也会保留更长时间                          │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 37.2 价格堆实现

```go
// priceHeap 价格堆
type priceHeap struct {
baseFee *uint256.Int // 当前基础费
list    []*types.Transaction // 交易列表
}

// 实现 heap.Interface
func (h *priceHeap) Len() int { return len(h.list) }

func (h *priceHeap) Less(i, j int) bool {
// 比较 effective tip
tipI := h.list[i].EffectiveGasTipIntCmp(h.list[j], h.baseFee)
if tipI != 0 {
return tipI > 0 // 高 tip 优先
}
// tip 相同时比较 nonce
return h.list[i].Nonce() < h.list[j].Nonce()
}

func (h *priceHeap) Swap(i, j int) {
h.list[i], h.list[j] = h.list[j], h.list[i]
}

func (h *priceHeap) Push(x interface{}) {
h.list = append(h.list, x.(*types.Transaction))
}

func (h *priceHeap) Pop() interface{} {
old := h.list
n := len(old)
x := old[n-1]
h.list = old[0: n-1]
return x
}
```

### 37.3 获取待打包交易

```go
// Pending 返回所有可执行交易
func (pool *LegacyPool) Pending(filter PendingFilter) map[common.Address][]*LazyTransaction {
pool.mu.Lock()
defer pool.mu.Unlock()

pending := make(map[common.Address][]*LazyTransaction, len(pool.pending))

for addr, list := range pool.pending {
txs := list.Flatten()

// 应用过滤器
if filter.MinTip != nil && filter.BaseFee != nil {
filtered := make(types.Transactions, 0, len(txs))
for _, tx := range txs {
if tx.EffectiveGasTipIntCmp(filter.MinTip, filter.BaseFee) >= 0 {
filtered = append(filtered, tx)
} else {
break // nonce 必须连续，遇到低价的就停止
}
}
txs = filtered
}

if len(txs) > 0 {
lazies := make([]*LazyTransaction, len(txs))
for i, tx := range txs {
lazies[i] = &LazyTransaction{
Hash:      tx.Hash(),
Tx:        tx,
Time:      tx.Time(),
GasFeeCap: tx.GasFeeCap(),
GasTipCap: tx.GasTipCap(),
}
}
pending[addr] = lazies
}
}
return pending
}

// ContentFrom 获取指定账户的所有交易
func (pool *LegacyPool) ContentFrom(addr common.Address) ([]*types.Transaction, []*types.Transaction) {
pool.mu.Lock()
defer pool.mu.Unlock()

var pending, queued types.Transactions

if list, ok := pool.pending[addr]; ok {
pending = list.Flatten()
}
if list, ok := pool.queue[addr]; ok {
queued = list.Flatten()
}
return pending, queued
}
```

---

## 38. Nonce 管理

### 38.1 Nonce 概念

```
┌─────────────────────────────────────────────────────────────────┐
│                      Nonce 机制                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Nonce = 账户发送的交易计数（从 0 开始）                        │
│                                                                 │
│  作用：                                                         │
│  1. 防止重放攻击：同一 nonce 只能用一次                         │
│  2. 确定执行顺序：必须按 nonce 顺序执行                         │
│  3. 交易替换：相同 nonce 可以替换交易                           │
│                                                                 │
│  示例：                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  链上状态：nonce = 5                                     │   │
│  │                                                          │   │
│  │  发送 nonce=5 ✓ → 执行后 nonce=6                        │   │
│  │  发送 nonce=4 ✗ → 拒绝（nonce too low）                 │   │
│  │  发送 nonce=7 → 进入 queue（等待 nonce=6）              │   │
│  │  再发 nonce=5 → 替换原交易（如果价格更高）              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 38.2 Nonce 追踪器

```go
// noncer Pending nonce 追踪器
type noncer struct {
fallback *state.StateDB
nonces   map[common.Address]uint64
lock     sync.Mutex
}

// get 获取账户的下一个可用 nonce
func (txn *noncer) get(addr common.Address) uint64 {
txn.lock.Lock()
defer txn.lock.Unlock()

if _, ok := txn.nonces[addr]; !ok {
// 从状态获取当前 nonce
txn.nonces[addr] = txn.fallback.GetNonce(addr)
}
return txn.nonces[addr]
}

// set 设置账户的 pending nonce
func (txn *noncer) set(addr common.Address, nonce uint64) {
txn.lock.Lock()
defer txn.lock.Unlock()

txn.nonces[addr] = nonce
}

// setIfLower 如果更低则设置
func (txn *noncer) setIfLower(addr common.Address, nonce uint64) {
txn.lock.Lock()
defer txn.lock.Unlock()

if _, ok := txn.nonces[addr]; !ok {
txn.nonces[addr] = txn.fallback.GetNonce(addr)
}
if txn.nonces[addr] > nonce {
txn.nonces[addr] = nonce
}
}
```

### 38.3 Nonce Gap 处理

```go
// promoteTx 将交易从 queue 提升到 pending
func (pool *LegacyPool) promoteTx(addr common.Address, hash common.Hash, tx *types.Transaction) bool {
if pool.pending[addr] == nil {
pool.pending[addr] = newList(true)
}
list := pool.pending[addr]

inserted, old := list.Add(tx, pool.config.PriceBump)
if !inserted {
// 价格太低，无法替换
pool.all.Remove(hash)
return false
}

if old != nil {
pool.all.Remove(old.Hash())
pool.priced.Removed(1)
}
pool.all.Add(tx, pool.locals.containsTx(tx))
pool.priced.Put(tx, pool.locals.containsTx(tx))
pool.journalTx(addr, tx)
pool.queueTxEvent(tx)

return true
}

// promoteExecutables 提升可执行交易
func (pool *LegacyPool) promoteExecutables(accounts []common.Address) []*types.Transaction {
var promoted []*types.Transaction

for _, addr := range accounts {
list := pool.queue[addr]
if list == nil {
continue
}

// 获取当前 nonce
nonce := pool.currentState.GetNonce(addr)

// 移除过低的 nonce 交易
forwards := list.Forward(nonce)
for _, tx := range forwards {
pool.all.Remove(tx.Hash())
}

// 提升连续 nonce 的交易
ready := list.Ready(nonce)
for _, tx := range ready {
if pool.promoteTx(addr, tx.Hash(), tx) {
promoted = append(promoted, tx)
}
}

// 删除超出限制的交易
if !pool.locals.contains(addr) {
caps := list.Cap(int(pool.config.AccountQueue))
for _, tx := range caps {
pool.all.Remove(tx.Hash())
}
}

// 删除空的 queue
if list.Empty() {
delete(pool.queue, addr)
delete(pool.beats, addr)
}
}
return promoted
}
```

---

## 39. Gas 价格机制

### 39.1 EIP-1559 Gas 价格模型

```
┌─────────────────────────────────────────────────────────────────┐
│                  EIP-1559 Gas 价格                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Legacy 交易：                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  gasPrice = 用户愿意支付的每单位 Gas 价格                │   │
│  │  矿工收入 = gasPrice * gasUsed                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  EIP-1559 交易：                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  maxFeePerGas = 用户愿意支付的最高价格                   │   │
│  │  maxPriorityFeePerGas = 用户愿意给矿工的最高小费         │   │
│  │  baseFee = 协议设定的基础费（会被销毁）                  │   │
│  │                                                          │   │
│  │  effectiveFee = min(maxFeePerGas, baseFee + maxPriorityFee)│ │
│  │  priorityFee = effectiveFee - baseFee                    │   │
│  │                                                          │   │
│  │  用户支付 = effectiveFee * gasUsed                      │   │
│  │  销毁 = baseFee * gasUsed                               │   │
│  │  矿工收入 = priorityFee * gasUsed                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  baseFee 动态调整：                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  目标区块使用率 = 50%                                    │   │
│  │  区块 > 50%: baseFee 上涨（最多 12.5%）                 │   │
│  │  区块 < 50%: baseFee 下降（最多 12.5%）                 │   │
│  │  区块 = 50%: baseFee 不变                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 39.2 Gas 价格相关代码

```go
// EffectiveGasTip 计算有效小费
func (tx *Transaction) EffectiveGasTip(baseFee *big.Int) (*big.Int, error) {
if baseFee == nil {
return tx.GasTipCap(), nil
}

var effectiveTip *big.Int

if tx.Type() == LegacyTxType {
// Legacy: effectiveTip = gasPrice - baseFee
effectiveTip = new(big.Int).Sub(tx.GasPrice(), baseFee)
} else {
// EIP-1559: effectiveTip = min(maxFeePerGas - baseFee, maxPriorityFeePerGas)
effectiveTip = new(big.Int).Sub(tx.GasFeeCap(), baseFee)
if effectiveTip.Cmp(tx.GasTipCap()) > 0 {
effectiveTip = tx.GasTipCap()
}
}

if effectiveTip.Sign() < 0 {
return nil, ErrGasFeeCapTooLow
}
return effectiveTip, nil
}

// CalcBaseFee 计算下一个区块的 baseFee
func CalcBaseFee(config *params.ChainConfig, parent *types.Header) *big.Int {
// EIP-1559 未激活
if !config.IsLondon(new(big.Int).Add(parent.Number, common.Big1)) {
return nil
}

parentGasTarget := parent.GasLimit / params.ElasticityMultiplier  // 目标 = limit / 2
parentBaseFee := parent.BaseFee

// 如果父区块使用量等于目标，baseFee 不变
if parent.GasUsed == parentGasTarget {
return parentBaseFee
}

var baseFee *big.Int

if parent.GasUsed > parentGasTarget {
// 区块使用过多，baseFee 上涨
gasUsedDelta := parent.GasUsed - parentGasTarget
baseFeeDelta := new(big.Int).Mul(parentBaseFee, new(big.Int).SetUint64(gasUsedDelta))
baseFeeDelta.Div(baseFeeDelta, new(big.Int).SetUint64(parentGasTarget))
baseFeeDelta.Div(baseFeeDelta, new(big.Int).SetUint64(params.BaseFeeChangeDenominator)) // 8

// 最少涨 1 wei
if baseFeeDelta.Cmp(common.Big1) < 0 {
baseFeeDelta = common.Big1
}
baseFee = new(big.Int).Add(parentBaseFee, baseFeeDelta)
} else {
// 区块使用不足，baseFee 下降
gasUsedDelta := parentGasTarget - parent.GasUsed
baseFeeDelta := new(big.Int).Mul(parentBaseFee, new(big.Int).SetUint64(gasUsedDelta))
baseFeeDelta.Div(baseFeeDelta, new(big.Int).SetUint64(parentGasTarget))
baseFeeDelta.Div(baseFeeDelta, new(big.Int).SetUint64(params.BaseFeeChangeDenominator))

baseFee = new(big.Int).Sub(parentBaseFee, baseFeeDelta)
}

return baseFee
}
```

---

## 40. 交易替换与取消

### 40.1 交易替换机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    交易替换机制                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  替换条件：                                                     │
│  1. 相同发送者                                                  │
│  2. 相同 nonce                                                  │
│  3. 新交易 Gas 价格 >= 旧交易 * (100 + PriceBump) / 100         │
│     默认 PriceBump = 10%                                        │
│                                                                 │
│  示例：                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  原交易：nonce=5, gasPrice=100 Gwei                      │   │
│  │  替换交易：nonce=5, gasPrice >= 110 Gwei                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  用途：                                                         │
│  • 加速交易：提高 Gas 价格                                      │
│  • 取消交易：发送 0 ETH 给自己                                  │
│  • 修改交易：完全替换交易内容                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 40.2 替换代码实现

```go
// list.Add 添加交易到列表（可能替换旧交易）
func (l *list) Add(tx *types.Transaction, priceBump uint64) (bool, *types.Transaction) {
old := l.txs.Get(tx.Nonce())

if old != nil {
// 检查是否满足替换条件
if !l.canReplace(old, tx, priceBump) {
return false, nil
}
l.txs.Put(tx)
return true, old
}

l.txs.Put(tx)
return true, nil
}

// canReplace 检查新交易是否可以替换旧交易
func (l *list) canReplace(old, new *types.Transaction, priceBump uint64) bool {
// 计算所需的最低价格
// newPrice >= oldPrice * (100 + priceBump) / 100

oldTip := old.GasTipCap()
oldFeeCap := old.GasFeeCap()

// 计算最小 tip
threshold := new(big.Int).Mul(oldTip, big.NewInt(100+int64(priceBump)))
threshold.Div(threshold, big.NewInt(100))

if new.GasTipCapIntCmp(threshold) < 0 {
return false
}

// 计算最小 feeCap
threshold = new(big.Int).Mul(oldFeeCap, big.NewInt(100+int64(priceBump)))
threshold.Div(threshold, big.NewInt(100))

return new.GasFeeCapIntCmp(threshold) >= 0
}
```

### 40.3 取消交易

```go
// CancelTransaction 取消交易
func CancelTransaction(client *ethclient.Client, privateKey *ecdsa.PrivateKey, nonce uint64) error {
// 获取当前 Gas 价格
ctx := context.Background()
gasPrice, _ := client.SuggestGasPrice(ctx)

// 提高 10% 以确保替换成功
newGasPrice := new(big.Int).Mul(gasPrice, big.NewInt(110))
newGasPrice.Div(newGasPrice, big.NewInt(100))

// 构建空交易（发送 0 ETH 给自己）
fromAddress := crypto.PubkeyToAddress(privateKey.PublicKey)

tx := types.NewTransaction(
nonce,
fromAddress, // 发给自己
big.NewInt(0), // 0 ETH
21000,         // 最小 Gas
newGasPrice,
nil, // 无数据
)

// 签名并发送
chainID, _ := client.NetworkID(ctx)
signedTx, _ := types.SignTx(tx, types.NewEIP155Signer(chainID), privateKey)

return client.SendTransaction(ctx, signedTx)
}
```

---

## 41. 实战：监控与分析 TxPool

### 41.1 获取 TxPool 内容

```go
// TxPoolMonitor TxPool 监控器
type TxPoolMonitor struct {
client  *rpc.Client
ethClient *ethclient.Client
}

// GetTxPoolContent 获取交易池全部内容
func (m *TxPoolMonitor) GetTxPoolContent() (*TxPoolContent, error) {
var result TxPoolContent
err := m.client.Call(&result, "txpool_content")
return &result, err
}

type TxPoolContent struct {
Pending map[string]map[string]*RPCTransaction `json:"pending"`
Queued  map[string]map[string]*RPCTransaction `json:"queued"`
}

type RPCTransaction struct {
Hash             string `json:"hash"`
Nonce            string `json:"nonce"`
From             string `json:"from"`
To               string `json:"to"`
Value            string `json:"value"`
Gas              string `json:"gas"`
GasPrice         string `json:"gasPrice"`
MaxFeePerGas     string `json:"maxFeePerGas,omitempty"`
MaxPriorityFee   string `json:"maxPriorityFeePerGas,omitempty"`
Input            string `json:"input"`
}

// GetTxPoolStatus 获取交易池状态
func (m *TxPoolMonitor) GetTxPoolStatus() (*TxPoolStatus, error) {
var result TxPoolStatus
err := m.client.Call(&result, "txpool_status")
return &result, err
}

type TxPoolStatus struct {
Pending hexutil.Uint `json:"pending"`
Queued  hexutil.Uint `json:"queued"`
}
```

### 41.2 订阅 Pending 交易

```go
// SubscribePendingTxs 订阅 pending 交易
func (m *TxPoolMonitor) SubscribePendingTxs(ctx context.Context) (<-chan common.Hash, error) {
ch := make(chan common.Hash, 100)

sub, err := m.client.EthSubscribe(ctx, ch, "newPendingTransactions")
if err != nil {
return nil, err
}

go func () {
<-sub.Err()
close(ch)
}()

return ch, nil
}

// SubscribeFullPendingTxs 订阅完整 pending 交易（需要特殊节点配置）
func (m *TxPoolMonitor) SubscribeFullPendingTxs(ctx context.Context) (<-chan *types.Transaction, error) {
ch := make(chan *types.Transaction, 100)

sub, err := m.client.EthSubscribe(ctx, ch, "newPendingTransactions", true)
if err != nil {
return nil, err
}

go func () {
<-sub.Err()
close(ch)
}()

return ch, nil
}

// MonitorPendingTxs 监控 pending 交易
func (m *TxPoolMonitor) MonitorPendingTxs(ctx context.Context, handler func (*types.Transaction)) error {
hashes, err := m.SubscribePendingTxs(ctx)
if err != nil {
return err
}

for hash := range hashes {
// 获取完整交易
tx, _, err := m.ethClient.TransactionByHash(ctx, hash)
if err != nil {
continue
}

handler(tx)
}

return nil
}
```

### 41.3 分析 DEX 交易

```go
// DEXAnalyzer DEX 交易分析器
type DEXAnalyzer struct {
uniswapRouter common.Address
sushiRouter   common.Address
routerABI     abi.ABI
}

// AnalyzeTx 分析交易
func (d *DEXAnalyzer) AnalyzeTx(tx *types.Transaction) *DEXTxInfo {
to := tx.To()
if to == nil {
return nil
}

// 检查是否是已知 DEX Router
if *to != d.uniswapRouter && *to != d.sushiRouter {
return nil
}

// 解码函数调用
method, err := d.routerABI.MethodById(tx.Data()[:4])
if err != nil {
return nil
}

info := &DEXTxInfo{
Hash:     tx.Hash(),
Method:   method.Name,
GasPrice: tx.GasPrice(),
Value:    tx.Value(),
}

// 解码参数
args, err := method.Inputs.Unpack(tx.Data()[4:])
if err != nil {
return info
}

// 根据方法解析参数
switch method.Name {
case "swapExactTokensForTokens":
info.AmountIn = args[0].(*big.Int)
info.AmountOutMin = args[1].(*big.Int)
info.Path = args[2].([]common.Address)
case "swapTokensForExactTokens":
info.AmountOut = args[0].(*big.Int)
info.AmountInMax = args[1].(*big.Int)
info.Path = args[2].([]common.Address)
case "swapExactETHForTokens":
info.AmountIn = tx.Value()
info.AmountOutMin = args[0].(*big.Int)
info.Path = args[1].([]common.Address)
}

return info
}

type DEXTxInfo struct {
Hash         common.Hash
Method       string
GasPrice     *big.Int
Value        *big.Int
AmountIn     *big.Int
AmountOut    *big.Int
AmountInMax  *big.Int
AmountOutMin *big.Int
Path         []common.Address
}
```

### 41.4 MEV 机会检测

```go
// MEVDetector MEV 机会检测器
type MEVDetector struct {
analyzer   *DEXAnalyzer
pools      map[common.Address]*Pool
minProfit  *big.Int
}

// DetectOpportunity 检测 MEV 机会
func (d *MEVDetector) DetectOpportunity(tx *types.Transaction) *MEVOpportunity {
info := d.analyzer.AnalyzeTx(tx)
if info == nil {
return nil
}

// 检查交易大小（小交易不值得攻击）
if info.AmountIn != nil && info.AmountIn.Cmp(big.NewInt(1e18)) < 0 {
return nil
}

// 计算滑点空间
slippage := d.calculateSlippage(info)

// 滑点太小，没有攻击空间
if slippage < 0.005 {  // 0.5%
return nil
}

// 计算潜在利润
profit := d.estimateProfit(info, slippage)

if profit.Cmp(d.minProfit) <= 0 {
return nil
}

return &MEVOpportunity{
Type:          "sandwich",
VictimTx:      tx,
ExpectedProfit: profit,
Slippage:      slippage,
}
}

// calculateSlippage 计算滑点
func (d *MEVDetector) calculateSlippage(info *DEXTxInfo) float64 {
if info.AmountIn == nil || info.AmountOutMin == nil {
return 0
}

// 获取当前市场价格
expectedOutput := d.getExpectedOutput(info.Path, info.AmountIn)
if expectedOutput.Sign() == 0 {
return 0
}

// 滑点 = (expected - minOutput) / expected
diff := new(big.Int).Sub(expectedOutput, info.AmountOutMin)
slippage := new(big.Float).SetInt(diff)
slippage.Quo(slippage, new(big.Float).SetInt(expectedOutput))

f, _ := slippage.Float64()
return f
}
```

### 41.5 完整监控示例

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/rpc"
)

func main() {
	ctx := context.Background()

	// 连接节点
	rpcClient, err := rpc.Dial("ws://localhost:8546")
	if err != nil {
		log.Fatal(err)
	}

	ethClient := ethclient.NewClient(rpcClient)

	// 创建监控器
	monitor := &TxPoolMonitor{
		client:    rpcClient,
		ethClient: ethClient,
	}

	// 获取当前状态
	status, _ := monitor.GetTxPoolStatus()
	fmt.Printf("TxPool Status: Pending=%d, Queued=%d\n",
		status.Pending, status.Queued)

	// 监控 pending 交易
	fmt.Println("Monitoring pending transactions...")

	err = monitor.MonitorPendingTxs(ctx, func(tx *types.Transaction) {
		from, _ := types.Sender(types.LatestSignerForChainID(tx.ChainId()), tx)

		fmt.Printf("New Tx: %s\n", tx.Hash().Hex())
		fmt.Printf("  From: %s\n", from.Hex())
		fmt.Printf("  To: %s\n", tx.To().Hex())
		fmt.Printf("  Value: %s ETH\n", weiToEth(tx.Value()))
		fmt.Printf("  Gas Price: %s Gwei\n", weiToGwei(tx.GasPrice()))
		fmt.Println()
	})

	if err != nil {
		log.Fatal(err)
	}
}

func weiToEth(wei *big.Int) string {
	eth := new(big.Float).SetInt(wei)
	eth.Quo(eth, new(big.Float).SetInt64(1e18))
	return eth.Text('f', 4)
}

func weiToGwei(wei *big.Int) string {
	gwei := new(big.Float).SetInt(wei)
	gwei.Quo(gwei, new(big.Float).SetInt64(1e9))
	return gwei.Text('f', 2)
}
```

---

## 总结

本文档详细介绍了 go-ethereum 交易池的设计与实现：

1. **TxPool 基础**：交易生命周期、核心职责、对 MEV 的重要性
2. **架构设计**：Pending/Queue 分区、索引结构、BlobPool
3. **交易验证**：完整的验证流程和检查项
4. **排序算法**：基于 Gas 价格的优先级排序
5. **Nonce 管理**：确保交易顺序执行
6. **Gas 价格**：EIP-1559 机制详解
7. **交易替换**：加速和取消交易
8. **实战监控**：订阅和分析 pending 交易

下一篇将介绍 EVM 模拟与本地执行。
