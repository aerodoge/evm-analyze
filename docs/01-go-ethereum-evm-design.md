# Go-Ethereum EVM 设计与实现详解

> 本文档详细讲解 go-ethereum 中 EVM（以太坊虚拟机）的设计思想、原理和架构实现，适合新手学习。

## 目录

- [1. EVM 基础概念](#1-evm-基础概念)
- [2. 整体架构设计](#2-整体架构设计)
- [3. 核心数据结构](#3-核心数据结构)
- [4. 指令集系统](#4-指令集系统)
- [5. 执行引擎](#5-执行引擎)
- [6. Gas 机制](#6-gas-机制)
- [7. 内存模型](#7-内存模型)
- [8. 存储模型](#8-存储模型)
- [9. 合约调用机制](#9-合约调用机制)
- [10. 预编译合约](#10-预编译合约)
- [11. 实战代码示例](#11-实战代码示例)

---

## 1. EVM 基础概念

### 1.1 什么是 EVM？

**EVM (Ethereum Virtual Machine)** 是以太坊的核心组件，可以把它理解为一台"世界计算机"：

```
┌─────────────────────────────────────────────────────────────┐
│                        现实世界类比                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    Computer                         Ethereum                │
│   ┌─────────┐                      ┌─────────┐              │
│   │   CPU   │  ─────────────────►  │   EVM   │              │
│   └─────────┘                      └─────────┘              │
│   ┌─────────┐                      ┌─────────┐              │
│   │   RAM   │  ─────────────────►  │ Memory  │              │
│   └─────────┘                      └─────────┘              │
│   ┌─────────┐                      ┌─────────┐              │
│   │  Disk   │  ─────────────────►  │ Storage │              │
│   └─────────┘                      └─────────┘              │
│   ┌─────────┐                      ┌─────────┐              │
│   │ Program │  ─────────────────►  │ Contract│              │
│   └─────────┘                      └─────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 EVM的核心特性

```
┌────────────────────────────────────────────────────────────────┐
│                        EVM 核心特性                              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. 确定性 (Deterministic)                                      │
│     ├── 相同输入 → 相同输出                                       │
│     ├── 全球所有节点执行结果一致                                   │
│     └── 保证区块链状态一致性                                      │
│                                                                │
│  2. 隔离性 (Isolated)                                           │
│     ├── 沙箱环境执行                                             │
│     ├── 合约无法访问外部系统                                      │
│     └── 合约间通过消息调用交互                                     │
│                                                                │
│  3. 可终止性 (Terminable)                                       │
│     ├── Gas机制防止无限循环                                      │
│     ├── 每个操作消耗固定Gas                                      │
│     └── Gas耗尽则执行终止                                        │
│                                                                │
│  4. 栈式架构 (Stack-based)                                      │
│     ├── 所有计算基于栈操作                                        │
│     ├── 栈深度最大1024                                           │
│     └── 每个栈元素256位                                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 为什么选择栈式架构？

```
┌─────────────────────────────────────────────────────────────────┐
│                  栈式 vs 寄存器式 架构对比                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  栈式架构 (EVM选择)                寄存器式架构                     │
│  ┌──────────────────┐            ┌──────────────────┐           │
│  │ 指令更简单紧凑     │            │ 指令需指定寄存器    │           │
│  │ ADD = 1字节      │             │ ADD R1,R2,R3     │           │
│  │                  │            │ = 多字节          │           │
│  ├──────────────────┤            ├──────────────────┤           │
│  │ 实现更简单        │             │ 实现更复杂         │           │
│  │ 无需寄存器分配     │            │ 需要寄存器分配      │           │
│  ├──────────────────┤            ├──────────────────┤           │
│  │ 代码体积更小       │            │ 执行可能更快       │           │
│  │ (链上存储节省Gas)  │            │ (但链上成本高)     │           │
│  └──────────────────┘            └──────────────────┘           │
│                                                                 │
│  EVM选择栈式的原因:                                               │
│  • 智能合约代码存储在链上，体积直接影响Gas费用                         │
│  • 简单架构更容易实现确定性执行                                      │
│  • 便于各种语言实现兼容的VM                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 整体架构设计

### 2.1 EVM 在 go-ethereum 中的位置

```
┌────────────────────────────────────────────────────────────────┐
│                    go-ethereum 整体架构                         │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     应用层 (Application)                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐     │   │
│  │  │ JSON-RPC│  │ GraphQL │  │   CLI   │  │ Console │     │   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘     │   │
│  └───────┼────────────┼────────────┼────────────┼──────────┘   │
│          │            │            │            │              │
│  ┌───────▼────────────▼────────────▼────────────▼──────────┐   │
│  │                     核心层 (Core)                        │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │              State Processor                     │   │   │
│  │  │  ┌────────────────────────────────────────────┐  │   │   │
│  │  │  │          ★ EVM (核心) ★                     │  │   │   │
│  │  │  │  ┌───────────┐ ┌─────────┐ ┌─────────────┐ │  │   │   │
│  │  │  │  │Interpreter│ │  Stack  │ │   Memory    │ │  │   │   │
│  │  │  │  └───────────┘ └─────────┘ └─────────────┘ │  │   │   │
│  │  │  │  ┌─────────┐ ┌─────────┐ ┌─────────────┐   │  │   │   │
│  │  │  │  │  OpCode │ │   Gas   │ │   Storage   │   │  │   │   │
│  │  │  │  └─────────┘ └─────────┘ └─────────────┘   │  │   │   │
│  │  │  └────────────────────────────────────────────┘  │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │   │
│  │  │ TxPool   │  │Blockchain│  │  Miner   │               │   │
│  │  └──────────┘  └──────────┘  └──────────┘               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     数据层 (Data)                        │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │   │
│  │  │ LevelDB  │  │  Trie    │  │   MPT    │               │   │
│  │  └──────────┘  └──────────┘  └──────────┘               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     网络层 (Network)                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │   │
│  │  │  DevP2P  │  │   ETH    │  │   LES    │               │   │
│  │  └──────────┘  └──────────┘  └──────────┘               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 EVM 源码目录结构

```
go-ethereum/core/vm/
│
├── evm.go              # EVM 主结构体和入口
│   └── 定义 EVM struct, Call/Create 等核心方法
│
├── interpreter.go      # 解释器，字节码执行引擎
│   └── 循环读取并执行 opcode
│
├── opcodes.go          # 操作码定义
│   └── 所有 256 个操作码的常量定义
│
├── instructions.go     # 指令实现
│   └── 每个操作码对应的具体执行逻辑
│
├── jump_table.go       # 跳转表
│   └── 操作码 → 执行函数的映射表
│
├── gas_table.go        # Gas 消耗表
│   └── 各操作的 Gas 计算函数
│
├── gas.go              # Gas 计算通用函数
│
├── memory.go           # 内存实现
│   └── 动态扩展的字节数组
│
├── stack.go            # 栈实现
│   └── 256位整数栈
│
├── contract.go         # 合约抽象
│   └── 合约代码、调用者、值等信息
│
├── contracts.go        # 预编译合约注册
│
├── errors.go           # 错误定义
│
└── analysis.go         # 代码分析(JUMPDEST验证)
```

### 2.3 EVM 核心组件关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVM 核心组件关系                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      ┌─────────────┐                            │
│                      │     EVM     │                            │
│                      │  (evm.go)   │                            │
│                      └──────┬──────┘                            │
│                             │                                   │
│            ┌────────────────┼────────────────┐                  │
│            │                │                │                  │
│            ▼                ▼                ▼                  │
│     ┌──────────┐    ┌──────────────┐   ┌──────────┐             │
│     │ StateDB  │    │ Interpreter  │   │ Context  │             │
│     │ (状态)    │    │  (解释器)     │   │ (上下文)  │            │
│     └──────────┘    └──────┬───────┘   └──────────┘             │
│                            │                                    │
│          ┌─────────────────┼─────────────────┐                  │
│          │                 │                 │                  │
│          ▼                 ▼                 ▼                  │
│    ┌──────────┐     ┌──────────┐      ┌──────────┐              │
│    │  Stack   │     │  Memory  │      │ Contract │              │
│    │  (栈)    │      │  (内存)  │      │  (合约)  │              │
│    │ 1024深度 │      │ 可扩展   │       │ 代码+数据│               │
│    │ 256bit项 │      │ 按字节   │      │          │              │
│    └──────────┘     └──────────┘      └────┬─────┘              │
│                                            │                    │
│                                            ▼                    │
│                                     ┌──────────┐               │
│                                     │ Storage  │               │
│                                     │ (持久存储)│               │
│                                     │ key→value│               │
│                                     │ 256→256  │               │
│                                     └──────────┘               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心数据结构

### 3.1 EVM 主结构体

```go
// 文件: core/vm/evm.go

// BlockContext 提供 EVM 执行时的区块信息
// 这些信息在整个区块的所有交易执行期间保持不变
type BlockContext struct {
    // CanTransfer 检查账户是否有足够余额进行转账
    CanTransfer CanTransferFunc
    
    // Transfer 执行实际的 ETH 转账
    Transfer TransferFunc
    
    // GetHash 获取指定区块号的区块哈希
    // 用于 BLOCKHASH 操作码
    GetHash GetHashFunc
    
    // 区块信息字段
    Coinbase    common.Address // 矿工/验证者地址 (COINBASE 操作码)
    GasLimit    uint64         // 区块 Gas 上限 (GASLIMIT 操作码)
    BlockNumber *big.Int       // 区块号 (NUMBER 操作码)
    Time        uint64         // 区块时间戳 (TIMESTAMP 操作码)
    Difficulty  *big.Int       // 挖矿难度 (DIFFICULTY 操作码)
    BaseFee     *big.Int       // EIP-1559 基础费用 (BASEFEE 操作码)
    Random      *common.Hash   // PREVRANDAO (合并后)
}

// TxContext 提供 EVM 执行时的交易信息
// 这些信息在每笔交易执行期间保持不变
type TxContext struct {
    Origin   common.Address // 交易发起者 (tx.from)
    GasPrice *big.Int       // Gas 价格
}

// EVM 是以太坊虚拟机的核心结构
type EVM struct {
    // Context 包含区块和交易上下文
    Context BlockContext
    TxContext
    
    // StateDB 状态数据库接口
    // 提供账户余额、合约代码、存储等读写能力
    StateDB StateDB
    
    // depth 当前调用深度 (最大 1024)
    depth int
    
    // chainConfig 链配置
    // 包含链 ID、各硬分叉激活区块号等
    chainConfig *params.ChainConfig
    
    // chainRules 当前区块适用的规则
    chainRules params.Rules
    
    // Config EVM 配置
    Config Config
    
    // interpreter EVM 解释器
    interpreter *EVMInterpreter
    
    // abort 终止标志
    abort atomic.Bool
    
    // callGasTemp 临时存储 CALL 类操作计算的 Gas
    callGasTemp uint64
}
```

**EVM 结构体的作用：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      EVM 结构体详解                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  一笔交易执行时，EVM 需要知道：                                     │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  BlockContext (区块上下文) - "我在哪个区块执行？"             │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ BlockNumber: 18000000     ← 当前区块号              │   │  │
│  │  │ Time: 1693526400          ← 区块时间戳              │   │  │
│  │  │ Coinbase: 0xabc...        ← 出块者地址              │   │  │
│  │  │ GasLimit: 30000000        ← 区块Gas上限             │   │  │
│  │  │ BaseFee: 20 gwei          ← EIP-1559基础费用        │   │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  TxContext (交易上下文) - "谁发起的这笔交易？"                │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ Origin: 0x123...          ← 最初的交易发起者(EOA)     │  │  │
│  │  │ GasPrice: 25 gwei         ← Gas价格                 │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  StateDB (状态数据库) - "区块链当前的状态是什么？"             │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ • 读写账户余额                                       │  │  │
│  │  │ • 读写合约代码                                       │  │  │
│  │  │ • 读写合约存储                                       │  │  │
│  │  │ • 创建/销毁账户                                      │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 解释器结构体

```go
// 文件: core/vm/interpreter.go

// Config 是解释器的配置
type Config struct {
    Tracer                  EVMLogger // 追踪器，用于调试
    NoBaseFee               bool      // 禁用基础费用检查
    EnablePreimageRecording bool      // 记录SHA3原像
    ExtraEips               []int     // 额外启用的EIP
}

// ScopeContext 包含每次合约调用的作用域数据
type ScopeContext struct {
    Memory   *Memory   // 内存
    Stack    *Stack    // 栈
    Contract *Contract // 当前执行的合约
}

// EVMInterpreter 代表EVM解释器
type EVMInterpreter struct {
    evm   *EVM       // 关联的EVM实例
    table *JumpTable // 操作码跳转表
    
    // hasher和hasherBuf用于KECCAK256操作
    hasher    crypto.KeccakState
    hasherBuf common.Hash
    
    // readOnly 只读模式标志
    // 在STATICCALL 中会设为true
    readOnly bool
    
    // returnData 上一次CALL的返回数据
    returnData []byte
}
```

### 3.3 合约结构体

```go
// 文件: core/vm/contract.go

// Contract 表示状态数据库中的以太坊合约
type Contract struct {
    // CallerAddress是调用者的地址（初始化此合约的账户）
    CallerAddress common.Address
    
    // caller 是调用者的合约引用
    caller ContractRef
    
    // self 是本合约的引用
    self ContractRef
    
    // jumpdests 是JUMPDEST分析结果的缓存
    // 用于验证跳转目标是否有效
    jumpdests map[common.Hash]bitvec
    
    // analysis 是当前代码的分析结果
    analysis bitvec
    
    // Code 是合约字节码
    Code []byte
    
    // CodeHash 是代码的keccak256哈希
    CodeHash common.Hash
    
    // CodeAddr 是代码所在的地址（可能与执行地址不同，如 DELEGATECALL）
    CodeAddr *common.Address
    
    // Input 是调用时传入的数据
    Input []byte
    
    // Gas 是可用的Gas
    Gas uint64
    
    // value 是调用时转账的ETH数量（wei）
    value *uint256.Int
}
```

**合约调用中的地址关系：**

```
┌─────────────────────────────────────────────────────────────────┐
│                 合约调用中的地址关系                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  场景: 用户 A 调用合约 B，合约 B 又调用合约 C                        │
│                                                                 │
│  ┌─────────┐    CALL     ┌─────────┐    CALL     ┌─────────┐    │
│  │ User A  │ ──────────► │Contract │ ──────────► │Contract │    │
│  │  (EOA)  │             │    B    │             │    C    │    │
│  │ 0xAAA   │             │ 0xBBB   │             │ 0xCCC   │    │
│  └─────────┘             └─────────┘             └─────────┘    │
│                                                                 │
│  在合约 C 的执行环境中:                                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  tx.origin (Origin)     = 0xAAA  ← 最初发起交易的EOA        │   │
│  │  msg.sender (Caller)    = 0xBBB  ← 直接调用者              │  │
│  │  address(this) (Self)   = 0xCCC  ← 当前合约地址            │   │
│  │  msg.value              = 转账金额                         │  │
│  │  msg.data (Input)       = 调用数据                         │  │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  DELEGATECALL 特殊情况:                                          │
│  ┌─────────┐ DELEGATECALL ┌─────────┐                           │
│  │Contract │ ───────────► │Contract │                           │
│  │    B    │              │    C    │ (只借用代码)                │
│  │ 0xBBB   │              │ 0xCCC   │                           │
│  └─────────┘              └─────────┘                           │
│                                                                 │
│  在 DELEGATECALL 中:                                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  msg.sender = 原来的调用者 (不是 B)                        │    │
│  │  address(this) = 0xBBB (仍然是 B 的地址!)                  │   │
│  │  storage 操作作用于 B 的存储                               │    │
│  │  只是执行 C 的代码                                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 栈结构体

```go
// 文件: core/vm/stack.go

// Stack 是用于基本栈操作的对象
type Stack struct {
    data []uint256.Int
}

// 创建新栈，预分配16个元素
func newstack() *Stack {
    return &Stack{data: make([]uint256.Int, 0, 16)}
}

// push 将新元素压入栈顶
func (st *Stack) push(d *uint256.Int) {
    st.data = append(st.data, *d)
}

// pop 弹出栈顶元素
func (st *Stack) pop() uint256.Int {
    ret := st.data[len(st.data)-1]
    st.data = st.data[:len(st.data)-1]
    return ret
}

// peek 查看栈顶元素（不弹出）
func (st *Stack) peek() *uint256.Int {
    return &st.data[len(st.data)-1]
}

// Back 返回从栈顶往下第n个元素
func (st *Stack) Back(n int) *uint256.Int {
    return &st.data[len(st.data)-n-1]
}

// swap 交换栈顶元素与第 n 个元素
func (st *Stack) swap(n int) {
    st.data[len(st.data)-n-1], st.data[len(st.data)-1] =
    st.data[len(st.data)-1], st.data[len(st.data)-n-1]
}

// dup 复制第 n 个元素到栈顶
func (st *Stack) dup(n int) {
    st.push(&st.data[len(st.data)-n])
}
```

**栈操作图解：**

```
┌─────────────────────────────────────────────────────────────────┐
│                        栈操作图解                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. PUSH 操作 (将值压入栈)                                       │
│  ┌───────────┐        ┌───────────┐                            │
│  │     3     │ ─────► │     5     │ ← 栈顶 (新)                 │
│  │     2     │        │     3     │                            │
│  │     1     │        │     2     │                            │
│  └───────────┘        │     1     │                            │
│   PUSH 5 前           └───────────┘                            │
│                        PUSH 5 后                                │
│                                                                 │
│  2. POP 操作 (弹出栈顶)                                          │
│  ┌───────────┐        ┌───────────┐                            │
│  │     5     │ ─────► │     3     │ ← 栈顶                      │
│  │     3     │        │     2     │   返回值: 5                 │
│  │     2     │        │     1     │                            │
│  │     1     │        └───────────┘                            │
│  └───────────┘                                                  │
│   POP 前               POP 后                                   │
│                                                                 │
│  3. SWAP1 操作 (交换栈顶两个元素)                                 │
│  ┌───────────┐        ┌───────────┐                            │
│  │     5     │ ─────► │     3     │ ← 栈顶                      │
│  │     3     │        │     5     │                            │
│  │     2     │        │     2     │                            │
│  └───────────┘        └───────────┘                            │
│   SWAP1 前             SWAP1 后                                 │
│                                                                 │
│  4. DUP2 操作 (复制第2个元素到栈顶)                               │
│  ┌───────────┐        ┌───────────┐                            │
│  │     5     │ ─────► │     3     │ ← 栈顶 (复制的)             │
│  │     3     │        │     5     │                            │
│  │     2     │        │     3     │                            │
│  └───────────┘        │     2     │                            │
│   DUP2 前             └───────────┘                            │
│                        DUP2 后                                  │
│                                                                 │
│  5. ADD 操作 (弹出两个元素，压入它们的和)                         │
│  ┌───────────┐        ┌───────────┐                            │
│  │     5     │ ─────► │     8     │ ← 栈顶 (5+3=8)              │
│  │     3     │        │     2     │                            │
│  │     2     │        └───────────┘                            │
│  └───────────┘                                                  │
│   ADD 前               ADD 后                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.5 内存结构体

```go
// 文件: core/vm/memory.go

// Memory 实现了一个简单的内存模型
type Memory struct {
    store       []byte // 底层存储
    lastGasCost uint64 // 上次内存扩展的Gas成本
}

// NewMemory 创建新的内存实例
func NewMemory() *Memory {
    return &Memory{}
}

// Set 设置offset开始的size字节为value
func (m *Memory) Set(offset, size uint64, value []byte) {
    if size > 0 {
    if offset+size > uint64(len(m.store)) {
        m.store = append(m.store, make([]byte, offset+size-uint64(len(m.store)))...)
    }
        copy(m.store[offset:], value)
    }
}

// Set32 在offset位置设置一个32字节的值
func (m *Memory) Set32(offset uint64, val *uint256.Int) {
    if offset+32 > uint64(len(m.store)) {
        m.store = append(m.store, make([]byte, offset+32-uint64(len(m.store)))...)
    }
    val.WriteToSlice(m.store[offset : offset+32])
}

// Resize 调整内存大小
func (m *Memory) Resize(size uint64) {
    if uint64(len(m.store)) < size {
        m.store = append(m.store, make([]byte, size-uint64(len(m.store)))...)
    }
}

// GetCopy 返回offset开始size字节的副本
func (m *Memory) GetCopy(offset, size uint64) []byte {
    if size == 0 {
        return nil
    }
    cpy := make([]byte, size)
    copy(cpy, m.store[offset:offset+size])
    return cpy
}

// GetPtr 返回内存的直接指针（零拷贝）
func (m *Memory) GetPtr(offset, size uint64) []byte {
    if size == 0 {
        return nil
    }
    return m.store[offset: offset+size]
}

// Len 返回当前内存长度
func (m *Memory) Len() int {
    return len(m.store)
}
```

---

## 4. 指令集系统

### 4.1 操作码定义

```go
// 文件: core/vm/opcodes.go

// OpCode 是 EVM 操作码
type OpCode byte

const (
    // 0x00 范围 - 算术操作
    STOP       OpCode = 0x00 // 停止执行
    ADD        OpCode = 0x01 // 加法
    MUL        OpCode = 0x02 // 乘法
    SUB        OpCode = 0x03 // 减法
    DIV        OpCode = 0x04 // 无符号除法
    SDIV       OpCode = 0x05 // 有符号除法
    MOD        OpCode = 0x06 // 无符号取模
    SMOD       OpCode = 0x07 // 有符号取模
    ADDMOD     OpCode = 0x08 // 模加法: (a + b) % N
    MULMOD     OpCode = 0x09 // 模乘法: (a * b) % N
    EXP        OpCode = 0x0a // 指数
    SIGNEXTEND OpCode = 0x0b // 符号扩展
    
    // 0x10 范围 - 比较和位操作
    LT     OpCode = 0x10 // 小于
    GT     OpCode = 0x11 // 大于
    SLT    OpCode = 0x12 // 有符号小于
    SGT    OpCode = 0x13 // 有符号大于
    EQ     OpCode = 0x14 // 相等
    ISZERO OpCode = 0x15 // 是否为零
    AND    OpCode = 0x16 // 按位与
    OR     OpCode = 0x17 // 按位或
    XOR    OpCode = 0x18 // 按位异或
    NOT    OpCode = 0x19 // 按位取反
    BYTE   OpCode = 0x1a // 获取第n个字节
    SHL    OpCode = 0x1b // 左移
    SHR    OpCode = 0x1c // 逻辑右移
    SAR    OpCode = 0x1d // 算术右移
    
    // 0x20 范围 - 加密操作
    KECCAK256 OpCode = 0x20 // Keccak-256 哈希
    
    // 0x30 范围 - 环境信息
    ADDRESS        OpCode = 0x30 // 当前合约地址
    BALANCE        OpCode = 0x31 // 地址余额
    ORIGIN         OpCode = 0x32 // 交易发起者
    CALLER         OpCode = 0x33 // 直接调用者
    CALLVALUE      OpCode = 0x34 // 调用附带的 wei
    CALLDATALOAD   OpCode = 0x35 // 加载调用数据
    CALLDATASIZE   OpCode = 0x36 // 调用数据大小
    CALLDATACOPY   OpCode = 0x37 // 复制调用数据到内存
    CODESIZE       OpCode = 0x38 // 代码大小
    CODECOPY       OpCode = 0x39 // 复制代码到内存
    GASPRICE       OpCode = 0x3a // Gas 价格
    EXTCODESIZE    OpCode = 0x3b // 外部代码大小
    EXTCODECOPY    OpCode = 0x3c // 复制外部代码
    RETURNDATASIZE OpCode = 0x3d // 返回数据大小
    RETURNDATACOPY OpCode = 0x3e // 复制返回数据
    EXTCODEHASH    OpCode = 0x3f // 外部代码哈希
    
    // 0x40 范围 - 区块信息
    BLOCKHASH   OpCode = 0x40 // 区块哈希
    COINBASE    OpCode = 0x41 // 矿工地址
    TIMESTAMP   OpCode = 0x42 // 区块时间戳
    NUMBER      OpCode = 0x43 // 区块号
    DIFFICULTY  OpCode = 0x44 // 难度 (合并后为 PREVRANDAO)
    GASLIMIT    OpCode = 0x45 // 区块 Gas 限制
    CHAINID     OpCode = 0x46 // 链 ID
    SELFBALANCE OpCode = 0x47 // 自身余额
    BASEFEE     OpCode = 0x48 // 基础费用
    
    // 0x50 范围 - 栈、内存、存储、流程控制
    POP      OpCode = 0x50 // 弹出栈顶
    MLOAD    OpCode = 0x51 // 从内存加载
    MSTORE   OpCode = 0x52 // 存储到内存
    MSTORE8  OpCode = 0x53 // 存储单字节到内存
    SLOAD    OpCode = 0x54 // 从存储加载
    SSTORE   OpCode = 0x55 // 存储到存储
    JUMP     OpCode = 0x56 // 无条件跳转
    JUMPI    OpCode = 0x57 // 条件跳转
    PC       OpCode = 0x58 // 程序计数器
    MSIZE    OpCode = 0x59 // 内存大小
    GAS      OpCode = 0x5a // 剩余 Gas
    JUMPDEST OpCode = 0x5b // 跳转目标标记
    TLOAD    OpCode = 0x5c // 瞬态加载 (EIP-1153)
    TSTORE   OpCode = 0x5d // 瞬态存储 (EIP-1153)
    MCOPY    OpCode = 0x5e // 内存复制 (EIP-5656)
    
    // 0x5f-0x7f - PUSH 操作
    PUSH0  OpCode = 0x5f // 压入 0
    PUSH1  OpCode = 0x60 // 压入 1 字节
    // ... PUSH2 到 PUSH31
    PUSH32 OpCode = 0x7f // 压入 32 字节
    
    // 0x80-0x8f - DUP 操作
    DUP1  OpCode = 0x80 // 复制第 1 个栈元素
    // ... DUP2 到 DUP15
    DUP16 OpCode = 0x8f // 复制第 16 个栈元素
    
    // 0x90-0x9f - SWAP 操作
    SWAP1  OpCode = 0x90 // 交换第 1 和第 2 个栈元素
    // ... SWAP2 到 SWAP15
    SWAP16 OpCode = 0x9f // 交换第 1 和第 17 个栈元素
    
    // 0xa0-0xa4 - 日志操作
    LOG0 OpCode = 0xa0 // 0 个主题的日志
    LOG1 OpCode = 0xa1 // 1 个主题的日志
    LOG2 OpCode = 0xa2 // 2 个主题的日志
    LOG3 OpCode = 0xa3 // 3 个主题的日志
    LOG4 OpCode = 0xa4 // 4 个主题的日志
    
    // 0xf0-0xff - 系统操作
    CREATE       OpCode = 0xf0 // 创建新合约
    CALL         OpCode = 0xf1 // 调用合约
    CALLCODE     OpCode = 0xf2 // (已废弃)
    RETURN       OpCode = 0xf3 // 返回
    DELEGATECALL OpCode = 0xf4 // 委托调用
    CREATE2      OpCode = 0xf5 // 确定性创建合约
    STATICCALL   OpCode = 0xfa // 静态调用 (只读)
    REVERT       OpCode = 0xfd // 回滚
    INVALID      OpCode = 0xfe // 无效指令
    SELFDESTRUCT OpCode = 0xff // 自毁合约
)
```

**操作码分类：**

```
┌─────────────────────────────────────────────────────────────────┐
│                       EVM 操作码分类                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  算术操作 (0x00-0x0b)                                            │
│  ADD, SUB, MUL, DIV, MOD, EXP, ADDMOD, MULMOD...                │
│  • 输入: 从栈弹出操作数                                          │
│  • 输出: 结果压入栈顶                                            │
│  • 特点: 所有运算都是 256 位整数运算                              │
│                                                                 │
│  比较和位操作 (0x10-0x1d)                                        │
│  LT, GT, EQ, ISZERO, AND, OR, XOR, NOT, SHL, SHR...             │
│  • 比较: 返回 1 (true) 或 0 (false)                              │
│  • 位操作: 对 256 位整数进行位操作                                │
│                                                                 │
│  环境信息 (0x30-0x3f)                                            │
│  ADDRESS, BALANCE, ORIGIN, CALLER, CALLVALUE...                 │
│  • 获取执行环境的各种信息                                        │
│                                                                 │
│  区块信息 (0x40-0x48)                                            │
│  BLOCKHASH, COINBASE, TIMESTAMP, NUMBER, GASLIMIT...            │
│  • 获取当前区块的信息                                            │
│                                                                 │
│  栈/内存/存储操作 (0x50-0x5f)                                    │
│  POP, MLOAD, MSTORE, SLOAD, SSTORE, JUMP, JUMPI...              │
│  • 栈: POP                                                      │
│  • 内存: MLOAD, MSTORE, MSTORE8 (临时存储)                       │
│  • 存储: SLOAD, SSTORE (持久存储, Gas 最贵)                      │
│  • 跳转: JUMP, JUMPI, JUMPDEST (流程控制)                        │
│                                                                 │
│  PUSH/DUP/SWAP (0x5f-0x9f)                                      │
│  PUSH0-PUSH32, DUP1-DUP16, SWAP1-SWAP16                         │
│  • PUSH: 将立即数压入栈 (从字节码读取)                           │
│  • DUP: 复制栈中元素到栈顶                                       │
│  • SWAP: 交换栈顶元素与其他位置元素                               │
│                                                                 │
│  日志操作 (0xa0-0xa4)                                            │
│  LOG0, LOG1, LOG2, LOG3, LOG4                                   │
│  • 发出事件日志 (链下可以监听)                                    │
│                                                                 │
│  系统操作 (0xf0-0xff)                                            │
│  CREATE, CALL, RETURN, DELEGATECALL, STATICCALL...              │
│  • CREATE/CREATE2: 创建新合约                                   │
│  • CALL系列: 调用其他合约                                        │
│  • RETURN/REVERT: 结束执行并返回                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 跳转表设计

```go
// 文件: core/vm/jump_table.go

// operation 定义了单个操作码的执行信息
type operation struct {
    execute     executionFunc // 执行函数
    constantGas uint64        // 固定 Gas 消耗
    dynamicGas  gasFunc       // 动态 Gas 计算函数
    minStack    int            // 最小栈深度
    maxStack    int            // 最大栈深度
    memorySize  memorySizeFunc // 内存大小计算
}

// JumpTable 是 256 个 operation 的数组
type JumpTable [256]*operation

// 不同硬分叉版本的跳转表
func newCancunInstructionSet() JumpTable {
    instructionSet := newShanghaiInstructionSet()
    enable1153(&instructionSet) // EIP-1153: 瞬态存储
    enable4844(&instructionSet) // EIP-4844: Blob 交易
    enable5656(&instructionSet) // EIP-5656: MCOPY
    enable6780(&instructionSet) // EIP-6780: 限制 SELFDESTRUCT
    return instructionSet
}
```

**跳转表工作原理：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      跳转表工作原理                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  字节码: 0x60 0x03 0x60 0x05 0x01                               │
│          ↓    ↓    ↓    ↓    ↓                                 │
│        PUSH1  3  PUSH1  5   ADD                                 │
│                                                                 │
│  执行流程:                                                       │
│                                                                 │
│  1. 读取操作码 0x60 (PUSH1)                                      │
│     JumpTable[0x60] = {                                         │
│         execute: opPush1,                                       │
│         constantGas: 3,                                         │
│         minStack: 0,                                            │
│         maxStack: 1,                                            │
│     }                                                           │
│                                                                 │
│  2. 读取操作码 0x01 (ADD)                                        │
│     JumpTable[0x01] = {                                         │
│         execute: opAdd,                                         │
│         constantGas: 3,                                         │
│         minStack: 2,                                            │
│         maxStack: -1,                                           │
│     }                                                           │
│                                                                 │
│  查表过程: O(1) 常数时间复杂度                                    │
│  opcode = bytecode[pc]                                          │
│  op = jumpTable[opcode]                                         │
│  op.execute(...)                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 指令实现示例

```go
// 文件: core/vm/instructions.go

// opAdd 实现ADD操作码
func opAdd(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    x, y := scope.Stack.pop(), scope.Stack.peek()
    y.Add(&x, y)
    return nil, nil
}

// opSub 实现SUB操作码
func opSub(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    x, y := scope.Stack.pop(), scope.Stack.peek()
    y.Sub(&x, y)
    return nil, nil
}

// opMload 从内存加载32字节
    func opMload(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    v := scope.Stack.peek()
    offset := v.Uint64()
    v.SetBytes(scope.Memory.GetPtr(offset, 32))
    return nil, nil
}

// opMstore 存储32字节到内存
func opMstore(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    mStart, val := scope.Stack.pop(), scope.Stack.pop()
    scope.Memory.Set32(mStart.Uint64(), &val)
    return nil, nil
}

// opSload 从存储加载值
func opSload(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    loc := scope.Stack.peek()
    hash := common.Hash(loc.Bytes32())
    val := interpreter.evm.StateDB.GetState(scope.Contract.Address(), hash)
    loc.SetBytes(val.Bytes())
    return nil, nil
}

// opSstore 存储值到存储
func opSstore(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    if interpreter.readOnly {
        return nil, ErrWriteProtection
    }
    loc := scope.Stack.pop()
    val := scope.Stack.pop()
    interpreter.evm.StateDB.SetState(
        scope.Contract.Address(),
        loc.Bytes32(),
        val.Bytes32(),
    )
    return nil, nil
}

// opJump 无条件跳转
func opJump(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    pos := scope.Stack.pop()
    if !scope.Contract.validJumpdest(&pos) {
        return nil, ErrInvalidJump
    }
    *pc = pos.Uint64() - 1
    return nil, nil
}

// opJumpi 条件跳转
func opJumpi(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    pos, cond := scope.Stack.pop(), scope.Stack.pop()
    if !cond.IsZero() {
        if !scope.Contract.validJumpdest(&pos) {
            return nil, ErrInvalidJump
        }
		*pc = pos.Uint64() - 1
	}
    return nil, nil
}

// opReturn 返回数据并停止执行
    func opReturn(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    offset, size := scope.Stack.pop(), scope.Stack.pop()
    ret := scope.Memory.GetPtr(offset.Uint64(), size.Uint64())
    return ret, errStopToken
}

// opRevert 回滚状态
func opRevert(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    offset, size := scope.Stack.pop(), scope.Stack.pop()
    ret := scope.Memory.GetPtr(offset.Uint64(), size.Uint64())
    return ret, ErrExecutionReverted
}
```

---

## 5. 执行引擎

### 5.1 解释器主循环

```go
// 文件: core/vm/interpreter.go

// Run 是EVM解释器的主执行循环
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    // 增加调用深度
    in.evm.depth++
    defer func () { in.evm.depth-- }()
    
    // 调用深度限制检查 (最大 1024)
    if in.evm.depth > int(params.CallCreateDepth) {
        return nil, ErrDepth
    }
    
    // 设置只读模式 (STATICCALL)
    if readOnly && !in.readOnly {
    in.readOnly = true
    defer func () { in.readOnly = false }()
    }
    
    // 重置返回数据
    in.returnData = nil

    // 空合约直接返回
    if len(contract.Code) == 0 {
        return nil, nil
    }
    
    var (
        op          OpCode
        mem = NewMemory()
        stack = newstack()
        callContext = &ScopeContext{
            Memory:   mem,
            Stack:    stack,
            Contract: contract,
        }
        pc = uint64(0)
        cost uint64
        res  []byte
    )
    
    contract.Input = input

    // 主执行循环
    for {
        // 检查终止标志
        if in.evm.abort.Load() {
            return nil, ErrAborted
        }
        
        // 获取当前操作码
        op = contract.GetOp(pc)
        operation := in.table[op]
        
        // 验证操作有效性
        if operation == nil {
			return nil, &ErrInvalidOpCode{opcode: op}
        }
    
        // 栈验证
        if sLen := stack.len(); sLen < operation.minStack {
            return nil, &ErrStackUnderflow{}
        } else if sLen > operation.maxStack {
            return nil, &ErrStackOverflow{}
        }
    
        // 只读模式写保护检查
        if in.readOnly && operation.writes {
            return nil, ErrWriteProtection
        }
    
        // 计算 Gas 消耗
        cost = operation.constantGas
        if operation.dynamicGas != nil {
            var dynamicCost uint64
            dynamicCost, err = operation.dynamicGas(in.evm, contract, stack, mem, cost)
            if err != nil {
                return nil, err
            }
            cost += dynamicCost
        }

        // 扣除 Gas
        if !contract.UseGas(cost) {
            return nil, ErrOutOfGas
        }
    
        // 内存扩展
        if operation.memorySize != nil {
            memSize, overflow := operation.memorySize(stack)
            if overflow {
                return nil, ErrGasUintOverflow
            }
            if memSize > 0 {
                mem.Resize(memSize)
            }
        }

        // 执行操作
        res, err = operation.execute(&pc, in, callContext)
        
        if err != nil {
            break
        }
        pc++
    }
    
    if err == errStopToken {
        err = nil
    }
    return res, err
}
```

**执行流程图解：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVM 执行循环详解                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  字节码: 0x60 0x03 0x60 0x05 0x01 0x60 0x00 0x52 0xf3           │
│          PUSH1 3  PUSH1 5   ADD  PUSH1 0   MSTORE RETURN       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PC=0: PUSH1 3                                            │  │
│  │   1. 读取 op=0x60                                         │  │
│  │   2. 查表获取 PUSH1 操作                                   │  │
│  │   3. 检查栈深度                                           │  │
│  │   4. 扣除 Gas: 3                                          │  │
│  │   5. 执行: 压入 3                                         │  │
│  │   栈: [3]                                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PC=2: PUSH1 5                                            │  │
│  │   执行: 压入 5                                            │  │
│  │   栈: [3, 5]                                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PC=4: ADD                                                │  │
│  │   执行: pop 5, pop 3, push 8                             │  │
│  │   栈: [8]                                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PC=5: PUSH1 0                                            │  │
│  │   栈: [8, 0]                                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PC=7: MSTORE                                             │  │
│  │   将 8 存储到 memory[0:32]                                │  │
│  │   栈: []                                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PC=8: RETURN                                             │  │
│  │   返回 memory[0:32]                                       │  │
│  │   结果: 0x0000...0008                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 EVM调用入口

```go
// 文件: core/vm/evm.go

// NewEVM 创建新的 EVM 实例
func NewEVM(blockCtx BlockContext, txCtx TxContext, statedb StateDB, chainConfig *params.ChainConfig, config Config) *EVM {
    evm := &EVM{
        Context:     blockCtx,
        TxContext:   txCtx,
        StateDB:     statedb,
        chainConfig: chainConfig,
        chainRules:  chainConfig.Rules(blockCtx.BlockNumber, blockCtx.Random != nil, blockCtx.Time),
        Config:      config,
    }
    evm.interpreter = NewEVMInterpreter(evm)
    return evm
}

// Call 执行合约调用
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *uint256.Int) (ret []byte, leftOverGas uint64, err error) {
    // 深度检查
    if evm.depth > int(params.CallCreateDepth) {
        return nil, gas, ErrDepth
    }
    
    // 余额检查
    if !value.IsZero() {
        if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value.ToBig()) {
            return nil, gas, ErrInsufficientBalance
        }
    }

    snapshot := evm.StateDB.Snapshot()
    
    // 检查预编译合约
    if p, isPrecompile := evm.precompile(addr); isPrecompile {
        ret, gas, err = RunPrecompiledContract(p, input, gas)
    } else {
        // 执行转账
        evm.Context.Transfer(evm.StateDB, caller.Address(), addr, value.ToBig())
        
        // 获取合约代码并执行
        code := evm.StateDB.GetCode(addr)
        if len(code) == 0 {
            ret, err = nil, nil
        } else {
            contract := NewContract(caller, AccountRef(addr), value, gas)
            contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), code)
            ret, err = evm.interpreter.Run(contract, input, false)
            gas = contract.Gas
        }
    }

    // 错误回滚
    if err != nil {
        evm.StateDB.RevertToSnapshot(snapshot)
        if err != ErrExecutionReverted {
            gas = 0
        }
    }
    return ret, gas, err
}

// DelegateCall 委托调用
func (evm *EVM) DelegateCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
    // 在调用者上下文中执行目标代码
    // msg.sender 和 address(this) 保持不变
    // ...
}

// StaticCall 静态调用 (只读)
func (evm *EVM) StaticCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
    // 禁止任何状态修改操作
    // readOnly = true
    // ...
}

// Create 创建合约
func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *uint256.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
    // 地址计算: keccak256(rlp([sender, nonce]))[12:]
    contractAddr = crypto.CreateAddress(caller.Address(), evm.StateDB.GetNonce(caller.Address()))
    return evm.create(caller, &codeAndHash{code: code}, gas, value, contractAddr, CREATE)
}

// Create2 确定性创建合约
func (evm *EVM) Create2(caller ContractRef, code []byte, gas uint64, endowment *uint256.Int, salt *uint256.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
    // 地址计算: keccak256(0xff ++ sender ++ salt ++ keccak256(code))[12:]
    contractAddr = crypto.CreateAddress2(caller.Address(), salt.Bytes32(), codeAndHash.Hash().Bytes())
    return evm.create(caller, codeAndHash, gas, endowment, contractAddr, CREATE2)
}
```

**不同调用类型对比：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    不同调用类型对比                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CALL - 普通调用                                                │
│  • msg.sender = 调用者合约                                       │
│  • address(this) = 被调用合约                                    │
│  • 存储操作影响被调用合约                                        │
│  • ETH 转移到被调用合约                                          │
│                                                                 │
│  DELEGATECALL - 委托调用                                        │
│  • msg.sender = 原调用者 (不变!)                                 │
│  • address(this) = 调用者合约 (不变!)                            │
│  • 存储操作影响调用者合约!                                       │
│  • 常用于代理模式和库调用                                        │
│                                                                 │
│  STATICCALL - 静态调用 (只读)                                   │
│  • 与 CALL 类似                                                  │
│  • 禁止 SSTORE, CREATE, SELFDESTRUCT, LOG 等                    │
│  • 禁止 ETH 转账                                                 │
│  • 用于 view/pure 函数调用                                       │
│                                                                 │
│  CREATE vs CREATE2                                              │
│  • CREATE: 地址依赖 nonce，不可预测                              │
│  • CREATE2: 地址可预测，适用于工厂模式                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Gas机制

### 6.1 Gas基本概念

```
┌─────────────────────────────────────────────────────────────────┐
│                       Gas 机制概述                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  为什么需要Gas?                                                 │
│  1. 防止无限循环(停机问题)                                      │
│  2. 防止DoS攻击                                                │
│  3. 资源计量和公平分配                                           │
│  4. 激励矿工/验证者                                              │
│                                                                 │
│  Gas 费用计算:                                                   │
│  总费用 = Gas消耗量 × Gas价格                                  │
│                                                                 │
│  EIP-1559 后:                                                   │
│  总费用 = Gas × (baseFee + priorityFee)                         │
│  • baseFee: 协议动态调整，被销毁                                 │
│  • priorityFee: 给验证者的小费                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Gas消耗常量

```go
// 文件: core/vm/gas.go

const (
    GasQuickStep   uint64 = 2 // 最简单的操作
    GasFastestStep uint64 = 3    // ADD, SUB 等
    GasFastStep    uint64 = 5    // 较快操作
    GasMidStep     uint64 = 8    // 中等操作
    GasSlowStep    uint64 = 10   // 较慢操作
    GasExtStep     uint64 = 20   // 外部操作
)

// 各操作的 Gas 消耗
const (
    GasVeryLow      uint64 = 3 // ADD, SUB, NOT, LT, GT
    GasLow          uint64 = 5 // MUL, DIV
    GasMid          uint64 = 8 // ADDMOD, MULMOD
    GasHigh         uint64 = 10  // JUMPI
    GasWarmAccess   uint64 = 100 // warm 访问
    GasColdAccess   uint64 = 2600  // cold 访问
    GasSset         uint64 = 20000 // SSTORE: 0 → 非0
    GasSreset       uint64 = 2900  // SSTORE: 非0 → 非0
    GasCreate       uint64 = 32000 // CREATE
    GasCallValue    uint64 = 9000 // CALL 带转账
    GasCallStipend  uint64 = 2300 // 转账赠送的 Gas
    GasNewAccount   uint64 = 25000 // 创建新账户
    GasMemory       uint64 = 3     // 内存每字
    GasKeccak256    uint64 = 30 // KECCAK256 基础
)
```

**常见操作Gas消耗：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    常见操作 Gas 消耗                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  算术操作 (便宜):                                                 │
│  ADD, SUB, LT, GT, EQ, ISZERO, AND, OR, XOR    = 3 Gas          │
│  MUL, DIV, MOD                                  = 5 Gas         │
│  EXP                                   = 10 + 50/byte Gas       │
│                                                                 │
│  栈操作 (便宜):                                                  │
│  POP                                            = 2 Gas         │
│  PUSH1-PUSH32, DUP1-DUP16, SWAP1-SWAP16        = 3 Gas          │
│                                                                 │
│  内存操作 (中等):                                                 │
│  MLOAD, MSTORE                          = 3 + 扩展成本           │
│  内存扩展成本 = 3 * words + words² / 512                          │
│                                                                 │
│  存储操作 (很贵!):                                                │
│  SLOAD (cold)                                = 2100 Gas         │
│  SLOAD (warm)                                = 100 Gas          │
│  SSTORE: 0 → 非0                             = 20000 Gas         │
│  SSTORE: 非0 → 非0                           = 2900 Gas          │
│  SSTORE: 非0 → 0                   = 2900 Gas (返还 4800)        │
│                                                                 │
│  调用操作 (贵):                                                   │
│  CALL (warm)                                 = 100 Gas          │
│  CALL (cold)                                 = 2600 Gas         │
│  CALL + 转账                                 + 9000 Gas          │
│  CALL + 创建新账户                           + 25000 Gas          │
│                                                                 │
│  创建操作 (最贵):                                                 │
│  CREATE/CREATE2                              = 32000 Gas        │
│  + 代码存储成本                              = 200/byte           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 动态Gas计算

```go
// 内存扩展的 Gas 成本
func memoryGasCost(mem *Memory, newMemSize uint64) (uint64, error) {
    if newMemSize == 0 {
        return 0, nil
    }
    
    newMemSizeWords := toWordSize(newMemSize)
    newMemSize = newMemSizeWords * 32
    
    if newMemSize > uint64(len(mem.store)) {
        // cost = 3 * words + words² / 512
        square := newMemSizeWords * newMemSizeWords
        linCoef := newMemSizeWords * params.MemoryGas
        quadCoef := square / params.QuadCoeffDiv
        newTotalFee := linCoef + quadCoef
        
        fee := newTotalFee - mem.lastGasCost
        mem.lastGasCost = newTotalFee
        return fee, nil
    }
    return 0, nil
}

// 63/64 规则 (EIP-150)
func callGas(isEip150 bool, availableGas, base uint64, callCost *uint256.Int) (uint64, error) {
    if isEip150 {
        // 最多传递 63/64 的可用 Gas
        availableGas = availableGas - base
        gas := availableGas - availableGas/64
        
        if !callCost.IsUint64() || gas < callCost.Uint64() {
            return gas, nil
        }
        return callCost.Uint64(), nil
    }
    return callCost.Uint64(), nil
}
```

**EIP-2929访问列表：**

```
┌─────────────────────────────────────────────────────────────────┐
│                   EIP-2929 访问列表                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  首次访问某地址/存储槽 = Cold (贵)                                 │
│  再次访问同一位置     = Warm (便宜)                                │
│                                                                 │
│  访问列表在每笔交易开始时初始化:                                     │
│  • tx.to 地址 (warm)                                             │
│  • tx.from 地址 (warm)                                           │
│  • 预编译合约地址 (warm)                                          │
│  • 交易指定的 accessList 中的地址/槽 (warm)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 内存模型

### 7.1 内存布局

```
┌─────────────────────────────────────────────────────────────────┐
│                        EVM 内存模型                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EVM 内存是一个可动态扩展的字节数组:                                 │
│                                                                 │
│  offset:0    32   64   96   128  160  192  224  256  ...        │
│         ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬───       │
│  memory:│    │    │    │    │    │    │    │    │    │          │
│         └────┴────┴────┴────┴────┴────┴────┴────┴────┴───       │
│                                                                 │
│  特点:                                                          │
│  1. 按字节寻址                                                   │
│  2. 读写单位: 按字(32字节)或按字节                                  │
│  3. 自动扩展: 访问超出当前大小时自动扩展                              │
│  4. 扩展成本: 二次方增长                                           │
│  5. 零初始化: 新扩展的内存全为 0                                    │
│  6. 易失性: 每次合约调用结束后内存被清空                              │
│                                                                 │
│  Solidity 内存布局惯例:                                           │
│  0x00 - 0x3f (64字节): 暂存空间 (用于哈希计算)                      │
│  0x40 - 0x5f (32字节): 空闲内存指针                                │
│  0x60 - 0x7f (32字节): 零槽                                      │
│  0x80 - ...         : 空闲内存区域                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. 存储模型

### 8.1 存储 vs 内存

```
┌─────────────────────────────────────────────────────────────────┐
│                      存储 vs 内存                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  特性       │   内存 (Memory)   │   存储 (Storage)                │
│  ──────────┼───────────────────┼────────────────────            │
│  生命周期   │   调用期间          │   永久                         │
│  大小      │   动态扩展          │   2^256 个槽位                  │
│  寻址      │   字节偏移          │   256位键                       │
│  数据大小   │   任意字节          │   256位值                      │
│  Gas成本   │   便宜             │   非常贵                        │
│  初始值     │   0               │   0                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Solidity存储布局

```
┌─────────────────────────────────────────────────────────────────┐
│                  Solidity 存储布局规则                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  contract Example {                                             │
│      uint256 a;           // slot 0                             │
│      uint256 b;           // slot 1                             │
│      uint128 c;           // slot 2 (低128位)                   │
│      uint128 d;           // slot 2 (高128位) - 打包!            │
│      mapping(address => uint) balances;  // slot 3              │
│      uint256[] arr;       // slot 4                             │
│  }                                                              │
│                                                                 │
│  Mapping 存储位置: keccak256(key . slot)                         │
│  动态数组: slot 存储长度, 元素在 keccak256(slot) + index            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 SSTORE Gas成本矩阵

```
┌─────────────────────────────────────────────────────────────────┐
│                   SSTORE Gas 成本矩阵                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  场景                            │ Gas 成本    │ 返还             │
│  ───────────────────────────────┼────────────┼───────────       │
│  current == new (无变化)         │    100     │     -            │
│                                 │            │                  │
│  original == current (首次修改): │            │                  │
│    original == 0 (创建)          │   20000    │     -            │
│    original != 0, new == 0 (删除)│   2900    │  +4800            │
│    original != 0, new != 0 (改)  │   2900    │     -            │
│                                 │            │                  │
│  original != current (已修改):   │    100     │   可能有          │
│                                                                 │
│  + EIP-2929 Cold 访问: +2100 Gas                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.4 瞬态存储 (EIP-1153)

```
┌─────────────────────────────────────────────────────────────────┐
│                   瞬态存储 (EIP-1153)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  特性         │  持久存储 (SSTORE) │ 瞬态存储 (TSTORE)             │
│  ────────────┼───────────────────┼───────────────────           │
│  生命周期     │  永久              │  交易期间                     │
│  Gas 成本    │  非常高 (20000)    │  便宜 (100)                   │
│  状态变化     │  会影响状态根       │  不影响状态根                  │
│  跨交易      │  保持              │  交易结束后清空                 │
│                                                                 │
│  使用场景:                                                       │
│  1. 重入锁 (比 SSTORE 便宜99%)                                    │
│  2. 跨合约通信 (同一交易内)                                        │
│  3. 闪电贷回调验证                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. 合约调用机制

### 9.1 CALL 完整执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    CALL 完整执行流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 交易处理器 (StateProcessor)                                  │
│     ├── 验证签名                                                 │
│     ├── 检查nonce                                                │
│     ├── 扣除固有Gas                                              │
│     └── 调用EVM                                                 │
│                              ↓                                  │
│  2. EVM.Call()                                                  │
│     ├── 创建状态快照                                              │
│     ├── 转账                                                     │
│     ├── 加载合约代码                                              │
│     └── 创建Contract对象                                         │
│                              ↓                                  │
│  3. Interpreter.Run()                                           │
│     ├── 主循环执行字节码                                          │
│     ├── 遇到CALL指令时递归调用                                     │
│     └── RETURN/REVERT结束执行                                    │
│                              ↓                                  │
│  4. 返回结果                                                     │
│     ├── 成功: 更新状态                                            │
│     └── 失败: 回滚到快照                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 CREATE vs CREATE2

```
┌─────────────────────────────────────────────────────────────────┐
│                 CREATE vs CREATE2 地址计算                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CREATE:                                                        │
│  address = keccak256(rlp([sender, nonce]))[12:]                 │
│  • 依赖 nonce，地址不可预测                                        │
│                                                                 │
│  CREATE2:                                                       │
│  address = keccak256(0xff ++ sender ++ salt ++ codehash)[12:]   │
│  • 地址完全确定性，可提前计算                                       │
│  • 适用于工厂模式、反事实部署                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. 预编译合约

### 10.1 预编译合约列表

| 地址   | 名称              | 功能            | Gas            |
|------|-----------------|---------------|----------------|
| 0x01 | ecRecover       | ECDSA 签名恢复    | 3000           |
| 0x02 | sha256          | SHA-256 哈希    | 60 + 12/word   |
| 0x03 | ripemd160       | RIPEMD-160 哈希 | 600 + 120/word |
| 0x04 | identity        | 数据复制          | 15 + 3/word    |
| 0x05 | modexp          | 模幂运算          | 动态计算           |
| 0x06 | bn256Add        | BN256 点加法     | 150            |
| 0x07 | bn256ScalarMul  | BN256 标量乘法    | 6000           |
| 0x08 | bn256Pairing    | BN256 配对检查    | 动态计算           |
| 0x09 | blake2f         | BLAKE2b 压缩    | 每轮 1           |
| 0x0a | pointEvaluation | KZG 点评估       | 50000          |

### 10.2 预编译合约接口

```go
// PrecompiledContract 是预编译合约的接口
type PrecompiledContract interface {
    RequiredGas(input []byte) uint64
    Run(input []byte) ([]byte, error)
}

// ecrecover 实现示例
type ecrecover struct{}

func (c *ecrecover) RequiredGas(input []byte) uint64 {
    return params.EcrecoverGas // 3000 Gas
}

func (c *ecrecover) Run(input []byte) ([]byte, error) {
    // 解析输入: msgHash || v || r || s
    // 恢复公钥并返回地址
}
```

---

## 11. 实战代码示例

### 11.1 模拟EVM执行

```go
package main

import (
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/rawdb"
	"github.com/ethereum/go-ethereum/core/state"
	"github.com/ethereum/go-ethereum/core/vm"
	"github.com/ethereum/go-ethereum/params"
)

func main() {
	// 1. 创建内存数据库和状态
	db := rawdb.NewMemoryDatabase()
	stateDB, _ := state.New(common.Hash{}, state.NewDatabase(db), nil)

	// 2. 创建测试账户
	sender := common.HexToAddress("0x1111111111111111111111111111111111111111")
	contractAddr := common.HexToAddress("0x2222222222222222222222222222222222222222")

	stateDB.AddBalance(sender, big.NewInt(1e18))

	// 3. 部署简单合约
	// 功能: CALLDATALOAD → SSTORE
	code := []byte{
		0x60, 0x00, // PUSH1 0x00
		0x35,       // CALLDATALOAD
		0x60, 0x00, // PUSH1 0x00
		0x55, // SSTORE
		0x00, // STOP
	}
	stateDB.SetCode(contractAddr, code)

	// 4. 创建 EVM
	blockContext := vm.BlockContext{
		CanTransfer: func(db vm.StateDB, addr common.Address, amount *big.Int) bool {
			return db.GetBalance(addr).Cmp(amount) >= 0
		},
		Transfer: func(db vm.StateDB, sender, recipient common.Address, amount *big.Int) {
			db.SubBalance(sender, amount)
			db.AddBalance(recipient, amount)
		},
		GetHash:     func(n uint64) common.Hash { return common.Hash{} },
		BlockNumber: big.NewInt(18000000),
		Time:        1700000000,
		BaseFee:     big.NewInt(1000000000),
	}

	txContext := vm.TxContext{
		Origin:   sender,
		GasPrice: big.NewInt(1000000000),
	}

	evm := vm.NewEVM(blockContext, txContext, stateDB, params.MainnetChainConfig, vm.Config{})

	// 5. 执行调用
	input := common.LeftPadBytes(big.NewInt(42).Bytes(), 32)
	ret, leftOverGas, err := evm.Call(
		vm.AccountRef(sender),
		contractAddr,
		input,
		100000,
		big.NewInt(0),
	)

	// 6. 输出结果
	fmt.Printf("返回数据: %x\n", ret)
	fmt.Printf("消耗 Gas: %d\n", 100000-leftOverGas)
	fmt.Printf("错误: %v\n", err)

	// 7. 验证存储
	storageValue := stateDB.GetState(contractAddr, common.Hash{})
	fmt.Printf("存储槽 0: %s\n", new(big.Int).SetBytes(storageValue.Bytes()))
}
```

### 11.2 字节码分析

```go
package main

import (
	"encoding/hex"
	"fmt"
	"github.com/ethereum/go-ethereum/core/vm"
)

func main() {
	bytecodeHex := "6080604052348015..." // 合约字节码
	bytecode, _ := hex.DecodeString(bytecodeHex)

	fmt.Println("=== 字节码分析 ===")

	pc := 0
	for pc < len(bytecode) {
		op := vm.OpCode(bytecode[pc])
		opName := op.String()

		if op >= vm.PUSH1 && op <= vm.PUSH32 {
			size := int(op - vm.PUSH1 + 1)
			if pc+size < len(bytecode) {
				data := bytecode[pc+1 : pc+1+size]
				fmt.Printf("%04x: %-12s %x\n", pc, opName, data)
				pc += size
			}
		} else {
			fmt.Printf("%04x: %s\n", pc, opName)
		}
		pc++
	}
}
```

---

## 总结

### 核心要点

1. **栈式架构**: 所有计算基于 1024 深度、256位元素的栈
2. **Gas 机制**: 防止无限循环，实现资源计量
3. **三层存储**: 栈(临时) → 内存(调用期间) → 存储(永久)
4. **确定性执行**: 相同输入必须产生相同输出
5. **隔离性**: 沙箱环境，合约间通过消息调用交互

### 关键文件

| 文件              | 功能           |
|-----------------|--------------|
| evm.go          | EVM 主结构和调用入口 |
| interpreter.go  | 解释器主循环       |
| opcodes.go      | 操作码定义        |
| instructions.go | 指令实现         |
| jump_table.go   | 操作码跳转表       |
| gas_table.go    | Gas 计算       |
| memory.go       | 内存实现         |
| stack.go        | 栈实现          |
| contract.go     | 合约抽象         |
| contracts.go    | 预编译合约        |

### 学习路径

1. 理解基础概念 (栈、内存、存储)
2. 阅读操作码定义 (opcodes.go)
3. 理解执行循环 (interpreter.go)
4. 学习指令实现 (instructions.go)
5. 掌握Gas机制 (gas_table.go)
6. 理解调用机制 (evm.go)
7. 实践代码调试和追踪
