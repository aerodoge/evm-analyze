# Go-Ethereum EVM 高级篇

> 本文档是 EVM 设计与实现的高级篇，涵盖状态管理、调试追踪、日志事件、异常处理、硬分叉演进、性能优化等深入主题。

## 目录

- [12. 状态管理与快照回滚](#12-状态管理与快照回滚)
- [13. 日志与事件系统](#13-日志与事件系统)
- [14. 异常处理机制](#14-异常处理机制)
- [15. EVM 调试与追踪](#15-evm-调试与追踪)
- [16. 硬分叉与 EIP 演进](#16-硬分叉与-eip-演进)
- [17. 性能优化技术](#17-性能优化技术)
- [18. 安全考虑](#18-安全考虑)
- [19. EVM 与其他组件交互](#19-evm-与其他组件交互)
- [20. 完整实战案例](#20-完整实战案例)

---

## 12. 状态管理与快照回滚

### 12.1 StateDB 接口

```go
// 文件: core/vm/interface.go

// StateDB 是EVM访问以太坊状态的接口
type StateDB interface {
    // 账户操作
    CreateAccount(common.Address)
    SubBalance(common.Address, *big.Int)
    AddBalance(common.Address, *big.Int)
    GetBalance(common.Address) *big.Int
    GetNonce(common.Address) uint64
    SetNonce(common.Address, uint64)
    
    // 代码操作
    GetCodeHash(common.Address) common.Hash
    GetCode(common.Address) []byte
    SetCode(common.Address, []byte)
    GetCodeSize(common.Address) int
    
    // 存储操作
    GetState(common.Address, common.Hash) common.Hash
    SetState(common.Address, common.Hash, common.Hash)
    GetCommittedState(common.Address, common.Hash) common.Hash
    
    // 瞬态存储 (EIP-1153)
    GetTransientState(addr common.Address, key common.Hash) common.Hash
    SetTransientState(addr common.Address, key, value common.Hash)
    
    // 快照和回滚
    Snapshot() int
    RevertToSnapshot(int)
    
    // 返还
    AddRefund(uint64)
    SubRefund(uint64)
    GetRefund() uint64
    
    // 日志
    AddLog(*types.Log)
    GetLogs(hash common.Hash, blockNumber uint64, blockHash common.Hash) []*types.Log
    
    // 访问列表 (EIP-2929)
    AddAddressToAccessList(addr common.Address)
    AddSlotToAccessList(addr common.Address, slot common.Hash)
    SlotInAccessList(addr common.Address, slot common.Hash) (addressOk bool, slotOk bool)
    
    // 自毁
    SelfDestruct(common.Address)
    HasSelfDestructed(common.Address) bool
    Selfdestruct6780(common.Address)
    
    // 账户状态
    Exist(common.Address) bool
    Empty(common.Address) bool
    
    // 预像记录
    AddPreimage(common.Hash, []byte)
}
```

### 12.2 快照机制

```go
// 文件: core/state/statedb.go

// StateDB 结构体
type StateDB struct {
    db         Database // 底层数据库
    trie       Trie     // 状态 trie
    
    // 状态对象缓存
    stateObjects        map[common.Address]*stateObject
    stateObjectsDirty   map[common.Address]struct{}
    stateObjectsDestruct map[common.Address]struct{}
    
    // 日志
    logs    map[common.Hash][]*types.Log
    logSize uint
    
    // 返还
    refund uint64
    
    // 访问列表
    accessList *accessList
    
    // 瞬态存储
    transientStorage transientStorage
    
    // 快照相关
    validRevisions []revision
    nextRevisionId int
    
    // journal 记录所有状态变化，用于回滚
    journal *journal
}

// revision 表示一个快照点
type revision struct {
    id           int
    journalIndex int
}

// Snapshot 创建状态快照，返回快照ID
func (s *StateDB) Snapshot() int {
    id := s.nextRevisionId
    s.nextRevisionId++
    s.validRevisions = append(s.validRevisions, revision{id, s.journal.length()})
    return id
}

// RevertToSnapshot 回滚到指定快照
func (s *StateDB) RevertToSnapshot(revid int) {
    // 找到对应的快照
    idx := sort.Search(len(s.validRevisions), func (i int) bool {
        return s.validRevisions[i].id >= revid
    })
    if idx == len(s.validRevisions) || s.validRevisions[idx].id != revid {
        panic(fmt.Errorf("revision id %v cannot be reverted", revid))
    }
    snapshot := s.validRevisions[idx]
    
    // 回滚 journal 中的所有变更
    s.journal.revert(s, snapshot.journalIndex)
    
    // 移除该快照之后的所有快照
    s.validRevisions = s.validRevisions[:idx]
}
```

### 12.3 Journal日志系统

```go
// 文件: core/state/journal.go

// journal 记录状态变更，支持回滚
type journal struct {
    entries []journalEntry // 变更条目
    dirties map[common.Address]int // 脏账户计数
}

// journalEntry 是单个状态变更的接口
type journalEntry interface {
    revert(*StateDB) // 回滚此变更
    dirtied() *common.Address // 返回被修改的地址
}

// 各种变更类型
type (
    // createObjectChange 记录账户创建
    createObjectChange struct {
        account *common.Address
    }
    
    // resetObjectChange 记录账户重置
    resetObjectChange struct {
        prev         *stateObject
        prevdestruct bool
    }
    
    // selfDestructChange 记录自毁
    selfDestructChange struct {
        account     *common.Address
        prev        bool
        prevbalance *big.Int
    }
    
    // balanceChange 记录余额变更
    balanceChange struct {
        account *common.Address
        prev    *big.Int
    }
    
    // nonceChange 记录 nonce 变更
    nonceChange struct {
        account *common.Address
        prev    uint64
    }
    
    // storageChange 记录存储变更
    storageChange struct {
        account       *common.Address
        key, prevalue common.Hash
    }
    
    // codeChange 记录代码变更
    codeChange struct {
        account            *common.Address
        prevcode, prevhash []byte
    }
    
    // refundChange 记录返还变更
    refundChange struct {
        prev uint64
    }
    
    // addLogChange 记录日志添加
    addLogChange struct {
        txhash common.Hash
    }
    
    // accessListAddAccountChange 记录访问列表变更
    accessListAddAccountChange struct {
        address common.Address
    }
    
    // accessListAddSlotChange 记录槽位访问列表变更
    accessListAddSlotChange struct {
        address common.Address
        slot    common.Hash
    }
    
    // transientStorageChange 记录瞬态存储变更
    transientStorageChange struct {
        account       *common.Address
        key, prevalue common.Hash
    }
)

// revert 方法示例
func (ch balanceChange) revert(s *StateDB) {
    s.getStateObject(*ch.account).setBalance(ch.prev)
}

func (ch storageChange) revert(s *StateDB) {
    s.getStateObject(*ch.account).setState(ch.key, ch.prevalue)
}

func (ch selfDestructChange) revert(s *StateDB) {
    obj := s.getStateObject(*ch.account)
    if obj != nil {
        obj.selfDestructed = ch.prev
        obj.setBalance(ch.prevbalance)
    }
}
```

**快照回滚机制图解：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      快照回滚机制                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  交易执行过程中的状态变化:                                          │
│                                                                 │
│  初始状态                                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  账户 A: balance=100, nonce=5                           │     │
│  │  账户 B: balance=50                                     │     │
│  │  合约 C: storage[0]=0                                   │     │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                    Snapshot() → id=0                            │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  操作 1: A 转账 30 给 B                                     │  │
│  │  journal 记录: balanceChange{A, 100}, balanceChange{B, 50} │  │
│  │                                                           │   │
│  │  账户 A: balance=70                                        │  │
│  │  账户 B: balance=80                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                    Snapshot() → id=1                            │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  操作 2: 调用合约 C, SSTORE(0, 42)                        │   │
│  │  journal 记录: storageChange{C, 0, 0}                   │   │
│  │                                                         │   │
│  │  合约 C: storage[0]=42                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                 │
│                              ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  操作 3: 合约执行失败! REVERT                              │   │
│  │                                                         │   │
│  │  RevertToSnapshot(1)                                    │   │
│  │  回滚 journal 从当前位置到快照 1 的所有条目                  │   │
│  │  → 恢复 storage[0]=0                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                 │
│                              ▼                                 │
│  回滚后状态                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  账户 A: balance=70  (保留，因为在快照1之前)                │    │
│  │  账户 B: balance=80  (保留)                              │    │
│  │  合约 C: storage[0]=0 (已回滚)                            │   │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 12.4 状态对象 (stateObject)

```go
// 文件: core/state/state_object.go

// stateObject 代表一个以太坊账户
type stateObject struct {
    address  common.Address
    addrHash common.Hash // 地址的 hash，用于 trie 键
    
    // 账户数据
    data types.StateAccount
    
    // 数据库引用
    db *StateDB
    
    // 写缓存
    trie Trie // 存储 trie
    code Code // 合约代码
    
    // 缓存标志
    dirtyStorage  Storage // 脏存储条目
    pendingStorage Storage // 待写入的存储
    originStorage Storage  // 原始存储（交易开始时）
    
    // 状态标志
    dirtyCode     bool  // 代码已修改
    selfDestructed bool // 已自毁
    deleted       bool // 已删除
}

// types.StateAccount 账户数据结构
type StateAccount struct {
    Nonce    uint64   // 交易计数
    Balance  *big.Int // 余额
    Root     common.Hash // 存储 trie 根哈希
    CodeHash []byte      // 代码哈希
}

// GetState 获取存储值
func (s *stateObject) GetState(db Database, key common.Hash) common.Hash {
    // 1. 首先检查脏缓存
    if value, dirty := s.dirtyStorage[key]; dirty {
        return value
    }
    // 2. 然后检查待处理缓存
    if value, pending := s.pendingStorage[key]; pending {
        return value
    }
    // 3. 最后从trie加载
    return s.GetCommittedState(db, key)
}

// SetState 设置存储值
func (s *stateObject) SetState(db Database, key, value common.Hash) {
    // 获取之前的值用于journal
    prev := s.GetState(db, key)
    if prev == value {
        return
    }
    // 记录变更到journal
    s.db.journal.append(storageChange{
        account:   &s.address,
        key:       key,
        prevalue:  prev,
    })
    s.setState(key, value)
}

func (s *stateObject) setState(key, value common.Hash) {
    s.dirtyStorage[key] = value
}
```

---

## 13. 日志与事件系统

### 13.1 日志结构

```go
// 文件: core/types/log.go

// Log 代表合约日志事件
type Log struct {
    // 合约地址
    Address common.Address `json:"address"`
    
    // 主题列表 (最多4个)
    // Topics[0] 通常是事件签名的 keccak256 哈希
    Topics []common.Hash `json:"topics"`
    
    // 数据 (非索引参数)
    Data []byte `json:"data"`
    
    // 以下字段由区块链填充
    BlockNumber uint64      `json:"blockNumber"`
    TxHash      common.Hash `json:"transactionHash"`
    TxIndex     uint        `json:"transactionIndex"`
    BlockHash   common.Hash `json:"blockHash"`
    Index       uint        `json:"logIndex"`
    
    // Removed 标志在链重组时设为 true
    Removed bool `json:"removed"`
}
```

### 13.2 LOG操作码实现

```go
// 文件: core/vm/instructions.go

// makeLog 创建 LOG0-LOG4 的执行函数
func makeLog(size int) executionFunc {
    return func (pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
        // 只读模式不能记录日志
        if interpreter.readOnly {
            return nil, ErrWriteProtection
        }
        
        // 从栈获取参数
        var (
            mStart = scope.Stack.pop() // 内存起始位置
            mSize = scope.Stack.pop() // 数据大小
        )
        
        // 获取主题
        topics := make([]common.Hash, size)
        for i := 0; i < size; i++ {
            addr := scope.Stack.pop()
            topics[i] = addr.Bytes32()
        }
        
        // 从内存获取数据
        data := scope.Memory.GetCopy(mStart.Uint64(), mSize.Uint64())
        
        // 创建日志并添加到 StateDB
        interpreter.evm.StateDB.AddLog(&types.Log{
            Address: scope.Contract.Address(),
            Topics:  topics,
            Data:    data,
            // BlockNumber, TxHash 等字段后续填充
        })
        
        return nil, nil
    }
}

// 初始化时注册
func init() {
    jumpTable[LOG0] = &operation{execute: makeLog(0), ...}
    jumpTable[LOG1] = &operation{execute: makeLog(1), ...}
    jumpTable[LOG2] = &operation{execute: makeLog(2), ...}
    jumpTable[LOG3] = &operation{execute: makeLog(3), ...}
    jumpTable[LOG4] = &operation{execute: makeLog(4), ...}
}
```

**日志系统图解：**

```
┌─────────────────────────────────────────────────────────────────┐
│                       日志系统详解                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Solidity 事件定义:                                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  event Transfer(                                         │   │
│  │      address indexed from,    // → topic[1]              │   │
│  │      address indexed to,      // → topic[2]              │   │
│  │      uint256 value            // → data                  │   │
│  │  );                                                      │   │
│  │                                                          │   │
│  │  emit Transfer(msg.sender, recipient, amount);           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  生成的日志结构:                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Log {                                                   │  │
│  │    Address: 0x合约地址                                    │  │
│  │    Topics: [                                             │  │
│  │      0: keccak256("Transfer(address,address,uint256)")   │  │
│  │         = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4 │  │
│  │           a11628f55a4df523b3ef                           │  │
│  │      1: 0x000...sender  (from, indexed)                  │  │
│  │      2: 0x000...recipient (to, indexed)                  │  │
│  │    ]                                                     │  │
│  │    Data: 0x000...amount (value, non-indexed)             │  │
│  │  }                                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  索引 vs 非索引参数:                                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  indexed (Topics):                                       │  │
│  │  • 最多 3 个 (+ 事件签名 = 4 个 topic)                      │  │
│  │  • 可用于过滤/搜索                                         │  │
│  │  • 每个固定 32 字节                                        │  │
│  │  • 值类型直接存储，引用类型存储哈希                           │  │
│  │                                                          │  │
│  │  non-indexed (Data):                                     │  │
│  │  • 数量不限                                               │  │
│  │  • 不能用于过滤                                            │  │
│  │  • ABI 编码，可变长度                                      │  │
│  │  • 成本更低                                               │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  LOG 操作码的 Gas 成本:                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  Gas = 375 (基础)                                         │  │
│  │      + 375 × topic数量                                    │  │
│  │      + 8 × data字节数                                     │   │
│  │      + 内存扩展成本                                        │   │
│  │                                                          │   │
│  │  LOG0: 375 + 8×len(data)                                 │   │
│  │  LOG1: 750 + 8×len(data)                                 │   │
│  │  LOG2: 1125 + 8×len(data)                                │   │
│  │  LOG3: 1500 + 8×len(data)                                │   │
│  │  LOG4: 1875 + 8×len(data)                                │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  日志存储位置:                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  • 日志不存储在状态 trie 中                                 │   │
│  │  • 存储在区块的 receipts 中                                │   │
│  │  • 形成 receipts trie (receiptsRoot)                     │   │
│  │  • 使用 Bloom Filter 加速日志过滤                           │  │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 13.3 Bloom Filter

```go
// 文件: core/types/bloom9.go

// Bloom 是2048位的布隆过滤器
type Bloom [BloomByteLength]byte

const (
    BloomByteLength = 256 // 256 字节 = 2048 位
    BloomBitLength = 8 * BloomByteLength
)

// CreateBloom 从一组日志创建布隆过滤器
func CreateBloom(receipts Receipts) Bloom {
    var bin Bloom
    for _, receipt := range receipts {
        for _, log := range receipt.Logs {
            bin.add(log.Address.Bytes())
            for _, topic := range log.Topics {
                bin.add(topic.Bytes())
            }
        }
    }
    return bin
}

// add 将数据添加到布隆过滤器
func (b *Bloom) add(d []byte) {
    // 计算 3 个位置
    i1, v1, i2, v2, i3, v3 := bloomValues(d)
    b[i1] |= v1
    b[i2] |= v2
    b[i3] |= v3
}

// bloomValues 计算要设置的 3 个位
func bloomValues(data []byte) (uint, byte, uint, byte, uint, byte) {
    sha := crypto.Keccak256(data)
    
    // 使用哈希的前 6 字节确定 3 个位置
    i1 := BloomByteLength - uint((binary.BigEndian.Uint16(sha)&0x7ff)>>3) - 1
    v1 := byte(1 << (sha[1] & 0x7))
    
    i2 := BloomByteLength - uint((binary.BigEndian.Uint16(sha[2:])&0x7ff)>>3) - 1
    v2 := byte(1 << (sha[3] & 0x7))
    
    i3 := BloomByteLength - uint((binary.BigEndian.Uint16(sha[4:])&0x7ff)>>3) - 1
    v3 := byte(1 << (sha[5] & 0x7))
    
    return i1, v1, i2, v2, i3, v3
}

// Test 检查数据是否可能在过滤器中
func (b Bloom) Test(topic []byte) bool {
    i1, v1, i2, v2, i3, v3 := bloomValues(topic)
    return (b[i1]&v1) != 0 && (b[i2]&v2) != 0 && (b[i3]&v3) != 0
}
```

---

## 14. 异常处理机制

### 14.1 EVM错误类型

```go
// 文件: core/vm/errors.go

var (
    // 执行错误
    ErrOutOfGas = errors.New("out of gas")
    ErrCodeStoreOutOfGas = errors.New("contract creation code storage out of gas")
    ErrDepth                    = errors.New("max call depth exceeded")
    ErrInsufficientBalance = errors.New("insufficient balance for transfer")
    ErrContractAddressCollision = errors.New("contract address collision")
    ErrExecutionReverted = errors.New("execution reverted")
    ErrMaxCodeSizeExceeded = errors.New("max code size exceeded")
    ErrInvalidJump = errors.New("invalid jump destination")
    ErrWriteProtection = errors.New("write protection")
    ErrReturnDataOutOfBounds = errors.New("return data out of bounds")
    ErrGasUintOverflow = errors.New("gas uint64 overflow")
    ErrInvalidCode = errors.New("invalid code: must not begin with 0xef")
    ErrNonceUintOverflow        = errors.New("nonce uint64 overflow")
    
    // 内部标记 (不是真正的错误)
    errStopToken = errors.New("stop token")
)

// ErrStackUnderflow 栈下溢
type ErrStackUnderflow struct {
    stackLen int
    required int
}

func (e *ErrStackUnderflow) Error() string {
    return fmt.Sprintf("stack underflow (%d <=> %d)", e.stackLen, e.required)
}

// ErrStackOverflow 栈溢出
type ErrStackOverflow struct {
    stackLen int
    limit    int
}

func (e *ErrStackOverflow) Error() string {
    return fmt.Sprintf("stack limit reached %d (%d)", e.stackLen, e.limit)
}

// ErrInvalidOpCode 无效操作码
type ErrInvalidOpCode struct {
    opcode OpCode
}

func (e *ErrInvalidOpCode) Error() string {
    return fmt.Sprintf("invalid opcode: %s", e.opcode)
}
```

### 14.2 异常处理流程

```go
// 文件: core/vm/interpreter.go

func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    // ... 初始化 ...
    
    // 主循环中的异常处理
    for {
        op = contract.GetOp(pc)
        operation := in.table[op]
        
        // 1. 无效操作码检查
        if operation == nil {
        return nil, &ErrInvalidOpCode{opcode: op}
        }
        
        // 2. 栈深度检查
        if sLen := stack.len(); sLen < operation.minStack {
            return nil, &ErrStackUnderflow{stackLen: sLen, required: operation.minStack}
        } else if sLen > operation.maxStack {
            return nil, &ErrStackOverflow{stackLen: sLen, limit: operation.maxStack}
        }
        
        // 3. 写保护检查
        if in.readOnly && operation.writes {
            return nil, ErrWriteProtection
        }
        
        // 4. Gas 检查
        if !contract.UseGas(cost) {
            return nil, ErrOutOfGas
        }
        
        // 5. 执行操作
        res, err = operation.execute(&pc, in, callContext)
        
        // 6. 错误处理
        if err != nil {
            break
        }
        pc++
    }
    
    // 处理特殊的停止标记
    if err == errStopToken {
        err = nil
    }
    
    return res, err
}
```

**异常处理流程图解：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     EVM 异常处理流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  执行操作码                                                      │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────┐                                                │
│  │ 操作码有效?   │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│    NO   │   YES                                                 │
│  ┌──────┴──────┐                                                │
│  │             │                                                │
│  ▼             ▼                                                │
│ ┌────────┐  ┌─────────────┐                                     │
│ │Invalid │  │  栈深度OK?   │                                     │
│ │OpCode  │  └──────┬──────┘                                     │
│ │错误     │         │                                            │
│ └────────┘    NO   │   YES                                      │
│            ┌───────┴───────┐                                    │
│            │               │                                    │
│            ▼               ▼                                    │
│         ┌─────────┐    ┌───────────────┐                        │
│         │Stack    │    │ 写保护检查?     │                        │
│         │Underflow│    │(STATICCALL中) │                         │
│         │/Overflow│    └──────┬────────┘                         │
│         └─────────┘           │                                  │
│                        NO     │   YES (是写操作且只读模式)         │
│                        ┌──────┴──────┐                           │
│                        │             │                           │
│                        ▼             ▼                           │
│                 ┌─────────┐   ┌─────────────┐                   │
│                 │ Gas足够? │   │WriteProtect │                   │
│                 └────┬────┘   │   错误       │                   │
│                      │        └─────────────┘                   │
│                 NO   │   YES                                    │
│               ┌──────┴──────┐                                   │
│               │             │                                   │
│               ▼             ▼                                   │
│          ┌────────┐    ┌──────────────┐                         │
│          │OutOfGas│    │   执行操作     │                        │
│          │  错误  │     └──────┬───────┘                        │
│          └────────┘           │                                 │
│                               ▼                                 │
│                        ┌───────────┐                            │
│                        │ 执行结果   │                            │
│                        └─────┬─────┘                            │
│                              │                                  │
│          ┌───────────────────┼───────────────────┐             │
│          │                   │                   │             │
│          ▼                   ▼                   ▼             │
│     ┌─────────┐        ┌──────────┐       ┌───────────┐        │
│     │  成功    │        │  REVERT  │       │  其他错误  │        │
│     │继续执行  │        │ 回滚状态   │       │回滚+消耗Gas│        │
│     │         │        │ 返还Gas   │       │           │        │
│     └─────────┘        └──────────┘       └───────────┘        │
│                                                                │
│  REVERT vs 其他错误:                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  REVERT:                                                 │  │
│  │  • 状态回滚到调用前                                        │   │
│  │  • 剩余 Gas 返还给调用者                                   │   │
│  │  • 可以返回错误信息                                        │   │
│  │  • require() 失败触发                                     │   │
│  │                                                          │  │
│  │  其他错误 (如 OutOfGas, InvalidJump):                      │  │
│  │  • 状态回滚到调用前                                         │  │
│  │  • 所有 Gas 被消耗 (不返还)                                 │  │
│  │  • 惩罚性质                                               │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 14.3 REVERT 与 require/revert/assert

```
┌─────────────────────────────────────────────────────────────────┐
│              Solidity 异常机制与 EVM 对应                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  require(condition, "message")                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  • 条件检查，用于输入验证、权限检查                           │   │
│  │  • 失败时触发 REVERT                                      │   │
│  │  • 剩余 Gas 返还                                          │   │
│  │  • 返回错误信息 (ABI 编码)                                 │   │
│  │                                                          │  │
│  │  EVM 行为:                                               │   │
│  │  REVERT + Error(string) 选择器 (0x08c379a0)               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  revert("message") / revert CustomError()                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  • 显式回滚                                               │   │
│  │  • 剩余 Gas 返还                                          │   │
│  │  • 可以使用自定义错误 (更省 Gas)                            │   │
│  │                                                          │  │
│  │  EVM 行为:                                                │  │
│  │  REVERT + 自定义错误选择器                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  assert(condition)                                              │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  • 内部错误检查，不应该发生的情况                                 │    │
│  │  • Solidity 0.8+ 触发 REVERT + Panic 错误                      │    │
│  │  • Solidity 0.8 之前触发 INVALID (消耗所有 Gas)                  │   │
│  │                                                               │   │
│  │  EVM 行为 (0.8+):                                              │   │
│  │  REVERT + Panic(uint256) 选择器 (0x4e487b71)                   │    │
│  │  Panic codes: 0x01=assert, 0x11=overflow, 0x12=div0...        │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  错误编码示例:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                                                                 │  │
│  │  require(false, "Not enough balance")                           │  │
│  │  返回数据:                                                       │  │
│  │  0x08c379a0                               // Error 选择器        │  │
│  │  0000000000000000000000000000000000000000000000000000020 // 偏移 │  |
│  │  0000000000000000000000000000000000000000000000000000012 // 长度 │  |
│  │  4e6f7420656e6f7567682062616c616e636500000000000000000000 // 消息 │ |
│  │                                                                  │  │
│  │  error InsufficientBalance(uint256 required, uint256 has);       │  |
│  │  revert InsufficientBalance(100, 50);                            │  │
│  │  返回数据:                                                        │  │
│  │  0xcf479181                               // 自定义错误选择器       │ |
│  │  0000000000000000000000000000000000000000000000000000064 // 100  │  |
│  │  0000000000000000000000000000000000000000000000000000032 // 50   │  |
│  │                                                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 15. EVM 调试与追踪

### 15.1 Tracer 接口

```go
// 文件: core/vm/logger.go

// EVMLogger 是 EVM 追踪器接口
type EVMLogger interface {
// 交易级别
CaptureTxStart(gasLimit uint64)
CaptureTxEnd(restGas uint64)

// 调用级别
CaptureStart(env *EVM, from common.Address, to common.Address, create bool, input []byte, gas uint64, value *big.Int)
CaptureEnd(output []byte, gasUsed uint64, err error)

// 每个操作
CaptureState(pc uint64, op OpCode, gas, cost uint64, scope *ScopeContext, rData []byte, depth int, err error)

// 错误
CaptureFault(pc uint64, op OpCode, gas, cost uint64, scope *ScopeContext, depth int, err error)

// 子调用
CaptureEnter(typ OpCode, from common.Address, to common.Address, input []byte, gas uint64, value *big.Int)
CaptureExit(output []byte, gasUsed uint64, err error)
}
```

### 15.2 结构化日志追踪器

```go
// 文件: eth/tracers/logger/logger.go

// StructLogger 结构化日志追踪器
type StructLogger struct {
    cfg Config
    env *vm.EVM
    
    storage    map[common.Address]Storage
    logs       []StructLog
    output     []byte
    err        error
    gasLimit   uint64
    usedGas    uint64
    
    interrupt atomic.Bool
    reason    error
}

// StructLog 单条日志
type StructLog struct {
    Pc            uint64                      `json:"pc"`
    Op            vm.OpCode                   `json:"op"`
    Gas           uint64                      `json:"gas"`
    GasCost       uint64                      `json:"gasCost"`
    Memory        []byte                      `json:"memory,omitempty"`
    MemorySize    int                         `json:"memSize"`
    Stack         []uint256.Int               `json:"stack"`
    ReturnData    []byte                      `json:"returnData,omitempty"`
    Storage       map[common.Hash]common.Hash `json:"storage,omitempty"`
    Depth         int                         `json:"depth"`
    RefundCounter uint64                      `json:"refund"`
    Err           error                       `json:"-"`
}

// CaptureState 捕获每个操作的状态
func (l *StructLogger) CaptureState(pc uint64, op vm.OpCode, gas, cost uint64, scope *vm.ScopeContext, rData []byte, depth int, err error) {
    // 检查是否被中断
    if l.interrupt.Load() {
        return
    }
    
    // 复制栈
    stack := make([]uint256.Int, len(scope.Stack.Data()))
    copy(stack, scope.Stack.Data())
    
    // 创建日志条目
    log := StructLog{
        Pc:            pc,
        Op:            op,
        Gas:           gas,
        GasCost:       cost,
        MemorySize:    scope.Memory.Len(),
        Stack:         stack,
        Depth:         depth,
        RefundCounter: l.env.StateDB.GetRefund(),
        Err:           err,
    }
    
    // 可选: 复制内存
    if l.cfg.EnableMemory {
        log.Memory = scope.Memory.GetCopy(0, uint64(scope.Memory.Len()))
    }
    
    // 可选: 复制返回数据
    if l.cfg.EnableReturnData && len(rData) > 0 {
        log.ReturnData = make([]byte, len(rData))
        copy(log.ReturnData, rData)
    }
    
    l.logs = append(l.logs, log)
}
```

### 15.3 调用追踪器

```go
// 文件: eth/tracers/native/call.go

// callTracer 追踪所有调用
type callTracer struct {
    env       *vm.EVM
    callstack []callFrame
    config    callTracerConfig
    interrupt atomic.Bool
    reason    error
}

// callFrame 调用帧
type callFrame struct {
    Type    vm.OpCode       `json:"type"`
    From    common.Address  `json:"from"`
    To      common.Address  `json:"to,omitempty"`
    Value   *big.Int        `json:"value,omitempty"`
    Gas     uint64          `json:"gas"`
    GasUsed uint64          `json:"gasUsed"`
    Input   []byte          `json:"input"`
    Output  []byte          `json:"output,omitempty"`
    Error   string          `json:"error,omitempty"`
    Calls   []callFrame     `json:"calls,omitempty"`
}

// CaptureEnter 捕获子调用开始
func (t *callTracer) CaptureEnter(typ vm.OpCode, from common.Address, to common.Address, input []byte, gas uint64, value *big.Int) {
    call := callFrame{
        Type:  typ,
        From:  from,
        To:    to,
        Input: common.CopyBytes(input),
        Gas:   gas,
        Value: value,
    }
    t.callstack = append(t.callstack, call)
}
    
// CaptureExit 捕获子调用结束 
func (t *callTracer) CaptureExit(output []byte, gasUsed uint64, err error) {
    size := len(t.callstack)
    if size <= 1 {
        return
    }

    // 弹出当前帧
    call := t.callstack[size-1]
    t.callstack = t.callstack[:size-1]
    
    call.GasUsed = gasUsed
    call.Output = common.CopyBytes(output)
    if err != nil {
        call.Error = err.Error()
    }
    
    // 添加到父帧的 Calls
    t.callstack[size-2].Calls = append(t.callstack[size-2].Calls, call)
}
```

### 15.4 使用调试追踪

```go
// 示例: 追踪交易执行

package main

import (
	"encoding/json"
	"fmt"

	"github.com/ethereum/go-ethereum/core/vm"
	"github.com/ethereum/go-ethereum/eth/tracers/logger"
)

func traceTransaction() {
	// 配置追踪器
	config := &logger.Config{
		EnableMemory:     true,
		EnableReturnData: true,
		DisableStack:     false,
		DisableStorage:   false,
	}

	tracer := logger.NewStructLogger(config)

	// 配置 EVM 使用追踪器
	vmConfig := vm.Config{
		Tracer: tracer,
	}

	evm := vm.NewEVM(blockCtx, txCtx, stateDB, chainConfig, vmConfig)

	// 执行调用
	ret, gas, err := evm.Call(caller, contractAddr, input, gasLimit, value)

	// 获取追踪日志
	logs := tracer.StructLogs()

	// 格式化输出
	for i, log := range logs {
		fmt.Printf("%4d: PC=%04d  OP=%-12s  GAS=%d  COST=%d  DEPTH=%d\n",
			i, log.Pc, log.Op.String(), log.Gas, log.GasCost, log.Depth)

		// 打印栈
		if len(log.Stack) > 0 {
			fmt.Printf("      Stack: ")
			for j := len(log.Stack) - 1; j >= 0; j-- {
				fmt.Printf("%s ", log.Stack[j].Hex())
			}
			fmt.Println()
		}

		// 打印内存
		if len(log.Memory) > 0 && len(log.Memory) <= 256 {
			fmt.Printf("      Memory: %x\n", log.Memory)
		}
	}

	// JSON 格式输出
	jsonLogs, _ := json.MarshalIndent(logs, "", "  ")
	fmt.Println(string(jsonLogs))
}
```

**追踪输出示例：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      追踪输出示例                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  执行: 简单加法合约 (3 + 5 = 8)                                  │
│                                                                 │
│  字节码: 6003 6005 01 6000 52 6020 6000 f3                      │
│                                                                 │
│  追踪:                                                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   0: PC=0000  OP=PUSH1        GAS=79978  COST=3  DEPTH=1 │  │
│  │      Stack: []                                           │  │
│  │                                                          │  │
│  │   1: PC=0002  OP=PUSH1        GAS=79975  COST=3  DEPTH=1 │  │
│  │      Stack: [0x3]                                        │  │
│  │                                                          │  │
│  │   2: PC=0004  OP=ADD          GAS=79972  COST=3  DEPTH=1 │  │
│  │      Stack: [0x3, 0x5]                                   │  │
│  │                                                          │  │
│  │   3: PC=0005  OP=PUSH1        GAS=79969  COST=3  DEPTH=1 │  │
│  │      Stack: [0x8]                                        │  │
│  │                                                          │  │
│  │   4: PC=0007  OP=MSTORE       GAS=79966  COST=6  DEPTH=1 │  │
│  │      Stack: [0x8, 0x0]                                   │  │
│  │      (内存扩展成本: 3)                                    │  │
│  │                                                          │  │
│  │   5: PC=0008  OP=PUSH1        GAS=79960  COST=3  DEPTH=1 │  │
│  │      Stack: []                                           │  │
│  │      Memory: 0x000...0008 (32字节)                       │  │
│  │                                                          │  │
│  │   6: PC=000a  OP=PUSH1        GAS=79957  COST=3  DEPTH=1 │  │
│  │      Stack: [0x20]                                       │  │
│  │                                                          │  │
│  │   7: PC=000c  OP=RETURN       GAS=79954  COST=0  DEPTH=1 │  │
│  │      Stack: [0x20, 0x0]                                  │  │
│  │      返回 memory[0:32]                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  总消耗 Gas: 79978 - 79954 = 24                                 │
│  返回值: 0x0000000000000000000000000000000000000000000000000008 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 16. 硬分叉与 EIP 演进

### 16.1 硬分叉历史

```
┌─────────────────────────────────────────────────────────────────┐
│                    以太坊硬分叉历史                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Frontier (2015.07)                                             │
│  └── 创世区块，最基础的 EVM                                      │
│                                                                 │
│  Homestead (2016.03, Block 1,150,000)                           │
│  └── EIP-2: 调整 CREATE Gas 成本                                │
│  └── EIP-7: DELEGATECALL 操作码                                 │
│                                                                 │
│  Tangerine Whistle (2016.10, Block 2,463,000)                   │
│  └── EIP-150: IO 密集操作 Gas 提升                              │
│  └── 63/64 规则                                                 │
│                                                                 │
│  Spurious Dragon (2016.11, Block 2,675,000)                     │
│  └── EIP-155: 重放攻击保护                                      │
│  └── EIP-160: EXP Gas 调整                                      │
│  └── EIP-161: 空账户清理                                        │
│                                                                 │
│  Byzantium (2017.10, Block 4,370,000)                           │
│  └── EIP-100: 难度调整                                          │
│  └── EIP-140: REVERT 操作码                                     │
│  └── EIP-196/197: BN256 预编译合约                              │
│  └── EIP-198: MODEXP 预编译合约                                 │
│  └── EIP-211: RETURNDATASIZE, RETURNDATACOPY                   │
│  └── EIP-214: STATICCALL 操作码                                 │
│                                                                 │
│  Constantinople (2019.02, Block 7,280,000)                      │
│  └── EIP-145: SHL, SHR, SAR 位移操作                           │
│  └── EIP-1014: CREATE2 操作码                                   │
│  └── EIP-1052: EXTCODEHASH 操作码                              │
│  └── EIP-1283: SSTORE Gas 计量 (后被 Petersburg 禁用)          │
│                                                                 │
│  Istanbul (2019.12, Block 9,069,000)                            │
│  └── EIP-152: BLAKE2 预编译合约                                 │
│  └── EIP-1108: BN256 Gas 降低                                   │
│  └── EIP-1344: CHAINID 操作码                                   │
│  └── EIP-1884: SLOAD 等 Gas 调整                               │
│  └── EIP-2028: Calldata Gas 降低                               │
│  └── EIP-2200: SSTORE Gas 计量重新启用                         │
│                                                                 │
│  Berlin (2021.04, Block 12,244,000)                             │
│  └── EIP-2565: MODEXP Gas 降低                                  │
│  └── EIP-2718: 类型化交易信封                                   │
│  └── EIP-2929: 访问列表 Gas 成本                                │
│  └── EIP-2930: 可选访问列表                                     │
│                                                                 │
│  London (2021.08, Block 12,965,000)                             │
│  └── EIP-1559: 费用市场改革                                     │
│  └── EIP-3198: BASEFEE 操作码                                   │
│  └── EIP-3529: Gas 返还减少                                     │
│  └── EIP-3541: 禁止 0xEF 开头的代码                             │
│                                                                 │
│  The Merge (2022.09)                                            │
│  └── EIP-3675: PoS 共识                                         │
│  └── EIP-4399: DIFFICULTY → PREVRANDAO                         │
│                                                                 │
│  Shanghai (2023.04)                                             │
│  └── EIP-3651: Warm COINBASE                                    │
│  └── EIP-3855: PUSH0 操作码                                     │
│  └── EIP-3860: Initcode 大小限制                               │
│  └── EIP-4895: 信标链提款                                       │
│                                                                 │
│  Cancun (2024.03)                                               │
│  └── EIP-1153: 瞬态存储 (TLOAD/TSTORE)                         │
│  └── EIP-4788: 信标根在 EVM                                     │
│  └── EIP-4844: Proto-Danksharding (Blob交易)                   │
│  └── EIP-5656: MCOPY 操作码                                     │
│  └── EIP-6780: SELFDESTRUCT 限制                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 16.2 EIP启用机制

```go
// 文件: params/config.go

// ChainConfig 链配置
type ChainConfig struct {
    ChainID *big.Int `json:"chainId"`
    
    // 硬分叉区块号
    HomesteadBlock      *big.Int `json:"homesteadBlock,omitempty"`
    DAOForkBlock        *big.Int `json:"daoForkBlock,omitempty"`
    EIP150Block         *big.Int `json:"eip150Block,omitempty"`
    EIP155Block         *big.Int `json:"eip155Block,omitempty"`
    EIP158Block         *big.Int `json:"eip158Block,omitempty"`
    ByzantiumBlock      *big.Int `json:"byzantiumBlock,omitempty"`
    ConstantinopleBlock *big.Int `json:"constantinopleBlock,omitempty"`
    PetersburgBlock     *big.Int `json:"petersburgBlock,omitempty"`
    IstanbulBlock       *big.Int `json:"istanbulBlock,omitempty"`
    MuirGlacierBlock    *big.Int `json:"muirGlacierBlock,omitempty"`
    BerlinBlock         *big.Int `json:"berlinBlock,omitempty"`
    LondonBlock         *big.Int `json:"londonBlock,omitempty"`
    
    // 时间戳触发的分叉 (合并后)
    MergeNetsplitBlock *big.Int `json:"mergeNetsplitBlock,omitempty"`
    ShanghaiTime       *uint64  `json:"shanghaiTime,omitempty"`
    CancunTime         *uint64  `json:"cancunTime,omitempty"`
    PragueTime         *uint64  `json:"pragueTime,omitempty"`
}

// Rules 当前区块的规则
type Rules struct {
    ChainID                                                 *big.Int
    IsHomestead, IsEIP150, IsEIP155, IsEIP158               bool
    IsByzantium, IsConstantinople, IsPetersburg, IsIstanbul bool
    IsBerlin, IsLondon                                      bool
    IsMerge, IsShanghai, IsCancun, IsPrague                 bool
}

// Rules 根据区块号/时间戳返回当前规则
func (c *ChainConfig) Rules(num *big.Int, isMerge bool, timestamp uint64) Rules {
    chainID := c.ChainID
    if chainID == nil {
        chainID = new(big.Int)
    }
    return Rules{
        ChainID:          new(big.Int).Set(chainID),
        IsHomestead:      c.IsHomestead(num),
        IsEIP150:         c.IsEIP150(num),
        IsEIP155:         c.IsEIP155(num),
        IsEIP158:         c.IsEIP158(num),
        IsByzantium:      c.IsByzantium(num),
        IsConstantinople: c.IsConstantinople(num),
        IsPetersburg:     c.IsPetersburg(num),
        IsIstanbul:       c.IsIstanbul(num),
        IsBerlin:         c.IsBerlin(num),
        IsLondon:         c.IsLondon(num),
        IsMerge:          isMerge,
        IsShanghai:       c.IsShanghai(num, timestamp),
        IsCancun:         c.IsCancun(num, timestamp),
        IsPrague:         c.IsPrague(num, timestamp),
    }
}
```

### 16.3 跳转表版本控制

```go
// 文件: core/vm/jump_table.go

// 根据规则选择正确的跳转表
func (evm *EVM) initJumpTable() *JumpTable {
    switch {
    case evm.chainRules.IsCancun:
        return newCancunInstructionSet()
    case evm.chainRules.IsShanghai:
        return newShanghaiInstructionSet()
    case evm.chainRules.IsMerge:
        return newMergeInstructionSet()
    case evm.chainRules.IsLondon:
        return newLondonInstructionSet()
    case evm.chainRules.IsBerlin:
        return newBerlinInstructionSet()
    case evm.chainRules.IsIstanbul:
        return newIstanbulInstructionSet()
    case evm.chainRules.IsConstantinople:
        return newConstantinopleInstructionSet()
    case evm.chainRules.IsByzantium:
        return newByzantiumInstructionSet()
    case evm.chainRules.IsEIP158:
        return newSpuriousDragonInstructionSet()
    case evm.chainRules.IsEIP150:   
		return newTangerineWhistleInstructionSet()
    case evm.chainRules.IsHomestead:
        return newHomesteadInstructionSet()
    default:
        return newFrontierInstructionSet()
    }
}

// 每个版本添加新指令
func newShanghaiInstructionSet() JumpTable {
    instructionSet := newMergeInstructionSet()
    enable3855(&instructionSet) // PUSH0
    enable3860(&instructionSet) // Limit initcode
    return instructionSet
}

func enable3855(jt *JumpTable) {
    jt[PUSH0] = &operation{
        execute:     opPush0,
        constantGas: GasQuickStep,
        minStack:    minStack(0, 1),
        maxStack:    maxStack(0, 1),
    }
}

func newCancunInstructionSet() JumpTable {
    instructionSet := newShanghaiInstructionSet()
    enable1153(&instructionSet) // TLOAD, TSTORE
    enable4844(&instructionSet) // BLOBHASH, BLOBBASEFEE
    enable5656(&instructionSet) // MCOPY
    enable6780(&instructionSet) // Modified SELFDESTRUCT
    return instructionSet
}
```

---

## 17. 性能优化技术

### 17.1 JIT 编译 (实验性)

```
┌─────────────────────────────────────────────────────────────────┐
│                      EVM 优化技术                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 解释执行 (当前 go-ethereum 使用)                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  优点:                                                    │  │
│  │  • 实现简单，易于维护                                      │  │
│  │  • 内存占用低                                             │  │
│  │  • 启动快，无编译延迟                                      │  │
│  │                                                          │  │
│  │  缺点:                                                    │  │
│  │  • 每次执行都需要解码和分发                                │  │
│  │  • 无法进行跨指令优化                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  2. JIT 编译 (evmone 等实现)                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  优点:                                                    │  │
│  │  • 运行时将字节码编译为本地代码                            │  │
│  │  • 可以进行优化 (常量折叠、死代码消除等)                    │  │
│  │  • 重复执行的代码性能大幅提升                              │  │
│  │                                                          │  │
│  │  缺点:                                                    │  │
│  │  • 编译开销                                               │  │
│  │  • 实现复杂                                               │  │
│  │  • 内存占用增加                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  3. EOF (EVM Object Format) - 未来方向                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • 结构化字节码格式                                        │  │
│  │  • 分离代码和数据                                          │  │
│  │  • 静态跳转验证                                            │  │
│  │  • 便于 JIT 编译和 AOT 编译                                │  │
│  │  • EIP-3540, 3670, 4200, 4750, 5450                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 17.2 常见优化

```go
// 1. 栈复用
// go-ethereum 使用对象池减少分配
var stackPool = sync.Pool{
    New: func () interface{} {
        return &Stack{data: make([]uint256.Int, 0, 16)}
    },
}

func newstack() *Stack {
    return stackPool.Get().(*Stack)
}

func returnStack(s *Stack) {
    s.data = s.data[:0]
    stackPool.Put(s)
}

// 2. 使用uint256而非big.Int
// holiman/uint256 库针对256位优化
// 避免堆分配，使用栈上的固定大小数组
type Int struct {
    arr [4]uint64 // 256 位 = 4 × 64 位
}

// 3. JUMPDEST分析缓存
// 合约代码的跳转目标分析结果被缓存
type Contract struct {
    jumpdests map[common.Hash]bitvec // 缓存分析结果
    analysis  bitvec
}

func (c *Contract) validJumpdest(dest *uint256.Int) bool {
    udest := dest.Uint64()
    // 检查是否在有效范围内
    if dest.IsUint64() && udest < uint64(len(c.Code)) {
        // 检查缓存的分析结果
        if c.analysis == nil {
            c.analysis = codeBitmap(c.Code)
        }
        return c.analysis.codeSegment(udest)
    }
    return false
}

// 4. 内存预分配
// 减少append导致的重新分配
func (m *Memory) Resize(size uint64) {
    if uint64(cap(m.store)) < size {
        // 分配更多空间以减少后续扩展
        newSize := size
        if size < 1024 {
            newSize = 1024
        } else {
            newSize = size * 2
        }
        newStore := make([]byte, size, newSize)
        copy(newStore, m.store)
        m.store = newStore
    } else {
        m.store = m.store[:size]
    }
}
```

### 17.3 Gas 优化的代码模式

```
┌─────────────────────────────────────────────────────────────────┐
│                  Solidity Gas 优化技巧                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 存储优化                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 差: 每次循环写入存储                                   │  │
│  │  for (uint i = 0; i < 100; i++) {                        │  │
│  │      counter++;  // SLOAD + SSTORE 每次迭代               │  │
│  │  }                                                        │  │
│  │                                                          │  │
│  │  // 好: 使用局部变量                                      │  │
│  │  uint local = counter;                                   │  │
│  │  for (uint i = 0; i < 100; i++) {                        │  │
│  │      local++;  // 只在内存/栈操作                         │  │
│  │  }                                                        │  │
│  │  counter = local;  // 只写入一次存储                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  2. 变量打包                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 差: 3 个存储槽                                        │  │
│  │  uint256 a;                                              │  │
│  │  uint256 b;                                              │  │
│  │  uint256 c;                                              │  │
│  │                                                          │  │
│  │  // 好: 1 个存储槽 (如果值够小)                           │  │
│  │  uint128 a;                                              │  │
│  │  uint64 b;                                               │  │
│  │  uint64 c;                                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  3. 使用 calldata 而非 memory                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 差: 复制到内存                                        │  │
│  │  function process(uint[] memory data) external { ... }   │  │
│  │                                                          │  │
│  │  // 好: 直接从 calldata 读取                              │  │
│  │  function process(uint[] calldata data) external { ... } │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  4. 短路求值                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 将便宜的检查放前面                                     │  │
│  │  require(x > 0 && expensiveCheck(), "error");            │  │
│  │  // x > 0 失败时不执行 expensiveCheck()                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  5. 使用自定义错误                                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 差: 字符串存储在合约中                                 │  │
│  │  require(x > 0, "Value must be positive");               │  │
│  │                                                          │  │
│  │  // 好: 只存储 4 字节选择器                               │  │
│  │  error ValueNotPositive();                               │  │
│  │  if (x <= 0) revert ValueNotPositive();                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  6. 使用 unchecked 算术 (确保安全时)                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 0.8+ 默认检查溢出，有 Gas 开销                        │  │
│  │  for (uint i = 0; i < arr.length;) {                     │  │
│  │      // do something                                     │  │
│  │      unchecked { i++; }  // i 不可能溢出                  │  │
│  │  }                                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 18. 安全考虑

### 18.1 常见漏洞

```
┌──────────────────────────────────────────────────────────────────┐
│                      常见 EVM 安全漏洞                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 重入攻击 (Reentrancy)                                         │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  // 易受攻击的代码                                           │  │
│  │  function withdraw() external {                            │  │
│  │      uint amount = balances[msg.sender];                   │  │
│  │      (bool success,) = msg.sender.call{value: amount}(""); │  │
│  │      balances[msg.sender] = 0;  // 状态更新在外部调用后!      │  │
│  │  }                                                         │  │
│  │                                                            │  │
│  │  // 安全: Checks-Effects-Interactions 模式                  │  │
│  │  function withdraw() external {                            │  │
│  │      uint amount = balances[msg.sender];                   │  │
│  │      balances[msg.sender] = 0;  // 先更新状态                │  │
│  │      (bool success,) = msg.sender.call{value: amount}(""); │  │
│  │  }                                                         │  │
│  │                                                            │  │
│  │  // 或使用重入锁                                             │  │
│  │  modifier nonReentrant() {                                 │  │
│  │      require(!locked);                                     │  │
│  │      locked = true;                                        │  │
│  │      _;                                                    │  │
│  │      locked = false;                                       │  │
│  │  }                                                         │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  2. 整数溢出/下溢 (0.8 之前)                                        │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  // 0.8 之前: uint8(255) + 1 = 0                           │  │
│  │  // 0.8+: 自动检查，溢出会 revert                            │  │
│  │                                                           │  │
│  │  // 0.8 之前需要使用 SafeMath 库                            │  │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. tx.origin 滥用                                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  // 危险: 钓鱼攻击                                         │   │
│  │  require(tx.origin == owner);  // 错误!                   │   │
│  │                                                          │   │
│  │  // 安全                                                  │  │
│  │  require(msg.sender == owner);                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. 外部调用失败未检查                                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  // 危险: 忽略返回值                                        │  │
│  │  token.transfer(to, amount);                             │   │
│  │                                                          │   │
│  │  // 安全: 检查返回值                                       │   │
│  │  require(token.transfer(to, amount), "Transfer failed"); │   │
│  │  // 或使用 SafeERC20                                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  5. 闪电贷价格操纵                                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  // 危险: 使用即时价格                                      │   │
│  │  uint price = getReserves() / totalSupply;               │   │
│  │                                                          │   │
│  │  // 安全: 使用时间加权平均价格 (TWAP)                        │   │
│  │  uint price = oracle.consult(token, period);             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  6. 存储碰撞 (代理模式)                                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  // 代理合约和实现合约的存储槽必须兼容                         │   │
│  │  // 使用 EIP-1967 标准槽位                                 │   │
│  │  bytes32 constant IMPL_SLOT = keccak256(                 |   |
|  |                  "eip1967.proxy.implementation") - 1;    │   |
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 18.2 EVM 级别的保护

```go
// EVM 内置的安全机制

// 1. 调用深度限制
const CallCreateDepth uint64 = 1024

func (evm *EVM) Call(...) {
    if evm.depth > int(params.CallCreateDepth) {
        return nil, gas, ErrDepth
    }
    // ...
}

// 2. 栈深度限制
const stackLimit uint64 = 1024

func (st *Stack) push(d *uint256.Int) {
    if len(st.data) >= int(stackLimit) {
    // 栈溢出
    }
    // ...
}

// 3. 63/64 规则防止调用深度攻击
func callGas(isEip150 bool, availableGas, base uint64, callCost *uint256.Int) (uint64, error) {
    if isEip150 {
        availableGas = availableGas - base
        gas := availableGas - availableGas/64
        // ...
    }
}

// 4. 代码大小限制
const MaxCodeSize = 24576 // 24KB

func (evm *EVM) create(...) {
    if len(ret) > params.MaxCodeSize {
        err = ErrMaxCodeSizeExceeded
    }
}

// 5. Initcode 大小限制 (EIP-3860)
const MaxInitCodeSize = 2 * MaxCodeSize // 48KB

// 6. 只读模式 (STATICCALL)
if interpreter.readOnly && operation.writes {
    return nil, ErrWriteProtection
}

// 7. Gas 机制本身就是对 DoS 的防护
if !contract.UseGas(cost) {
    return nil, ErrOutOfGas
}
```

---

## 19. EVM 与其他组件交互

### 19.1 交易处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    交易处理完整流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 交易进入交易池                                               │
│     ├── 签名验证                                                │
│     ├── nonce 检查                                              │
│     ├── 余额检查                                                │
│     └── Gas 价格/基础费用检查                                   │
│                                                                 │
│  2. 区块构建/验证                                                │
│     ├── 从交易池选择交易                                        │
│     └── 按 Gas 价格排序                                         │
│                                                                 │
│  3. StateProcessor.Process()                                    │
│     │                                                           │
│     └── 对每笔交易:                                              │
│         ├── ApplyTransaction()                                  │
│         │   ├── 创建 EVM 实例                                   │
│         │   ├── 计算固有 Gas (21000 + calldata)                 │
│         │   ├── 扣除 Gas 费用                                   │
│         │   │                                                   │
│         │   ├── EVM.Call() 或 EVM.Create()                     │
│         │   │   └── Interpreter.Run()                          │
│         │   │       └── 执行字节码                              │
│         │   │                                                   │
│         │   ├── 计算 Gas 返还                                   │
│         │   ├── 退还剩余 Gas                                    │
│         │   └── 支付矿工费用                                    │
│         │                                                       │
│         └── 生成收据 (Receipt)                                  │
│             ├── 状态根/状态码                                   │
│             ├── 累计 Gas                                        │
│             ├── 日志                                            │
│             └── Bloom Filter                                    │
│                                                                 │
│  4. 状态提交                                                     │
│     ├── 更新 StateDB                                            │
│     ├── 计算新状态根                                            │
│     └── 写入数据库                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 19.2 代码示例: 完整交易执行

```go
// 文件: core/state_processor.go

// Process 执行区块中的所有交易
func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, uint64, error) {
    var (
        receipts    types.Receipts
        usedGas = new(uint64)
        header = block.Header()
        blockHash = block.Hash()
        blockNumber = block.Number()
        allLogs     []*types.Log
        gp = new(GasPool).AddGas(block.GasLimit())
    )
    
    // 创建区块上下文
    blockContext := NewEVMBlockContext(header, p.bc, nil)
    vmenv := vm.NewEVM(blockContext, vm.TxContext{}, statedb, p.config, cfg)
    
    // 处理每笔交易
    for i, tx := range block.Transactions() {
        msg, err := TransactionToMessage(tx, types.MakeSigner(p.config, header.Number), header.BaseFee)
        if err != nil {
            return nil, nil, 0, fmt.Errorf("could not apply tx %d [%v]: %w", i, tx.Hash().Hex(), err)
        }
        
        statedb.SetTxContext(tx.Hash(), i)
        
        receipt, err := applyTransaction(msg, p.config, gp, statedb, blockNumber, blockHash, tx, usedGas, vmenv)
        if err != nil {
            return nil, nil, 0, fmt.Errorf("could not apply tx %d [%v]: %w", i, tx.Hash().Hex(), err)
        }
        
        receipts = append(receipts, receipt)
        allLogs = append(allLogs, receipt.Logs...)
    }
    
    // 完成区块处理
    p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles(), withdrawals)
    
    return receipts, allLogs, *usedGas, nil
}
    
// applyTransaction 执行单笔交易
func applyTransaction(msg *Message, config *params.ChainConfig, gp *GasPool, statedb *state.StateDB, blockNumber *big.Int, blockHash common.Hash, tx *types.Transaction, usedGas *uint64, evm *vm.EVM) (*types.Receipt, error) {
    // 创建状态转换对象
    stateTransition := NewStateTransition(evm, msg, gp)
    
    // 执行交易
    result, err := stateTransition.TransitionDb()
    if err != nil {
        return nil, err
    }
    
    *usedGas += result.UsedGas
    
    // 创建收据
    receipt := &types.Receipt{
        Type:              tx.Type(),
        PostState:         nil,
        CumulativeGasUsed: *usedGas,
        TxHash:            tx.Hash(),
        GasUsed:           result.UsedGas,
        Logs:              statedb.GetLogs(tx.Hash(), blockNumber.Uint64(), blockHash),
        BlockHash:         blockHash,
        BlockNumber:       blockNumber,
        TransactionIndex:  uint(statedb.TxIndex()),
    }
    
    if result.Failed() {
        receipt.Status = types.ReceiptStatusFailed
    } else {
        receipt.Status = types.ReceiptStatusSuccessful
    }
    
    // 设置合约地址 (如果是创建交易)
    if msg.To == nil {
        receipt.ContractAddress = crypto.CreateAddress(evm.TxContext.Origin, tx.Nonce())
    }
    
    // 创建 Bloom Filter
    receipt.Bloom = types.CreateBloom(types.Receipts{receipt})
    
    return receipt, nil
}
```

---

## 20. 完整实战案例

### 20.1 模拟DEX交易

```go
package main

import (
	"fmt"
	"math/big"
	"strings"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/rawdb"
	"github.com/ethereum/go-ethereum/core/state"
	"github.com/ethereum/go-ethereum/core/vm"
	"github.com/ethereum/go-ethereum/params"
)

// 简化的 DEX 合约 ABI
const dexABI = `[
    {
        "name": "swap",
        "type": "function",
        "inputs": [
            {"name": "tokenIn", "type": "address"},
            {"name": "tokenOut", "type": "address"},
            {"name": "amountIn", "type": "uint256"}
        ],
        "outputs": [
            {"name": "amountOut", "type": "uint256"}
        ]
    },
    {
        "name": "getReserves",
        "type": "function",
        "inputs": [],
        "outputs": [
            {"name": "reserve0", "type": "uint256"},
            {"name": "reserve1", "type": "uint256"}
        ]
    }
]`

func main() {
	// 初始化
	db := rawdb.NewMemoryDatabase()
	stateDB, _ := state.New(common.Hash{}, state.NewDatabase(db), nil)

	// 地址
	trader := common.HexToAddress("0x1111111111111111111111111111111111111111")
	dexContract := common.HexToAddress("0x2222222222222222222222222222222222222222")
	tokenA := common.HexToAddress("0xAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA")
	tokenB := common.HexToAddress("0xBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB")

	// 给交易者一些 ETH
	stateDB.AddBalance(trader, big.NewInt(10e18))

	// 部署模拟 DEX 合约
	// 实际中这会是编译后的字节码
	// 这里我们直接设置存储来模拟

	// 设置储备量
	// reserve0 (tokenA) = 1000e18
	// reserve1 (tokenB) = 2000e18
	reserve0Slot := common.HexToHash("0x0")
	reserve1Slot := common.HexToHash("0x1")
	stateDB.SetState(dexContract, reserve0Slot, common.BigToHash(big.NewInt(1000e18)))
	stateDB.SetState(dexContract, reserve1Slot, common.BigToHash(big.NewInt(2000e18)))

	// 解析 ABI
	parsedABI, err := abi.JSON(strings.NewReader(dexABI))
	if err != nil {
		panic(err)
	}

	// 创建 EVM
	blockCtx := vm.BlockContext{
		CanTransfer: func(db vm.StateDB, addr common.Address, amount *big.Int) bool {
			return db.GetBalance(addr).Cmp(amount) >= 0
		},
		Transfer: func(db vm.StateDB, from, to common.Address, amount *big.Int) {
			db.SubBalance(from, amount)
			db.AddBalance(to, amount)
		},
		GetHash:     func(n uint64) common.Hash { return common.Hash{} },
		BlockNumber: big.NewInt(18000000),
		Time:        1700000000,
		BaseFee:     big.NewInt(30e9), // 30 gwei
	}

	txCtx := vm.TxContext{
		Origin:   trader,
		GasPrice: big.NewInt(50e9), // 50 gwei
	}

	evm := vm.NewEVM(blockCtx, txCtx, stateDB, params.MainnetChainConfig, vm.Config{})

	// 编码 swap 调用
	amountIn := big.NewInt(100e18) // 100 tokenA
	callData, err := parsedABI.Pack("swap", tokenA, tokenB, amountIn)
	if err != nil {
		panic(err)
	}

	fmt.Println("=== DEX 交易模拟 ===")
	fmt.Printf("交易者: %s\n", trader.Hex())
	fmt.Printf("DEX 合约: %s\n", dexContract.Hex())
	fmt.Printf("输入代币: %s\n", tokenA.Hex())
	fmt.Printf("输出代币: %s\n", tokenB.Hex())
	fmt.Printf("输入金额: %s\n", amountIn.String())
	fmt.Printf("调用数据: 0x%x\n", callData)

	// 由于没有真实合约代码，我们模拟计算
	// AMM 公式: amountOut = reserveOut * amountIn / (reserveIn + amountIn)
	reserveIn := big.NewInt(1000e18)
	reserveOut := big.NewInt(2000e18)

	numerator := new(big.Int).Mul(reserveOut, amountIn)
	denominator := new(big.Int).Add(reserveIn, amountIn)
	amountOut := new(big.Int).Div(numerator, denominator)

	fmt.Printf("\n=== 交易结果 (模拟计算) ===\n")
	fmt.Printf("输出金额: %s tokenB\n", amountOut.String())
	fmt.Printf("汇率: 1 tokenA = %f tokenB\n", float64(amountOut.Int64())/float64(amountIn.Int64()))

	// 如果有真实合约代码，可以这样调用:
	// ret, leftOverGas, err := evm.Call(
	//     vm.AccountRef(trader),
	//     dexContract,
	//     callData,
	//     1000000, // gas limit
	//     big.NewInt(0),
	// )
}
```

### 20.2 分析真实交易

```go
package main

import (
	"context"
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/vm"
	"github.com/ethereum/go-ethereum/eth/tracers/logger"
	"github.com/ethereum/go-ethereum/ethclient"
)

// 分析主网上的真实交易
func analyzeTransaction() {
	// 连接到节点 (需要 archive 节点进行历史追踪)
	client, err := ethclient.Dial("https://mainnet.infura.io/v3/YOUR_PROJECT_ID")
	if err != nil {
		panic(err)
	}

	// 交易哈希
	txHash := common.HexToHash("0x...")

	// 获取交易
	tx, isPending, err := client.TransactionByHash(context.Background(), txHash)
	if err != nil {
		panic(err)
	}
	if isPending {
		fmt.Println("交易还在 pending")
		return
	}

	fmt.Printf("=== 交易分析 ===\n")
	fmt.Printf("哈希: %s\n", tx.Hash().Hex())
	fmt.Printf("Gas 限制: %d\n", tx.Gas())
	fmt.Printf("Gas 价格: %s\n", tx.GasPrice().String())
	fmt.Printf("目标地址: %s\n", tx.To().Hex())
	fmt.Printf("Value: %s wei\n", tx.Value().String())
	fmt.Printf("数据长度: %d bytes\n", len(tx.Data()))

	// 获取收据
	receipt, err := client.TransactionReceipt(context.Background(), txHash)
	if err != nil {
		panic(err)
	}

	fmt.Printf("\n=== 执行结果 ===\n")
	fmt.Printf("状态: %d (1=成功, 0=失败)\n", receipt.Status)
	fmt.Printf("Gas 使用: %d\n", receipt.GasUsed)
	fmt.Printf("日志数量: %d\n", len(receipt.Logs))

	// 打印日志
	for i, log := range receipt.Logs {
		fmt.Printf("\n日志 #%d:\n", i)
		fmt.Printf("  合约: %s\n", log.Address.Hex())
		fmt.Printf("  主题数: %d\n", len(log.Topics))
		for j, topic := range log.Topics {
			fmt.Printf("  Topic[%d]: %s\n", j, topic.Hex())
		}
		fmt.Printf("  数据: 0x%x\n", log.Data)
	}

	// 如果需要完整的执行追踪，需要使用 debug_traceTransaction RPC
	// 这需要节点启用 debug API
}

// 使用 debug_traceTransaction (需要节点支持)
func traceTransactionDebug() {
	// 配置追踪器
	tracerConfig := &logger.Config{
		EnableMemory:     false, // 内存追踪很大
		EnableReturnData: true,
		DisableStack:     false,
		DisableStorage:   false,
	}

	// 调用 debug_traceTransaction
	// 返回完整的操作码执行日志

	fmt.Println("追踪交易需要使用 debug_traceTransaction RPC")
	fmt.Println("示例: curl -X POST --data '{")
	fmt.Println(`  "jsonrpc":"2.0",`)
	fmt.Println(`  "method":"debug_traceTransaction",`)
	fmt.Println(`  "params":["0x...", {"tracer": "callTracer"}],`)
	fmt.Println(`  "id":1`)
	fmt.Println("}' http://localhost:8545")
}
```

### 20.3 实现简单的 EVM 模拟器

```go
package main

import (
	"encoding/hex"
	"fmt"
	"math/big"

	"github.com/holiman/uint256"
)

// SimpleEVM 简单的 EVM 实现
type SimpleEVM struct {
	code    []byte
	pc      uint64
	stack   []*uint256.Int
	memory  []byte
	storage map[string]*uint256.Int
	gas     uint64
	stopped bool
}

// NewSimpleEVM 创建简单 EVM
func NewSimpleEVM(code []byte, gas uint64) *SimpleEVM {
	return &SimpleEVM{
		code:    code,
		pc:      0,
		stack:   make([]*uint256.Int, 0),
		memory:  make([]byte, 0),
		storage: make(map[string]*uint256.Int),
		gas:     gas,
		stopped: false,
	}
}

// 操作码常量
const (
	STOP     = 0x00
	ADD      = 0x01
	MUL      = 0x02
	SUB      = 0x03
	DIV      = 0x04
	LT       = 0x10
	GT       = 0x11
	EQ       = 0x14
	ISZERO   = 0x15
	POP      = 0x50
	MLOAD    = 0x51
	MSTORE   = 0x52
	JUMP     = 0x56
	JUMPI    = 0x57
	JUMPDEST = 0x5b
	PUSH1    = 0x60
	PUSH32   = 0x7f
	DUP1     = 0x80
	SWAP1    = 0x90
	RETURN   = 0xf3
)

// Push 压入栈
func (e *SimpleEVM) Push(v *uint256.Int) {
	e.stack = append(e.stack, v)
}

// Pop 弹出栈
func (e *SimpleEVM) Pop() *uint256.Int {
	if len(e.stack) == 0 {
		panic("stack underflow")
	}
	v := e.stack[len(e.stack)-1]
	e.stack = e.stack[:len(e.stack)-1]
	return v
}

// Peek 查看栈顶
func (e *SimpleEVM) Peek() *uint256.Int {
	if len(e.stack) == 0 {
		panic("stack underflow")
	}
	return e.stack[len(e.stack)-1]
}

// UseGas 消耗 Gas
func (e *SimpleEVM) UseGas(amount uint64) bool {
	if e.gas < amount {
		return false
	}
	e.gas -= amount
	return true
}

// Run 运行 EVM
func (e *SimpleEVM) Run() ([]byte, error) {
	for !e.stopped && e.pc < uint64(len(e.code)) {
		op := e.code[e.pc]

		switch {
		case op == STOP:
			e.UseGas(0)
			e.stopped = true

		case op == ADD:
			if !e.UseGas(3) {
				return nil, fmt.Errorf("out of gas")
			}
			a, b := e.Pop(), e.Pop()
			result := new(uint256.Int).Add(a, b)
			e.Push(result)
			e.pc++

		case op == MUL:
			if !e.UseGas(5) {
				return nil, fmt.Errorf("out of gas")
			}
			a, b := e.Pop(), e.Pop()
			result := new(uint256.Int).Mul(a, b)
			e.Push(result)
			e.pc++

		case op == SUB:
			if !e.UseGas(3) {
				return nil, fmt.Errorf("out of gas")
			}
			a, b := e.Pop(), e.Pop()
			result := new(uint256.Int).Sub(a, b)
			e.Push(result)
			e.pc++

		case op == LT:
			if !e.UseGas(3) {
				return nil, fmt.Errorf("out of gas")
			}
			a, b := e.Pop(), e.Pop()
			if a.Lt(b) {
				e.Push(uint256.NewInt(1))
			} else {
				e.Push(uint256.NewInt(0))
			}
			e.pc++

		case op == EQ:
			if !e.UseGas(3) {
				return nil, fmt.Errorf("out of gas")
			}
			a, b := e.Pop(), e.Pop()
			if a.Eq(b) {
				e.Push(uint256.NewInt(1))
			} else {
				e.Push(uint256.NewInt(0))
			}
			e.pc++

		case op == ISZERO:
			if !e.UseGas(3) {
				return nil, fmt.Errorf("out of gas")
			}
			a := e.Pop()
			if a.IsZero() {
				e.Push(uint256.NewInt(1))
			} else {
				e.Push(uint256.NewInt(0))
			}
			e.pc++

		case op == POP:
			if !e.UseGas(2) {
				return nil, fmt.Errorf("out of gas")
			}
			e.Pop()
			e.pc++

		case op == MSTORE:
			if !e.UseGas(3) {
				return nil, fmt.Errorf("out of gas")
			}
			offset := e.Pop().Uint64()
			value := e.Pop()

			// 扩展内存
			needed := offset + 32
			if uint64(len(e.memory)) < needed {
				extension := make([]byte, needed-uint64(len(e.memory)))
				e.memory = append(e.memory, extension...)
			}

			// 写入值 (大端序)
			value.WriteToSlice(e.memory[offset : offset+32])
			e.pc++

		case op == MLOAD:
			if !e.UseGas(3) {
				return nil, fmt.Errorf("out of gas")
			}
			offset := e.Pop().Uint64()

			// 确保内存够大
			if uint64(len(e.memory)) < offset+32 {
				extension := make([]byte, offset+32-uint64(len(e.memory)))
				e.memory = append(e.memory, extension...)
			}

			value := new(uint256.Int).SetBytes(e.memory[offset : offset+32])
			e.Push(value)
			e.pc++

		case op >= PUSH1 && op <= PUSH32:
			if !e.UseGas(3) {
				return nil, fmt.Errorf("out of gas")
			}
			size := int(op - PUSH1 + 1)

			// 读取立即数
			start := e.pc + 1
			end := start + uint64(size)
			if end > uint64(len(e.code)) {
				end = uint64(len(e.code))
			}

			value := new(uint256.Int).SetBytes(e.code[start:end])
			e.Push(value)
			e.pc = end

		case op == JUMP:
			if !e.UseGas(8) {
				return nil, fmt.Errorf("out of gas")
			}
			dest := e.Pop().Uint64()
			if dest >= uint64(len(e.code)) || e.code[dest] != JUMPDEST {
				return nil, fmt.Errorf("invalid jump destination")
			}
			e.pc = dest

		case op == JUMPI:
			if !e.UseGas(10) {
				return nil, fmt.Errorf("out of gas")
			}
			dest := e.Pop().Uint64()
			cond := e.Pop()
			if !cond.IsZero() {
				if dest >= uint64(len(e.code)) || e.code[dest] != JUMPDEST {
					return nil, fmt.Errorf("invalid jump destination")
				}
				e.pc = dest
			} else {
				e.pc++
			}

		case op == JUMPDEST:
			if !e.UseGas(1) {
				return nil, fmt.Errorf("out of gas")
			}
			e.pc++

		case op == RETURN:
			if !e.UseGas(0) {
				return nil, fmt.Errorf("out of gas")
			}
			offset := e.Pop().Uint64()
			size := e.Pop().Uint64()

			// 确保内存够大
			if uint64(len(e.memory)) < offset+size {
				extension := make([]byte, offset+size-uint64(len(e.memory)))
				e.memory = append(e.memory, extension...)
			}

			ret := make([]byte, size)
			copy(ret, e.memory[offset:offset+size])
			e.stopped = true
			return ret, nil

		default:
			return nil, fmt.Errorf("unknown opcode: 0x%02x at pc=%d", op, e.pc)
		}

		// 打印状态
		fmt.Printf("PC=%04d  OP=0x%02x  GAS=%d  STACK=%v\n", e.pc, op, e.gas, e.stack)
	}

	return nil, nil
}

func main() {
	// 测试代码: 3 + 5 = 8, 存入内存并返回
	// PUSH1 3, PUSH1 5, ADD, PUSH1 0, MSTORE, PUSH1 32, PUSH1 0, RETURN
	code, _ := hex.DecodeString("6003600501600052602060006000f3")

	fmt.Println("=== 简单 EVM 执行 ===")
	fmt.Printf("字节码: 0x%x\n", code)
	fmt.Println()

	evm := NewSimpleEVM(code, 100000)
	result, err := evm.Run()

	if err != nil {
		fmt.Printf("错误: %v\n", err)
	} else {
		fmt.Printf("\n返回值: 0x%x\n", result)
		fmt.Printf("十进制: %s\n", new(big.Int).SetBytes(result).String())
		fmt.Printf("剩余 Gas: %d\n", evm.gas)
	}
}
```

---

## 总结

### 关键知识点回顾

1. **状态管理**: Snapshot/Revert机制通过Journal实现原子性回滚
2. **日志系统**: LOG0-LOG4操作码，Bloom Filter加速过滤
3. **异常处理**: REVERT返还Gas，其他错误消耗所有Gas
4. **调试追踪**: Tracer接口捕获每个操作的执行状态
5. **硬分叉**: 通过版本化的JumpTable支持EIP演进
6. **性能优化**: 对象池、uint256、缓存等技术
7. **安全机制**: 深度限制、Gas 机制、只读模式等

### 学习资源

- [go-ethereum 源码](https://github.com/ethereum/go-ethereum)
- [以太坊黄皮书](https://ethereum.github.io/yellowpaper/paper.pdf)
- [EVM 操作码参考](https://www.evm.codes/)
- [EIP 列表](https://eips.ethereum.org/)
- [Solidity 文档](https://docs.soliditylang.org/)

### 下一步学习建议

1. 阅读go-ethereum源码中的`core/vm`目录
2. 使用Remix调试简单合约
3. 学习Foundry/Hardhat 的调试功能
4. 研究MEV相关的EVM优化技术
5. 了解L2 EVM变体(zkEVM, Optimistic EVM)
