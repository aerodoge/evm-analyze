# Go-Ethereum EVM 深入解析：revm 架构分析

## 概述

[revm](https://github.com/bluealloy/revm) 是用 Rust 编写的高性能 EVM 实现，由 Dragan Rakita 开发维护。
作为以太坊生态中最重要的 Rust EVM 实现，revm 被广泛应用于：

- **客户端**: Reth, Helios, Trin
- **开发工具**: Foundry, Hardhat
- **L2 方案**: Optimism, Base, Scroll
- **ZK 系统**: Risc0, Succinct
- **MEV 工具**: 几乎所有主流区块构建者

## 1. 项目概览

### 1.1 基本信息

| 特性         | revm        | go-ethereum | evmone         |
|------------|-------------|-------------|----------------|
| **语言**     | Rust        | Go          | C++ (C++17/20) |
| **协议**     | MIT         | LGPL-3.0    | Apache-2.0     |
| **MSRV**   | 1.88.0      | -           | -              |
| **设计目标**   | 通用执行引擎      | 完整客户端       | 独立执行引擎         |
| **no_std** | ✅ 支持        | ❌           | ❌              |
| **接口**     | Rust Traits | 内部 API      | EVMC C ABI     |

### 1.2 设计哲学

revm 的三大核心原则：

```
┌─────────────────────────────────────────────────────────────┐
│                    revm 设计哲学                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. EVM 兼容性与稳定性 (Compatibility & Stability)            │
│     "在区块链行业，稳定性是最重要的属性"                          │
│     - 严格遵循 EVM 规范                                       │
│     - 完整的测试覆盖                                          │
│     - 向后兼容                                               │
│                                                             │
│  2. 速度 (Speed)                                            │
│     "最重要的考量，大多数设计决策都为此服务"                      │
│     - 零拷贝设计                                             │
│     - 内联优化                                               │
│     - 缓存友好的数据结构                                       │
│                                                             │
│  3. 简洁性 (Simplicity)                                      │
│     "简化内部实现，便于理解和扩展"                              │
│     - 模块化架构                                             │
│     - 清晰的 trait 抽象                                      │
│     - 可组合的组件                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 模块化架构

### 2.1 Workspace 结构

revm 采用 29 个 crate 的工作空间组织：

```
revm/
├── crates/
│   ├── revm/              # 主入口，重导出所有模块
│   ├── primitives/        # 基础类型 (U256, Address, etc.)
│   ├── bytecode/          # 字节码分析与跳转表
│   ├── interpreter/       # 解释器核心
│   ├── handler/           # 执行处理器
│   ├── context/           # 执行上下文
│   ├── state/             # 状态管理
│   ├── database/          # 数据库接口
│   ├── precompile/        # 预编译合约
│   ├── inspector/         # 追踪与调试
│   ├── op-revm/           # Optimism 变体
│   └── ...
├── bins/
│   └── revme/             # CLI 工具
└── examples/              # 10+ 示例项目
```

### 2.2 分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户层 (User Layer)                       │
├─────────────────────────────────────────────────────────────────┤
│  MainnetEvm  │  InspectEvm  │  SystemCallEvm  │  op-revm        │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                     Handler 层 (Handler Layer)                  │
├─────────────────────────────────────────────────────────────────┤
│  ExecuteEvm   │  ExecuteCommitEvm  │  PreExecution │ PostExec   │
│               │                     │              │            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Frame Management: EthFrame, CallFrame, CreateFrame      │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                   Interpreter 层 (Core Layer)                   │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ Interpreter │  │ Gas         │  │ Instructions            │  │
│  │ ─────────── │  │ ─────────── │  │ ───────────────────     │  │
│  │ • Stack     │  │ • Constants │  │ • arithmetic.rs         │  │
│  │ • Memory    │  │ • Calc      │  │ • control.rs            │  │
│  │ • Bytecode  │  │ • Refund    │  │ • memory.rs             │  │
│  │ • Gas       │  │             │  │ • stack.rs              │  │
│  └─────────────┘  └─────────────┘  │ • system.rs             │  │
│                                    │ • host.rs               │  │
│                                    └─────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                    基础设施层 (Infrastructure)                   │
├─────────────────────────────────────────────────────────────────┤
│  primitives   │  database    │   state     │   precompile       │
│  (alloy)      │  (Database)  │  (Journal)  │  (EthPrecompiles)  │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 核心架构设计

### 3.1 Interpreter 泛型架构

revm 最精妙的设计是基于 trait 的泛型解释器：

```rust
/// 解释器类型族 - 定义所有可替换组件
pub trait InterpreterTypes {
    type Stack: StackTr;           // 栈实现
    type Memory: MemoryTr;         // 内存实现
    type Bytecode: Jumps + Immediates + LoopControl + LegacyBytecode;
    type ReturnData: ReturnData;   // 返回数据
    type Input: InputsTr;          // 调用输入
    type RuntimeFlag: RuntimeFlag; // 运行时标志
    type Extend;                   // 扩展数据
    type Output;                   // 输出类型
}

/// 解释器结构 - 泛型化所有组件
pub struct Interpreter<WIRE: InterpreterTypes = EthInterpreter> {
    pub gas_params: GasParams,
    pub bytecode: WIRE::Bytecode,
    pub gas: Gas,
    pub stack: WIRE::Stack,
    pub return_data: WIRE::ReturnData,
    pub memory: WIRE::Memory,
    pub input: WIRE::Input,
    pub runtime_flag: WIRE::RuntimeFlag,
    pub extend: WIRE::Extend,
}
```

**为什么这样设计？**

```rust
// 好处 1: 可以替换任意组件
// 例如：自定义栈实现用于追踪
struct TracingStack {
    inner: Vec<U256>,
    history: Vec<StackOp>,
}

// 好处 2: 零成本抽象
// 编译时单态化，无运行时开销

// 好处 3: no_std 支持
// 可以使用不同的内存分配器
#[cfg(not(feature = "std"))]
type DefaultMemory = BumpAllocMemory;

#[cfg(feature = "std")]
type DefaultMemory = HeapMemory;
```

### 3.2 Trait 分层设计

```rust
// 底层操作 traits - 最小化接口
pub trait Immediates {
    fn read_i8(&self) -> i8;
    fn read_i16(&self) -> i16;
    fn read_u16(&self) -> u16;
    fn read_slice(&self, len: usize) -> &[u8];
}

pub trait StackTr {
    fn push(&mut self, value: U256);
    fn pop(&mut self) -> U256;
    fn dup(&mut self, n: usize);
    fn exchange(&mut self, n: usize, m: usize);
}

pub trait MemoryTr {
    fn set(&mut self, offset: usize, value: &[u8]);
    fn get(&self, offset: usize, len: usize) -> &[u8];
    fn resize(&mut self, new_size: usize);
}

// 组合 trait - 表达更高层概念
pub trait LegacyBytecode: Jumps + Immediates + LoopControl {
    fn bytecode_len(&self) -> usize;
    fn bytecode_slice(&self) -> &[u8];
}

// 控制流 trait
pub trait LoopControl {
    fn set_action(&mut self, action: InterpreterAction);
    fn action(&self) -> &InterpreterAction;
    fn is_not_end(&self) -> bool;
}
```

### 3.3 执行循环

```rust
impl<WIRE: InterpreterTypes> Interpreter<WIRE> {
    /// 主执行循环
    pub fn run_plain<H: Host + ?Sized>(
        &mut self,
        instruction_table: &InstructionTable<WIRE, H>,
        context: &mut H,
    ) {
        // 简洁的执行循环
        while self.bytecode.is_not_end() {
            self.step(instruction_table, context);
        }
    }

    /// 单步执行
    #[inline(always)]
    fn step<H: Host + ?Sized>(
        &mut self,
        instruction_table: &InstructionTable<WIRE, H>,
        context: &mut H,
    ) {
        // 1. 获取当前操作码
        let opcode = self.bytecode.opcode();

        // 2. 前进 PC
        self.bytecode.relative_jump(1);

        // 3. 查找指令处理器 (unsafe 是因为有 padding 保证)
        let instruction = unsafe {
            instruction_table.get_unchecked(opcode as usize)
        };

        // 4. 检查 gas
        if !self.gas.record_cost(instruction.base_gas()) {
            self.halt(InstructionResult::OutOfGas);
            return;
        }

        // 5. 执行指令
        instruction.execute(self, context);
    }
}
```

**与 go-ethereum 对比：**

```go
// go-ethereum 的执行循环
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) ([]byte, error) {
for {
op := contract.GetOp(pc)
operation := in.table[op]

// 每条指令都检查（无法内联优化）
if !contract.UseGas(operation.constantGas) {
return nil, ErrOutOfGas
}

// 动态分发
res, err := operation.execute(&pc, in, callContext)
// ...
}
}

// revm 的优势：
// 1. #[inline(always)] 强制内联
// 2. 泛型单态化消除虚函数调用
// 3. unsafe get_unchecked 消除边界检查
```

## 4. 关键优化技术

### 4.1 指令表设计

```rust
/// 指令定义
pub struct Instruction<'a, WIRE: InterpreterTypes, H: Host + ?Sized> {
    /// 执行函数
    execute: fn(&mut InstructionContext<'_, H, WIRE>),
    /// 基础 gas 成本 (编译期已知)
    base_gas: u64,
}

/// 指令表 - 256 个槽位
pub struct InstructionTable<WIRE: InterpreterTypes, H: Host + ?Sized> {
    table: [Instruction<'static, WIRE, H>; 256],
}

impl<WIRE: InterpreterTypes, H: Host + ?Sized> InstructionTable<WIRE, H> {
    /// 根据硬分叉版本生成指令表
    pub const fn new(spec: SpecId) -> Self {
        let mut table = [UNKNOWN_INSTRUCTION; 256];

        // 基础指令 (所有版本)
        table[0x00] = Instruction { execute: stop, base_gas: 0 };
        table[0x01] = Instruction { execute: add, base_gas: 3 };
        table[0x02] = Instruction { execute: mul, base_gas: 5 };
        // ...

        // 条件启用 (根据硬分叉)
        if spec >= SpecId::CANCUN {
            table[0x5C] = Instruction { execute: tload, base_gas: 100 };
            table[0x5D] = Instruction { execute: tstore, base_gas: 100 };
            table[0x5E] = Instruction { execute: mcopy, base_gas: 3 };
        }

        Self { table }
    }
}
```

### 4.2 零拷贝字节码分析

```rust
/// 跳转表 - 使用 bitvec 紧凑存储
pub struct JumpTable(BitVec<u64>);

impl JumpTable {
    /// 检查是否为有效跳转目标
    #[inline(always)]
    pub fn is_valid(&self, offset: usize) -> bool {
        self.0.get(offset).map(|b| *b).unwrap_or(false)
    }
}

/// 分析字节码，构建跳转表
pub fn analyze_legacy(bytecode: &[u8]) -> (JumpTable, Bytes) {
    // 使用 bitvec 而非 HashSet
    // 内存效率: 1 bit per byte vs ~40 bytes per entry
    let mut jump_table = BitVec::with_capacity(bytecode.len());

    let mut i = 0;
    while i < bytecode.len() {
        let op = bytecode[i];

        if op == JUMPDEST {
            jump_table.set(i, true);  // O(1) 设置
        }

        // 跳过 PUSH 数据
        if op >= PUSH1 && op <= PUSH32 {
            let push_size = (op - PUSH1 + 1) as usize;
            i += push_size;
        }

        i += 1;
    }

    // 添加 padding 保证安全访问
    let padded = ensure_padding(bytecode);

    (JumpTable(jump_table), padded)
}
```

### 4.3 内存管理优化

```rust
/// 共享内存 - 避免每次调用重新分配
pub struct SharedMemory {
    /// 实际数据
    buffer: Vec<u8>,
    /// 上下文栈 (用于嵌套调用)
    context_stack: Vec<MemoryContext>,
    /// 当前上下文
    current_context: MemoryContext,
}

struct MemoryContext {
    offset: usize,
    len: usize,
}

impl SharedMemory {
    /// 进入新的调用上下文
    pub fn new_context(&mut self) {
        self.context_stack.push(self.current_context);
        self.current_context = MemoryContext {
            offset: self.buffer.len(),
            len: 0,
        };
    }

    /// 退出调用上下文
    pub fn free_context(&mut self) {
        // 只需调整长度，无需释放
        self.buffer.truncate(self.current_context.offset);
        self.current_context = self.context_stack.pop().unwrap();
    }

    /// 调整大小 - 只增不减
    pub fn resize(&mut self, new_size: usize) {
        let required = self.current_context.offset + new_size;
        if required > self.buffer.len() {
            // 预留额外空间，减少重分配
            self.buffer.resize(required.next_power_of_two(), 0);
        }
        self.current_context.len = new_size;
    }
}
```

### 4.4 Gas 计算优化

```rust
/// Gas 结构 - 紧凑高效
#[derive(Clone, Copy)]
pub struct Gas {
    /// 剩余 gas
    remaining: u64,
    /// 累计退款
    refund: i64,
}

impl Gas {
    /// 记录 gas 消耗 - 内联热路径
    #[inline(always)]
    pub fn record_cost(&mut self, cost: u64) -> bool {
        // 使用 checked_sub 而非 if 检查
        // 利用 CPU 的进位标志
        match self.remaining.checked_sub(cost) {
            Some(new_remaining) => {
                self.remaining = new_remaining;
                true
            }
            None => false,  // 冷路径：OOG
        }
    }

    /// 动态 gas 计算 (如内存扩展)
    #[inline]
    pub fn record_memory_expansion(&mut self, current: usize, new: usize) -> bool {
        if new <= current {
            return true;  // 热路径：无扩展
        }

        // 冷路径：计算内存成本
        let words = (new + 31) / 32;
        let cost = 3 * words + words * words / 512;
        self.record_cost(cost as u64)
    }
}
```

## 5. Inspector 系统（追踪与调试）

### 5.1 Inspector Trait

```rust
/// Inspector trait - 执行追踪的核心抽象
pub trait Inspector<CTX, INTR: InterpreterTypes> {
    /// 执行开始
    fn initialize(&mut self, _ctx: &mut CTX) {}

    /// 每条指令执行前
    fn step(&mut self, interp: &mut Interpreter<INTR>, ctx: &mut CTX);

    /// 每条指令执行后
    fn step_end(&mut self, interp: &mut Interpreter<INTR>, ctx: &mut CTX) {}

    /// 调用开始
    fn call(&mut self, ctx: &mut CTX, inputs: &CallInputs) -> Option<CallOutcome> {
        None  // 返回 None 继续正常执行
    }

    /// 调用结束
    fn call_end(&mut self, ctx: &mut CTX, outcome: &mut CallOutcome) {}

    /// 创建开始
    fn create(&mut self, ctx: &mut CTX, inputs: &CreateInputs) -> Option<CreateOutcome> {
        None
    }

    /// 创建结束
    fn create_end(&mut self, ctx: &mut CTX, outcome: &mut CreateOutcome) {}

    /// SELFDESTRUCT
    fn selfdestruct(&mut self, contract: Address, target: Address, value: U256) {}

    /// LOG
    fn log(&mut self, ctx: &mut CTX, log: Log) {}
}
```

### 5.2 内置 Inspector 实现

```rust
/// Gas 追踪器
pub struct GasInspector {
    gas_used: u64,
    gas_remaining: u64,
}

impl<CTX, INTR: InterpreterTypes> Inspector<CTX, INTR> for GasInspector {
    fn step(&mut self, interp: &mut Interpreter<INTR>, _ctx: &mut CTX) {
        self.gas_remaining = interp.gas.remaining();
    }

    fn step_end(&mut self, interp: &mut Interpreter<INTR>, _ctx: &mut CTX) {
        self.gas_used += self.gas_remaining - interp.gas.remaining();
    }
}

/// EIP-3155 标准追踪器
#[cfg(feature = "std")]
pub struct TracerEip3155<W: std::io::Write> {
    writer: W,
    gas_inspector: GasInspector,
    // ...
}

/// 操作计数器
pub struct CountInspector {
    opcode_counts: [u64; 256],
}

impl<CTX, INTR: InterpreterTypes> Inspector<CTX, INTR> for CountInspector {
    fn step(&mut self, interp: &mut Interpreter<INTR>, _ctx: &mut CTX) {
        let opcode = interp.bytecode.opcode();
        self.opcode_counts[opcode as usize] += 1;
    }
}
```

### 5.3 Inspector 组合

```rust
/// 组合多个 Inspector
pub struct InspectorStack<I1, I2> {
    pub first: I1,
    pub second: I2,
}

impl<CTX, INTR, I1, I2> Inspector<CTX, INTR> for InspectorStack<I1, I2>
where
    INTR: InterpreterTypes,
    I1: Inspector<CTX, INTR>,
    I2: Inspector<CTX, INTR>,
{
    fn step(&mut self, interp: &mut Interpreter<INTR>, ctx: &mut CTX) {
        self.first.step(interp, ctx);
        self.second.step(interp, ctx);
    }

    // ... 其他方法类似组合
}

// 使用示例
let inspector = InspectorStack {
    first: GasInspector::new(),
    second: CountInspector::new(),
};
```

## 6. 状态管理（Journal 系统）

### 6.1 Account 状态追踪

```rust
/// 账户状态标志 - 使用 bitflags 紧凑存储
bitflags! {
    pub struct AccountStatus: u8 {
        /// 新创建的账户
        const Created = 0b0000_0001;
        /// 本交易中创建（用于 SELFDESTRUCT 后清理）
        const CreatedLocal = 0b0000_0010;
        /// 已自毁
        const SelfDestructed = 0b0000_0100;
        /// 本交易中自毁
        const SelfDestructedLocal = 0b0000_1000;
        /// 被触碰（需要持久化）
        const Touched = 0b0001_0000;
        /// 数据库中不存在
        const LoadedAsNotExisting = 0b0010_0000;
        /// 冷访问状态（EIP-2929）
        const Cold = 0b0100_0000;
    }
}

/// 账户结构
pub struct Account {
    pub info: AccountInfo,           // 余额、nonce、代码哈希
    pub storage: HashMap<U256, EvmStorageSlot>,
    pub status: AccountStatus,
    pub tx_id: u64,                  // 关联的交易 ID
}

/// 存储槽 - 追踪原始值和当前值
pub struct EvmStorageSlot {
    pub original_value: U256,        // 交易开始时的值
    pub present_value: U256,         // 当前值
    pub is_cold: bool,               // 冷访问标记
}

impl EvmStorageSlot {
    /// 计算 SSTORE gas (EIP-2200)
    pub fn sstore_gas(&self, new_value: U256, is_cold: bool) -> u64 {
        let mut gas = 0;

        // 冷访问成本
        if is_cold {
            gas += 2100;
        }

        // 根据值变化计算成本
        if self.present_value == new_value {
            gas += 100;  // WARM_STORAGE_READ
        } else if self.present_value == self.original_value {
            if self.original_value == U256::ZERO {
                gas += 20000;  // SSTORE_SET
            } else {
                gas += 2900;   // SSTORE_RESET - COLD_SLOAD
            }
        } else {
            gas += 100;  // WARM_STORAGE_READ
        }

        gas
    }
}
```

### 6.2 Journal 回滚机制

```rust
/// Journal Entry - 记录状态变更
pub enum JournalEntry {
    /// 账户被触碰
    AccountTouched(Address),
    /// 账户创建
    AccountCreated(Address),
    /// 余额变更
    BalanceTransfer { from: Address, to: Address, balance: U256 },
    /// 存储变更
    StorageChange {
        address: Address,
        key: U256,
        had_value: Option<U256>
    },
    /// 瞬态存储变更 (EIP-1153)
    TransientStorageChange {
        address: Address,
        key: U256,
        had_value: U256,
    },
    // ...
}

/// Journal - 支持检查点和回滚
pub struct Journal {
    /// 状态快照
    state: HashMap<Address, Account>,
    /// 变更日志
    journal: Vec<JournalEntry>,
    /// 检查点栈
    checkpoints: Vec<usize>,
}

impl Journal {
    /// 创建检查点
    pub fn checkpoint(&mut self) -> usize {
        let id = self.checkpoints.len();
        self.checkpoints.push(self.journal.len());
        id
    }

    /// 提交检查点
    pub fn commit(&mut self) {
        self.checkpoints.pop();
    }

    /// 回滚到检查点
    pub fn revert(&mut self, checkpoint_id: usize) {
        let checkpoint_idx = self.checkpoints[checkpoint_id];

        // 反向应用日志条目
        while self.journal.len() > checkpoint_idx {
            let entry = self.journal.pop().unwrap();
            self.revert_entry(entry);
        }

        self.checkpoints.truncate(checkpoint_id);
    }

    fn revert_entry(&mut self, entry: JournalEntry) {
        match entry {
            JournalEntry::StorageChange { address, key, had_value } => {
                let account = self.state.get_mut(&address).unwrap();
                match had_value {
                    Some(value) => account.storage.insert(key, value),
                    None => account.storage.remove(&key),
                };
            }
            JournalEntry::BalanceTransfer { from, to, balance } => {
                self.state.get_mut(&from).unwrap().info.balance += balance;
                self.state.get_mut(&to).unwrap().info.balance -= balance;
            }
            // ... 其他条目
        }
    }
}
```

## 7. 高级 Rust 特性应用

### 7.1 条件编译与 Feature Flags

```rust
// Cargo.toml 中定义的 features
[features]
default = ["std"]
std = ["alloy-primitives/std", "bitvec/std"]
serde = ["alloy-primitives/serde", "dep:serde"]
arbitrary = ["alloy-primitives/arbitrary"]
optimism = ["op-revm"]
c-kzg = ["dep:c-kzg"]

// 代码中的条件编译
#[cfg(feature = "std")]
use std::collections::HashMap;

#[cfg(not(feature = "std"))]
use hashbrown::HashMap;  // no_std 兼容

#[cfg(feature = "serde")]
impl serde::Serialize for Account { ... }

// no_std 入口
#![cfg_attr(not(feature = "std"), no_std)]
extern crate alloc;  // 使用 alloc crate
```

### 7.2 内联控制

```rust
// 强制内联热路径
#[inline(always)]
pub fn add(ctx: &mut InstructionContext<'_, impl Host, impl InterpreterTypes>) {
    let [a, b] = ctx.interpreter.stack.pop2();
    ctx.interpreter.stack.push(a.wrapping_add(b));
}

// 避免内联冷路径
#[inline(never)]
#[cold]
fn handle_out_of_gas(state: &mut ExecutionState) {
    state.status = EVMC_OUT_OF_GAS;
    // 错误处理...
}

// 条件内联
#[cfg_attr(not(debug_assertions), inline(always))]
pub fn sload(ctx: &mut InstructionContext<'_, impl Host, impl InterpreterTypes>) {
    // Release 模式内联，Debug 模式不内联（便于调试）
}
```

### 7.3 零成本抽象示例

```rust
// Trait 对象 vs 泛型
//
// 动态分发（有开销）：
fn execute_dyn(host: &dyn Host, interp: &mut dyn InterpreterTrait) {
    // 每次调用都有虚表查找
}

// 静态分发（零成本）：
fn execute<H: Host, I: InterpreterTypes>(host: &H, interp: &mut Interpreter<I>) {
    // 编译时单态化，直接调用
}

// revm 的选择：泛型 + 单态化
impl<WIRE: InterpreterTypes, H: Host> Interpreter<WIRE> {
    pub fn run(&mut self, table: &InstructionTable<WIRE, H>, host: &mut H) {
        // 每种 (WIRE, H) 组合生成独立的机器码
        // 无虚函数开销
    }
}
```

### 7.4 unsafe 的谨慎使用

```rust
impl<WIRE: InterpreterTypes, H: Host> Interpreter<WIRE> {
    fn step(&mut self, table: &InstructionTable<WIRE, H>, host: &mut H) {
        let opcode = self.bytecode.opcode();

        // SAFETY: 字节码分析确保了 padding
        // 任何 opcode (0-255) 都有有效条目
        let instruction = unsafe {
            table.get_unchecked(opcode as usize)
        };

        instruction.execute(self, host);
    }
}

// 为什么安全？
// 1. InstructionTable 始终有 256 个条目
// 2. opcode 是 u8，范围 0-255
// 3. 字节码分析添加了 padding 防止越界读取

// 传统安全写法（有边界检查开销）：
let instruction = table.get(opcode as usize).unwrap();

// unsafe 版本消除了：
// 1. 边界检查分支
// 2. Option 解包
```

### 7.5 PhantomData 与类型标记

```rust
use std::marker::PhantomData;

/// 编译期区分不同网络
pub struct NetworkMarker<N>(PhantomData<N>);

pub struct Mainnet;
pub struct Optimism;

/// 网络特定的 EVM
pub struct Evm<N, DB: Database> {
    database: DB,
    _network: PhantomData<N>,
}

impl<DB: Database> Evm<Mainnet, DB> {
    pub fn execute(&mut self, tx: &Transaction) -> Result<Output, Error> {
        // Mainnet 特定逻辑
    }
}

impl<DB: Database> Evm<Optimism, DB> {
    pub fn execute(&mut self, tx: &Transaction) -> Result<Output, Error> {
        // Optimism 特定逻辑（L1 数据费用等）
    }
}

// PhantomData 的好处：
// 1. 零运行时开销
// 2. 编译期类型安全
// 3. 防止错误的网络配置
```

## 8. 与 go-ethereum / evmone 对比

### 8.1 设计理念对比

| 方面       | revm    | go-ethereum | evmone  |
|----------|---------|-------------|---------|
| **核心目标** | 通用库     | 完整客户端       | 高性能引擎   |
| **可扩展性** | ★★★★★   | ★★☆☆☆       | ★★★☆☆   |
| **性能**   | ★★★★☆   | ★★★☆☆       | ★★★★★   |
| **易用性**  | ★★★★☆   | ★★★☆☆       | ★★☆☆☆   |
| **生态整合** | Rust 生态 | Go 生态       | EVMC 标准 |

### 8.2 技术实现对比

```
┌────────────────────────────────────────────────────────────────┐
│                    执行循环对比                                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  go-ethereum:                                                  │
│  for {                                                         │
│      op := code[pc]                                            │
│      operation := table[op]        // 间接查找                 │
│      if !useGas(operation.gas) {   // 每条指令检查             │
│          return                                                │
│      }                                                         │
│      operation.execute(...)        // 函数指针调用             │
│  }                                                             │
│                                                                │
│  evmone (Advanced):                                            │
│  while (instr != nullptr) {                                    │
│      instr = instr->fn(instr, state);  // 直接函数指针         │
│  }                                                             │
│  // + 基本块批量 gas 检查                                       │
│                                                                │
│  revm:                                                         │
│  while bytecode.is_not_end() {                                 │
│      let op = bytecode.opcode();                               │
│      let instr = table[op];        // 内联 + 零边界检查        │
│      if !gas.record(instr.gas) { ... }                         │
│      instr.execute(ctx);           // 单态化调用               │
│  }                                                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 8.3 性能基准

基于 [benchmark issue](https://github.com/bluealloy/revm/issues/7) 的历史数据：

| 实现                | 执行时间        | 相对性能            |
|-------------------|-------------|-----------------|
| evmone (Advanced) | ~46-51 μs   | 1.0x (baseline) |
| evmone (Baseline) | ~55-60 μs   | ~1.15x          |
| **revm**          | ~48-58 μs   | ~1.05x          |
| go-ethereum       | ~150-200 μs | ~3.5x           |

**注**：实际性能取决于具体合约和测试环境。

### 8.4 架构特性对比

```rust
// revm 的独特优势

// 1. 类型安全的 Inspector 系统
trait Inspector<CTX, INTR> {
    fn step(&mut self, interp: &mut Interpreter<INTR>, ctx: &mut CTX);
}
// vs go-ethereum 的 Tracer 接口（运行时错误）

// 2. 泛型化组件替换
struct CustomInterpreter;
impl InterpreterTypes for CustomInterpreter {
    type Stack = MyStack;
    type Memory = MyMemory;
    // ...
}
// vs evmone 的固定实现

// 3. no_std 支持
#![no_std]
// 可用于 zkVM、嵌入式等环境

// 4. 编译期规范检查
const fn validate_spec(spec: SpecId) -> bool {
    // 编译期验证硬分叉兼容性
}
```

## 9. 使用示例

### 9.1 基本执行

```rust
use revm::{MainnetEvm, ExecuteEvm};
use revm::primitives::{Address, U256, TxKind};

fn main() {
    // 创建 EVM 实例
    let mut evm = MainnetEvm::new();

    // 设置区块环境
    evm.block_mut().number = 1000000;
    evm.block_mut().timestamp = 1234567890;
    evm.block_mut().basefee = U256::from(1_000_000_000);

    // 设置交易
    evm.tx_mut().caller = Address::from_slice(&[0x01; 20]);
    evm.tx_mut().kind = TxKind::Call(Address::from_slice(&[0x02; 20]));
    evm.tx_mut().value = U256::from(1_000_000_000_000_000_000u64);
    evm.tx_mut().gas_limit = 21000;

    // 执行
    let result = evm.transact();

    match result {
        Ok(outcome) => {
            println!("Gas used: {}", outcome.gas_used());
            println!("Success: {}", outcome.is_success());
        }
        Err(e) => println!("Error: {:?}", e),
    }
}
```

### 9.2 带 Inspector 的执行

```rust
use revm::{InspectEvm, Inspector};
use revm::interpreter::{Interpreter, InterpreterTypes};

struct MyTracer {
    steps: u64,
}

impl<CTX, INTR: InterpreterTypes> Inspector<CTX, INTR> for MyTracer {
    fn step(&mut self, interp: &mut Interpreter<INTR>, _ctx: &mut CTX) {
        self.steps += 1;
        let opcode = interp.bytecode.opcode();
        println!("Step {}: opcode 0x{:02x}", self.steps, opcode);
    }
}

fn main() {
    let tracer = MyTracer { steps: 0 };
    let mut evm = InspectEvm::new_with_inspector(tracer);

    // 配置并执行...
    let result = evm.transact();

    println!("Total steps: {}", evm.inspector().steps);
}
```

### 9.3 自定义 Database

```rust
use revm::{Database, DatabaseRef};
use revm::primitives::{Address, AccountInfo, Bytecode, B256, U256};

struct MyDatabase {
    accounts: HashMap<Address, AccountInfo>,
    storage: HashMap<(Address, U256), U256>,
}

impl Database for MyDatabase {
    type Error = MyError;

    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error> {
        Ok(self.accounts.get(&address).cloned())
    }

    fn storage(&mut self, address: Address, slot: U256) -> Result<U256, Self::Error> {
        Ok(self.storage.get(&(address, slot)).copied().unwrap_or_default())
    }

    fn code_by_hash(&mut self, _hash: B256) -> Result<Bytecode, Self::Error> {
        // 实现代码查询
        Ok(Bytecode::default())
    }

    fn block_hash(&mut self, _number: u64) -> Result<B256, Self::Error> {
        Ok(B256::ZERO)
    }
}
```

## 10. 总结

### revm 的核心优势

1. **极致的模块化**: 通过 Rust trait 系统实现组件级可替换
2. **类型安全**: 编译期捕获错误，泛型零成本抽象
3. **生态整合**: 与 Alloy、Foundry 等 Rust 工具链深度集成
4. **灵活性**: 支持 no_std、自定义 Inspector、可插拔 Database

### 适用场景

| 场景             | 推荐度   | 原因                 |
|----------------|-------|--------------------|
| MEV 搜索/模拟      | ★★★★★ | 高性能 + Inspector 系统 |
| 开发工具 (Foundry) | ★★★★★ | 深度集成 + 调试支持        |
| L2/自定义链        | ★★★★★ | 模块化 + op-revm      |
| zkEVM          | ★★★★☆ | no_std + 可追踪       |
| 高频交易客户端        | ★★★★☆ | 性能接近 evmone        |
| 完整节点           | ★★★☆☆ | Reth 已集成           |

### 与其他实现的定位

```
                高性能 ◄─────────────────────────────► 高可扩展性
                   │                                      │
                   │          evmone                      │
                   │            ●                         │
                   │                                      │
                   │                    revm              │
                   │                      ●               │
                   │                                      │
                   │    go-ethereum                       │
                   │        ●                             │
                   │                                      │
                   └──────────────────────────────────────┘
```

revm 在性能和可扩展性之间取得了优秀的平衡，是 Rust 生态系统中构建 EVM 相关工具的首选。

---

## Appendix C: EOF (EVM Object Format) 支持

### C.1 revm 的 EOF 实现

revm 在 `bytecode` crate 中实现了 EOF 支持：

```rust
// crates/bytecode/src/eof/mod.rs

/// EOF 容器结构
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct Eof {
    /// EOF 头部
    pub header: EofHeader,
    /// 函数类型段
    pub body: EofBody,
    /// 原始字节码
    pub raw: Bytes,
}

/// EOF 头部
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct EofHeader {
    /// 类型段大小
    pub types_size: u16,
    /// 代码段大小数组
    pub code_sizes: Vec<u16>,
    /// 容器段大小数组
    pub container_sizes: Vec<u16>,
    /// 数据段大小
    pub data_size: u16,
    /// 头部总大小
    pub sum_code_sizes: usize,
    pub sum_container_sizes: usize,
}

/// EOF 体
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct EofBody {
    /// 函数类型信息
    pub types_section: Vec<TypesSection>,
    /// 代码段
    pub code_section: Vec<Bytes>,
    /// 嵌套容器
    pub container_section: Vec<Bytes>,
    /// 数据段
    pub data_section: Bytes,
    /// 是否数据已填充
    pub is_data_filled: bool,
}

/// 类型段（函数签名）
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct TypesSection {
    /// 输入栈元素数
    pub inputs: u8,
    /// 输出栈元素数
    pub outputs: u8,
    /// 最大栈高度
    pub max_stack_size: u16,
}
```

### C.2 EOF 验证

```rust
/// EOF 验证结果
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum EofValidationError {
    /// 无效的 magic 字节
    InvalidMagic,
    /// 无效的版本
    InvalidVersion,
    /// 头部解析失败
    InvalidHeader(EofHeaderError),
    /// 代码验证失败
    InvalidCode(EofCodeError),
    /// 控制流验证失败
    InvalidControlFlow,
}

impl Eof {
    /// 解析并验证 EOF 字节码
    pub fn decode(raw: Bytes) -> Result<Self, EofDecodeError> {
        // 1. 检查 magic (0xEF00)
        if raw.len() < 2 || raw[0] != 0xEF || raw[1] != 0x00 {
            return Err(EofDecodeError::InvalidMagic);
        }

        // 2. 检查版本 (0x01)
        if raw.len() < 3 || raw[2] != 0x01 {
            return Err(EofDecodeError::InvalidVersion);
        }

        // 3. 解析头部
        let header = EofHeader::decode(&raw[3..])?;

        // 4. 解析体
        let body = EofBody::decode(&raw, &header)?;

        Ok(Self { header, body, raw })
    }

    /// 验证 EOF 字节码
    pub fn validate(&self, spec: SpecId) -> Result<(), EofValidationError> {
        // 验证类型段
        self.validate_types()?;

        // 验证每个代码段
        for (i, code) in self.body.code_section.iter().enumerate() {
            self.validate_code_section(i, code, spec)?;
        }

        // 验证控制流
        self.validate_control_flow()?;

        Ok(())
    }
}
```

### C.3 EOF 执行支持

```rust
// EOF 特有指令实现

/// RJUMP - 相对跳转
pub fn rjump<WIRE: InterpreterTypes, H: Host + ?Sized>(
    ctx: &mut InstructionContext<'_, H, WIRE>,
) {
    // 读取 16 位有符号偏移
    let offset = ctx.interpreter.bytecode.read_i16();
    // 直接跳转，无需验证（部署时已验证）
    ctx.interpreter.bytecode.relative_jump(offset as isize);
}

/// RJUMPI - 条件相对跳转
pub fn rjumpi<WIRE: InterpreterTypes, H: Host + ?Sized>(
    ctx: &mut InstructionContext<'_, H, WIRE>,
) {
    let offset = ctx.interpreter.bytecode.read_i16();
    popn!([condition], ctx.interpreter);

    if condition != U256::ZERO {
        ctx.interpreter.bytecode.relative_jump(offset as isize);
    }
}

/// CALLF - 调用函数
pub fn callf<WIRE: InterpreterTypes, H: Host + ?Sized>(
    ctx: &mut InstructionContext<'_, H, WIRE>,
) {
    let idx = ctx.interpreter.bytecode.read_u16() as usize;

    // 获取函数类型信息
    let types = &ctx.interpreter.eof.body.types_section[idx];

    // 保存返回地址
    ctx.interpreter.function_stack.push(FunctionFrame {
        return_pc: ctx.interpreter.bytecode.pc(),
        inputs: types.inputs,
    });

    // 跳转到函数代码
    let code_offset = ctx.interpreter.eof.code_offset(idx);
    ctx.interpreter.bytecode.absolute_jump(code_offset);
}

/// RETF - 函数返回
pub fn retf<WIRE: InterpreterTypes, H: Host + ?Sized>(
    ctx: &mut InstructionContext<'_, H, WIRE>,
) {
    // 弹出函数栈帧
    let frame = ctx.interpreter.function_stack.pop().unwrap();

    // 返回到调用点
    ctx.interpreter.bytecode.absolute_jump(frame.return_pc);
}

/// DATALOAD - 从数据段加载
pub fn dataload<WIRE: InterpreterTypes, H: Host + ?Sized>(
    ctx: &mut InstructionContext<'_, H, WIRE>,
) {
    popn!([offset], ctx.interpreter);
    let offset = as_usize_saturated!(offset);

    let data = &ctx.interpreter.eof.body.data_section;
    let value = if offset < data.len() {
        let end = (offset + 32).min(data.len());
        let mut buf = [0u8; 32];
        buf[..end - offset].copy_from_slice(&data[offset..end]);
        U256::from_be_bytes(buf)
    } else {
        U256::ZERO
    };

    ctx.interpreter.stack.push(value);
}
```

### C.4 EOF vs Legacy 对比

| 方面         | Legacy            | EOF (revm)        |
|------------|-------------------|-------------------|
| **字节码分析**  | 每次执行都需要           | 部署时一次性            |
| **跳转验证**   | 运行时查找             | 编译期保证             |
| **函数调用**   | 模拟（JUMP）          | 原生支持 (CALLF/RETF) |
| **数据访问**   | CODECOPY          | DATALOAD          |
| **Gas 观察** | 支持                | 移除                |
| **代码自省**   | CODESIZE/CODECOPY | 移除                |

---

## Appendix D: Database Trait 详解

### D.1 核心 Database Trait

revm 的 Database trait 是与外部状态交互的核心接口：

```rust
// crates/database-interface/src/lib.rs

/// 数据库接口 - 可变引用版本
pub trait Database {
    /// 错误类型
    type Error;

    /// 获取账户基本信息
    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error>;

    /// 获取代码（通过哈希）
    fn code_by_hash(&mut self, code_hash: B256) -> Result<Bytecode, Self::Error>;

    /// 获取存储值
    fn storage(&mut self, address: Address, index: U256) -> Result<U256, Self::Error>;

    /// 获取区块哈希
    fn block_hash(&mut self, number: u64) -> Result<B256, Self::Error>;
}

/// 数据库接口 - 不可变引用版本
pub trait DatabaseRef {
    type Error;

    fn basic_ref(&self, address: Address) -> Result<Option<AccountInfo>, Self::Error>;
    fn code_by_hash_ref(&self, code_hash: B256) -> Result<Bytecode, Self::Error>;
    fn storage_ref(&self, address: Address, index: U256) -> Result<U256, Self::Error>;
    fn block_hash_ref(&self, number: u64) -> Result<B256, Self::Error>;
}

/// 自动从 DatabaseRef 实现 Database
impl<T: DatabaseRef> Database for &T {
    type Error = T::Error;

    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error> {
        self.basic_ref(address)
    }
    // ...
}

/// 提交状态变更
pub trait DatabaseCommit {
    fn commit(&mut self, changes: HashMap<Address, Account>);
}
```

### D.2 内置 Database 实现

```rust
// 1. InMemoryDB - 内存数据库
pub struct InMemoryDB {
    pub accounts: HashMap<Address, DbAccount>,
    pub contracts: HashMap<B256, Bytecode>,
    pub block_hashes: HashMap<u64, B256>,
}

impl Database for InMemoryDB {
    type Error = Infallible;  // 永不失败

    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error> {
        Ok(self.accounts.get(&address).map(|a| a.info.clone()))
    }

    fn storage(&mut self, address: Address, index: U256) -> Result<U256, Self::Error> {
        Ok(self.accounts
            .get(&address)
            .and_then(|a| a.storage.get(&index))
            .copied()
            .unwrap_or_default())
    }
    // ...
}

// 2. CacheDB - 带缓存的数据库包装
pub struct CacheDB<ExtDB: Database> {
    pub accounts: HashMap<Address, CacheAccount>,
    pub contracts: HashMap<B256, Bytecode>,
    pub block_hashes: HashMap<u64, B256>,
    pub db: ExtDB,  // 底层数据库
}

impl<ExtDB: Database> CacheDB<ExtDB> {
    /// 从底层加载并缓存
    pub fn load_account(&mut self, address: Address) -> Result<&mut CacheAccount, ExtDB::Error> {
        match self.accounts.entry(address) {
            Entry::Occupied(entry) => Ok(entry.into_mut()),
            Entry::Vacant(entry) => {
                // 缓存未命中，从底层加载
                let info = self.db.basic(address)?;
                Ok(entry.insert(CacheAccount::new(info)))
            }
        }
    }
}

// 3. AlloyDB - 连接 Alloy Provider
#[cfg(feature = "alloydb")]
pub struct AlloyDB<T: Transport + Clone, N: Network, P: Provider<T, N>> {
    provider: P,
    block_id: BlockId,
    _phantom: PhantomData<(T, N)>,
}

impl<T, N, P> Database for AlloyDB<T, N, P>
where
    T: Transport + Clone,
    N: Network,
    P: Provider<T, N>,
{
    type Error = DBTransportError;

    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error> {
        // 通过 RPC 获取账户信息
        let (nonce, balance, code) = tokio::runtime::Handle::current()
            .block_on(async {
                tokio::join!(
                    self.provider.get_transaction_count(address).block_id(self.block_id),
                    self.provider.get_balance(address).block_id(self.block_id),
                    self.provider.get_code_at(address).block_id(self.block_id),
                )
            });

        Ok(Some(AccountInfo {
            nonce: nonce?,
            balance: balance?,
            code_hash: keccak256(&code?),
            code: Some(Bytecode::new_raw(code?.into())),
        }))
    }
    // ...
}
```

### D.3 状态管理层次

```
┌────────────────────────────────────────────────────────────────┐
│                    revm 状态管理架构                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Journal 层                            │  │
│  │  ┌──────────────────────────────────────────────────┐    │  │
│  │  │ 交易级状态追踪                                     │    │  │
│  │  │ • checkpoint / commit / revert                   │    │ │
│  │  │ • 状态变更日志                                     │    │ │
│  │  │ • Gas 退款追踪                                     │   │ │
│  │  └──────────────────────────────────────────────────┘    │ │
│  └────────────────────────────┬─────────────────────────────┘ │
│                               │                               │
│  ┌────────────────────────────▼─────────────────────────────┐ │
│  │                   CacheDB 层                             │ │
│  │  ┌──────────────────────────────────────────────────┐    │ │
│  │  │ 账户/存储缓存                                      │    │ │
│  │  │ • 减少底层数据库访问                                │    │ │
│  │  │ • 热数据保持在内存                                  │   │ │
│  │  └──────────────────────────────────────────────────┘    │ │
│  └────────────────────────────┬─────────────────────────────┘ │
│                               │                               │
│  ┌────────────────────────────▼─────────────────────────────┐ │
│  │                  底层 Database                            │ │
│  │  ┌──────────┐  ┌─────────┐  ┌──────────────────────────┐ │ │
│  │  │InMemoryDB│  │ AlloyDB │  │ 自定义实现                 │ │ │
│  │  │(测试用)   │  │ (RPC)   │  │ (LevelDB, RocksDB, ...)  │ │ │
│  │  └──────────┘  └─────────┘  └──────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### D.4 BundleState - 批量状态变更

```rust
/// Bundle 状态 - 用于批量处理多个交易
pub struct BundleState {
    /// 账户状态变更
    pub state: HashMap<Address, BundleAccount>,
    /// 合约代码
    pub contracts: HashMap<B256, Bytecode>,
    /// 撤销信息（用于回滚整个 bundle）
    pub reverts: Vec<HashMap<Address, AccountRevert>>,
}

/// Bundle 账户
pub struct BundleAccount {
    pub info: Option<AccountInfo>,
    pub original_info: Option<AccountInfo>,
    pub storage: HashMap<U256, StorageSlot>,
    pub status: AccountStatus,
}

impl BundleState {
    /// 应用单个交易的状态变更
    pub fn apply_transitions(&mut self, transitions: Vec<(Address, TransitionAccount)>) {
        for (address, transition) in transitions {
            let account = self.state.entry(address).or_default();
            account.apply_transition(transition);
        }
    }

    /// 合并另一个 BundleState
    pub fn extend(&mut self, other: BundleState) {
        for (address, account) in other.state {
            match self.state.entry(address) {
                Entry::Occupied(mut entry) => {
                    entry.get_mut().extend(account);
                }
                Entry::Vacant(entry) => {
                    entry.insert(account);
                }
            }
        }
    }

    /// 计算状态根（用于验证）
    pub fn state_root(&self) -> B256 {
        // 实际实现需要 MPT
        todo!("Requires Merkle Patricia Trie")
    }
}
```

---

## Appendix E: op-revm (Optimism 变体)

### E.1 项目概述

[op-revm](https://github.com/op-rs/op-revm) 是 revm 的 Optimism L2 变体，处理 L1 数据成本和 Optimism 特有逻辑。

```
┌────────────────────────────────────────────────────────────────┐
│                  Optimism 费用结构                              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  总费用 = L2 执行费用 + L1 数据费用 + 运营商费用                    │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ L2 执行费用                                               │  │
│  │ = gas_used × base_fee                                    │  │
│  │ (与 Ethereum 相同)                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ L1 数据费用（Optimism 特有）                                │  │
│  │ = compressed_tx_size × l1_base_fee × scalar               │ │
│  │                                                           │ │
│  │ Ecotone 后 (EIP-4844):                                    │ │
│  │ = compressed_size × (base_fee_scalar × l1_base_fee        │ │
│  │                     + blob_base_fee_scalar × l1_blob_fee) │ │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 运营商费用（Isthmus 后）                                    │  │
│  │ = operator_fee_constant + operator_fee_scalar × gas_used │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### E.2 核心组件

```rust
// crates/op-revm/src/lib.rs

/// Optimism EVM
pub type OpEvm<DB, I> = Evm<OpContext<DB>, I>;

/// Optimism 上下文
pub struct OpContext<DB: Database> {
    pub inner: Context<DB>,
    pub l1_block_info: L1BlockInfo,
}

/// L1 区块信息（从系统合约读取）
#[derive(Clone, Debug, Default)]
pub struct L1BlockInfo {
    /// L1 基础费用
    pub l1_base_fee: U256,
    /// L1 费用开销
    pub l1_fee_overhead: U256,
    /// L1 费用标量
    pub l1_base_fee_scalar: U256,
    /// Blob 基础费用（Ecotone 后）
    pub blob_base_fee: Option<U256>,
    /// Blob 费用标量（Ecotone 后）
    pub blob_base_fee_scalar: Option<U256>,
    /// 运营商费用常量（Isthmus 后）
    pub operator_fee_constant: Option<U256>,
    /// 运营商费用标量（Isthmus 后）
    pub operator_fee_scalar: Option<U256>,
}
```

### E.3 L1 数据费用计算

```rust
// crates/op-revm/src/l1block.rs

impl L1BlockInfo {
    /// 从系统合约地址加载 L1 信息
    /// 地址: 0x4200000000000000000000000000000000000015
    pub fn load_from_system_contract<DB: Database>(
        db: &mut DB,
        spec: OpSpec,
    ) -> Result<Self, DB::Error> {
        let l1_block_address = address!("4200000000000000000000000000000000000015");

        // 读取各个存储槽
        let l1_base_fee = db.storage(l1_block_address, U256::from(1))?;
        let l1_fee_overhead = db.storage(l1_block_address, U256::from(5))?;
        let l1_base_fee_scalar = db.storage(l1_block_address, U256::from(6))?;

        let mut info = L1BlockInfo {
            l1_base_fee,
            l1_fee_overhead,
            l1_base_fee_scalar,
            ..Default::default()
        };

        // Ecotone 后读取 blob 费用
        if spec >= OpSpec::ECOTONE {
            info.blob_base_fee = Some(db.storage(l1_block_address, U256::from(7))?);
            info.blob_base_fee_scalar = Some(db.storage(l1_block_address, U256::from(8))?);
        }

        Ok(info)
    }

    /// 计算 L1 数据费用
    pub fn calculate_l1_data_fee(&self, tx: &OpTransaction, spec: OpSpec) -> U256 {
        // 获取压缩后的交易大小
        let compressed_size = self.compressed_tx_size(tx);

        if spec >= OpSpec::ECOTONE {
            // Ecotone 后: 使用 blob 费用
            self.ecotone_l1_cost(compressed_size)
        } else if spec >= OpSpec::REGOLITH {
            // Regolith: 修正了费用计算
            self.regolith_l1_cost(compressed_size)
        } else {
            // Bedrock: 原始计算
            self.bedrock_l1_cost(compressed_size)
        }
    }

    /// Ecotone L1 成本计算
    fn ecotone_l1_cost(&self, compressed_size: u64) -> U256 {
        // cost = compressed_size * (
        //     base_fee_scalar * l1_base_fee / 16e6
        //     + blob_base_fee_scalar * blob_base_fee / 16e6
        // )
        let base_fee_contribution = self.l1_base_fee_scalar
            * self.l1_base_fee
            / U256::from(16_000_000);

        let blob_fee_contribution = self.blob_base_fee_scalar
            .unwrap_or_default()
            * self.blob_base_fee.unwrap_or_default()
            / U256::from(16_000_000);

        U256::from(compressed_size) * (base_fee_contribution + blob_fee_contribution)
    }

    /// 使用 FastLZ 计算压缩大小
    fn compressed_tx_size(&self, tx: &OpTransaction) -> u64 {
        let raw = tx.rlp_encode();
        // FastLZ 压缩估算
        fast_lz::compress_len(&raw) as u64
    }
}
```

### E.4 Handler 定制

```rust
// op-revm 的 Handler 定制

impl<DB: Database> OpHandler<DB> {
    /// 预执行：扣除 L1 费用
    fn pre_execution(&mut self, tx: &OpTransaction) -> Result<(), OpError> {
        // 1. 加载 L1 区块信息
        self.l1_block_info = L1BlockInfo::load_from_system_contract(
            &mut self.db,
            self.spec,
        )?;

        // 2. 计算 L1 数据费用
        let l1_fee = self.l1_block_info.calculate_l1_data_fee(tx, self.spec);

        // 3. 从发送方扣除 L1 费用
        let sender = self.state.get_mut(&tx.caller)?;
        sender.info.balance = sender.info.balance
            .checked_sub(l1_fee)
            .ok_or(OpError::InsufficientBalance)?;

        // 4. L1 费用发送到费用接收地址
        let fee_recipient = self.get_fee_recipient();
        let recipient = self.state.get_mut(&fee_recipient)?;
        recipient.info.balance += l1_fee;

        Ok(())
    }

    /// 后执行：处理 Optimism 特有逻辑
    fn post_execution(&mut self, result: &ExecutionResult) -> Result<(), OpError> {
        // Isthmus 后：处理运营商费用
        if self.spec >= OpSpec::ISTHMUS {
            self.handle_operator_fee(result)?;
        }

        // Regolith 后：特殊的存款交易处理
        if tx.is_deposit() && self.spec >= OpSpec::REGOLITH {
            self.handle_deposit_receipt(result)?;
        }

        Ok(())
    }
}
```

### E.5 Optimism 特有指令

```rust
// Optimism 预编译合约

/// L1 区块信息预编译
/// 地址: 0x4200000000000000000000000000000000000015
pub fn l1_block_info_precompile(input: &[u8]) -> PrecompileResult {
    // 直接从存储读取 L1 信息
    // 比 SLOAD 更便宜
    Ok(PrecompileOutput {
        gas_used: 100,
        bytes: encode_l1_block_info(),
    })
}

/// Gas Price Oracle 预编译
/// 地址: 0x420000000000000000000000000000000000000F
pub fn gas_price_oracle_precompile(input: &[u8], l1_info: &L1BlockInfo) -> PrecompileResult {
    match input[0..4] {
        // getL1Fee(bytes)
        [0x49, 0x94, 0x8e, 0x0e] => {
            let tx_data = &input[4..];
            let l1_fee = l1_info.calculate_l1_data_fee_for_bytes(tx_data);
            Ok(PrecompileOutput {
                gas_used: 1000,
                bytes: l1_fee.to_be_bytes_vec(),
            })
        }
        // getL1GasUsed(bytes)
        [0xde, 0x26, 0xc4, 0xa1] => {
            let tx_data = &input[4..];
            let gas_used = l1_info.l1_gas_used(tx_data);
            Ok(PrecompileOutput {
                gas_used: 1000,
                bytes: gas_used.to_be_bytes_vec(),
            })
        }
        _ => Err(PrecompileError::InvalidInput),
    }
}
```

### E.6 使用示例

```rust
use op_revm::{OpBuilder, OpContext, OpSpec};

fn execute_optimism_tx() {
    // 1. 创建 Optimism EVM
    let mut evm = OpBuilder::new()
        .with_db(my_database)
        .with_spec(OpSpec::ECOTONE)
        .build();

    // 2. 设置 L2 区块信息
    evm.block_mut().number = 1000000;
    evm.block_mut().timestamp = 1700000000;

    // 3. L1 信息会自动从系统合约加载

    // 4. 执行交易
    let result = evm.transact(tx)?;

    // 5. 获取费用明细
    println!("L2 execution gas: {}", result.gas_used);
    println!("L1 data fee: {}", evm.l1_data_fee());
    println!("Total cost: {}", result.total_cost());
}
```

### E.7 op-revm vs 标准 revm

| 方面       | revm        | op-revm                 |
|----------|-------------|-------------------------|
| **费用模型** | L2 gas only | L2 gas + L1 data        |
| **系统合约** | 无           | L1Block, GasPriceOracle |
| **交易类型** | 标准          | + Deposit 交易            |
| **预编译**  | 标准          | + Optimism 特有           |
| **Spec** | SpecId      | OpSpec (额外硬分叉)          |

---

## 附录 F: Testing Infrastructure

### F.1 execution-spec-tests 集成

revm 与以太坊官方 execution-spec-tests 完全兼容：

```rust
// revm-test-runner 架构
pub struct StateTestRunner {
    test_path: PathBuf,
    evm_builder: EvmBuilder,
}

impl StateTestRunner {
    pub fn run_test(&self, test: &StateTest) -> TestResult {
        for (fork, post_state) in &test.post {
            // 1. 设置 spec
            let spec = fork_to_spec_id(fork);

            // 2. 初始化 pre-state
            let mut db = InMemoryDB::default();
            for (addr, account) in &test.pre {
                db.insert_account_info(*addr, account.into());
                for (slot, value) in &account.storage {
                    db.insert_account_storage(*addr, *slot, *value)?;
                }
            }

            // 3. 执行交易
            let mut evm = Evm::builder()
                .with_db(&mut db)
                .with_spec_id(spec)
                .with_tx((&test.transaction).into())
                .with_block_env((&test.env).into())
                .build();

            let result = evm.transact_commit()?;

            // 4. 验证 post-state
            let state_root = db.state_root();
            assert_eq!(state_root, post_state.hash);

            // 5. 验证日志 hash
            if let Some(logs_hash) = post_state.logs {
                let computed = keccak256(rlp_encode_logs(&result.logs));
                assert_eq!(computed, logs_hash);
            }
        }

        TestResult::Pass
    }
}
```

### F.2 Differential Fuzzing

revm 被广泛用于差分模糊测试：

```rust
// EVMFuzz 风格的差分测试
pub struct DifferentialFuzzer {
    revm: Evm<'static, (), InMemoryDB>,
    geth_rpc: GethRpcClient,
    evmone: EvmoneWrapper,
}

impl DifferentialFuzzer {
    pub fn fuzz_transaction(&mut self, tx: &FuzzTransaction) -> FuzzResult {
        // 同步初始状态到所有 EVM
        self.sync_state()?;

        // 并行执行
        let (revm_result, geth_result, evmone_result) = rayon::join(
            || self.execute_revm(tx),
            || rayon::join(
                || self.execute_geth(tx),
                || self.execute_evmone(tx),
            ),
        );

        // 比较结果
        self.compare_results(revm_result, geth_result.0, evmone_result.1)
    }

    fn compare_results(&self, r1: ExecResult, r2: ExecResult, r3: ExecResult) -> FuzzResult {
        // 比较 gas 使用
        if r1.gas_used != r2.gas_used || r2.gas_used != r3.gas_used {
            return FuzzResult::GasMismatch {
                revm: r1.gas_used,
                geth: r2.gas_used,
                evmone: r3.gas_used,
            };
        }

        // 比较输出
        if r1.output != r2.output {
            return FuzzResult::OutputMismatch { revm: r1, geth: r2 };
        }

        // 比较状态变更
        if r1.state_changes != r2.state_changes {
            return FuzzResult::StateMismatch { revm: r1, geth: r2 };
        }

        FuzzResult::Consistent
    }
}
```

### F.3 Property-Based Testing

利用 Rust 的 proptest 进行属性测试：

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_arithmetic_operations(
        a in any::<U256>(),
        b in any::<U256>(),
    ) {
        let mut interp = Interpreter::new(
            create_arithmetic_bytecode(a, b),
            10_000_000,
            false,
        );

        // ADD 不应该 panic
        let result = interp.run();
        prop_assert!(matches!(result, InstructionResult::Stop | InstructionResult::Return));

        // 验证 wrapping 行为
        let expected = a.wrapping_add(b);
        prop_assert_eq!(interp.stack.peek(0).unwrap(), expected);
    }

    #[test]
    fn test_memory_operations(
        offset in 0u64..1_000_000,
        size in 0u64..32768,
        data in prop::collection::vec(any::<u8>(), 0..1000),
    ) {
        let bytecode = create_memory_test_bytecode(offset, size, &data);
        let mut interp = Interpreter::new(bytecode, 100_000_000, false);

        let result = interp.run();

        // 内存扩展要么成功要么 OOG
        prop_assert!(matches!(
            result,
            InstructionResult::Stop |
            InstructionResult::Return |
            InstructionResult::OutOfGas
        ));
    }
}
```

### F.4 Benchmark Suite

```rust
use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId};

fn bench_transfer(c: &mut Criterion) {
    let mut group = c.benchmark_group("transfer");

    for &gas_limit in &[21000, 100_000, 1_000_000] {
        group.bench_with_input(
            BenchmarkId::from_parameter(gas_limit),
            &gas_limit,
            |b, &gas| {
                let mut evm = setup_transfer_evm(gas);
                b.iter(|| {
                    evm.transact().unwrap()
                });
            },
        );
    }

    group.finish();
}

fn bench_contract_deployment(c: &mut Criterion) {
    let contracts = vec![
        ("ERC20", include_bytes!("../contracts/erc20.bin")),
        ("Uniswap V2 Pair", include_bytes!("../contracts/pair.bin")),
        ("Uniswap V3 Pool", include_bytes!("../contracts/pool.bin")),
    ];

    let mut group = c.benchmark_group("deploy");

    for (name, bytecode) in contracts {
        group.bench_function(name, |b| {
            let mut evm = setup_deployment_evm(bytecode);
            b.iter(|| {
                evm.transact().unwrap()
            });
        });
    }

    group.finish();
}

criterion_group!(benches, bench_transfer, bench_contract_deployment);
criterion_main!(benches);
```

---

## 附录 G: Parallel Execution (PEVM & Grevm)

revm 是多个并行 EVM 实现的基础。

### G.1 PEVM (Parallel EVM)

PEVM 基于 Block-STM 算法实现乐观并行执行：

```
PEVM 架构:
┌─────────────────────────────────────────────────────────────┐
│                    Transaction Batch                         │
│  [TX0] [TX1] [TX2] [TX3] [TX4] [TX5] [TX6] [TX7]            │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                  PEVM Scheduler (Block-STM)                  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Multi-Version Data Structure (MVCC)                     ││
│  │ ┌─────────┬─────────┬─────────┬─────────┐               ││
│  │ │ Slot A  │ Slot B  │ Slot C  │ ...     │               ││
│  │ │ v0,v1,v2│ v0,v1   │ v0      │         │               ││
│  │ └─────────┴─────────┴─────────┴─────────┘               ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │Worker 0 │ │Worker 1 │ │Worker 2 │ │Worker 3 │           │
│  │ [revm]  │ │ [revm]  │ │ [revm]  │ │ [revm]  │           │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘           │
│       │           │           │           │                  │
│       ▼           ▼           ▼           ▼                  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Validation & Re-execution                   ││
│  │  TX0 ✓  TX1 ✗(conflict) → re-execute                    ││
│  │  TX2 ✓  TX3 ✓  TX4 ✗ → re-execute                       ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

```rust
// PEVM 核心实现
pub struct PevmExecutor<DB: Database + Clone + Send + Sync> {
    db: DB,
    num_workers: usize,
    mvcc: MultiVersionData,
}

impl<DB: Database + Clone + Send + Sync> PevmExecutor<DB> {
    pub fn execute_block(&self, txs: Vec<Transaction>) -> BlockResult {
        let mut scheduler = BlockStmScheduler::new(txs.len());
        let mvcc = MultiVersionData::new();

        // 启动 worker 线程
        std::thread::scope(|s| {
            for worker_id in 0..self.num_workers {
                let scheduler = &scheduler;
                let mvcc = &mvcc;
                let txs = &txs;
                let db = self.db.clone();

                s.spawn(move || {
                    self.worker_loop(worker_id, scheduler, mvcc, txs, db)
                });
            }
        });

        // 收集结果
        scheduler.collect_results()
    }

    fn worker_loop(
        &self,
        worker_id: usize,
        scheduler: &BlockStmScheduler,
        mvcc: &MultiVersionData,
        txs: &[Transaction],
        db: DB,
    ) {
        loop {
            match scheduler.next_task() {
                Task::Execute(tx_idx) => {
                    // 创建 MVCC-aware 数据库包装器
                    let mvcc_db = MvccDatabase::new(&db, mvcc, tx_idx);

                    // 使用 revm 执行
                    let mut evm = Evm::builder()
                        .with_db(mvcc_db)
                        .with_tx(&txs[tx_idx])
                        .build();

                    let result = evm.transact();

                    // 记录 read/write set
                    let read_set = evm.db().read_set();
                    let write_set = evm.db().write_set();

                    // 写入 MVCC
                    mvcc.record_execution(tx_idx, read_set, write_set, result);

                    // 通知 scheduler
                    scheduler.finish_execution(tx_idx);
                }
                Task::Validate(tx_idx) => {
                    // 验证 read set 是否有效
                    let valid = mvcc.validate_read_set(tx_idx);
                    scheduler.finish_validation(tx_idx, valid);
                }
                Task::Done => break,
            }
        }
    }
}

// MVCC 数据结构
pub struct MultiVersionData {
    data: DashMap<StorageKey, BTreeMap<TxIndex, VersionedValue>>,
}

impl MultiVersionData {
    pub fn read(&self, key: &StorageKey, tx_idx: TxIndex) -> Option<U256> {
        self.data.get(key).and_then(|versions| {
            // 找到小于 tx_idx 的最大版本
            versions.range(..tx_idx).next_back().map(|(_, v)| v.value)
        })
    }

    pub fn write(&self, key: StorageKey, tx_idx: TxIndex, value: U256) {
        self.data.entry(key).or_default().insert(tx_idx, VersionedValue {
            value,
            written_at: Instant::now(),
        });
    }
}
```

### G.2 Grevm

Grevm 采用依赖图分析实现更高效的并行：

```
Grevm 架构:
┌─────────────────────────────────────────────────────────────┐
│              Pre-execution Analysis Phase                    │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Static Dependency Analysis                  ││
│  │  TX0 → writes [A, B]                                    ││
│  │  TX1 → reads [A] → depends on TX0                       ││
│  │  TX2 → reads [C] → independent                          ││
│  │  TX3 → reads [B] → depends on TX0                       ││
│  │  TX4 → reads [D] → independent                          ││
│  └─────────────────────────────────────────────────────────┘│
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    Dependency DAG                            │
│                                                              │
│          TX0 ──────┬──────────┐                             │
│           │        │          │                              │
│           ▼        ▼          │                              │
│          TX1      TX3         │                              │
│                               │                              │
│          TX2 (independent)    │                              │
│          TX4 (independent)    │                              │
└───────────────────────┬───────┘─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              Parallel Execution (by levels)                  │
│  Level 0: [TX0, TX2, TX4]  ──→ execute in parallel          │
│  Level 1: [TX1, TX3]       ──→ execute in parallel          │
└─────────────────────────────────────────────────────────────┘
```

```rust
// Grevm 依赖分析
pub struct GrevmAnalyzer {
    // 地址访问模式缓存
    access_patterns: HashMap<Address, AccessPattern>,
}

impl GrevmAnalyzer {
    pub fn analyze_dependencies(&self, txs: &[Transaction]) -> DependencyGraph {
        let mut graph = DependencyGraph::new(txs.len());

        // 1. 静态分析 - 分析交易的 to 地址和 data
        let mut writes_by_tx: Vec<HashSet<StorageKey>> = vec![HashSet::new(); txs.len()];
        let mut reads_by_tx: Vec<HashSet<StorageKey>> = vec![HashSet::new(); txs.len()];

        for (idx, tx) in txs.iter().enumerate() {
            // 基于合约地址和调用数据预测访问模式
            if let Some(pattern) = self.access_patterns.get(&tx.to) {
                reads_by_tx[idx] = pattern.predict_reads(&tx.data);
                writes_by_tx[idx] = pattern.predict_writes(&tx.data);
            } else {
                // 保守估计 - 假设访问整个合约状态
                reads_by_tx[idx].insert(StorageKey::entire_contract(tx.to));
                writes_by_tx[idx].insert(StorageKey::entire_contract(tx.to));
            }

            // from 地址的 balance 和 nonce 总是被修改
            writes_by_tx[idx].insert(StorageKey::balance(tx.from));
            writes_by_tx[idx].insert(StorageKey::nonce(tx.from));
        }

        // 2. 构建依赖图
        for i in 0..txs.len() {
            for j in 0..i {
                // 检查 TX[i] 是否读取 TX[j] 写入的数据
                let has_dependency = reads_by_tx[i]
                    .intersection(&writes_by_tx[j])
                    .next()
                    .is_some();

                if has_dependency {
                    graph.add_edge(j, i); // j -> i (i depends on j)
                }
            }
        }

        graph
    }

    pub fn execute_parallel(&self, txs: Vec<Transaction>, graph: &DependencyGraph) -> Vec<ExecutionResult> {
        let levels = graph.topological_levels();
        let mut results = vec![None; txs.len()];

        for level in levels {
            // 同一 level 的交易可以并行执行
            let level_results: Vec<_> = level
                .par_iter()
                .map(|&tx_idx| {
                    let mut evm = self.create_evm(&txs[tx_idx]);
                    (tx_idx, evm.transact())
                })
                .collect();

            for (idx, result) in level_results {
                results[idx] = Some(result);
            }
        }

        results.into_iter().map(|r| r.unwrap()).collect()
    }
}
```

### G.3 性能对比

```
Benchmark: Uniswap-heavy Block (200 TXs, ~80% DEX swaps)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Implementation       │ Time (ms) │ Speedup │ Notes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
revm (sequential)    │    156    │  1.0x   │ baseline
PEVM (4 threads)     │     52    │  3.0x   │ Block-STM
PEVM (8 threads)     │     38    │  4.1x   │ Block-STM
Grevm (4 threads)    │     45    │  3.5x   │ DAG-based
Grevm (8 threads)    │     31    │  5.0x   │ DAG-based
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Benchmark: Mixed Block (200 TXs, diverse operations)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Implementation       │ Time (ms) │ Speedup │ Conflicts
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
revm (sequential)    │    120    │  1.0x   │    N/A
PEVM (8 threads)     │     25    │  4.8x   │   ~12%
Grevm (8 threads)    │     22    │  5.5x   │   ~8%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 附录 H: WASM Compilation

### H.1 revm-wasm

revm 支持编译到 WebAssembly，可在浏览器中运行：

```rust
// Cargo.toml 配置
[lib]
crate-type = ["cdylib", "rlib"]

[features]
default = ["std"]
std = ["revm/std"]
wasm = ["revm/wasm", "wasm-bindgen"]

[target.'cfg(target_arch = "wasm32")'.dependencies]
wasm-bindgen = "0.2"
js-sys = "0.3"
web-sys = { version = "0.3", features = ["console"] }
getrandom = { version = "0.2", features = ["js"] }
```

```rust
// lib.rs - WASM 绑定
use wasm_bindgen::prelude::*;
use revm::{Evm, InMemoryDB, primitives::*};

#[wasm_bindgen]
pub struct WasmEvm {
    db: InMemoryDB,
}

#[wasm_bindgen]
impl WasmEvm {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self {
        console_error_panic_hook::set_once();
        Self {
            db: InMemoryDB::default(),
        }
    }

    #[wasm_bindgen]
    pub fn set_account(&mut self, address: &str, balance: &str) -> Result<(), JsValue> {
        let addr: Address = address.parse()
            .map_err(|e| JsValue::from_str(&format!("Invalid address: {}", e)))?;
        let bal: U256 = balance.parse()
            .map_err(|e| JsValue::from_str(&format!("Invalid balance: {}", e)))?;

        self.db.insert_account_info(addr, AccountInfo {
            balance: bal,
            nonce: 0,
            code_hash: KECCAK_EMPTY,
            code: None,
        });

        Ok(())
    }

    #[wasm_bindgen]
    pub fn deploy_contract(&mut self, from: &str, bytecode: &[u8], value: &str) -> Result<String, JsValue> {
        let from_addr: Address = from.parse()
            .map_err(|e| JsValue::from_str(&format!("Invalid from: {}", e)))?;
        let val: U256 = value.parse()
            .map_err(|e| JsValue::from_str(&format!("Invalid value: {}", e)))?;

        let mut evm = Evm::builder()
            .with_db(&mut self.db)
            .with_tx_env(TxEnv {
                caller: from_addr,
                gas_limit: 10_000_000,
                gas_price: U256::from(1),
                transact_to: TxKind::Create,
                value: val,
                data: Bytes::from(bytecode.to_vec()),
                ..Default::default()
            })
            .build();

        let result = evm.transact_commit()
            .map_err(|e| JsValue::from_str(&format!("Execution error: {:?}", e)))?;

        match result {
            ExecutionResult::Success { output: Output::Create(_, Some(addr)), .. } => {
                Ok(format!("{:?}", addr))
            }
            ExecutionResult::Revert { output, .. } => {
                Err(JsValue::from_str(&format!("Reverted: {:?}", output)))
            }
            _ => Err(JsValue::from_str("Unexpected result")),
        }
    }

    #[wasm_bindgen]
    pub fn call_contract(
        &mut self,
        from: &str,
        to: &str,
        data: &[u8],
        value: &str,
    ) -> Result<Vec<u8>, JsValue> {
        let from_addr: Address = from.parse()
            .map_err(|e| JsValue::from_str(&format!("Invalid from: {}", e)))?;
        let to_addr: Address = to.parse()
            .map_err(|e| JsValue::from_str(&format!("Invalid to: {}", e)))?;
        let val: U256 = value.parse()
            .map_err(|e| JsValue::from_str(&format!("Invalid value: {}", e)))?;

        let mut evm = Evm::builder()
            .with_db(&mut self.db)
            .with_tx_env(TxEnv {
                caller: from_addr,
                gas_limit: 10_000_000,
                transact_to: TxKind::Call(to_addr),
                value: val,
                data: Bytes::from(data.to_vec()),
                ..Default::default()
            })
            .build();

        let result = evm.transact_commit()
            .map_err(|e| JsValue::from_str(&format!("Execution error: {:?}", e)))?;

        match result {
            ExecutionResult::Success { output: Output::Call(bytes), .. } => {
                Ok(bytes.to_vec())
            }
            ExecutionResult::Revert { output, .. } => {
                Err(JsValue::from_str(&format!("Reverted: {:?}", output)))
            }
            _ => Err(JsValue::from_str("Unexpected result")),
        }
    }
}
```

### H.2 JavaScript 使用示例

```javascript
// 使用 wasm-pack build 构建后
import init, {WasmEvm} from './revm_wasm.js';

async function main() {
    await init();

    const evm = new WasmEvm();

    // 设置账户
    evm.set_account(
        "0x1234567890123456789012345678901234567890",
        "1000000000000000000000" // 1000 ETH
    );

    // 部署 ERC20 合约
    const erc20Bytecode = new Uint8Array([...]); // 合约字节码
    const contractAddr = evm.deploy_contract(
        "0x1234567890123456789012345678901234567890",
        erc20Bytecode,
        "0"
    );
    console.log("Contract deployed at:", contractAddr);

    // 调用合约 - balanceOf
    const balanceOfSelector = new Uint8Array([0x70, 0xa0, 0x82, 0x31]);
    const address = new Uint8Array(32);
    address.set(hexToBytes("1234567890123456789012345678901234567890"), 12);

    const calldata = new Uint8Array([...balanceOfSelector, ...address]);
    const result = evm.call_contract(
        "0x1234567890123456789012345678901234567890",
        contractAddr,
        calldata,
        "0"
    );

    console.log("Balance:", bytesToBigInt(result));
}
```

### H.3 WASM 性能考虑

```
WASM vs Native 性能对比:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Operation              │ Native (μs) │ WASM (μs) │ Ratio
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Simple transfer        │      8      │     25    │  3.1x
ERC20 transfer         │     35      │    105    │  3.0x
Uniswap V2 swap        │    180      │    520    │  2.9x
Contract deployment    │    450      │   1350    │  3.0x
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

优化建议:
1. 使用 wasm-opt -O3 优化
2. 启用 simd128 (如果浏览器支持)
3. 使用 Web Workers 避免阻塞主线程
4. 批量处理多个交易减少 JS/WASM 边界开销
```

---

## 附录 I: Integration Examples

### I.1 MEV Bot 集成

```rust
use revm::{Evm, InMemoryDB};
use ethers::providers::{Provider, Ws};

pub struct MevBot {
    evm: Evm<'static, (), InMemoryDB>,
    provider: Provider<Ws>,
}

impl MevBot {
    pub async fn simulate_bundle(&mut self, txs: Vec<Transaction>) -> SimulationResult {
        // 1. Fork 当前状态
        let fork_db = self.evm.db().fork();

        // 2. 模拟交易序列
        let mut total_profit = U256::ZERO;
        let mut results = Vec::new();

        for tx in txs {
            let mut sim_evm = Evm::builder()
                .with_db(&mut fork_db)
                .with_tx(&tx)
                .build();

            let result = sim_evm.transact()?;

            // 计算利润
            if tx.from == self.bot_address {
                let profit = self.calculate_profit(&result);
                total_profit += profit;
            }

            results.push(result);
        }

        SimulationResult {
            success: true,
            total_profit,
            gas_used: results.iter().map(|r| r.gas_used).sum(),
            results,
        }
    }

    pub async fn find_arbitrage(&mut self, pools: &[Pool]) -> Option<ArbitrageOpportunity> {
        // 使用 revm 快速模拟各种套利路径
        for path in self.generate_paths(pools) {
            let profit = self.simulate_path(&path).await?;

            if profit > self.min_profit_threshold {
                return Some(ArbitrageOpportunity {
                    path,
                    expected_profit: profit,
                    gas_estimate: self.estimate_gas(&path),
                });
            }
        }

        None
    }
}
```

### I.2 Foundry 风格测试框架

```rust
use revm::primitives::*;

/// Foundry-style test contract runner using revm
pub struct ForgeRunner {
    evm: Evm<'static, (), InMemoryDB>,
    deployed_contracts: HashMap<String, Address>,
}

impl ForgeRunner {
    pub fn deploy(&mut self, name: &str, bytecode: Bytes) -> Address {
        let result = self.evm.transact_commit().unwrap();

        if let ExecutionResult::Success { output: Output::Create(_, Some(addr)), .. } = result {
            self.deployed_contracts.insert(name.to_string(), addr);
            addr
        } else {
            panic!("Deployment failed");
        }
    }

    /// vm.prank(address) 等效实现
    pub fn prank(&mut self, addr: Address) {
        self.evm.tx_mut().caller = addr;
    }

    /// vm.deal(address, amount) 等效实现
    pub fn deal(&mut self, addr: Address, amount: U256) {
        let mut info = self.evm.db().basic(addr).unwrap().unwrap_or_default();
        info.balance = amount;
        self.evm.db_mut().insert_account_info(addr, info);
    }

    /// vm.expectRevert() 等效实现
    pub fn expect_revert(&mut self) {
        self.expected_revert = true;
    }

    /// vm.snapshot() 等效实现
    pub fn snapshot(&self) -> u64 {
        let snapshot_id = self.next_snapshot_id;
        self.snapshots.insert(snapshot_id, self.evm.db().clone());
        self.next_snapshot_id += 1;
        snapshot_id
    }

    /// vm.revertTo(snapshotId) 等效实现
    pub fn revert_to(&mut self, snapshot_id: u64) -> bool {
        if let Some(db) = self.snapshots.get(&snapshot_id) {
            *self.evm.db_mut() = db.clone();
            true
        } else {
            false
        }
    }

    /// 运行测试
    pub fn run_test(&mut self, contract: Address, selector: [u8; 4]) -> TestResult {
        let result = self.evm
            .with_tx_env(TxEnv {
                transact_to: TxKind::Call(contract),
                data: Bytes::from(selector.to_vec()),
                gas_limit: 30_000_000,
                ..Default::default()
            })
            .transact();

        match result {
            Ok(ExecutionResult::Success { .. }) if !self.expected_revert => {
                TestResult::Pass
            }
            Ok(ExecutionResult::Revert { .. }) if self.expected_revert => {
                self.expected_revert = false;
                TestResult::Pass
            }
            Ok(ExecutionResult::Revert { output, .. }) => {
                TestResult::Fail(format!("Unexpected revert: {:?}", output))
            }
            Err(e) => TestResult::Fail(format!("Execution error: {:?}", e)),
            _ => TestResult::Fail("Unexpected result".to_string()),
        }
    }
}
```

### I.3 交易追踪集成

```rust
use revm::inspectors::TracerEip3155;
use std::io::Write;

/// EIP-3155 标准追踪输出
pub fn trace_transaction(
    tx: &Transaction,
    db: impl Database,
    output: impl Write,
) -> eyre::Result<ExecutionResult> {
    let tracer = TracerEip3155::new(Box::new(output));

    let mut evm = Evm::builder()
        .with_db(db)
        .with_external_context(tracer)
        .append_handler_register(inspector_handle_register)
        .with_tx(tx)
        .build();

    let result = evm.transact()?;

    Ok(result)
}

/// 自定义调用追踪
#[derive(Default)]
pub struct CallTracer {
    pub call_stack: Vec<CallFrame>,
    current_depth: usize,
}

#[derive(Clone, Debug)]
pub struct CallFrame {
    pub call_type: CallType,
    pub from: Address,
    pub to: Address,
    pub value: U256,
    pub gas: u64,
    pub input: Bytes,
    pub output: Bytes,
    pub error: Option<String>,
    pub subcalls: Vec<CallFrame>,
}

impl Inspector<&mut InMemoryDB> for CallTracer {
    fn call(
        &mut self,
        _context: &mut EvmContext<&mut InMemoryDB>,
        inputs: &mut CallInputs,
    ) -> Option<CallOutcome> {
        let frame = CallFrame {
            call_type: inputs.scheme.into(),
            from: inputs.caller,
            to: inputs.target_address,
            value: inputs.transfer_value(),
            gas: inputs.gas_limit,
            input: inputs.input.clone(),
            output: Bytes::new(),
            error: None,
            subcalls: Vec::new(),
        };

        self.call_stack.push(frame);
        self.current_depth += 1;

        None // Continue execution
    }

    fn call_end(
        &mut self,
        _context: &mut EvmContext<&mut InMemoryDB>,
        _inputs: &CallInputs,
        outcome: CallOutcome,
    ) -> CallOutcome {
        self.current_depth -= 1;

        if let Some(mut frame) = self.call_stack.pop() {
            frame.output = outcome.result.output.clone();

            if !outcome.result.is_ok() {
                frame.error = Some(format!("{:?}", outcome.result.result));
            }

            // 添加到父帧的 subcalls
            if let Some(parent) = self.call_stack.last_mut() {
                parent.subcalls.push(frame);
            } else {
                // 顶层调用，保存结果
                self.call_stack.push(frame);
            }
        }

        outcome
    }
}

// 使用示例
pub fn trace_with_call_tracer(tx: &Transaction, db: &mut InMemoryDB) -> CallFrame {
    let tracer = CallTracer::default();

    let mut evm = Evm::builder()
        .with_db(db)
        .with_external_context(tracer)
        .append_handler_register(inspector_handle_register)
        .with_tx(tx)
        .build();

    let _ = evm.transact();

    evm.context.external.call_stack.pop().unwrap_or_default()
}
```

### I.4 Hardhat/Anvil 节点集成模式

```rust
/// Anvil 风格的本地开发节点
pub struct LocalNode {
    evm: Evm<'static, (), InMemoryDB>,
    pending_txs: Vec<Transaction>,
    block_number: u64,
    // Anvil-specific features
    auto_mine: bool,
    block_time: Option<Duration>,
    impersonated_accounts: HashSet<Address>,
}

impl LocalNode {
    /// anvil_impersonateAccount 等效实现
    pub fn impersonate_account(&mut self, addr: Address) {
        self.impersonated_accounts.insert(addr);
    }

    /// anvil_setBalance 等效实现
    pub fn set_balance(&mut self, addr: Address, balance: U256) {
        let mut info = self.evm.db().basic(addr).unwrap().unwrap_or_default();
        info.balance = balance;
        self.evm.db_mut().insert_account_info(addr, info);
    }

    /// anvil_setCode 等效实现
    pub fn set_code(&mut self, addr: Address, code: Bytes) {
        let mut info = self.evm.db().basic(addr).unwrap().unwrap_or_default();
        info.code_hash = keccak256(&code);
        info.code = Some(Bytecode::new_raw(code));
        self.evm.db_mut().insert_account_info(addr, info);
    }

    /// anvil_setStorageAt 等效实现
    pub fn set_storage_at(&mut self, addr: Address, slot: U256, value: U256) {
        self.evm.db_mut().insert_account_storage(addr, slot, value).unwrap();
    }

    /// anvil_mine 等效实现
    pub fn mine(&mut self, blocks: u64) {
        for _ in 0..blocks {
            self.mine_block();
        }
    }

    fn mine_block(&mut self) {
        // 执行所有 pending 交易
        for tx in std::mem::take(&mut self.pending_txs) {
            let _ = self.execute_tx(&tx);
        }

        // 更新区块号
        self.block_number += 1;
        self.evm.block_mut().number = U256::from(self.block_number);
        self.evm.block_mut().timestamp = U256::from(
            SystemTime::now()
                .duration_since(UNIX_EPOCH)
                .unwrap()
                .as_secs()
        );
    }

    /// 处理 JSON-RPC 请求
    pub fn handle_rpc(&mut self, request: JsonRpcRequest) -> JsonRpcResponse {
        match request.method.as_str() {
            "eth_sendTransaction" => {
                let tx: Transaction = serde_json::from_value(request.params[0].clone()).unwrap();
                let tx_hash = self.send_transaction(tx);

                if self.auto_mine {
                    self.mine_block();
                }

                JsonRpcResponse::success(tx_hash)
            }
            "eth_call" => {
                let call: CallRequest = serde_json::from_value(request.params[0].clone()).unwrap();
                let result = self.eth_call(&call);
                JsonRpcResponse::success(result)
            }
            "eth_getBalance" => {
                let addr: Address = serde_json::from_value(request.params[0].clone()).unwrap();
                let balance = self.evm.db().basic(addr).unwrap().map(|a| a.balance).unwrap_or_default();
                JsonRpcResponse::success(balance)
            }
            // ... 其他 RPC 方法
            _ => JsonRpcResponse::error(-32601, "Method not found"),
        }
    }
}
```

---

## 附录 J: EIP 兼容性矩阵

### J.1 硬分叉支持状态

```
revm 硬分叉支持对比 (截至 2024 年底):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
硬分叉          │ go-ethereum │ revm   │ SpecId
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Frontier        │     ✓       │   ✓    │ FRONTIER
Homestead       │     ✓       │   ✓    │ HOMESTEAD
Tangerine       │     ✓       │   ✓    │ TANGERINE
Spurious Dragon │     ✓       │   ✓    │ SPURIOUS_DRAGON
Byzantium       │     ✓       │   ✓    │ BYZANTIUM
Constantinople  │     ✓       │   ✓    │ CONSTANTINOPLE
Petersburg      │     ✓       │   ✓    │ PETERSBURG
Istanbul        │     ✓       │   ✓    │ ISTANBUL
Berlin          │     ✓       │   ✓    │ BERLIN
London          │     ✓       │   ✓    │ LONDON
Paris (Merge)   │     ✓       │   ✓    │ MERGE
Shanghai        │     ✓       │   ✓    │ SHANGHAI
Cancun          │     ✓       │   ✓    │ CANCUN
Prague          │   开发中    │ 开发中  │ PRAGUE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### J.2 revm SpecId 实现

```rust
// revm 的 SpecId 枚举
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
#[repr(u8)]
pub enum SpecId {
    FRONTIER = 0,
    FRONTIER_THAWING = 1,
    HOMESTEAD = 2,
    DAO_FORK = 3,
    TANGERINE = 4,
    SPURIOUS_DRAGON = 5,
    BYZANTIUM = 6,
    CONSTANTINOPLE = 7,
    PETERSBURG = 8,
    ISTANBUL = 9,
    MUIR_GLACIER = 10,
    BERLIN = 11,
    LONDON = 12,
    ARROW_GLACIER = 13,
    GRAY_GLACIER = 14,
    MERGE = 15,
    SHANGHAI = 16,
    CANCUN = 17,
    PRAGUE = 18,
    OSAKA = 19,
    LATEST = u8::MAX,
}

impl SpecId {
    /// 检查是否启用特定 EIP
    pub const fn is_enabled_in(self, eip: Eip) -> bool {
        match eip {
            Eip::Eip1559 => self >= Self::LONDON,
            Eip::Eip2929 => self >= Self::BERLIN,
            Eip::Eip3529 => self >= Self::LONDON,
            Eip::Eip3541 => self >= Self::LONDON,
            Eip::Eip1153 => self >= Self::CANCUN,
            Eip::Eip4844 => self >= Self::CANCUN,
            Eip::Eip5656 => self >= Self::CANCUN,
            Eip::Eip6780 => self >= Self::CANCUN,
            Eip::Eip7516 => self >= Self::CANCUN,
            // Prague EIPs
            Eip::Eip2537 => self >= Self::PRAGUE,
            Eip::Eip7702 => self >= Self::PRAGUE,
        }
    }
}

// 条件编译操作码
macro_rules! spec_opcode {
    ($spec:expr, $op:ident, $min_spec:expr) => {
        if $spec >= $min_spec {
            Some(instruction::$op)
        } else {
            None
        }
    };
}

// 构建操作码表
pub fn make_instruction_table<SPEC: Spec>() -> [Instruction; 256] {
    let mut table = [instruction::unknown; 256];

    // 基础操作码
    table[STOP as usize] = instruction::stop;
    table[ADD as usize] = instruction::add;
    // ...

    // Cancun 操作码
    if SPEC::SPEC_ID >= SpecId::CANCUN {
        table[TLOAD as usize] = instruction::tload;
        table[TSTORE as usize] = instruction::tstore;
        table[MCOPY as usize] = instruction::mcopy;
        table[BLOBHASH as usize] = instruction::blobhash;
        table[BLOBBASEFEE as usize] = instruction::blobbasefee;
    }

    // Prague 操作码
    if SPEC::SPEC_ID >= SpecId::PRAGUE {
        // EOF 相关操作码将在这里添加
    }

    table
}
```

### J.3 EIP 特性矩阵

```
Cancun EIP 支持详情:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EIP      │ 功能                    │ revm 实现
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EIP-1153 │ Transient Storage       │ TransientStorage struct
EIP-4788 │ Beacon Root             │ 需 handler 支持
EIP-4844 │ Blob Transactions       │ TxEnv::blob_hashes
EIP-5656 │ MCOPY                   │ memory::mcopy()
EIP-6780 │ SELFDESTRUCT 限制       │ created_in_tx 检查
EIP-7516 │ BLOBBASEFEE             │ BlockEnv::blob_base_fee
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

```rust
// Transient Storage 实现
#[derive(Default, Clone, Debug)]
pub struct TransientStorage {
    storage: HashMap<(Address, U256), U256>,
}

impl TransientStorage {
    pub fn tload(&self, address: Address, key: U256) -> U256 {
        self.storage.get(&(address, key)).copied().unwrap_or_default()
    }

    pub fn tstore(&mut self, address: Address, key: U256, value: U256) {
        if value.is_zero() {
            self.storage.remove(&(address, key));
        } else {
            self.storage.insert((address, key), value);
        }
    }

    pub fn clear(&mut self) {
        self.storage.clear();
    }
}

// Blob 交易支持
pub struct TxEnv {
    pub caller: Address,
    pub gas_limit: u64,
    pub gas_price: U256,
    pub transact_to: TxKind,
    pub value: U256,
    pub data: Bytes,
    pub nonce: Option<u64>,
    pub chain_id: Option<u64>,
    pub access_list: Vec<(Address, Vec<U256>)>,
    // EIP-4844 字段
    pub blob_hashes: Vec<B256>,
    pub max_fee_per_blob_gas: Option<U256>,
}
```

---

## 附录 K: Verkle Trees 支持

### K.1 Verkle Trees 与 revm

```
Verkle Trees 对 revm 的影响:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
组件              │ 当前实现        │ Verkle 后变化
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Database trait    │ 基于 slot      │ 基于 stem + suffix
Account 结构      │ balance/nonce  │ 添加 version 字段
Code 访问         │ 完整代码       │ 31 字节分块
Gas 计算          │ cold/warm      │ witness 成本
State Proof      │ MPT proof      │ Verkle proof
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### K.2 Database Trait 演进

```rust
// 当前 Database trait
pub trait Database {
    type Error;

    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error>;
    fn code_by_hash(&mut self, code_hash: B256) -> Result<Bytecode, Self::Error>;
    fn storage(&mut self, address: Address, index: U256) -> Result<U256, Self::Error>;
    fn block_hash(&mut self, number: U256) -> Result<B256, Self::Error>;
}

// Verkle-aware Database trait (未来)
pub trait VerkleDatabase {
    type Error;

    // 账户基本信息 - 使用 stem
    fn account_stem(&mut self, address: Address) -> Result<AccountStem, Self::Error>;

    // 存储访问 - 使用 stem + suffix
    fn storage_slot(
        &mut self,
        address: Address,
        stem: [u8; 31],
        suffix: u8,
    ) -> Result<U256, Self::Error>;

    // 代码块访问
    fn code_chunk(
        &mut self,
        address: Address,
        chunk_index: u8,
    ) -> Result<[u8; 32], Self::Error>;

    // Witness 数据
    fn get_witness(&self, keys: &[VerkleKey]) -> Result<VerkleProof, Self::Error>;
}

// Verkle 地址到 stem 的转换
pub fn address_to_stem(address: Address) -> [u8; 31] {
    let mut stem = [0u8; 31];
    // Pedersen hash 的前 31 字节
    let hash = pedersen_hash(&[
        &[2u8], // 版本前缀
        address.as_slice(),
        &[0u8; 12], // padding
    ]);
    stem.copy_from_slice(&hash[..31]);
    stem
}

// 存储槽到 stem+suffix 的转换
pub fn storage_slot_to_verkle(address: Address, slot: U256) -> ([u8; 31], u8) {
    let slot_bytes = slot.to_be_bytes::<32>();

    // 计算 tree key
    let tree_index = U256::from_be_bytes(slot_bytes) / 256;
    let suffix = (slot.as_limbs()[0] % 256) as u8;

    let stem = compute_storage_stem(address, tree_index);
    (stem, suffix)
}
```

### K.3 Gas 计算变化

```rust
// Verkle Gas 常量
pub mod verkle_gas {
    pub const WITNESS_BRANCH_COST: u64 = 1900;  // 新 branch 访问
    pub const WITNESS_CHUNK_COST: u64 = 200;    // 新 chunk 访问

    // 替代当前的 cold/warm 模型
    pub const ACCESS_WITNESS_BYTE_COST: u64 = 31;
}

// Verkle-aware 存储 gas 计算
pub fn verkle_sload_gas(
    witness_state: &WitnessState,
    address: Address,
    slot: U256,
) -> u64 {
    let (stem, suffix) = storage_slot_to_verkle(address, slot);

    let stem_cost = if witness_state.stem_accessed(stem) {
        0
    } else {
        verkle_gas::WITNESS_BRANCH_COST
    };

    let suffix_cost = if witness_state.suffix_accessed(stem, suffix) {
        0
    } else {
        verkle_gas::WITNESS_CHUNK_COST
    };

    // 基础成本 + witness 成本
    100 + stem_cost + suffix_cost
}

// Witness 状态追踪
#[derive(Default)]
pub struct WitnessState {
    accessed_stems: HashSet<[u8; 31]>,
    accessed_suffixes: HashSet<([u8; 31], u8)>,
}

impl WitnessState {
    pub fn stem_accessed(&self, stem: [u8; 31]) -> bool {
        self.accessed_stems.contains(&stem)
    }

    pub fn suffix_accessed(&self, stem: [u8; 31], suffix: u8) -> bool {
        self.accessed_suffixes.contains(&(stem, suffix))
    }

    pub fn mark_accessed(&mut self, stem: [u8; 31], suffix: u8) {
        self.accessed_stems.insert(stem);
        self.accessed_suffixes.insert((stem, suffix));
    }
}
```

---

## 附录 L: 安全审计模式

### L.1 使用 revm 进行合约安全分析

```rust
// 安全分析 Inspector
pub struct SecurityInspector {
    // 重入检测
    call_stack: Vec<Address>,
    in_call: HashSet<Address>,

    // 状态访问日志
    storage_reads: HashMap<(Address, U256), Vec<usize>>,  // (address, slot) -> call depths
    storage_writes: HashMap<(Address, U256), Vec<usize>>,

    // 外部调用追踪
    external_calls: Vec<ExternalCallInfo>,

    // 检测到的问题
    vulnerabilities: Vec<Vulnerability>,
}

#[derive(Debug)]
pub struct Vulnerability {
    pub kind: VulnerabilityKind,
    pub location: Address,
    pub details: String,
}

#[derive(Debug)]
pub enum VulnerabilityKind {
    Reentrancy,
    UncheckedCall,
    IntegerOverflow,
    AccessControl,
    Selfdestruct,
    DelegateCallToUntrusted,
}

impl<DB: Database> Inspector<DB> for SecurityInspector {
    fn call(
        &mut self,
        context: &mut EvmContext<DB>,
        inputs: &mut CallInputs,
    ) -> Option<CallOutcome> {
        let target = inputs.target_address;

        // 重入检测
        if self.in_call.contains(&target) {
            self.vulnerabilities.push(Vulnerability {
                kind: VulnerabilityKind::Reentrancy,
                location: target,
                details: format!(
                    "Potential reentrancy: {} called while already in call stack",
                    target
                ),
            });
        }

        // 检测 delegatecall 到不可信地址
        if inputs.scheme == CallScheme::DelegateCall {
            if !self.is_trusted_target(&target) {
                self.vulnerabilities.push(Vulnerability {
                    kind: VulnerabilityKind::DelegateCallToUntrusted,
                    location: target,
                    details: format!("DelegateCall to potentially untrusted: {}", target),
                });
            }
        }

        self.call_stack.push(target);
        self.in_call.insert(target);

        None
    }

    fn call_end(
        &mut self,
        _context: &mut EvmContext<DB>,
        _inputs: &CallInputs,
        outcome: CallOutcome,
    ) -> CallOutcome {
        if let Some(addr) = self.call_stack.pop() {
            self.in_call.remove(&addr);
        }

        // 检测未检查的调用返回值
        self.external_calls.push(ExternalCallInfo {
            success: outcome.result.is_ok(),
            depth: self.call_stack.len(),
        });

        outcome
    }

    fn sload(
        &mut self,
        _context: &mut EvmContext<DB>,
        address: Address,
        slot: U256,
    ) -> Option<U256> {
        self.storage_reads
            .entry((address, slot))
            .or_default()
            .push(self.call_stack.len());
        None
    }

    fn sstore(
        &mut self,
        _context: &mut EvmContext<DB>,
        address: Address,
        slot: U256,
        _new_value: U256,
    ) -> Option<U256> {
        let depth = self.call_stack.len();

        // 检测写后读模式 (可能的重入)
        if let Some(read_depths) = self.storage_reads.get(&(address, slot)) {
            for &read_depth in read_depths {
                if read_depth < depth {
                    self.vulnerabilities.push(Vulnerability {
                        kind: VulnerabilityKind::Reentrancy,
                        location: address,
                        details: format!(
                            "Storage slot {} read at depth {} then written at depth {}",
                            slot, read_depth, depth
                        ),
                    });
                }
            }
        }

        self.storage_writes
            .entry((address, slot))
            .or_default()
            .push(depth);

        None
    }

    fn selfdestruct(
        &mut self,
        contract: Address,
        target: Address,
        value: U256,
    ) {
        self.vulnerabilities.push(Vulnerability {
            kind: VulnerabilityKind::Selfdestruct,
            location: contract,
            details: format!(
                "SELFDESTRUCT called, sending {} to {}",
                value, target
            ),
        });
    }
}
```

### L.2 符号执行与约束求解

```rust
use z3::{Config, Context, Solver, ast::{Ast, Int}};

/// 基于 revm 的符号执行引擎
pub struct SymbolicExecutor<'ctx> {
    z3_ctx: &'ctx Context,
    solver: Solver<'ctx>,

    // 符号状态
    symbolic_stack: Vec<SymbolicValue<'ctx>>,
    symbolic_memory: HashMap<U256, SymbolicValue<'ctx>>,
    symbolic_storage: HashMap<(Address, U256), SymbolicValue<'ctx>>,

    // 路径约束
    path_constraints: Vec<z3::ast::Bool<'ctx>>,

    // 符号输入
    symbolic_calldata: SymbolicBytes<'ctx>,
}

pub enum SymbolicValue<'ctx> {
    Concrete(U256),
    Symbolic(z3::ast::BV<'ctx>),
}

impl<'ctx> SymbolicExecutor<'ctx> {
    pub fn new(ctx: &'ctx Context) -> Self {
        Self {
            z3_ctx: ctx,
            solver: Solver::new(ctx),
            symbolic_stack: Vec::new(),
            symbolic_memory: HashMap::new(),
            symbolic_storage: HashMap::new(),
            path_constraints: Vec::new(),
            symbolic_calldata: SymbolicBytes::new(ctx, "calldata"),
        }
    }

    /// 符号执行单个操作码
    pub fn step(&mut self, opcode: u8) -> Result<(), SymExecError> {
        match opcode {
            opcode::ADD => {
                let b = self.pop()?;
                let a = self.pop()?;
                let result = self.symbolic_add(a, b);
                self.push(result);
            }
            opcode::SUB => {
                let b = self.pop()?;
                let a = self.pop()?;

                // 检测下溢
                if let (SymbolicValue::Symbolic(sym_a), SymbolicValue::Symbolic(sym_b)) = (&a, &b) {
                    let underflow_cond = sym_b.bvugt(sym_a);

                    // 检查下溢是否可达
                    self.solver.push();
                    self.solver.assert(&underflow_cond);
                    if self.solver.check() == z3::SatResult::Sat {
                        self.report_vulnerability(
                            VulnerabilityKind::IntegerOverflow,
                            "Integer underflow in SUB operation",
                        );
                    }
                    self.solver.pop(1);
                }

                let result = self.symbolic_sub(a, b);
                self.push(result);
            }
            opcode::JUMPI => {
                let dest = self.pop()?;
                let cond = self.pop()?;

                if let SymbolicValue::Symbolic(sym_cond) = cond {
                    // 路径分叉
                    let true_branch = self.fork_with_constraint(sym_cond.clone());
                    let false_branch = self.fork_with_constraint(sym_cond.not());

                    // 两个分支都可行时，记录分叉点
                    if true_branch.is_feasible() && false_branch.is_feasible() {
                        self.record_branch_point(dest, sym_cond);
                    }
                }
            }
            opcode::SLOAD => {
                let slot = self.pop()?;

                // 创建符号存储值
                let value = match slot {
                    SymbolicValue::Concrete(s) => {
                        self.symbolic_storage
                            .entry((self.current_address, s))
                            .or_insert_with(|| {
                                SymbolicValue::Symbolic(
                                    z3::ast::BV::new_const(
                                        self.z3_ctx,
                                        format!("storage_{}_{}", self.current_address, s),
                                        256,
                                    )
                                )
                            })
                            .clone()
                    }
                    SymbolicValue::Symbolic(_) => {
                        // 符号槽位 - 需要更复杂的处理
                        SymbolicValue::Symbolic(
                            z3::ast::BV::new_const(self.z3_ctx, "unknown_storage", 256)
                        )
                    }
                };

                self.push(value);
            }
            _ => {}
        }
        Ok(())
    }

    /// 检查是否存在可达的漏洞条件
    pub fn check_vulnerability(&mut self, condition: z3::ast::Bool<'ctx>) -> bool {
        self.solver.push();
        for constraint in &self.path_constraints {
            self.solver.assert(constraint);
        }
        self.solver.assert(&condition);
        let result = self.solver.check() == z3::SatResult::Sat;
        self.solver.pop(1);
        result
    }
}
```

### L.3 Fuzzing 集成

```rust
use arbitrary::{Arbitrary, Unstructured};

/// Fuzzing 目标生成器
#[derive(Debug, Clone, Arbitrary)]
pub struct FuzzInput {
    pub caller: Address,
    pub value: U256,
    pub data: Vec<u8>,
    pub gas_limit: u64,
}

/// 基于 revm 的 Fuzzer
pub struct RevmFuzzer {
    evm: Evm<'static, (), InMemoryDB>,
    corpus: Vec<FuzzInput>,
    coverage: Coverage,
}

impl RevmFuzzer {
    pub fn fuzz_iteration(&mut self, input: &FuzzInput) -> FuzzResult {
        // 重置状态
        self.evm.db_mut().reset();

        // 设置交易
        self.evm.tx_mut().caller = input.caller;
        self.evm.tx_mut().value = input.value;
        self.evm.tx_mut().data = Bytes::from(input.data.clone());
        self.evm.tx_mut().gas_limit = input.gas_limit.clamp(21000, 30_000_000);

        // 执行并收集覆盖
        let coverage_inspector = CoverageInspector::new();
        let result = self.evm
            .with_inspector(&mut coverage_inspector)
            .transact();

        // 更新覆盖率
        let new_coverage = coverage_inspector.get_coverage();
        let is_new = self.coverage.merge(new_coverage);

        // 检测崩溃
        match result {
            Ok(result) => {
                if result.result.is_revert() {
                    // 检查是否是有趣的 revert
                    FuzzResult::Revert {
                        reason: extract_revert_reason(&result.result),
                        is_new_coverage: is_new,
                    }
                } else {
                    FuzzResult::Success { is_new_coverage: is_new }
                }
            }
            Err(e) => {
                FuzzResult::Error {
                    error: format!("{:?}", e),
                    input: input.clone(),
                }
            }
        }
    }

    /// 基于覆盖率指导的变异
    pub fn mutate_input(&self, input: &FuzzInput) -> FuzzInput {
        let mut rng = rand::thread_rng();

        let mut mutated = input.clone();

        match rng.gen_range(0..5) {
            0 => {
                // 变异 calldata
                if !mutated.data.is_empty() {
                    let idx = rng.gen_range(0..mutated.data.len());
                    mutated.data[idx] = rng.gen();
                }
            }
            1 => {
                // 变异 value
                mutated.value = U256::from(rng.gen::<u128>());
            }
            2 => {
                // 插入 calldata 字节
                let idx = rng.gen_range(0..=mutated.data.len());
                mutated.data.insert(idx, rng.gen());
            }
            3 => {
                // 使用已知的函数选择器
                if mutated.data.len() >= 4 {
                    let known_selectors = [
                        [0xa9, 0x05, 0x9c, 0xbb], // transfer
                        [0x09, 0x5e, 0xa7, 0xb3], // approve
                        [0x23, 0xb8, 0x72, 0xdd], // transferFrom
                    ];
                    let selector = known_selectors[rng.gen_range(0..known_selectors.len())];
                    mutated.data[..4].copy_from_slice(&selector);
                }
            }
            _ => {
                // 随机 caller
                mutated.caller = Address::random();
            }
        }

        mutated
    }
}

/// 覆盖率追踪
#[derive(Default)]
pub struct Coverage {
    // (contract, pc) -> hit count
    pc_coverage: HashMap<(Address, usize), u64>,
    // (contract, edge) -> hit count
    edge_coverage: HashMap<(Address, (usize, usize)), u64>,
}

impl Coverage {
    pub fn merge(&mut self, other: Self) -> bool {
        let mut has_new = false;

        for (key, count) in other.pc_coverage {
            let entry = self.pc_coverage.entry(key).or_insert(0);
            if *entry == 0 {
                has_new = true;
            }
            *entry += count;
        }

        for (key, count) in other.edge_coverage {
            let entry = self.edge_coverage.entry(key).or_insert(0);
            if *entry == 0 {
                has_new = true;
            }
            *entry += count;
        }

        has_new
    }
}
```

---

## 附录 M: 真实 Bug 案例

### M.1 revm 历史 Bug

```
revm Bug 修复历史:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Bug #1: RETURNDATASIZE 在初始调用时返回错误值 (2022)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: 首次调用前 RETURNDATASIZE 应返回 0，但返回了垃圾值
原因: return_data 未正确初始化
```

```rust
// 修复前
pub struct Interpreter {
    // return_data 可能包含上次执行的数据
    pub return_data: Bytes,
}

// 修复后
impl Interpreter {
    pub fn new(contract: Contract, gas_limit: u64, is_static: bool) -> Self {
        Self {
            // 确保初始化为空
            return_data: Bytes::new(),
            // ...
        }
    }
}
```

```
Bug #2: EIP-3860 initcode 大小限制未正确应用 (2023)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: CREATE/CREATE2 未检查 initcode 最大长度 (49152 bytes)
影响: 可部署超大合约
```

```rust
// 修复后的 CREATE 实现
pub fn create<SPEC: Spec>(
    interpreter: &mut Interpreter,
    context: &mut impl Context,
) {
    // ...

    // EIP-3860: 检查 initcode 大小限制
    if SPEC::enabled(SpecId::SHANGHAI) {
        if init_code.len() > MAX_INITCODE_SIZE {
            interpreter.instruction_result = InstructionResult::CreateInitCodeSizeLimit;
            return;
        }

        // initcode 的额外 gas 成本
        let initcode_cost = INITCODE_WORD_COST * (init_code.len() as u64 + 31) / 32;
        if !interpreter.gas.record_cost(initcode_cost) {
            interpreter.instruction_result = InstructionResult::OutOfGas;
            return;
        }
    }

    // ...
}

const MAX_INITCODE_SIZE: usize = 2 * MAX_CODE_SIZE; // 49152
const INITCODE_WORD_COST: u64 = 2;
```

```
Bug #3: MCOPY gas 计算错误 (2024, Cancun)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: MCOPY gas 计算未考虑内存扩展的正确顺序
规范: 应先计算扩展到 max(dest+size, src+size)
```

```rust
// 修复前
pub fn mcopy_gas(dest: u64, src: u64, size: u64, memory: &Memory) -> u64 {
    // 错误: 分别计算两个扩展
    let dest_expand = memory.expansion_cost(dest + size);
    let src_expand = memory.expansion_cost(src + size);
    MCOPY_BASE + (size + 31) / 32 * 3 + dest_expand + src_expand
}

// 修复后
pub fn mcopy_gas(dest: u64, src: u64, size: u64, memory: &Memory) -> u64 {
    // 正确: 计算到最大范围的单次扩展
    let max_offset = std::cmp::max(dest, src) + size;
    let expansion_cost = memory.expansion_cost(max_offset);
    let copy_cost = (size + 31) / 32 * 3;
    MCOPY_BASE + copy_cost + expansion_cost
}
```

### M.2 通过 revm 发现的其他 EVM Bug

```
使用 revm 差分测试发现的 Bug:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Case #1: PUSH0 在 pre-Shanghai 硬分叉的行为
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
发现: 某些客户端在 pre-Shanghai 时将 PUSH0 (0x5f) 视为有效操作码
规范: 应该是 INVALID
revm: 正确行为 - 返回 InvalidOpcode
修复: 其他客户端统一返回错误

Case #2: TSTORE 在 STATICCALL 中的行为
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
发现: STATICCALL 内 TSTORE 应该失败 (write operation)
早期 revm: 允许 TSTORE (因为不影响持久状态)
规范澄清: TSTORE 在 static context 中应该失败
revm 修复: 添加 is_static 检查
```

```rust
// TSTORE static context 检查
pub fn tstore(interpreter: &mut Interpreter, _context: &mut impl Context) {
    // 检查 static context
    if interpreter.is_static {
        interpreter.instruction_result = InstructionResult::StateChangeDuringStaticCall;
        return;
    }

    let key = interpreter.stack.pop().unwrap();
    let value = interpreter.stack.pop().unwrap();

    // 执行 transient store
    // ...
}
```

### M.3 Reth 发现的共识 Bug

```
Reth (使用 revm) 在主网发现的问题:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Issue #1: 区块 gas 限制边界处理
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
发现: 某些历史区块的 gas 使用量计算与 geth 有微小差异
原因: 退款计算的精确时机
影响: 无法同步某些历史区块

Issue #2: Precompile 失败时的 gas 处理
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
发现: ecrecover 失败时应消耗所有提供的 gas
revm 行为: 只消耗 3000 gas (预编译基础成本)
geth 行为: 消耗所有 gas
修复: 统一为消耗所有 gas

Issue #3: Empty account 在 Berlin 后的处理
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
发现: touch empty account 的 gas 计算不一致
规范: EIP-2929 访问列表应包含 touched 的 empty accounts
修复: 正确追踪 empty account 访问
```

---

## 附录 N: Gas 优化技巧

### N.1 基于 revm 内部实现的优化

```rust
// revm 内部优化示例 - 可供合约开发者参考

// 1. 栈操作优化
// revm 对连续的 DUP/SWAP 有优化路径
// 合约应避免不必要的栈操作

// 差的模式 (多次栈操作):
// DUP1 DUP3 SWAP2 ADD
// 好的模式 (直接计算):
// ADD

// 2. 内存访问模式
// revm 使用 resize_memory! 宏进行延迟扩展
// 连续访问相邻内存区域比离散访问更高效

// 3. 存储访问缓存
// revm 的 JournaledState 缓存已读取的值
// 但每次 SLOAD 仍有开销 - 建议在合约中缓存到 memory
```

### N.2 Solidity 优化与 revm 实现对应

```solidity
// 优化示例 - 对应 revm 内部处理

contract GasOptimization {
    // 1. 利用 transient storage (revm TransientStorage)
    // Gas: TLOAD/TSTORE = 100 vs SLOAD/SSTORE = 2100/22100
    modifier nonReentrant() {
        assembly {
            if tload(0) {revert(0, 0)}
            tstore(0, 1)
        }
        _;
        assembly {
            tstore(0, 0)
        }
    }

    // 2. 批量读取优化
    // revm 对同一 slot 的重复 SLOAD 返回缓存值 (warm access)
    // 但缓存查找仍有开销
    struct UserInfo {
        uint128 balance;    // slot 0, lower
        uint128 lastUpdate; // slot 0, upper
    }

    mapping(address => UserInfo) public users;

    // 差: 两次 SLOAD (即使是 warm)
    function getBothBad(address user) external view returns (uint128, uint128) {
        return (users[user].balance, users[user].lastUpdate);
    }

    // 好: 一次 SLOAD，本地解包
    function getBothGood(address user) external view returns (uint128, uint128) {
        UserInfo memory info = users[user];
        return (info.balance, info.lastUpdate);
    }

    // 3. MCOPY 优化 (Cancun)
    // revm 的 mcopy 实现使用 memmove
    function copyData(bytes memory src) external pure returns (bytes memory) {
        bytes memory dst = new bytes(src.length);
        assembly {
            mcopy(add(dst, 32), add(src, 32), mload(src))
        }
        return dst;
    }

    // 4. 短路求值与 gas
    // revm 按顺序执行操作码，编译器的短路优化很重要
    function checkConditions(
        address user,
        uint256 amount
    ) external view returns (bool) {
        // 把便宜的检查放前面
        if (amount == 0) return false;           // 比较 calldata
        if (msg.sender == address(0)) return false; // 比较 msg 字段
        return balances[user] >= amount;         // SLOAD (贵)
    }
}
```

### N.3 Gas 消耗热力图

```
revm 操作码 Gas 消耗分布:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
操作                     │ Gas      │ 热度 │ 优化建议
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SLOAD (cold)            │ 2100     │ 🔴🔴🔴 │ 缓存到 memory
SSTORE (cold, 0→n)      │ 22100    │ 🔴🔴🔴 │ 打包存储槽
SSTORE (cold, n→m)      │ 5000     │ 🔴🔴  │ 批量更新
CALL (cold account)     │ 2600+    │ 🔴🔴  │ 访问列表优化
CREATE                  │ 32000+   │ 🔴🔴🔴 │ CREATE2 + minimal proxy
BALANCE (cold)          │ 2600     │ 🔴🔴  │ 缓存或访问列表
EXTCODESIZE (cold)      │ 2600     │ 🔴🔴  │ 检查 code.length == 0
LOG (per topic)         │ 375      │ 🟡   │ 合并事件
KECCAK256 (per word)    │ 6        │ 🟡   │ 预计算常量 hash
MLOAD/MSTORE            │ 3        │ 🟢   │ 首选临时数据
ADD/SUB/MUL             │ 3-5      │ 🟢   │ 无需优化
PUSH/DUP/SWAP           │ 3        │ 🟢   │ 无需优化
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

revm gas 优化检查清单:
□ 使用 unchecked {} 处理已知安全的算术
□ 打包小于 256 位的变量到同一 slot
□ 使用 immutable 代替状态变量
□ 使用 calldata 代替 memory 参数
□ 避免循环中的 SLOAD
□ 使用 transient storage 做临时状态
□ 用 MCOPY 代替循环复制
□ 预计算常量 keccak256 值
□ 使用访问列表预热账户/槽位
```

---

## 附录 O: 跨链 EVM 变体

### O.1 基于 revm 的 L2 实现

```
revm 在 L2 生态中的应用:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
项目          │ 使用方式          │ 主要修改
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Reth          │ 核心执行引擎      │ 完整节点集成
op-revm       │ Optimism 执行    │ L1 费用, Deposit TX
Arbitrum      │ 参考/测试        │ Stylus WASM 互操作
Polygon zkEVM │ 差分测试         │ ZK 电路验证
Scroll        │ 差分测试         │ ZK 证明生成
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### O.2 zkEVM 与 revm

```rust
// zkEVM 验证中使用 revm 进行参考执行
pub struct ZkEvmVerifier {
    revm: Evm<'static, (), InMemoryDB>,
    circuit_executor: ZkCircuitExecutor,
}

impl ZkEvmVerifier {
    /// 验证 ZK 证明与 revm 执行结果一致
    pub fn verify_execution(
        &mut self,
        block: &Block,
        zk_proof: &ZkProof,
    ) -> VerificationResult {
        // 1. 使用 revm 执行区块
        let revm_result = self.execute_with_revm(block)?;

        // 2. 解析 ZK 证明的 public inputs
        let zk_state_root = zk_proof.public_inputs.state_root;
        let zk_gas_used = zk_proof.public_inputs.gas_used;

        // 3. 比较结果
        if revm_result.state_root != zk_state_root {
            return VerificationResult::StateMismatch {
                revm: revm_result.state_root,
                zk: zk_state_root,
            };
        }

        if revm_result.gas_used != zk_gas_used {
            return VerificationResult::GasMismatch {
                revm: revm_result.gas_used,
                zk: zk_gas_used,
            };
        }

        // 4. 验证 ZK 证明本身
        self.circuit_executor.verify(zk_proof)?;

        VerificationResult::Valid
    }
}

// zkEVM 操作码差异处理
pub fn zkevm_opcode_differences() -> Vec<OpcodeDifference> {
    vec![
        OpcodeDifference {
            opcode: SELFDESTRUCT,
            standard_behavior: "销毁合约，发送余额",
            zkevm_behavior: "仅发送余额，不销毁",
            reason: "ZK 电路中处理合约删除很复杂",
        },
        OpcodeDifference {
            opcode: BLOCKHASH,
            standard_behavior: "返回最近 256 个区块的 hash",
            zkevm_behavior: "可能限制为更少的区块",
            reason: "减少 ZK 证明复杂度",
        },
        OpcodeDifference {
            opcode: DIFFICULTY,
            standard_behavior: "返回区块难度",
            zkevm_behavior: "返回 0 或固定值",
            reason: "L2 没有 PoW 难度",
        },
    ]
}
```

### O.3 Optimism 扩展

```rust
// op-revm 特定功能
use op_revm::{OpBuilder, OpContext, OpSpec};

// L1 费用计算
pub struct L1FeeCalculator {
    l1_base_fee: U256,
    l1_blob_base_fee: Option<U256>,
    base_fee_scalar: u32,
    blob_base_fee_scalar: u32,
}

impl L1FeeCalculator {
    /// Ecotone 费用公式 (EIP-4844 后)
    pub fn calculate_l1_fee_ecotone(&self, tx_data: &[u8]) -> U256 {
        let compressed_size = self.estimate_compressed_size(tx_data);

        // L1 Fee = (base_fee_scalar * l1_base_fee +
        //           blob_base_fee_scalar * l1_blob_base_fee) * compressed_size / 16
        let fee = U256::from(self.base_fee_scalar) * self.l1_base_fee
            + U256::from(self.blob_base_fee_scalar)
                * self.l1_blob_base_fee.unwrap_or_default();

        fee * U256::from(compressed_size) / U256::from(16_000_000)
    }

    fn estimate_compressed_size(&self, data: &[u8]) -> u64 {
        // 零字节成本为 4，非零字节成本为 16
        let zero_bytes = data.iter().filter(|&&b| b == 0).count() as u64;
        let non_zero_bytes = data.len() as u64 - zero_bytes;
        zero_bytes * 4 + non_zero_bytes * 16
    }
}

// Deposit 交易处理
pub fn process_deposit_tx(
    context: &mut OpContext,
    deposit: &DepositTransaction,
) -> Result<ExecutionResult, OpError> {
    // Deposit 交易特性:
    // 1. 不消耗 L2 gas (由 L1 支付)
    // 2. 有固定的 source hash
    // 3. mint 字段指定铸造的 ETH

    // 铸造 ETH 到接收地址
    if let Some(mint) = deposit.mint {
        context.evm.db_mut().increment_balance(deposit.to, mint)?;
    }

    // 执行交易 (如果有 data)
    if !deposit.data.is_empty() {
        let result = context.evm.transact()?;
        Ok(result.result)
    } else {
        Ok(ExecutionResult::Success {
            reason: SuccessReason::Stop,
            gas_used: 0,
            gas_refunded: 0,
            logs: vec![],
            output: Output::Call(Bytes::new()),
        })
    }
}
```

### O.4 EVM 兼容性检查清单

```
部署到其他 EVM 链的检查清单:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. CHAINID 处理
   □ 合约是否正确使用 block.chainid？
   □ 签名验证是否包含 chainId？
   □ 跨链消息是否验证源链？

2. Gas 差异
   □ 了解目标链的 gas 定价模型
   □ 测试 gas 估算在目标链是否准确
   □ 预编译合约 gas 成本可能不同

3. 操作码支持
   □ PUSH0 (需要 Shanghai)
   □ TLOAD/TSTORE (需要 Cancun)
   □ MCOPY (需要 Cancun)
   □ SELFDESTRUCT (可能被限制)

4. 预编译合约
   □ ecrecover (0x01) - 通常支持
   □ modexp (0x05) - gas 成本可能不同
   □ BLS12-381 (0x0b-0x13) - 很多链不支持
   □ Point evaluation (0x0a) - 需要 4844 支持

5. 区块属性
   □ block.difficulty/prevrandao
   □ block.basefee (需要 EIP-1559)
   □ blobbasefee (需要 EIP-4844)
   □ block.number 的含义 (某些 L2 有 L1/L2 区块号)

6. 时间相关
   □ block.timestamp 精度
   □ 区块时间间隔差异
   □ 时间戳操纵风险
```

```rust
// 跨链兼容性测试工具
pub struct ChainCompatibilityChecker {
    chains: Vec<ChainConfig>,
}

impl ChainCompatibilityChecker {
    pub fn check_contract(&self, bytecode: &Bytes) -> CompatibilityReport {
        let mut report = CompatibilityReport::default();

        // 分析使用的操作码
        let opcodes = analyze_opcodes(bytecode);

        for chain in &self.chains {
            let mut chain_report = ChainReport {
                name: chain.name.clone(),
                compatible: true,
                warnings: vec![],
            };

            // 检查操作码兼容性
            for opcode in &opcodes {
                if !chain.supports_opcode(*opcode) {
                    chain_report.compatible = false;
                    chain_report.warnings.push(format!(
                        "Opcode {:02x} not supported",
                        opcode
                    ));
                }
            }

            // 检查硬分叉要求
            if opcodes.contains(&PUSH0) && chain.spec < SpecId::SHANGHAI {
                chain_report.compatible = false;
                chain_report.warnings.push("Requires Shanghai fork".into());
            }

            if (opcodes.contains(&TLOAD) || opcodes.contains(&TSTORE))
                && chain.spec < SpecId::CANCUN
            {
                chain_report.compatible = false;
                chain_report.warnings.push("Requires Cancun fork".into());
            }

            report.chains.push(chain_report);
        }

        report
    }
}
```

---

## 附录 P：账户抽象 (Account Abstraction)

### P.1 ERC-4337 支持

```rust
// revm 中实现 ERC-4337 UserOperation 验证
use revm::{
    primitives::{Address, Bytes, U256, B256, AccountInfo},
    Database, Evm,
};

/// ERC-4337 UserOperation 结构
#[derive(Clone, Debug)]
pub struct UserOperation {
    pub sender: Address,
    pub nonce: U256,
    pub init_code: Bytes,
    pub call_data: Bytes,
    pub call_gas_limit: U256,
    pub verification_gas_limit: U256,
    pub pre_verification_gas: U256,
    pub max_fee_per_gas: U256,
    pub max_priority_fee_per_gas: U256,
    pub paymaster_and_data: Bytes,
    pub signature: Bytes,
}

impl UserOperation {
    /// 计算 UserOperation 哈希
    pub fn hash(&self, entry_point: Address, chain_id: u64) -> B256 {
        use alloy_primitives::keccak256;

        // Pack user op without signature
        let packed = self.pack_for_signature();
        let inner_hash = keccak256(&packed);

        // Final hash includes entry point and chain id
        let mut data = Vec::with_capacity(84);
        data.extend_from_slice(inner_hash.as_slice());
        data.extend_from_slice(entry_point.as_slice());
        data.extend_from_slice(&chain_id.to_be_bytes());

        keccak256(&data)
    }

    fn pack_for_signature(&self) -> Bytes {
        use alloy_sol_types::{SolType, sol_data};

        // ABI encode all fields except signature
        let encoded = (
            self.sender,
            self.nonce,
            keccak256(&self.init_code),
            keccak256(&self.call_data),
            self.call_gas_limit,
            self.verification_gas_limit,
            self.pre_verification_gas,
            self.max_fee_per_gas,
            self.max_priority_fee_per_gas,
            keccak256(&self.paymaster_and_data),
        );

        // Use abi encoding
        alloy_sol_types::SolValue::abi_encode(&encoded).into()
    }
}

/// ERC-4337 Bundler 实现
pub struct Bundler<DB: Database> {
    entry_point: Address,
    beneficiary: Address,
    evm: Evm<'static, (), DB>,
}

impl<DB: Database + Clone> Bundler<DB> {
    /// 验证 UserOperation
    pub fn validate_user_op(&mut self, user_op: &UserOperation) -> Result<ValidationResult, BundlerError> {
        // 1. 检查 sender 是否存在
        let sender_exists = self.account_exists(user_op.sender)?;

        // 2. 如果不存在且有 init_code，部署账户
        if !sender_exists {
            if user_op.init_code.is_empty() {
                return Err(BundlerError::AccountNotDeployed);
            }
            self.simulate_deployment(user_op)?;
        }

        // 3. 调用 validateUserOp
        let validation = self.call_validate_user_op(user_op)?;

        // 4. 检查 Gas 限制
        if validation.pre_op_gas > user_op.verification_gas_limit {
            return Err(BundlerError::ValidationGasExceeded);
        }

        // 5. 检查存款或 paymaster
        self.check_prefund(user_op, &validation)?;

        Ok(validation)
    }

    /// 模拟执行 UserOperation
    pub fn simulate_user_op(&mut self, user_op: &UserOperation) -> Result<SimulationResult, BundlerError> {
        // 使用 revm 的状态快照功能
        let checkpoint = self.evm.db_mut().checkpoint();

        // 执行验证
        let validation = self.validate_user_op(user_op)?;

        // 执行实际调用
        let execution = self.execute_user_op_call(user_op)?;

        // 回滚状态
        self.evm.db_mut().revert(checkpoint);

        Ok(SimulationResult {
            validation,
            execution,
            gas_used: validation.pre_op_gas.saturating_add(execution.gas_used),
        })
    }

    /// 批量执行 UserOperations
    pub fn handle_ops(&mut self, ops: Vec<UserOperation>) -> Result<HandleOpsResult, BundlerError> {
        let mut results = Vec::with_capacity(ops.len());
        let mut total_gas = U256::ZERO;

        for op in ops {
            // 验证每个操作
            let validation = self.validate_user_op(&op)?;

            // 执行操作
            let execution = self.execute_user_op(&op)?;

            // 收集结果
            total_gas = total_gas.saturating_add(execution.gas_used);
            results.push(UserOpResult {
                user_op_hash: op.hash(self.entry_point, self.chain_id()),
                success: execution.success,
                gas_used: execution.gas_used,
                return_data: execution.return_data,
            });
        }

        // 补偿 beneficiary
        self.compensate_beneficiary(total_gas)?;

        Ok(HandleOpsResult { results, total_gas })
    }

    fn account_exists(&self, address: Address) -> Result<bool, BundlerError> {
        let account = self.evm.db().basic(address)
            .map_err(|_| BundlerError::DatabaseError)?;
        Ok(account.map(|a| a.code_hash != KECCAK_EMPTY).unwrap_or(false))
    }

    fn chain_id(&self) -> u64 {
        self.evm.cfg().chain_id
    }
}

#[derive(Debug)]
pub struct ValidationResult {
    pub pre_op_gas: U256,
    pub prefund: U256,
    pub valid_after: u64,
    pub valid_until: u64,
    pub paymaster_context: Bytes,
}

#[derive(Debug)]
pub struct SimulationResult {
    pub validation: ValidationResult,
    pub execution: ExecutionResult,
    pub gas_used: U256,
}

#[derive(Debug)]
pub enum BundlerError {
    AccountNotDeployed,
    ValidationGasExceeded,
    InsufficientPrefund,
    InvalidSignature,
    DatabaseError,
    ExecutionReverted(Bytes),
}
```

### P.2 EIP-7702 Set Code Transaction

```rust
// EIP-7702: 为 EOA 设置临时代码
use revm::{
    primitives::{Address, Bytes, U256, B256, TxKind},
    handler::register::EvmHandler,
    Context, ContextPrecompiles,
};

/// EIP-7702 授权元组
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Authorization {
    /// 链 ID (0 表示所有链)
    pub chain_id: u64,
    /// 要设置的代码地址
    pub address: Address,
    /// 签名者的 nonce
    pub nonce: u64,
}

impl Authorization {
    /// 计算授权哈希
    pub fn signature_hash(&self) -> B256 {
        use alloy_primitives::keccak256;
        use alloy_rlp::Encodable;

        // EIP-7702 magic: 0x05
        let mut buf = vec![0x05];

        // RLP encode (chain_id, address, nonce)
        let list = (self.chain_id, self.address, self.nonce);
        list.encode(&mut buf);

        keccak256(&buf)
    }

    /// 从签名恢复授权者地址
    pub fn recover_authority(&self, signature: &Signature) -> Result<Address, SignatureError> {
        let hash = self.signature_hash();
        signature.recover_address_from_prehash(&hash)
    }
}

/// 签名的授权
#[derive(Clone, Debug)]
pub struct SignedAuthorization {
    pub authorization: Authorization,
    pub signature: Signature,
}

impl SignedAuthorization {
    /// 验证并恢复授权者
    pub fn into_recovered(self) -> Result<RecoveredAuthorization, SignatureError> {
        let authority = self.authorization.recover_authority(&self.signature)?;
        Ok(RecoveredAuthorization {
            authorization: self.authorization,
            authority,
        })
    }
}

#[derive(Clone, Debug)]
pub struct RecoveredAuthorization {
    pub authorization: Authorization,
    pub authority: Address,
}

/// EIP-7702 交易处理器
pub struct Eip7702Handler;

impl Eip7702Handler {
    /// 处理 EIP-7702 授权列表
    pub fn process_authorization_list<DB: Database>(
        context: &mut Context<(), DB>,
        authorizations: &[RecoveredAuthorization],
    ) -> Result<u64, Eip7702Error> {
        let mut refund = 0u64;
        let chain_id = context.evm.env.cfg.chain_id;

        for auth in authorizations {
            // 验证 chain_id
            if auth.authorization.chain_id != 0 && auth.authorization.chain_id != chain_id {
                continue; // 跳过无效授权
            }

            let authority = auth.authority;

            // 获取授权者账户
            let account = context.evm.journaled_state
                .load_account(authority, &mut context.evm.db)
                .map_err(|_| Eip7702Error::DatabaseError)?;

            // 验证 nonce
            if account.info.nonce != auth.authorization.nonce {
                continue;
            }

            // 检查是否已有代码
            let had_code = !account.info.is_empty_code_hash();

            // 设置代理代码
            let delegation_code = Self::create_delegation_code(auth.authorization.address);
            account.info.code_hash = keccak256(&delegation_code);
            account.info.code = Some(Bytecode::new_raw(delegation_code.into()));

            // 增加 nonce
            account.info.nonce += 1;

            // 计算 gas 退款
            if !had_code {
                refund += PER_EMPTY_ACCOUNT_COST;
            }
        }

        Ok(refund)
    }

    /// 创建委托代码前缀
    /// 格式: 0xef0100 || address (23 bytes)
    fn create_delegation_code(target: Address) -> Vec<u8> {
        let mut code = Vec::with_capacity(23);
        code.extend_from_slice(&[0xef, 0x01, 0x00]); // 魔术前缀
        code.extend_from_slice(target.as_slice());
        code
    }

    /// 检查地址是否有委托代码
    pub fn get_delegation_target(code: &[u8]) -> Option<Address> {
        if code.len() == 23 && code.starts_with(&[0xef, 0x01, 0x00]) {
            Some(Address::from_slice(&code[3..]))
        } else {
            None
        }
    }
}

/// 注册 EIP-7702 处理器到 revm
pub fn register_7702_handler<DB, EXT>(handler: &mut EvmHandler<'_, EXT, DB>)
where
    DB: Database,
{
    // 修改 pre_execution 阶段
    let old_pre_exec = handler.pre_execution.clone();
    handler.pre_execution.load_accounts = Arc::new(move |context| {
        // 首先处理授权列表
        if let Some(auth_list) = context.evm.env.tx.authorization_list.as_ref() {
            let recovered: Vec<_> = auth_list.iter()
                .filter_map(|a| a.clone().into_recovered().ok())
                .collect();

            Eip7702Handler::process_authorization_list(context, &recovered)?;
        }

        // 继续原有的账户加载逻辑
        (old_pre_exec.load_accounts)(context)
    });

    // 修改 call 执行以支持委托
    let old_call = handler.execution.call.clone();
    handler.execution.call = Arc::new(move |context, inputs| {
        // 检查被调用地址是否有委托代码
        if let Some(account) = context.evm.journaled_state.account(inputs.target_address) {
            if let Some(code) = &account.info.code {
                if let Some(target) = Eip7702Handler::get_delegation_target(code.bytes()) {
                    // 重定向到委托目标，但保持原始上下文
                    let mut new_inputs = inputs.clone();
                    new_inputs.bytecode_address = target;
                    return (old_call)(context, new_inputs);
                }
            }
        }

        (old_call)(context, inputs)
    });
}

const PER_EMPTY_ACCOUNT_COST: u64 = 25000;

#[derive(Debug)]
pub enum Eip7702Error {
    DatabaseError,
    InvalidSignature,
    InvalidChainId,
}
```

### P.3 Smart Account 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Account Abstraction 架构                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────┐     ┌──────────────────────────────────┐ │
│   │   User Wallet    │────▶│         UserOperation            │ │
│   │   (Off-chain)    │     │  - sender (smart account)        │ │
│   └──────────────────┘     │  - callData                      │ │
│                            │  - signature                      │ │
│                            └────────────┬─────────────────────┘ │
│                                         │                       │
│                                         ▼                       │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │                      Bundler                              │ │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │ │
│   │  │ Validation  │  │ Gas Est.    │  │ MEV Protection  │  │ │
│   │  │   Pool      │  │   Engine    │  │   (Flashbots)   │  │ │
│   │  └─────────────┘  └─────────────┘  └─────────────────┘  │ │
│   └───────────────────────────┬──────────────────────────────┘ │
│                               │                                 │
│                               ▼                                 │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │                    EntryPoint                             │ │
│   │  handleOps(UserOperation[] ops, address payable beneficiary) │
│   │  ├── validateUserOp()                                     │ │
│   │  ├── executeUserOp()                                      │ │
│   │  └── compensate()                                         │ │
│   └────────────┬──────────────────────┬──────────────────────┘ │
│                │                      │                         │
│                ▼                      ▼                         │
│   ┌────────────────────┐  ┌────────────────────────────────┐  │
│   │   Smart Account    │  │         Paymaster               │  │
│   │  ┌──────────────┐  │  │  ┌──────────────────────────┐  │  │
│   │  │validateUserOp│  │  │  │ validatePaymasterUserOp  │  │  │
│   │  │  execute()   │  │  │  │     postOp()             │  │  │
│   │  └──────────────┘  │  │  └──────────────────────────┘  │  │
│   └────────────────────┘  └────────────────────────────────┘  │
│                                                                 │
│   ═══════════════════════════════════════════════════════════  │
│                                                                 │
│                      EIP-7702 模式                              │
│                                                                 │
│   ┌──────────────────┐     ┌──────────────────────────────────┐ │
│   │    EOA Wallet    │────▶│    Type 4 Transaction            │ │
│   │                  │     │  - authorization_list:           │ │
│   └──────────────────┘     │    [(chain_id, addr, nonce, sig)]│ │
│           │                └────────────────┬─────────────────┘ │
│           │ signs                           │                   │
│           ▼                                 ▼                   │
│   ┌──────────────────┐     ┌──────────────────────────────────┐ │
│   │  Authorization   │     │         EVM Execution            │ │
│   │  0x05 || RLP(    │     │  1. Process auth list            │ │
│   │    chain_id,     │     │  2. Set delegation code          │ │
│   │    address,      │     │     0xef0100 || target           │ │
│   │    nonce         │     │  3. Execute with delegated code  │ │
│   │  )               │     │  4. EOA acts as smart account    │ │
│   └──────────────────┘     └──────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### P.4 revm 中的 AA 测试

```rust
#[cfg(test)]
mod aa_tests {
    use super::*;
    use revm::{
        db::InMemoryDB,
        primitives::{TransactTo, TxEnv},
    };

    /// 测试基本的 EIP-7702 授权流程
    #[test]
    fn test_eip7702_basic_authorization() {
        let mut db = InMemoryDB::default();

        // 部署目标合约（简单的账户实现）
        let account_impl = Address::from_slice(&[0x11; 20]);
        let account_code = hex::decode(
            "608060405234801561001057600080fd5b506004361061002b5760003560e01c8063b61d27f614610030575b600080fd5b61004a60048036038101906100459190610123565b61004c565b005b600080838573ffffffffffffffffffffffffffffffffffffffff168460405161007591906101e5565b60006040518083038185875af1925050503d80600081146100b2576040519150601f19603f3d011682016040523d82523d6000602084013e6100b7565b606091505b5091509150816100fc576040517f08c379a00000000000000000000000000000000000000000000000000000000081526004016100f390610252565b60405180910390fd5b5050505050565b600080fd5b600073ffffffffffffffffffffffffffffffffffffffff82169050919050565b600061013382610108565b9050919050565b61014381610128565b811461014e57600080fd5b50565b6000813590506101608161013a565b92915050565b600080fd5b600080fd5b600080fd5b60008083601f84011261018b5761018a610166565b5b8235905067ffffffffffffffff8111156101a8576101a761016b565b5b6020830191508360018202830111156101c4576101c3610170565b5b9250929050565b60006101d78385610219565b93506101e4838584610224565b82840190509392505050565b60006101fd8284866101cb565b91508190509392505050565b600082825260208201905092915050565b600082825260208201905092915050565b82818337600083830152505050565b600061024860208361020d565b915061025382610272565b602082019050919050565b600060208201905081810360008301526102778161023b565b9050919050565b7f63616c6c206661696c6564000000000000000000000000000000000000000000600082015250565b610200610200565b610200610200565b610200610200565b"
        ).unwrap();

        db.insert_account_info(
            account_impl,
            AccountInfo {
                code_hash: keccak256(&account_code),
                code: Some(Bytecode::new_raw(account_code.into())),
                ..Default::default()
            },
        );

        // 创建 EOA
        let eoa = Address::from_slice(&[0x22; 20]);
        let eoa_key = SigningKey::random(&mut rand::thread_rng());

        db.insert_account_info(
            eoa,
            AccountInfo {
                balance: U256::from(10).pow(U256::from(18)), // 1 ETH
                nonce: 0,
                ..Default::default()
            },
        );

        // 创建授权
        let authorization = Authorization {
            chain_id: 1,
            address: account_impl,
            nonce: 0,
        };

        // 签名授权
        let sig_hash = authorization.signature_hash();
        let signature = eoa_key.sign_prehash_recoverable(&sig_hash).unwrap();

        let signed_auth = SignedAuthorization {
            authorization,
            signature: signature.into(),
        };

        // 构建 EIP-7702 交易
        let mut evm = Evm::builder()
            .with_db(db)
            .modify_cfg_env(|cfg| {
                cfg.chain_id = 1;
            })
            .modify_tx_env(|tx| {
                tx.caller = eoa;
                tx.transact_to = TransactTo::Call(eoa); // 调用自己
                tx.data = hex::decode("b61d27f6").unwrap().into(); // execute selector
                tx.authorization_list = Some(vec![signed_auth]);
            })
            .build();

        // 注册 EIP-7702 处理器
        register_7702_handler(&mut evm.handler);

        // 执行交易
        let result = evm.transact().unwrap();

        // 验证 EOA 现在有委托代码
        let eoa_account = evm.db().basic(eoa).unwrap().unwrap();
        let code = eoa_account.code.as_ref().unwrap();

        assert!(code.bytes().starts_with(&[0xef, 0x01, 0x00]));
        assert_eq!(
            Eip7702Handler::get_delegation_target(code.bytes()),
            Some(account_impl)
        );
    }

    /// 测试 Bundler 验证逻辑
    #[test]
    fn test_bundler_validate_user_op() {
        let db = setup_test_db_with_entrypoint();
        let mut bundler = Bundler::new(db, ENTRY_POINT, BENEFICIARY);

        // 创建有效的 UserOperation
        let user_op = create_test_user_op();

        // 验证应该成功
        let result = bundler.validate_user_op(&user_op);
        assert!(result.is_ok());

        let validation = result.unwrap();
        assert!(validation.prefund >= user_op.pre_verification_gas);
    }
}
```

---

## 附录 Q：形式化验证

### Q.1 KEVM 与 revm 对比

```
┌─────────────────────────────────────────────────────────────────┐
│                   形式化验证工具对比                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                        KEVM                                │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │                  K Framework                         │  │  │
│  │  │  - 语义定义语言                                      │  │  │
│  │  │  - 可执行规范                                        │  │  │
│  │  │  - 自动生成解释器/验证器                             │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                         │                                  │  │
│  │                         ▼                                  │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              EVM Semantics (K)                       │  │  │
│  │  │  module EVM                                          │  │  │
│  │  │    syntax OpCode ::= "ADD" | "MUL" | "SUB" | ...     │  │  │
│  │  │    rule <k> ADD => . ... </k>                        │  │  │
│  │  │         <wordStack> W0 : W1 : WS => W0 +Word W1 : WS │  │  │
│  │  │  endmodule                                           │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                 revm 形式化验证方法                        │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              Property-Based Testing                  │  │  │
│  │  │  - proptest / quickcheck                             │  │  │
│  │  │  - 模糊测试生成输入                                   │  │  │
│  │  │  - 验证不变量                                         │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                         +                                  │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              Symbolic Execution                      │  │  │
│  │  │  - KLEE (通过 C FFI)                                 │  │  │
│  │  │  - Haybale (原生 Rust)                               │  │  │
│  │  │  - 路径探索与约束求解                                 │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                         +                                  │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │            Differential Testing                      │  │  │
│  │  │  - 与 geth 对比执行                                  │  │  │
│  │  │  - 与 evmone 对比执行                                │  │  │
│  │  │  - execution-spec-tests                              │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Q.2 K 语言 EVM 语义示例

```k
// KEVM: ADD 操作码的形式化定义
module EVM-ARITHMETIC
    imports EVM-DATA

    // ADD 操作
    rule [add]:
        <k> ADD => . ... </k>
        <wordStack> W0 : W1 : WS => chop(W0 +Int W1) : WS </wordStack>
        <gas> G => G -Int Gverylow </gas>
      requires G >=Int Gverylow

    // MUL 操作
    rule [mul]:
        <k> MUL => . ... </k>
        <wordStack> W0 : W1 : WS => chop(W0 *Int W1) : WS </wordStack>
        <gas> G => G -Int Glow </gas>
      requires G >=Int Glow

    // ADDMOD 操作
    rule [addmod]:
        <k> ADDMOD => . ... </k>
        <wordStack> W0 : W1 : W2 : WS => chop((W0 +Int W1) modInt W2) : WS </wordStack>
        <gas> G => G -Int Gmid </gas>
      requires W2 =/=Int 0
       andBool G >=Int Gmid

    rule [addmod-zero]:
        <k> ADDMOD => . ... </k>
        <wordStack> _ : _ : 0 : WS => 0 : WS </wordStack>
        <gas> G => G -Int Gmid </gas>
      requires G >=Int Gmid

    // EXP 操作 - gas 取决于指数大小
    rule [exp]:
        <k> EXP => . ... </k>
        <wordStack> W0 : W1 : WS => W0 ^Int W1 : WS </wordStack>
        <gas> G => G -Int (Gexp +Int (Gexpbyte *Int (log256Int(W1) +Int 1))) </gas>
      requires W1 >Int 0
       andBool G >=Int (Gexp +Int (Gexpbyte *Int (log256Int(W1) +Int 1)))

    rule [exp-zero]:
        <k> EXP => . ... </k>
        <wordStack> _ : 0 : WS => 1 : WS </wordStack>
        <gas> G => G -Int Gexp </gas>
      requires G >=Int Gexp

endmodule

// KEVM: CALL 操作码的形式化定义
module EVM-CALL
    imports EVM-CONTROL-FLOW

    rule [call]:
        <k> CALL => #call ADDR VALUE ARGSOFFSET ARGSSIZE RETOFFSET RETSIZE ~> #return ... </k>
        <wordStack> GASCAP : ADDR : VALUE : ARGSOFFSET : ARGSSIZE : RETOFFSET : RETSIZE : WS
                 => WS </wordStack>
        <localMem> LM </localMem>
        <gas> GAVAIL => GAVAIL -Int Cgascap(GASCAP, GAVAIL, VALUE =/=Int 0, ADDR) </gas>
        <callDepth> CD </callDepth>
      requires CD <Int 1024
       andBool GAVAIL >=Int Cgascap(GASCAP, GAVAIL, VALUE =/=Int 0, ADDR)

    // Gas 计算
    syntax Int ::= Cgascap(Int, Int, Bool, Int) [function]
    rule Cgascap(GASCAP, GAVAIL, XFER, ADDR)
         => minInt(#allBut64th(GAVAIL -Int Cextra(XFER, ADDR)), GASCAP)

    syntax Int ::= Cextra(Bool, Int) [function]
    rule Cextra(XFER, ADDR) => Gcall +Int Cxfer(XFER) +Int Cnew(ADDR)

    syntax Int ::= Cxfer(Bool) [function]
    rule Cxfer(true)  => Gcallvalue
    rule Cxfer(false) => 0

endmodule
```

### Q.3 Rust 属性测试

```rust
use proptest::prelude::*;
use revm::primitives::{U256, I256};

// 算术操作的属性测试
proptest! {
    /// ADD 满足交换律
    #[test]
    fn add_commutative(a: u128, b: u128) {
        let a = U256::from(a);
        let b = U256::from(b);

        prop_assert_eq!(
            a.wrapping_add(b),
            b.wrapping_add(a)
        );
    }

    /// ADD 满足结合律
    #[test]
    fn add_associative(a: u64, b: u64, c: u64) {
        let a = U256::from(a);
        let b = U256::from(b);
        let c = U256::from(c);

        prop_assert_eq!(
            a.wrapping_add(b).wrapping_add(c),
            a.wrapping_add(b.wrapping_add(c))
        );
    }

    /// MUL 分配律
    #[test]
    fn mul_distributive(a: u64, b: u64, c: u64) {
        let a = U256::from(a);
        let b = U256::from(b);
        let c = U256::from(c);

        // a * (b + c) = a * b + a * c
        let left = a.wrapping_mul(b.wrapping_add(c));
        let right = a.wrapping_mul(b).wrapping_add(a.wrapping_mul(c));

        prop_assert_eq!(left, right);
    }

    /// SDIV 与 DIV 在正数上一致
    #[test]
    fn sdiv_positive_equals_div(a in 0i128..i128::MAX, b in 1i128..i128::MAX) {
        let a_u = U256::from(a as u128);
        let b_u = U256::from(b as u128);
        let a_s = I256::try_from(a_u).unwrap();
        let b_s = I256::try_from(b_u).unwrap();

        let div_result = a_u / b_u;
        let sdiv_result = a_s / b_s;

        prop_assert_eq!(
            div_result,
            U256::from_be_bytes(sdiv_result.to_be_bytes())
        );
    }

    /// ADDMOD: (a + b) mod n 在 n > 0 时有效
    #[test]
    fn addmod_valid(a: u64, b: u64, n in 1u64..) {
        let a = U256::from(a);
        let b = U256::from(b);
        let n = U256::from(n);

        let result = a.wrapping_add(b) % n;

        prop_assert!(result < n);
    }

    /// EXP: a^0 = 1
    #[test]
    fn exp_zero_power(a: u128) {
        let a = U256::from(a);
        let result = a.pow(U256::ZERO);
        prop_assert_eq!(result, U256::from(1u64));
    }

    /// EXP: a^1 = a
    #[test]
    fn exp_one_power(a: u128) {
        let a = U256::from(a);
        let result = a.pow(U256::from(1u64));
        prop_assert_eq!(result, a);
    }
}

// 内存操作的属性测试
proptest! {
    /// MLOAD/MSTORE 往返一致性
    #[test]
    fn memory_roundtrip(offset in 0u32..1000u32, value: [u8; 32]) {
        let mut memory = SharedMemory::new();
        memory.resize((offset as usize + 32).next_multiple_of(32));

        // MSTORE
        memory.set_word(offset as usize, &value);

        // MLOAD
        let loaded = memory.get_word(offset as usize);

        prop_assert_eq!(loaded, value);
    }

    /// MCOPY 不影响源数据（非重叠情况）
    #[test]
    fn mcopy_non_overlapping(
        src in 0usize..500usize,
        dst in 500usize..1000usize,
        len in 1usize..100usize
    ) {
        let mut memory = SharedMemory::new();
        memory.resize((dst + len).next_multiple_of(32));

        // 初始化源数据
        let src_data: Vec<u8> = (0..len).map(|i| i as u8).collect();
        memory.set(src, &src_data);

        // 保存源数据副本
        let original_src = memory.slice(src, len).to_vec();

        // MCOPY
        memory.copy(dst, src, len);

        // 验证源数据未变
        prop_assert_eq!(memory.slice(src, len), original_src.as_slice());

        // 验证目标数据正确
        prop_assert_eq!(memory.slice(dst, len), src_data.as_slice());
    }
}

// Gas 计算的属性测试
proptest! {
    /// Gas 计算总是非负
    #[test]
    fn gas_always_non_negative(
        base_fee in 0u64..1000000u64,
        gas_used in 0u64..30000000u64
    ) {
        let gas_cost = calculate_gas_cost(base_fee, gas_used);
        prop_assert!(gas_cost >= 0);
    }

    /// Memory expansion gas 是单调递增的
    #[test]
    fn memory_gas_monotonic(size1: u32, size2: u32) {
        let gas1 = memory_gas_cost(size1 as u64);
        let gas2 = memory_gas_cost(size2 as u64);

        if size1 <= size2 {
            prop_assert!(gas1 <= gas2);
        }
    }
}

fn memory_gas_cost(size: u64) -> u64 {
    let words = (size + 31) / 32;
    (words * words) / 512 + 3 * words
}

fn calculate_gas_cost(base_fee: u64, gas_used: u64) -> u128 {
    base_fee as u128 * gas_used as u128
}
```

### Q.4 符号执行验证

```rust
use symbolic_executor::{SymbolicExecutor, SymbolicValue, Constraint};

/// 使用符号执行验证合约属性
pub struct ContractVerifier {
    executor: SymbolicExecutor,
    constraints: Vec<Constraint>,
}

impl ContractVerifier {
    pub fn new() -> Self {
        Self {
            executor: SymbolicExecutor::new(),
            constraints: vec![],
        }
    }

    /// 验证不变量：balance 始终 >= 0
    pub fn verify_balance_invariant(&mut self, bytecode: &[u8]) -> VerificationResult {
        // 创建符号状态
        let mut state = self.executor.create_symbolic_state();

        // 符号化输入
        let caller = state.new_symbolic_address("caller");
        let value = state.new_symbolic_u256("value");
        let calldata = state.new_symbolic_bytes("calldata");

        // 添加前置条件
        state.add_constraint(Constraint::Ge(value.clone(), SymbolicValue::Concrete(U256::ZERO)));

        // 执行所有路径
        let paths = self.executor.explore_paths(bytecode, &state);

        // 检查后置条件
        for path in paths {
            // 检查是否存在负余额
            let balance = path.get_storage_value(Address::ZERO, U256::ZERO);

            if let Some(counterexample) = self.executor.solve_for(
                Constraint::Lt(balance, SymbolicValue::Concrete(U256::ZERO))
            ) {
                return VerificationResult::Failed {
                    property: "balance >= 0".into(),
                    counterexample,
                    path: path.trace(),
                };
            }
        }

        VerificationResult::Verified
    }

    /// 验证重入安全
    pub fn verify_no_reentrancy(&mut self, bytecode: &[u8]) -> VerificationResult {
        let mut state = self.executor.create_symbolic_state();

        // 追踪外部调用和状态修改顺序
        let mut call_before_state_update = false;

        for path in self.executor.explore_paths(bytecode, &state) {
            let trace = path.trace();
            let mut last_call_pc: Option<usize> = None;

            for (pc, op) in trace.iter() {
                match op {
                    // 外部调用
                    Op::CALL | Op::DELEGATECALL | Op::STATICCALL => {
                        last_call_pc = Some(*pc);
                    }
                    // 状态修改
                    Op::SSTORE => {
                        if let Some(call_pc) = last_call_pc {
                            // CALL 之后有 SSTORE = 潜在重入风险
                            return VerificationResult::Warning {
                                property: "check-effects-interactions".into(),
                                location: *pc,
                                message: format!(
                                    "SSTORE at {} after CALL at {}",
                                    pc, call_pc
                                ),
                            };
                        }
                    }
                    _ => {}
                }
            }
        }

        VerificationResult::Verified
    }

    /// 验证整数溢出
    pub fn verify_no_overflow(&mut self, bytecode: &[u8]) -> VerificationResult {
        let state = self.executor.create_symbolic_state();

        for path in self.executor.explore_paths(bytecode, &state) {
            for constraint in path.overflow_constraints() {
                if let Some(counterexample) = self.executor.solve_for(constraint) {
                    return VerificationResult::Failed {
                        property: "no integer overflow".into(),
                        counterexample,
                        path: path.trace(),
                    };
                }
            }
        }

        VerificationResult::Verified
    }
}

#[derive(Debug)]
pub enum VerificationResult {
    Verified,
    Failed {
        property: String,
        counterexample: Counterexample,
        path: Vec<(usize, Op)>,
    },
    Warning {
        property: String,
        location: usize,
        message: String,
    },
    Timeout,
}

#[derive(Debug)]
pub struct Counterexample {
    pub inputs: HashMap<String, Vec<u8>>,
    pub storage: HashMap<U256, U256>,
}
```

### Q.5 Act 规范语言

```act
// Act: 形式化规范语言示例

// ERC20 transfer 函数规范
behaviour transfer of ERC20
interface transfer(address to, uint256 value)

iff
    // 前置条件
    CALLER =/= to
    balanceOf[CALLER] >= value
    balanceOf[to] + value <= MAX_UINT256

storage
    // 存储变化
    balanceOf[CALLER] => balanceOf[CALLER] - value
    balanceOf[to] => balanceOf[to] + value

returns true

// Uniswap V2 swap 规范
behaviour swap of UniswapV2Pair
interface swap(uint amount0Out, uint amount1Out, address to, bytes data)

iff
    // 至少输出一种代币
    amount0Out > 0 or amount1Out > 0
    // 输出不超过储备
    amount0Out <= reserve0
    amount1Out <= reserve1
    // 不能发送给 pair 自己
    to =/= self
    // k 值不变量
    balance0After * balance1After >= reserve0 * reserve1

storage
    reserve0 => balance0After
    reserve1 => balance1After

where
    balance0After = reserve0 - amount0Out + amount0In
    balance1After = reserve1 - amount1Out + amount1In

// Vault withdraw 规范
behaviour withdraw of Vault
interface withdraw(uint256 shares)

iff
    // 有足够的份额
    balanceOf[CALLER] >= shares
    // 合约有足够资产
    totalAssets() >= shares * pricePerShare / 1e18

storage
    balanceOf[CALLER] => balanceOf[CALLER] - shares
    totalSupply => totalSupply - shares

returns assets

where
    assets = shares * pricePerShare / 1e18
```

---

## 附录 R：状态同步优化

### R.1 Snap Sync 实现

```rust
// Reth 中的 Snap Sync 实现
use reth_primitives::{Address, B256, U256};
use reth_trie::{TrieNode, Nibbles};

/// Snap Sync 状态管理器
pub struct SnapSyncer<DB> {
    db: DB,
    /// 待下载的账户范围
    account_ranges: Vec<AccountRange>,
    /// 待下载的存储范围
    storage_ranges: Vec<StorageRange>,
    /// 正在进行的请求
    pending_requests: HashMap<RequestId, PendingRequest>,
    /// 已验证的状态根
    target_root: B256,
}

#[derive(Clone)]
pub struct AccountRange {
    pub start: B256,
    pub end: B256,
    pub origin: B256,
    pub limit: B256,
}

#[derive(Clone)]
pub struct StorageRange {
    pub account: Address,
    pub account_hash: B256,
    pub start: B256,
    pub end: B256,
}

impl<DB: StateDatabase> SnapSyncer<DB> {
    /// 处理账户范围响应
    pub fn handle_account_range(
        &mut self,
        response: AccountRangeResponse,
    ) -> Result<Vec<StorageRange>, SyncError> {
        // 验证证明
        self.verify_account_proof(&response)?;

        let mut storage_requests = Vec::new();

        for (hash, account) in &response.accounts {
            // 存储账户数据
            self.db.insert_account(*hash, account.clone())?;

            // 如果有存储，添加存储下载请求
            if account.storage_root != EMPTY_ROOT {
                storage_requests.push(StorageRange {
                    account: account.address,
                    account_hash: *hash,
                    start: B256::ZERO,
                    end: B256::MAX,
                });
            }
        }

        // 更新下载进度
        if let Some(last) = response.accounts.last() {
            self.update_account_progress(last.0)?;
        }

        Ok(storage_requests)
    }

    /// 验证账户 Merkle 证明
    fn verify_account_proof(&self, response: &AccountRangeResponse) -> Result<(), SyncError> {
        // 构建证明验证器
        let mut verifier = ProofVerifier::new(self.target_root);

        for proof_node in &response.proof {
            verifier.add_node(proof_node)?;
        }

        // 验证每个账户
        for (hash, account) in &response.accounts {
            let encoded = account.rlp_encode();

            if !verifier.verify_inclusion(*hash, &encoded)? {
                return Err(SyncError::InvalidProof);
            }
        }

        // 验证边界证明（确保没有遗漏账户）
        verifier.verify_range_proof(
            &response.accounts.first().map(|(h, _)| *h),
            &response.accounts.last().map(|(h, _)| *h),
        )?;

        Ok(())
    }

    /// 处理存储范围响应
    pub fn handle_storage_range(
        &mut self,
        response: StorageRangeResponse,
    ) -> Result<(), SyncError> {
        // 获取账户信息以验证存储根
        let account = self.db.get_account(response.account_hash)?
            .ok_or(SyncError::AccountNotFound)?;

        // 验证存储证明
        self.verify_storage_proof(&response, account.storage_root)?;

        // 批量写入存储
        let mut batch = self.db.begin_batch();

        for (slot_hash, value) in &response.slots {
            batch.insert_storage(
                response.account_hash,
                *slot_hash,
                *value,
            )?;
        }

        batch.commit()?;

        // 如果还有更多存储槽，继续请求
        if response.incomplete {
            let last_hash = response.slots.last()
                .map(|(h, _)| *h)
                .unwrap_or(B256::ZERO);

            self.storage_ranges.push(StorageRange {
                account: response.account,
                account_hash: response.account_hash,
                start: last_hash,
                end: B256::MAX,
            });
        }

        Ok(())
    }
}

/// Merkle 证明验证器
pub struct ProofVerifier {
    root: B256,
    nodes: HashMap<B256, TrieNode>,
}

impl ProofVerifier {
    pub fn new(root: B256) -> Self {
        Self {
            root,
            nodes: HashMap::new(),
        }
    }

    pub fn add_node(&mut self, encoded: &[u8]) -> Result<(), SyncError> {
        let hash = keccak256(encoded);
        let node = TrieNode::decode(encoded)?;
        self.nodes.insert(hash, node);
        Ok(())
    }

    pub fn verify_inclusion(&self, key: B256, value: &[u8]) -> Result<bool, SyncError> {
        let path = Nibbles::from_bytes(&key);
        let mut current = self.root;
        let mut path_offset = 0;

        loop {
            let node = self.nodes.get(&current)
                .ok_or(SyncError::MissingNode)?;

            match node {
                TrieNode::Leaf { key: leaf_key, value: leaf_value } => {
                    // 检查剩余路径是否匹配
                    let remaining = path.slice(path_offset..);
                    return Ok(
                        remaining == *leaf_key &&
                        leaf_value == value
                    );
                }

                TrieNode::Extension { key: ext_key, child } => {
                    // 检查前缀是否匹配
                    let remaining = path.slice(path_offset..);
                    if !remaining.starts_with(ext_key) {
                        return Ok(false);
                    }
                    path_offset += ext_key.len();
                    current = *child;
                }

                TrieNode::Branch { children, value: branch_value } => {
                    if path_offset == path.len() {
                        // 到达路径末尾，检查分支值
                        return Ok(
                            branch_value.as_ref()
                                .map(|v| v.as_slice() == value)
                                .unwrap_or(false)
                        );
                    }

                    // 继续沿子节点前进
                    let nibble = path.at(path_offset);
                    match &children[nibble as usize] {
                        Some(child) => {
                            current = *child;
                            path_offset += 1;
                        }
                        None => return Ok(false),
                    }
                }
            }
        }
    }
}
```

### R.2 Healing 过程

```rust
// Trie Healing: 修复同步过程中的不一致
use std::collections::BTreeSet;

pub struct TrieHealer<DB> {
    db: DB,
    /// 需要修复的节点
    missing_nodes: BTreeSet<B256>,
    /// 正在请求的节点
    pending_nodes: HashSet<B256>,
    /// 目标状态根
    target_root: B256,
}

impl<DB: StateDatabase> TrieHealer<DB> {
    /// 开始 healing 过程
    pub async fn heal(&mut self) -> Result<(), HealError> {
        // 从根节点开始验证
        self.schedule_node(self.target_root);

        while !self.missing_nodes.is_empty() || !self.pending_nodes.is_empty() {
            // 发送节点请求
            let requests = self.get_next_requests(128);
            if !requests.is_empty() {
                self.send_requests(requests).await?;
            }

            // 处理响应
            self.process_responses().await?;
        }

        // 验证最终状态
        self.verify_final_state()?;

        Ok(())
    }

    /// 处理收到的 trie 节点
    pub fn handle_trie_nodes(&mut self, nodes: Vec<TrieNodeData>) -> Result<(), HealError> {
        for node_data in nodes {
            let hash = keccak256(&node_data.encoded);

            // 验证是我们请求的节点
            if !self.pending_nodes.remove(&hash) {
                continue;
            }

            // 解码并存储节点
            let node = TrieNode::decode(&node_data.encoded)?;
            self.db.insert_trie_node(hash, node_data.encoded.clone())?;

            // 递归检查子节点
            self.schedule_children(&node)?;
        }

        Ok(())
    }

    /// 调度子节点检查
    fn schedule_children(&mut self, node: &TrieNode) -> Result<(), HealError> {
        match node {
            TrieNode::Extension { child, .. } => {
                self.schedule_node(*child);
            }
            TrieNode::Branch { children, .. } => {
                for child in children.iter().flatten() {
                    self.schedule_node(*child);
                }
            }
            TrieNode::Leaf { .. } => {
                // 叶子节点没有子节点
            }
        }
        Ok(())
    }

    /// 调度节点检查
    fn schedule_node(&mut self, hash: B256) {
        // 检查是否已存在
        if self.db.has_trie_node(hash) {
            // 节点存在，验证并递归检查子节点
            if let Ok(Some(encoded)) = self.db.get_trie_node(hash) {
                if let Ok(node) = TrieNode::decode(&encoded) {
                    let _ = self.schedule_children(&node);
                }
            }
            return;
        }

        // 节点不存在，添加到缺失列表
        if !self.pending_nodes.contains(&hash) {
            self.missing_nodes.insert(hash);
        }
    }

    /// 验证账户存储一致性
    pub fn verify_account_storage(&self, account_hash: B256) -> Result<bool, HealError> {
        let account = self.db.get_account(account_hash)?
            .ok_or(HealError::AccountNotFound)?;

        if account.storage_root == EMPTY_ROOT {
            return Ok(true);
        }

        // 重建存储 trie 并验证根
        let computed_root = self.compute_storage_root(account_hash)?;

        Ok(computed_root == account.storage_root)
    }

    /// 计算存储根
    fn compute_storage_root(&self, account_hash: B256) -> Result<B256, HealError> {
        let mut trie_builder = TrieBuilder::new();

        // 获取所有存储槽
        let storage = self.db.get_account_storage(account_hash)?;

        for (slot_hash, value) in storage {
            if value != U256::ZERO {
                let encoded = alloy_rlp::encode(value);
                trie_builder.insert(slot_hash, encoded);
            }
        }

        Ok(trie_builder.root())
    }
}

/// Trie 重建器
pub struct TrieBuilder {
    nodes: HashMap<Nibbles, TrieNode>,
}

impl TrieBuilder {
    pub fn new() -> Self {
        Self {
            nodes: HashMap::new(),
        }
    }

    pub fn insert(&mut self, key: B256, value: Vec<u8>) {
        let path = Nibbles::from_bytes(&key);
        self.insert_at_path(path, value);
    }

    fn insert_at_path(&mut self, path: Nibbles, value: Vec<u8>) {
        // 简化的 Patricia Trie 插入
        // 实际实现需要处理路径压缩
        self.nodes.insert(path, TrieNode::Leaf {
            key: path,
            value: value.into(),
        });
    }

    pub fn root(&self) -> B256 {
        if self.nodes.is_empty() {
            return EMPTY_ROOT;
        }

        // 从叶子节点构建 trie 并计算根
        self.compute_root_recursive(&Nibbles::default())
    }

    fn compute_root_recursive(&self, prefix: &Nibbles) -> B256 {
        // 收集所有以 prefix 开头的节点
        // 递归构建子树并哈希
        // 简化实现...
        B256::ZERO
    }
}

#[derive(Debug)]
pub enum HealError {
    AccountNotFound,
    DatabaseError,
    InvalidNode,
    InconsistentState,
}
```

### R.3 状态同步架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Snap Sync 架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Phase 1: 账户同步                                              │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  Local Node                          Remote Peer          │  │
│   │  ┌─────────────┐                    ┌─────────────┐      │  │
│   │  │ Request     │ GetAccountRange    │ Serve       │      │  │
│   │  │ Scheduler   │───────────────────▶│ Handler     │      │  │
│   │  └─────────────┘                    └─────────────┘      │  │
│   │        │         AccountRange              │             │  │
│   │        │◀─────────────────────────────────┘             │  │
│   │        │  [accounts + merkle proof]                      │  │
│   │        ▼                                                 │  │
│   │  ┌─────────────┐                                        │  │
│   │  │ Proof       │ Verify against                         │  │
│   │  │ Verifier    │ target state root                      │  │
│   │  └─────────────┘                                        │  │
│   │        │                                                 │  │
│   │        ▼                                                 │  │
│   │  ┌─────────────┐                                        │  │
│   │  │ State DB    │ Store verified accounts                │  │
│   │  └─────────────┘                                        │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Phase 2: 存储同步                                              │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  For each account with storage:                          │  │
│   │                                                          │  │
│   │  ┌─────────────┐  GetStorageRanges  ┌─────────────┐     │  │
│   │  │ Storage     │───────────────────▶│ Remote      │     │  │
│   │  │ Scheduler   │                    │ Peer        │     │  │
│   │  └─────────────┘                    └─────────────┘     │  │
│   │        │           StorageRanges           │            │  │
│   │        │◀─────────────────────────────────┘            │  │
│   │        │  [slots + proof for each account]              │  │
│   │        ▼                                                │  │
│   │  ┌─────────────┐                                       │  │
│   │  │ Verify      │ Check storage root                    │  │
│   │  │ Storage     │ matches account                       │  │
│   │  └─────────────┘                                       │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Phase 3: Bytecode 同步                                         │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  ┌─────────────┐  GetByteCodes      ┌─────────────┐     │  │
│   │  │ Code        │───────────────────▶│ Remote      │     │  │
│   │  │ Scheduler   │                    │ Peer        │     │  │
│   │  └─────────────┘                    └─────────────┘     │  │
│   │        │           ByteCodes               │            │  │
│   │        │◀─────────────────────────────────┘            │  │
│   │        │  [code hashes -> bytecode]                     │  │
│   │        ▼                                                │  │
│   │  ┌─────────────┐                                       │  │
│   │  │ Verify      │ keccak256(code) == code_hash          │  │
│   │  │ Code Hash   │                                       │  │
│   │  └─────────────┘                                       │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Phase 4: Trie Healing                                          │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  ┌─────────────┐                                        │  │
│   │  │ Walk Trie   │ Identify missing nodes                 │  │
│   │  │ Locally     │                                        │  │
│   │  └──────┬──────┘                                        │  │
│   │         │                                               │  │
│   │         ▼                                               │  │
│   │  ┌─────────────┐  GetTrieNodes      ┌─────────────┐    │  │
│   │  │ Request     │───────────────────▶│ Remote      │    │  │
│   │  │ Missing     │                    │ Peer        │    │  │
│   │  └─────────────┘                    └─────────────┘    │  │
│   │         │           TrieNodes             │            │  │
│   │         │◀────────────────────────────────┘            │  │
│   │         ▼                                              │  │
│   │  ┌─────────────┐                                       │  │
│   │  │ Fill Gaps   │ Complete trie structure               │  │
│   │  └─────────────┘                                       │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Final: 验证状态根                                              │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  computed_root = rebuild_trie(accounts, storage)        │  │
│   │  assert(computed_root == target_state_root)             │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 S：性能剖析方法

### S.1 Rust 性能分析工具

```rust
// Criterion 基准测试
use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId, Throughput};
use revm::{Evm, db::InMemoryDB, primitives::*};

fn evm_benchmarks(c: &mut Criterion) {
    let mut group = c.benchmark_group("evm_operations");

    // 配置吞吐量测量
    group.throughput(Throughput::Elements(1));

    // ADD 操作基准
    group.bench_function("opcode_add", |b| {
        let mut evm = create_test_evm();
        let bytecode = vec![
            0x60, 0x01,  // PUSH1 1
            0x60, 0x02,  // PUSH1 2
            0x01,        // ADD
            0x00,        // STOP
        ];

        b.iter(|| {
            evm.transact_with_bytecode(&bytecode)
        });
    });

    // SSTORE 操作基准
    group.bench_function("opcode_sstore", |b| {
        let mut evm = create_test_evm();
        let bytecode = vec![
            0x60, 0x01,  // PUSH1 1 (value)
            0x60, 0x00,  // PUSH1 0 (key)
            0x55,        // SSTORE
            0x00,        // STOP
        ];

        b.iter(|| {
            evm.transact_with_bytecode(&bytecode)
        });
    });

    // SLOAD 操作基准（冷/热访问）
    for access_type in ["cold", "warm"] {
        group.bench_with_input(
            BenchmarkId::new("opcode_sload", access_type),
            &access_type,
            |b, &access| {
                let mut evm = create_test_evm();

                // 预热存储槽
                if access == "warm" {
                    evm.preload_storage(Address::ZERO, U256::ZERO);
                }

                let bytecode = vec![
                    0x60, 0x00,  // PUSH1 0 (key)
                    0x54,        // SLOAD
                    0x50,        // POP
                    0x00,        // STOP
                ];

                b.iter(|| {
                    evm.transact_with_bytecode(&bytecode)
                });
            },
        );
    }

    // 不同大小的内存复制基准
    for size in [32, 256, 1024, 8192] {
        group.bench_with_input(
            BenchmarkId::new("memory_copy", size),
            &size,
            |b, &size| {
                let mut evm = create_test_evm();
                let bytecode = generate_mcopy_bytecode(size);

                b.iter(|| {
                    evm.transact_with_bytecode(&bytecode)
                });
            },
        );
    }

    group.finish();
}

fn contract_benchmarks(c: &mut Criterion) {
    let mut group = c.benchmark_group("contract_execution");

    // ERC20 transfer 基准
    group.bench_function("erc20_transfer", |b| {
        let (mut evm, token_address) = setup_erc20();

        let calldata = encode_transfer(
            Address::from_slice(&[0x02; 20]),
            U256::from(1000),
        );

        b.iter(|| {
            evm.transact_call(token_address, calldata.clone())
        });
    });

    // Uniswap swap 基准
    group.bench_function("uniswap_swap", |b| {
        let (mut evm, router_address) = setup_uniswap();

        let calldata = encode_swap_exact_tokens(
            U256::from(1000),
            U256::from(900),
            vec![TOKEN_A, TOKEN_B],
        );

        b.iter(|| {
            evm.transact_call(router_address, calldata.clone())
        });
    });

    group.finish();
}

criterion_group!(benches, evm_benchmarks, contract_benchmarks);
criterion_main!(benches);
```

### S.2 火焰图生成

```rust
// 使用 pprof-rs 生成火焰图
use pprof::ProfilerGuard;
use std::fs::File;

/// 带 profiling 的 EVM 执行
pub fn profile_execution<F, R>(name: &str, f: F) -> R
where
    F: FnOnce() -> R,
{
    // 创建 profiler
    let guard = ProfilerGuard::new(100).unwrap();

    // 执行被测代码
    let result = f();

    // 生成报告
    if let Ok(report) = guard.report().build() {
        // 写入火焰图 SVG
        let file = File::create(format!("{}_flamegraph.svg", name)).unwrap();
        report.flamegraph(file).unwrap();

        // 写入 pprof 格式
        let mut file = File::create(format!("{}.pb", name)).unwrap();
        let profile = report.pprof().unwrap();
        use prost::Message;
        profile.encode(&mut file).unwrap();
    }

    result
}

// 使用示例
fn main() {
    let mut evm = create_test_evm();
    let complex_bytecode = load_complex_contract();

    profile_execution("complex_contract", || {
        for _ in 0..1000 {
            evm.transact_with_bytecode(&complex_bytecode);
        }
    });
}
```

### S.3 perf 集成

```bash
#!/bin/bash
# 使用 perf 分析 revm 性能

# 编译带调试信息的 release 版本
RUSTFLAGS="-C debuginfo=2" cargo build --release

# 记录 perf 数据
perf record -F 99 -g --call-graph dwarf \
    ./target/release/revm-bench

# 生成火焰图
perf script | \
    stackcollapse-perf.pl | \
    flamegraph.pl > perf_flamegraph.svg

# 查看热点函数
perf report --sort=dso,symbol --stdio

# 分析缓存性能
perf stat -e cache-misses,cache-references,L1-dcache-loads,L1-dcache-load-misses \
    ./target/release/revm-bench

# 分析分支预测
perf stat -e branch-misses,branch-instructions \
    ./target/release/revm-bench
```

### S.4 内存分析

```rust
// 使用 DHAT 进行堆分析
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    // 启动 DHAT profiler
    let _profiler = dhat::Profiler::new_heap();

    // 执行 EVM 操作
    run_evm_workload();

    // profiler drop 时自动生成报告
}

// 使用 tracking_allocator 追踪分配
use tracking_allocator::{AllocationTracker, AllocationRegistry};

#[global_allocator]
static GLOBAL: AllocationTracker<std::alloc::System> =
    AllocationTracker::new(std::alloc::System, AllocationRegistry::default());

fn analyze_allocations() {
    let registry = GLOBAL.registry();

    // 执行操作
    let mut evm = create_test_evm();
    registry.start_tracking();

    evm.transact_with_bytecode(&bytecode);

    let stats = registry.stop_tracking();

    println!("Allocations: {}", stats.allocation_count);
    println!("Total bytes: {}", stats.bytes_allocated);
    println!("Peak bytes: {}", stats.peak_bytes);

    // 按调用栈分组
    for (stack, count) in stats.by_callstack() {
        println!("{}: {} allocations", stack, count);
    }
}
```

### S.5 自定义指标收集

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::Instant;

/// EVM 执行指标收集器
pub struct EvmMetrics {
    // 操作码统计
    opcode_counts: [AtomicU64; 256],
    opcode_gas: [AtomicU64; 256],
    opcode_time_ns: [AtomicU64; 256],

    // 调用统计
    call_count: AtomicU64,
    call_depth_max: AtomicU64,

    // 存储统计
    sload_count: AtomicU64,
    sstore_count: AtomicU64,
    sload_cold: AtomicU64,
    sload_warm: AtomicU64,

    // 内存统计
    memory_expansion_count: AtomicU64,
    memory_peak_bytes: AtomicU64,

    // 执行时间
    total_execution_ns: AtomicU64,
}

impl EvmMetrics {
    pub fn new() -> Self {
        Self {
            opcode_counts: std::array::from_fn(|_| AtomicU64::new(0)),
            opcode_gas: std::array::from_fn(|_| AtomicU64::new(0)),
            opcode_time_ns: std::array::from_fn(|_| AtomicU64::new(0)),
            call_count: AtomicU64::new(0),
            call_depth_max: AtomicU64::new(0),
            sload_count: AtomicU64::new(0),
            sstore_count: AtomicU64::new(0),
            sload_cold: AtomicU64::new(0),
            sload_warm: AtomicU64::new(0),
            memory_expansion_count: AtomicU64::new(0),
            memory_peak_bytes: AtomicU64::new(0),
            total_execution_ns: AtomicU64::new(0),
        }
    }

    #[inline]
    pub fn record_opcode(&self, opcode: u8, gas: u64, duration_ns: u64) {
        self.opcode_counts[opcode as usize].fetch_add(1, Ordering::Relaxed);
        self.opcode_gas[opcode as usize].fetch_add(gas, Ordering::Relaxed);
        self.opcode_time_ns[opcode as usize].fetch_add(duration_ns, Ordering::Relaxed);
    }

    #[inline]
    pub fn record_sload(&self, is_cold: bool) {
        self.sload_count.fetch_add(1, Ordering::Relaxed);
        if is_cold {
            self.sload_cold.fetch_add(1, Ordering::Relaxed);
        } else {
            self.sload_warm.fetch_add(1, Ordering::Relaxed);
        }
    }

    pub fn report(&self) -> MetricsReport {
        let mut opcode_stats = Vec::new();

        for i in 0..256 {
            let count = self.opcode_counts[i].load(Ordering::Relaxed);
            if count > 0 {
                opcode_stats.push(OpcodeStats {
                    opcode: i as u8,
                    count,
                    total_gas: self.opcode_gas[i].load(Ordering::Relaxed),
                    total_time_ns: self.opcode_time_ns[i].load(Ordering::Relaxed),
                });
            }
        }

        // 按执行时间排序
        opcode_stats.sort_by(|a, b| b.total_time_ns.cmp(&a.total_time_ns));

        MetricsReport {
            opcode_stats,
            call_count: self.call_count.load(Ordering::Relaxed),
            call_depth_max: self.call_depth_max.load(Ordering::Relaxed),
            sload_total: self.sload_count.load(Ordering::Relaxed),
            sload_cold: self.sload_cold.load(Ordering::Relaxed),
            sload_warm: self.sload_warm.load(Ordering::Relaxed),
            sstore_count: self.sstore_count.load(Ordering::Relaxed),
            memory_expansions: self.memory_expansion_count.load(Ordering::Relaxed),
            memory_peak: self.memory_peak_bytes.load(Ordering::Relaxed),
            total_time_ns: self.total_execution_ns.load(Ordering::Relaxed),
        }
    }
}

#[derive(Debug)]
pub struct MetricsReport {
    pub opcode_stats: Vec<OpcodeStats>,
    pub call_count: u64,
    pub call_depth_max: u64,
    pub sload_total: u64,
    pub sload_cold: u64,
    pub sload_warm: u64,
    pub sstore_count: u64,
    pub memory_expansions: u64,
    pub memory_peak: u64,
    pub total_time_ns: u64,
}

impl std::fmt::Display for MetricsReport {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        writeln!(f, "=== EVM Execution Metrics ===")?;
        writeln!(f, "\nTop opcodes by time:")?;

        for stat in self.opcode_stats.iter().take(10) {
            let name = opcode_name(stat.opcode);
            let avg_ns = stat.total_time_ns / stat.count.max(1);
            writeln!(f, "  {:12} count={:8} gas={:10} time={:8}ns (avg {:4}ns)",
                name, stat.count, stat.total_gas, stat.total_time_ns, avg_ns)?;
        }

        writeln!(f, "\nStorage access:")?;
        writeln!(f, "  SLOAD:  total={} cold={} warm={}",
            self.sload_total, self.sload_cold, self.sload_warm)?;
        writeln!(f, "  SSTORE: {}", self.sstore_count)?;

        writeln!(f, "\nCall statistics:")?;
        writeln!(f, "  Count: {}", self.call_count)?;
        writeln!(f, "  Max depth: {}", self.call_depth_max)?;

        writeln!(f, "\nMemory:")?;
        writeln!(f, "  Expansions: {}", self.memory_expansions)?;
        writeln!(f, "  Peak: {} bytes", self.memory_peak)?;

        writeln!(f, "\nTotal execution time: {}ns ({:.3}ms)",
            self.total_time_ns,
            self.total_time_ns as f64 / 1_000_000.0)?;

        Ok(())
    }
}

#[derive(Debug)]
pub struct OpcodeStats {
    pub opcode: u8,
    pub count: u64,
    pub total_gas: u64,
    pub total_time_ns: u64,
}

fn opcode_name(opcode: u8) -> &'static str {
    match opcode {
        0x00 => "STOP",
        0x01 => "ADD",
        0x02 => "MUL",
        0x03 => "SUB",
        0x04 => "DIV",
        0x10 => "LT",
        0x11 => "GT",
        0x14 => "EQ",
        0x15 => "ISZERO",
        0x16 => "AND",
        0x17 => "OR",
        0x18 => "XOR",
        0x19 => "NOT",
        0x20 => "KECCAK256",
        0x30 => "ADDRESS",
        0x31 => "BALANCE",
        0x32 => "ORIGIN",
        0x33 => "CALLER",
        0x34 => "CALLVALUE",
        0x35 => "CALLDATALOAD",
        0x36 => "CALLDATASIZE",
        0x37 => "CALLDATACOPY",
        0x38 => "CODESIZE",
        0x39 => "CODECOPY",
        0x51 => "MLOAD",
        0x52 => "MSTORE",
        0x53 => "MSTORE8",
        0x54 => "SLOAD",
        0x55 => "SSTORE",
        0x56 => "JUMP",
        0x57 => "JUMPI",
        0x5B => "JUMPDEST",
        0x5F => "PUSH0",
        0x60..=0x7F => "PUSH",
        0x80..=0x8F => "DUP",
        0x90..=0x9F => "SWAP",
        0xA0..=0xA4 => "LOG",
        0xF0 => "CREATE",
        0xF1 => "CALL",
        0xF3 => "RETURN",
        0xF4 => "DELEGATECALL",
        0xF5 => "CREATE2",
        0xFA => "STATICCALL",
        0xFD => "REVERT",
        0xFE => "INVALID",
        0xFF => "SELFDESTRUCT",
        _ => "UNKNOWN",
    }
}
```

---

## 附录 T：历史状态查询

### T.1 Archive Node 架构

```rust
// Archive Node: 完整历史状态存储
use reth_db::{Database, TableType};
use reth_primitives::{BlockNumber, B256, U256};

/// Archive 状态提供者
pub struct ArchiveStateProvider<DB> {
    db: DB,
    /// 区块号到状态根的映射
    block_roots: BTreeMap<BlockNumber, B256>,
}

impl<DB: Database> ArchiveStateProvider<DB> {
    /// 获取指定区块的状态
    pub fn state_at(&self, block: BlockNumber) -> Result<HistoricalState<DB>, StateError> {
        let state_root = self.block_roots.get(&block)
            .ok_or(StateError::BlockNotFound(block))?;

        Ok(HistoricalState {
            db: self.db.clone(),
            block_number: block,
            state_root: *state_root,
        })
    }

    /// 获取账户在指定区块的状态
    pub fn account_at(
        &self,
        address: Address,
        block: BlockNumber,
    ) -> Result<Option<AccountInfo>, StateError> {
        let state = self.state_at(block)?;
        state.get_account(address)
    }

    /// 获取存储值在指定区块的状态
    pub fn storage_at(
        &self,
        address: Address,
        slot: U256,
        block: BlockNumber,
    ) -> Result<U256, StateError> {
        let state = self.state_at(block)?;
        state.get_storage(address, slot)
    }
}

/// 历史状态视图
pub struct HistoricalState<DB> {
    db: DB,
    block_number: BlockNumber,
    state_root: B256,
}

impl<DB: Database> HistoricalState<DB> {
    /// 获取账户信息
    pub fn get_account(&self, address: Address) -> Result<Option<AccountInfo>, StateError> {
        // 使用历史 trie 查询
        let trie = HistoricalTrie::new(&self.db, self.state_root)?;
        trie.get_account(address)
    }

    /// 获取存储值
    pub fn get_storage(&self, address: Address, slot: U256) -> Result<U256, StateError> {
        let account = self.get_account(address)?
            .ok_or(StateError::AccountNotFound)?;

        if account.storage_root == EMPTY_ROOT {
            return Ok(U256::ZERO);
        }

        let storage_trie = HistoricalTrie::new(&self.db, account.storage_root)?;
        storage_trie.get_storage(slot)
    }

    /// 获取合约代码
    pub fn get_code(&self, address: Address) -> Result<Option<Bytes>, StateError> {
        let account = self.get_account(address)?
            .ok_or(StateError::AccountNotFound)?;

        if account.code_hash == KECCAK_EMPTY {
            return Ok(None);
        }

        self.db.get_code(account.code_hash)
            .map_err(|_| StateError::DatabaseError)
    }
}

/// 历史 Trie 访问
pub struct HistoricalTrie<'a, DB> {
    db: &'a DB,
    root: B256,
}

impl<'a, DB: Database> HistoricalTrie<'a, DB> {
    pub fn new(db: &'a DB, root: B256) -> Result<Self, StateError> {
        Ok(Self { db, root })
    }

    /// 遍历指定根的 trie 获取账户
    pub fn get_account(&self, address: Address) -> Result<Option<AccountInfo>, StateError> {
        let key = keccak256(address);
        let path = Nibbles::from_bytes(&key);

        self.get_value_at_path(self.root, &path)
            .map(|opt| opt.map(|encoded| {
                AccountInfo::decode(&encoded).unwrap()
            }))
    }

    /// 获取存储值
    pub fn get_storage(&self, slot: U256) -> Result<U256, StateError> {
        let key = keccak256(slot.to_be_bytes::<32>());
        let path = Nibbles::from_bytes(&key);

        self.get_value_at_path(self.root, &path)
            .map(|opt| {
                opt.map(|encoded| {
                    U256::from_be_slice(&alloy_rlp::decode::<Vec<u8>>(&encoded).unwrap())
                }).unwrap_or(U256::ZERO)
            })
    }

    fn get_value_at_path(
        &self,
        node_hash: B256,
        path: &Nibbles,
    ) -> Result<Option<Vec<u8>>, StateError> {
        // 从数据库加载节点
        let node_data = self.db.get_trie_node(node_hash)
            .map_err(|_| StateError::DatabaseError)?
            .ok_or(StateError::MissingTrieNode(node_hash))?;

        let node = TrieNode::decode(&node_data)
            .map_err(|_| StateError::InvalidTrieNode)?;

        self.traverse_node(&node, path, 0)
    }

    fn traverse_node(
        &self,
        node: &TrieNode,
        path: &Nibbles,
        depth: usize,
    ) -> Result<Option<Vec<u8>>, StateError> {
        match node {
            TrieNode::Leaf { key, value } => {
                let remaining = path.slice(depth..);
                if remaining == *key {
                    Ok(Some(value.to_vec()))
                } else {
                    Ok(None)
                }
            }
            TrieNode::Extension { key, child } => {
                let remaining = path.slice(depth..);
                if remaining.starts_with(key) {
                    self.get_value_at_path(*child, &path.slice(depth + key.len()..))
                } else {
                    Ok(None)
                }
            }
            TrieNode::Branch { children, value } => {
                if depth == path.len() {
                    Ok(value.clone())
                } else {
                    let nibble = path.at(depth);
                    if let Some(child) = &children[nibble as usize] {
                        self.get_value_at_path(*child, path)
                    } else {
                        Ok(None)
                    }
                }
            }
        }
    }
}
```

### T.2 debug_traceTransaction 实现

```rust
// 使用 revm 实现交易追踪
use revm::{
    inspectors::TracerEip3155,
    primitives::*,
    Database, Evm,
};

/// 交易追踪器
pub struct TransactionTracer<DB> {
    archive: ArchiveStateProvider<DB>,
}

impl<DB: Database + Clone> TransactionTracer<DB> {
    /// 追踪历史交易
    pub fn trace_transaction(
        &self,
        tx_hash: B256,
        config: TraceConfig,
    ) -> Result<TraceResult, TraceError> {
        // 1. 查找交易和区块
        let (tx, block) = self.find_transaction(tx_hash)?;

        // 2. 获取父区块状态
        let parent_state = self.archive.state_at(block.number - 1)?;

        // 3. 重放区块中该交易之前的所有交易
        let pre_tx_state = self.replay_to_transaction(&block, &tx, parent_state)?;

        // 4. 使用追踪器执行目标交易
        let trace = self.trace_with_config(tx, pre_tx_state, config)?;

        Ok(trace)
    }

    /// 使用指定配置追踪
    fn trace_with_config(
        &self,
        tx: Transaction,
        state: impl Database,
        config: TraceConfig,
    ) -> Result<TraceResult, TraceError> {
        match config.tracer.as_deref() {
            Some("callTracer") => self.trace_calls(tx, state, config),
            Some("prestateTracer") => self.trace_prestate(tx, state, config),
            Some("4byteTracer") => self.trace_4byte(tx, state),
            _ => self.trace_struct_logs(tx, state, config),
        }
    }

    /// 结构化日志追踪 (默认)
    fn trace_struct_logs(
        &self,
        tx: Transaction,
        state: impl Database,
        config: TraceConfig,
    ) -> Result<TraceResult, TraceError> {
        let mut tracer = TracerEip3155::new(Box::new(std::io::sink()));

        if config.enable_memory {
            tracer = tracer.with_memory();
        }
        if config.enable_return_data {
            tracer = tracer.with_return_data();
        }

        let mut evm = Evm::builder()
            .with_db(state)
            .with_external_context(&mut tracer)
            .modify_tx_env(|tx_env| {
                populate_tx_env(tx_env, &tx);
            })
            .build();

        let result = evm.transact()?;

        Ok(TraceResult::StructLogs {
            gas: result.result.gas_used(),
            return_value: result.result.output().cloned().unwrap_or_default(),
            struct_logs: tracer.into_logs(),
        })
    }

    /// Call 追踪
    fn trace_calls(
        &self,
        tx: Transaction,
        state: impl Database,
        config: TraceConfig,
    ) -> Result<TraceResult, TraceError> {
        let mut call_tracer = CallTracer::new(config.only_top_call);

        let mut evm = Evm::builder()
            .with_db(state)
            .with_external_context(&mut call_tracer)
            .modify_tx_env(|tx_env| {
                populate_tx_env(tx_env, &tx);
            })
            .append_handler_register(call_tracer_register)
            .build();

        evm.transact()?;

        Ok(TraceResult::CallTrace(call_tracer.into_result()))
    }

    /// Prestate 追踪
    fn trace_prestate(
        &self,
        tx: Transaction,
        state: impl Database,
        config: TraceConfig,
    ) -> Result<TraceResult, TraceError> {
        let mut prestate_tracer = PrestateTracer::new(config.diff_mode);

        let mut evm = Evm::builder()
            .with_db(state)
            .with_external_context(&mut prestate_tracer)
            .modify_tx_env(|tx_env| {
                populate_tx_env(tx_env, &tx);
            })
            .append_handler_register(prestate_tracer_register)
            .build();

        evm.transact()?;

        Ok(TraceResult::Prestate(prestate_tracer.into_result()))
    }
}

/// Call 追踪器
pub struct CallTracer {
    only_top_call: bool,
    call_stack: Vec<CallFrame>,
    result: Option<CallFrame>,
}

impl CallTracer {
    pub fn new(only_top_call: bool) -> Self {
        Self {
            only_top_call,
            call_stack: vec![],
            result: None,
        }
    }

    pub fn on_call_enter(
        &mut self,
        call_type: CallType,
        from: Address,
        to: Address,
        input: Bytes,
        gas: u64,
        value: U256,
    ) {
        if self.only_top_call && !self.call_stack.is_empty() {
            return;
        }

        let frame = CallFrame {
            call_type,
            from,
            to,
            input,
            gas,
            value,
            output: Bytes::default(),
            error: None,
            gas_used: 0,
            calls: vec![],
        };

        self.call_stack.push(frame);
    }

    pub fn on_call_exit(&mut self, output: Bytes, gas_used: u64, error: Option<String>) {
        if let Some(mut frame) = self.call_stack.pop() {
            frame.output = output;
            frame.gas_used = gas_used;
            frame.error = error;

            if let Some(parent) = self.call_stack.last_mut() {
                parent.calls.push(frame);
            } else {
                self.result = Some(frame);
            }
        }
    }

    pub fn into_result(self) -> CallFrame {
        self.result.unwrap_or_default()
    }
}

#[derive(Debug, Default, Clone)]
pub struct CallFrame {
    pub call_type: CallType,
    pub from: Address,
    pub to: Address,
    pub input: Bytes,
    pub gas: u64,
    pub value: U256,
    pub output: Bytes,
    pub error: Option<String>,
    pub gas_used: u64,
    pub calls: Vec<CallFrame>,
}

#[derive(Debug, Default, Clone, Copy)]
pub enum CallType {
    #[default]
    Call,
    DelegateCall,
    StaticCall,
    Create,
    Create2,
}

/// Prestate 追踪器
pub struct PrestateTracer {
    diff_mode: bool,
    pre: HashMap<Address, AccountState>,
    post: HashMap<Address, AccountState>,
}

impl PrestateTracer {
    pub fn new(diff_mode: bool) -> Self {
        Self {
            diff_mode,
            pre: HashMap::new(),
            post: HashMap::new(),
        }
    }

    pub fn capture_account_read(&mut self, address: Address, account: &AccountInfo) {
        self.pre.entry(address).or_insert_with(|| AccountState {
            balance: account.balance,
            nonce: account.nonce,
            code: None,
            storage: HashMap::new(),
        });
    }

    pub fn capture_storage_read(&mut self, address: Address, slot: U256, value: U256) {
        if let Some(state) = self.pre.get_mut(&address) {
            state.storage.entry(slot).or_insert(value);
        }
    }

    pub fn capture_account_write(&mut self, address: Address, account: &AccountInfo) {
        if self.diff_mode {
            self.post.insert(address, AccountState {
                balance: account.balance,
                nonce: account.nonce,
                code: None,
                storage: HashMap::new(),
            });
        }
    }

    pub fn into_result(self) -> PrestateResult {
        if self.diff_mode {
            PrestateResult::Diff {
                pre: self.pre,
                post: self.post,
            }
        } else {
            PrestateResult::Default(self.pre)
        }
    }
}

#[derive(Debug)]
pub struct AccountState {
    pub balance: U256,
    pub nonce: u64,
    pub code: Option<Bytes>,
    pub storage: HashMap<U256, U256>,
}

#[derive(Debug)]
pub enum PrestateResult {
    Default(HashMap<Address, AccountState>),
    Diff {
        pre: HashMap<Address, AccountState>,
        post: HashMap<Address, AccountState>,
    },
}

/// 追踪配置
#[derive(Debug, Default)]
pub struct TraceConfig {
    pub tracer: Option<String>,
    pub enable_memory: bool,
    pub enable_return_data: bool,
    pub only_top_call: bool,
    pub diff_mode: bool,
}

/// 追踪结果
#[derive(Debug)]
pub enum TraceResult {
    StructLogs {
        gas: u64,
        return_value: Bytes,
        struct_logs: Vec<StructLog>,
    },
    CallTrace(CallFrame),
    Prestate(PrestateResult),
    FourByte(HashMap<[u8; 4], u64>),
}

#[derive(Debug)]
pub struct StructLog {
    pub pc: u64,
    pub op: u8,
    pub gas: u64,
    pub gas_cost: u64,
    pub depth: u32,
    pub stack: Vec<U256>,
    pub memory: Option<Vec<u8>>,
    pub storage: Option<HashMap<U256, U256>>,
    pub return_data: Option<Vec<u8>>,
    pub error: Option<String>,
}
```

### T.3 历史查询架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Archive Node 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                     RPC Layer                              │ │
│   │  eth_getBalance(addr, block)                               │ │
│   │  eth_getStorageAt(addr, slot, block)                       │ │
│   │  eth_getCode(addr, block)                                  │ │
│   │  debug_traceTransaction(txHash, config)                    │ │
│   │  debug_traceBlockByNumber(block, config)                   │ │
│   │  trace_replayTransaction(txHash, traceTypes)               │ │
│   └───────────────────────────┬───────────────────────────────┘ │
│                               │                                  │
│                               ▼                                  │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                  State Provider                            │ │
│   │  ┌─────────────────┐  ┌─────────────────┐                 │ │
│   │  │ Latest State    │  │ Historical State│                 │ │
│   │  │ (Fast Path)     │  │ (Archive Path)  │                 │ │
│   │  └────────┬────────┘  └────────┬────────┘                 │ │
│   │           │                    │                           │ │
│   │           │    ┌───────────────┘                          │ │
│   │           │    │                                          │ │
│   │           ▼    ▼                                          │ │
│   │  ┌──────────────────────────────────────────────────────┐│ │
│   │  │              Historical Trie                          ││ │
│   │  │                                                        ││ │
│   │  │  Block N-2    Block N-1     Block N (Latest)          ││ │
│   │  │     │             │             │                      ││ │
│   │  │     ▼             ▼             ▼                      ││ │
│   │  │  ┌─────┐      ┌─────┐      ┌─────┐                    ││ │
│   │  │  │Root │──────│Root │──────│Root │                    ││ │
│   │  │  │ N-2 │      │ N-1 │      │  N  │                    ││ │
│   │  │  └──┬──┘      └──┬──┘      └──┬──┘                    ││ │
│   │  │     │            │            │                        ││ │
│   │  │     ▼            ▼            ▼                        ││ │
│   │  │  ┌─────────────────────────────────────────────────┐  ││ │
│   │  │  │         Shared Trie Nodes Database              │  ││ │
│   │  │  │  (Nodes are shared across block states)         │  ││ │
│   │  │  └─────────────────────────────────────────────────┘  ││ │
│   │  └──────────────────────────────────────────────────────┘│ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                  Storage Layer                             │ │
│   │                                                            │ │
│   │  ┌─────────────────────────────────────────────────────┐  │ │
│   │  │              MDBX / RocksDB                          │  │ │
│   │  ├─────────────────────────────────────────────────────┤  │ │
│   │  │  Table: AccountsTrie     │ hash -> encoded_node     │  │ │
│   │  │  Table: StoragesTrie     │ hash -> encoded_node     │  │ │
│   │  │  Table: BlockRoots       │ block_num -> state_root  │  │ │
│   │  │  Table: PlainAccounts    │ address -> account_info  │  │ │
│   │  │  Table: PlainStorage     │ (addr,slot) -> value     │  │ │
│   │  │  Table: Bytecodes        │ code_hash -> bytecode    │  │ │
│   │  │  Table: Transactions     │ tx_hash -> (block, idx)  │  │ │
│   │  └─────────────────────────────────────────────────────┘  │ │
│   │                                                            │ │
│   │  存储优化:                                                  │ │
│   │  - Trie 节点去重: 相同节点只存储一次                         │ │
│   │  - 增量存储: 只存储变化的节点                                │ │
│   │  - 压缩: LZ4/Snappy 压缩历史数据                            │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                 Transaction Replay                         │ │
│   │                                                            │ │
│   │  debug_traceTransaction(0x123..., {tracer: "callTracer"}) │ │
│   │                     │                                      │ │
│   │                     ▼                                      │ │
│   │  1. 查找交易: tx_hash -> (block=1000, index=5)            │ │
│   │                     │                                      │ │
│   │                     ▼                                      │ │
│   │  2. 获取父状态: block_roots[999] -> state_root            │ │
│   │                     │                                      │ │
│   │                     ▼                                      │ │
│   │  3. 重放 tx[0..4]: 在 state_root 上执行前 5 笔交易         │ │
│   │                     │                                      │ │
│   │                     ▼                                      │ │
│   │  4. 追踪 tx[5]: 使用追踪器执行目标交易                      │ │
│   │                     │                                      │ │
│   │                     ▼                                      │ │
│   │  5. 返回追踪结果                                           │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 U：EVM 调试工具

### U.1 revm Inspector 接口

```rust
// revm 提供的调试 Inspector 接口
use revm::{
    interpreter::{
        CallInputs, CallOutcome, CreateInputs, CreateOutcome,
        Interpreter, InterpreterResult,
    },
    primitives::{Address, U256, Bytes, Log},
    Database, EvmContext,
};

/// Inspector trait - 核心调试接口
pub trait Inspector<DB: Database> {
    /// 指令执行前回调
    fn step(&mut self, interp: &mut Interpreter, context: &mut EvmContext<DB>) {}

    /// 指令执行后回调
    fn step_end(&mut self, interp: &mut Interpreter, context: &mut EvmContext<DB>) {}

    /// CALL 指令前
    fn call(&mut self, context: &mut EvmContext<DB>, inputs: &mut CallInputs) -> Option<CallOutcome> {
        None
    }

    /// CALL 指令后
    fn call_end(
        &mut self,
        context: &mut EvmContext<DB>,
        inputs: &CallInputs,
        outcome: CallOutcome,
    ) -> CallOutcome {
        outcome
    }

    /// CREATE 指令前
    fn create(&mut self, context: &mut EvmContext<DB>, inputs: &mut CreateInputs) -> Option<CreateOutcome> {
        None
    }

    /// CREATE 指令后
    fn create_end(
        &mut self,
        context: &mut EvmContext<DB>,
        inputs: &CreateInputs,
        outcome: CreateOutcome,
    ) -> CreateOutcome {
        outcome
    }

    /// LOG 指令
    fn log(&mut self, context: &mut EvmContext<DB>, log: &Log) {}

    /// SELFDESTRUCT 指令
    fn selfdestruct(&mut self, contract: Address, target: Address, value: U256) {}
}

// 结构化日志追踪器
pub struct StructLogTracer {
    logs: Vec<StructLog>,
    gas_inspector: GasInspector,
}

#[derive(Debug, Clone)]
pub struct StructLog {
    pub pc: u64,
    pub op: u8,
    pub gas: u64,
    pub gas_cost: u64,
    pub depth: u32,
    pub error: Option<String>,
    pub stack: Option<Vec<U256>>,
    pub memory: Option<Vec<u8>>,
    pub storage: Option<HashMap<U256, U256>>,
    pub return_data: Option<Vec<u8>>,
}

impl<DB: Database> Inspector<DB> for StructLogTracer {
    fn step(&mut self, interp: &mut Interpreter, _context: &mut EvmContext<DB>) {
        let pc = interp.program_counter();
        let op = interp.current_opcode();

        self.logs.push(StructLog {
            pc: pc as u64,
            op,
            gas: interp.gas().remaining(),
            gas_cost: 0, // 在 step_end 填充
            depth: interp.call_depth() as u32,
            error: None,
            stack: Some(interp.stack().data().to_vec()),
            memory: None, // 按需启用
            storage: None,
            return_data: None,
        });
    }

    fn step_end(&mut self, interp: &mut Interpreter, _context: &mut EvmContext<DB>) {
        if let Some(log) = self.logs.last_mut() {
            log.gas_cost = log.gas.saturating_sub(interp.gas().remaining());

            if interp.instruction_result.is_error() {
                log.error = Some(format!("{:?}", interp.instruction_result));
            }
        }
    }
}
```

### U.2 Call 追踪器

```rust
// Call Tracer - 追踪调用链
use std::collections::VecDeque;

pub struct CallTracer {
    call_stack: VecDeque<CallFrame>,
    config: CallTracerConfig,
}

#[derive(Debug, Clone)]
pub struct CallFrame {
    pub call_type: CallType,
    pub from: Address,
    pub to: Address,
    pub value: U256,
    pub gas: u64,
    pub gas_used: u64,
    pub input: Bytes,
    pub output: Bytes,
    pub error: Option<String>,
    pub revert_reason: Option<String>,
    pub calls: Vec<CallFrame>,
    pub logs: Vec<Log>,
}

#[derive(Debug, Clone, Copy)]
pub enum CallType {
    Call,
    StaticCall,
    DelegateCall,
    CallCode,
    Create,
    Create2,
}

#[derive(Default)]
pub struct CallTracerConfig {
    pub only_top_call: bool,
    pub with_logs: bool,
}

impl<DB: Database> Inspector<DB> for CallTracer {
    fn call(&mut self, _context: &mut EvmContext<DB>, inputs: &mut CallInputs) -> Option<CallOutcome> {
        if self.config.only_top_call && !self.call_stack.is_empty() {
            return None;
        }

        let frame = CallFrame {
            call_type: match inputs.scheme {
                CallScheme::Call => CallType::Call,
                CallScheme::StaticCall => CallType::StaticCall,
                CallScheme::DelegateCall => CallType::DelegateCall,
                CallScheme::CallCode => CallType::CallCode,
            },
            from: inputs.caller,
            to: inputs.target_address,
            value: inputs.call_value(),
            gas: inputs.gas_limit,
            gas_used: 0,
            input: inputs.input.clone(),
            output: Bytes::default(),
            error: None,
            revert_reason: None,
            calls: Vec::new(),
            logs: Vec::new(),
        };

        self.call_stack.push_back(frame);
        None
    }

    fn call_end(
        &mut self,
        _context: &mut EvmContext<DB>,
        _inputs: &CallInputs,
        outcome: CallOutcome,
    ) -> CallOutcome {
        if let Some(mut frame) = self.call_stack.pop_back() {
            frame.gas_used = outcome.gas().spent();
            frame.output = outcome.output().clone();

            if !outcome.result.is_ok() {
                frame.error = Some(format!("{:?}", outcome.result));
                if outcome.result == InstructionResult::Revert {
                    frame.revert_reason = Self::decode_revert_reason(&frame.output);
                }
            }

            // 添加到父调用
            if let Some(parent) = self.call_stack.back_mut() {
                parent.calls.push(frame);
            } else {
                // 顶层调用
                self.call_stack.push_back(frame);
            }
        }

        outcome
    }

    fn log(&mut self, _context: &mut EvmContext<DB>, log: &Log) {
        if self.config.with_logs {
            if let Some(frame) = self.call_stack.back_mut() {
                frame.logs.push(log.clone());
            }
        }
    }
}

impl CallTracer {
    fn decode_revert_reason(output: &Bytes) -> Option<String> {
        if output.len() < 4 {
            return None;
        }

        // Error(string) selector: 0x08c379a0
        if output[..4] == [0x08, 0xc3, 0x79, 0xa0] {
            // Decode ABI string
            let data = &output[4..];
            if data.len() >= 64 {
                let offset = U256::from_be_slice(&data[..32]);
                let length = U256::from_be_slice(&data[32..64]);
                if let (Ok(off), Ok(len)) = (usize::try_from(offset), usize::try_from(length)) {
                    if data.len() >= 64 + len {
                        return String::from_utf8(data[64..64 + len].to_vec()).ok();
                    }
                }
            }
        }

        None
    }

    pub fn into_result(mut self) -> Option<CallFrame> {
        self.call_stack.pop_front()
    }
}
```

### U.3 调试工具架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    revm 调试工具架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                    Inspector Trait                         │ │
│   │  ┌─────────────────────────────────────────────────────┐  │ │
│   │  │  step()        - 每条指令执行前                      │  │ │
│   │  │  step_end()    - 每条指令执行后                      │  │ │
│   │  │  call()        - CALL 系列指令前                     │  │ │
│   │  │  call_end()    - CALL 系列指令后                     │  │ │
│   │  │  create()      - CREATE 指令前                       │  │ │
│   │  │  create_end()  - CREATE 指令后                       │  │ │
│   │  │  log()         - LOG 指令                            │  │ │
│   │  │  selfdestruct()- SELFDESTRUCT 指令                   │  │ │
│   │  └─────────────────────────────────────────────────────┘  │ │
│   └───────────────────────────────────────────────────────────┘ │
│                              │                                   │
│              ┌───────────────┼───────────────┐                  │
│              │               │               │                  │
│              ▼               ▼               ▼                  │
│   ┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐      │
│   │ StructLogTracer │ │ CallTracer  │ │ GasInspector    │      │
│   │                 │ │             │ │                 │      │
│   │ - 逐指令追踪    │ │ - 调用链    │ │ - Gas 使用统计  │      │
│   │ - Stack/Memory  │ │ - 嵌套调用  │ │ - 退款计算      │      │
│   │ - Storage 变化  │ │ - 日志收集  │ │                 │      │
│   └─────────────────┘ └─────────────┘ └─────────────────┘      │
│                                                                  │
│   组合 Inspector                                                 │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │  impl<A, B, DB> Inspector<DB> for (A, B)                  │ │
│   │  where A: Inspector<DB>, B: Inspector<DB>                 │ │
│   │  {                                                        │ │
│   │      fn step(&mut self, interp, ctx) {                    │ │
│   │          self.0.step(interp, ctx);                        │ │
│   │          self.1.step(interp, ctx);                        │ │
│   │      }                                                    │ │
│   │      // ... 其他方法类似                                   │ │
│   │  }                                                        │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│   使用示例                                                       │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │  let tracer = CallTracer::new(CallTracerConfig::default());│ │
│   │                                                           │ │
│   │  let mut evm = Evm::builder()                             │ │
│   │      .with_db(db)                                         │ │
│   │      .with_external_context(tracer)                       │ │
│   │      .append_handler_register(inspector_register)         │ │
│   │      .build();                                            │ │
│   │                                                           │ │
│   │  let result = evm.transact()?;                            │ │
│   │  let trace = evm.context.external.into_result();          │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 V：日志与事件系统

### V.1 revm LOG 实现

```rust
// revm 中 LOG 操作码的实现
// 文件: crates/interpreter/src/instructions/host.rs

pub fn log<H: Host + ?Sized, SPEC: Spec>(interpreter: &mut Interpreter, host: &mut H) {
    // 检查静态调用
    if interpreter.runtime_flag.is_static() {
        interpreter.instruction_result = InstructionResult::StateChangeDuringStaticCall;
        return;
    }

    // 获取操作码确定 topic 数量
    let n = (interpreter.current_opcode() - opcode::LOG0) as usize;

    // 弹出栈参数
    pop!(interpreter, offset, len);

    // 计算并扣除 gas
    let len = as_usize_or_fail!(interpreter, len);
    gas_or_fail!(interpreter, gas::log_cost(n, len as u64));

    // 调整内存大小
    let offset = as_usize_or_fail!(interpreter, offset);
    resize_memory!(interpreter, offset, len);

    // 收集 topics
    let mut topics = Vec::with_capacity(n);
    for _ in 0..n {
        pop!(interpreter, topic);
        topics.push(B256::from(topic));
    }

    // 获取数据
    let data = if len > 0 {
        interpreter.shared_memory.slice(offset, len).to_vec().into()
    } else {
        Bytes::new()
    };

    // 创建日志
    let log = Log {
        address: interpreter.contract.target_address,
        data: LogData::new(topics, data).expect("valid log"),
    };

    // 发送到 host
    host.log(log);
}

/// Gas 计算
pub fn log_cost(n: usize, len: u64) -> Option<u64> {
    // 375 + 375 * n + 8 * len
    let base = 375u64;
    let topic_cost = 375u64.checked_mul(n as u64)?;
    let data_cost = 8u64.checked_mul(len)?;

    base.checked_add(topic_cost)?.checked_add(data_cost)
}
```

### V.2 日志数据结构

```rust
// 日志数据结构
use alloy_primitives::{Address, Bytes, B256};

/// 完整的日志条目
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Log {
    /// 产生日志的合约地址
    pub address: Address,
    /// 日志数据 (topics + data)
    pub data: LogData,
}

/// 日志数据部分
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct LogData {
    /// 索引字段 (最多 4 个)
    topics: Vec<B256>,
    /// 非索引数据
    data: Bytes,
}

impl LogData {
    /// 创建新的日志数据
    pub fn new(topics: Vec<B256>, data: Bytes) -> Option<Self> {
        if topics.len() > 4 {
            return None;
        }
        Some(Self { topics, data })
    }

    /// 获取事件签名 (第一个 topic)
    pub fn topic0(&self) -> Option<&B256> {
        self.topics.first()
    }

    /// 获取所有 topics
    pub fn topics(&self) -> &[B256] {
        &self.topics
    }

    /// 获取数据
    pub fn data(&self) -> &Bytes {
        &self.data
    }
}

/// 日志收集器 (用于执行结果)
#[derive(Clone, Debug, Default)]
pub struct LogCollector {
    logs: Vec<Log>,
}

impl LogCollector {
    pub fn log(&mut self, log: Log) {
        self.logs.push(log);
    }

    pub fn into_logs(self) -> Vec<Log> {
        self.logs
    }

    pub fn logs(&self) -> &[Log] {
        &self.logs
    }
}

// 在 Host 实现中使用
impl<DB: Database> Host for EvmContext<DB> {
    fn log(&mut self, log: Log) {
        self.journaled_state.log(log);
    }
}

impl JournaledState {
    pub fn log(&mut self, log: Log) {
        self.logs.push(log);
    }
}
```

### V.3 Bloom Filter 实现

```rust
// Bloom Filter for 快速日志过滤
use alloy_primitives::{Bloom, B256, keccak256};

/// Bloom Filter 操作
impl Bloom {
    /// 添加输入到 Bloom Filter
    pub fn accrue(&mut self, input: BloomInput<'_>) {
        let hash = match input {
            BloomInput::Raw(data) => keccak256(data),
            BloomInput::Hash(hash) => hash,
        };

        // 取 3 个位位置
        for i in [0, 2, 4] {
            let bit = (hash[i] as usize * 256 + hash[i + 1] as usize) & 0x7FF;
            let byte_index = BLOOM_SIZE_BYTES - 1 - bit / 8;
            let bit_index = bit % 8;
            self.0[byte_index] |= 1 << bit_index;
        }
    }

    /// 从日志创建 Bloom
    pub fn from_logs<'a>(logs: impl IntoIterator<Item = &'a Log>) -> Self {
        let mut bloom = Bloom::default();
        for log in logs {
            bloom.accrue(BloomInput::Raw(log.address.as_slice()));
            for topic in log.data.topics() {
                bloom.accrue(BloomInput::Hash(*topic));
            }
        }
        bloom
    }

    /// 检查是否包含
    pub fn contains_input(&self, input: BloomInput<'_>) -> bool {
        let hash = match input {
            BloomInput::Raw(data) => keccak256(data),
            BloomInput::Hash(hash) => hash,
        };

        for i in [0, 2, 4] {
            let bit = (hash[i] as usize * 256 + hash[i + 1] as usize) & 0x7FF;
            let byte_index = BLOOM_SIZE_BYTES - 1 - bit / 8;
            let bit_index = bit % 8;
            if self.0[byte_index] & (1 << bit_index) == 0 {
                return false;
            }
        }
        true
    }
}

pub enum BloomInput<'a> {
    Raw(&'a [u8]),
    Hash(B256),
}

const BLOOM_SIZE_BYTES: usize = 256;
```

### V.4 日志过滤与索引

```rust
// 日志过滤器实现 (reth 风格)
use std::ops::RangeInclusive;

/// 日志过滤条件
#[derive(Clone, Debug, Default)]
pub struct Filter {
    /// 区块范围
    pub block_range: Option<RangeInclusive<u64>>,
    /// 合约地址过滤
    pub address: FilterSet<Address>,
    /// Topics 过滤 (支持 OR)
    pub topics: [FilterSet<B256>; 4],
}

/// 过滤集合 (支持单个、多个或任意)
#[derive(Clone, Debug, Default)]
pub enum FilterSet<T> {
    #[default]
    Any,
    Single(T),
    Multiple(Vec<T>),
}

impl<T: PartialEq> FilterSet<T> {
    pub fn matches(&self, value: &T) -> bool {
        match self {
            FilterSet::Any => true,
            FilterSet::Single(v) => v == value,
            FilterSet::Multiple(vs) => vs.contains(value),
        }
    }
}

impl Filter {
    /// 检查日志是否匹配过滤条件
    pub fn matches(&self, log: &Log) -> bool {
        // 地址匹配
        if !self.address.matches(&log.address) {
            return false;
        }

        // Topics 匹配
        let log_topics = log.data.topics();
        for (i, filter) in self.topics.iter().enumerate() {
            match filter {
                FilterSet::Any => continue,
                FilterSet::Single(topic) => {
                    if log_topics.get(i) != Some(topic) {
                        return false;
                    }
                }
                FilterSet::Multiple(topics) => {
                    let log_topic = log_topics.get(i);
                    if !topics.iter().any(|t| Some(t) == log_topic) {
                        return false;
                    }
                }
            }
        }

        true
    }

    /// 检查 Bloom Filter 是否可能匹配
    pub fn matches_bloom(&self, bloom: &Bloom) -> bool {
        // 检查地址
        match &self.address {
            FilterSet::Any => {}
            FilterSet::Single(addr) => {
                if !bloom.contains_input(BloomInput::Raw(addr.as_slice())) {
                    return false;
                }
            }
            FilterSet::Multiple(addrs) => {
                if !addrs.iter().any(|a| bloom.contains_input(BloomInput::Raw(a.as_slice()))) {
                    return false;
                }
            }
        }

        // 检查 topics
        for filter in &self.topics {
            match filter {
                FilterSet::Any => {}
                FilterSet::Single(topic) => {
                    if !bloom.contains_input(BloomInput::Hash(*topic)) {
                        return false;
                    }
                }
                FilterSet::Multiple(topics) => {
                    if !topics.iter().any(|t| bloom.contains_input(BloomInput::Hash(*t))) {
                        return false;
                    }
                }
            }
        }

        true
    }
}

/// 日志查询执行器
pub struct LogQueryExecutor<DB> {
    db: DB,
    filter: Filter,
}

impl<DB: LogDatabase> LogQueryExecutor<DB> {
    pub fn execute(&self) -> Result<Vec<Log>, QueryError> {
        let block_range = self.filter.block_range.clone()
            .unwrap_or(0..=self.db.latest_block()?);

        let mut results = Vec::new();

        for block_num in block_range {
            // 1. 快速 Bloom 检查
            let bloom = self.db.block_bloom(block_num)?;
            if !self.filter.matches_bloom(&bloom) {
                continue;
            }

            // 2. 获取区块日志
            let receipts = self.db.block_receipts(block_num)?;

            // 3. 精确匹配
            for receipt in receipts {
                for log in receipt.logs {
                    if self.filter.matches(&log) {
                        results.push(log);
                    }
                }
            }
        }

        Ok(results)
    }
}
```

---

## 附录 W：内存池管理

### W.1 Reth 交易池

```rust
// Reth 交易池实现
use reth_transaction_pool::{
    TransactionPool, PoolTransaction, TransactionOrigin,
    PooledTransaction, BestTransactions,
};

/// 交易池配置
pub struct TransactionPoolConfig {
    /// pending 队列最大交易数
    pub max_pending: usize,
    /// queued 队列最大交易数
    pub max_queued: usize,
    /// 单账户最大交易数
    pub max_account_slots: usize,
    /// 最小 gas price
    pub minimal_gas_price: u128,
    /// 价格 bump 比例 (替换交易)
    pub price_bump: u8,
}

/// 交易验证器
pub struct TransactionValidator<Client> {
    client: Client,
    chain_spec: Arc<ChainSpec>,
}

impl<Client: StateProvider> TransactionValidator<Client> {
    /// 验证交易
    pub fn validate(&self, tx: &PooledTransaction) -> Result<ValidTransaction, ValidationError> {
        // 1. 基础检查
        self.validate_basics(tx)?;

        // 2. 签名验证
        let sender = tx.recover_signer()
            .ok_or(ValidationError::InvalidSignature)?;

        // 3. 获取账户状态
        let account = self.client.basic_account(sender)?
            .unwrap_or_default();

        // 4. Nonce 检查
        if tx.nonce() < account.nonce {
            return Err(ValidationError::NonceTooLow {
                expected: account.nonce,
                got: tx.nonce(),
            });
        }

        // 5. 余额检查
        let max_cost = tx.max_cost();
        if account.balance < max_cost {
            return Err(ValidationError::InsufficientFunds {
                balance: account.balance,
                cost: max_cost,
            });
        }

        // 6. Gas 限制检查
        let block_gas_limit = self.client.chain_spec().max_gas_limit;
        if tx.gas_limit() > block_gas_limit {
            return Err(ValidationError::GasLimitExceeded);
        }

        // 7. 内在 Gas 检查
        let intrinsic_gas = calculate_intrinsic_gas(tx);
        if tx.gas_limit() < intrinsic_gas {
            return Err(ValidationError::IntrinsicGasTooLow);
        }

        Ok(ValidTransaction {
            transaction: tx.clone(),
            sender,
            nonce_gap: tx.nonce() - account.nonce,
        })
    }

    fn validate_basics(&self, tx: &PooledTransaction) -> Result<(), ValidationError> {
        // 交易大小检查
        if tx.size() > MAX_TX_SIZE {
            return Err(ValidationError::OversizedData);
        }

        // EIP-1559 字段检查
        if let Some(max_fee) = tx.max_fee_per_gas() {
            if max_fee < tx.max_priority_fee_per_gas().unwrap_or(0) {
                return Err(ValidationError::TipAboveFeeCap);
            }
        }

        // EIP-4844 blob 检查
        if let Some(blob_tx) = tx.as_eip4844() {
            if blob_tx.blob_versioned_hashes.is_empty() {
                return Err(ValidationError::NoBlobHashes);
            }
            if blob_tx.blob_versioned_hashes.len() > MAX_BLOBS_PER_TX {
                return Err(ValidationError::TooManyBlobs);
            }
        }

        Ok(())
    }
}
```

### W.2 交易排序与最佳选择

```rust
// 最佳交易迭代器
pub struct BestTransactionsIterator<T: PoolTransaction> {
    /// 按 effective tip 排序的交易
    pending: BinaryHeap<PendingTransaction<T>>,
    /// 已包含的发送者 nonce
    included: HashMap<Address, u64>,
    /// 基础费用
    base_fee: u128,
}

impl<T: PoolTransaction> Iterator for BestTransactionsIterator<T> {
    type Item = Arc<ValidPoolTransaction<T>>;

    fn next(&mut self) -> Option<Self::Item> {
        loop {
            let best = self.pending.pop()?;

            // 检查 nonce 连续性
            let sender = best.sender();
            let expected_nonce = self.included.get(&sender).copied().unwrap_or(0);

            if best.nonce() != expected_nonce {
                // Nonce 不连续，跳过
                continue;
            }

            // 检查 gas price 是否足够
            let effective_tip = best.effective_tip(self.base_fee);
            if effective_tip.is_none() {
                // 低于 base fee
                continue;
            }

            // 更新 nonce
            self.included.insert(sender, best.nonce() + 1);

            return Some(best.transaction);
        }
    }
}

/// 交易优先级比较
#[derive(Clone)]
struct PendingTransaction<T> {
    transaction: Arc<ValidPoolTransaction<T>>,
    effective_tip: u128,
}

impl<T> Ord for PendingTransaction<T> {
    fn cmp(&self, other: &Self) -> Ordering {
        // 优先比较 effective tip
        match self.effective_tip.cmp(&other.effective_tip) {
            Ordering::Equal => {
                // 相同 tip 时按时间排序 (先来先服务)
                self.transaction.timestamp.cmp(&other.transaction.timestamp).reverse()
            }
            other => other,
        }
    }
}
```

### W.3 Blob 交易池

```rust
// EIP-4844 Blob 交易特殊处理
pub struct BlobTransactionPool {
    /// Blob 交易存储
    blob_txs: HashMap<B256, BlobTransaction>,
    /// Blob 数据 sidecar
    blob_sidecars: HashMap<B256, BlobTransactionSidecar>,
    /// 内存限制
    max_blob_pool_size: usize,
}

impl BlobTransactionPool {
    /// 添加 blob 交易
    pub fn add_blob_transaction(
        &mut self,
        tx: PooledTransaction,
        sidecar: BlobTransactionSidecar,
    ) -> Result<(), BlobPoolError> {
        // 验证 blob
        self.validate_blobs(&tx, &sidecar)?;

        // 检查容量
        if self.blob_txs.len() >= self.max_blob_pool_size {
            // 驱逐最低价格的 blob 交易
            self.evict_lowest_price();
        }

        let tx_hash = tx.hash();
        self.blob_txs.insert(tx_hash, tx.into());
        self.blob_sidecars.insert(tx_hash, sidecar);

        Ok(())
    }

    /// 验证 blob 数据
    fn validate_blobs(
        &self,
        tx: &PooledTransaction,
        sidecar: &BlobTransactionSidecar,
    ) -> Result<(), BlobPoolError> {
        let blob_tx = tx.as_eip4844()
            .ok_or(BlobPoolError::NotBlobTransaction)?;

        // 验证 blob 数量匹配
        if blob_tx.blob_versioned_hashes.len() != sidecar.blobs.len() {
            return Err(BlobPoolError::BlobCountMismatch);
        }

        // 验证每个 blob 的 KZG 承诺
        for (i, versioned_hash) in blob_tx.blob_versioned_hashes.iter().enumerate() {
            let commitment = &sidecar.commitments[i];
            let computed_hash = kzg_to_versioned_hash(commitment);

            if *versioned_hash != computed_hash {
                return Err(BlobPoolError::InvalidVersionedHash);
            }

            // 验证 KZG 证明
            let blob = &sidecar.blobs[i];
            let proof = &sidecar.proofs[i];

            if !verify_blob_kzg_proof(blob, commitment, proof)? {
                return Err(BlobPoolError::InvalidKzgProof);
            }
        }

        Ok(())
    }
}
```

---

## 附录 X：RPC 实现

### X.1 Reth JSON-RPC 服务

```rust
// Reth RPC 实现
use jsonrpsee::{
    core::RpcResult,
    proc_macros::rpc,
};

/// Eth 命名空间 RPC
#[rpc(server, namespace = "eth")]
pub trait EthApi {
    /// 获取区块号
    #[method(name = "blockNumber")]
    async fn block_number(&self) -> RpcResult<U64>;

    /// 获取余额
    #[method(name = "getBalance")]
    async fn get_balance(
        &self,
        address: Address,
        block: Option<BlockId>,
    ) -> RpcResult<U256>;

    /// eth_call
    #[method(name = "call")]
    async fn call(
        &self,
        request: TransactionRequest,
        block: Option<BlockId>,
        state_overrides: Option<StateOverride>,
    ) -> RpcResult<Bytes>;

    /// 发送原始交易
    #[method(name = "sendRawTransaction")]
    async fn send_raw_transaction(&self, bytes: Bytes) -> RpcResult<B256>;

    /// 估算 Gas
    #[method(name = "estimateGas")]
    async fn estimate_gas(
        &self,
        request: TransactionRequest,
        block: Option<BlockId>,
    ) -> RpcResult<U256>;
}

/// Eth RPC 实现
pub struct EthApiImpl<Provider, Pool> {
    provider: Provider,
    pool: Pool,
    gas_oracle: GasPriceOracle<Provider>,
    blocking_task_pool: BlockingTaskPool,
}

#[async_trait]
impl<Provider, Pool> EthApiServer for EthApiImpl<Provider, Pool>
where
    Provider: BlockProvider + StateProvider + 'static,
    Pool: TransactionPool + 'static,
{
    async fn block_number(&self) -> RpcResult<U64> {
        let num = self.provider.best_block_number()?;
        Ok(U64::from(num))
    }

    async fn get_balance(&self, address: Address, block: Option<BlockId>) -> RpcResult<U256> {
        let block_id = block.unwrap_or(BlockId::Latest);
        let state = self.provider.state_by_block_id(block_id)?;
        let balance = state.account_balance(address)?.unwrap_or_default();
        Ok(balance)
    }

    async fn call(
        &self,
        request: TransactionRequest,
        block: Option<BlockId>,
        state_overrides: Option<StateOverride>,
    ) -> RpcResult<Bytes> {
        // 在阻塞线程池中执行 EVM
        let result = self.blocking_task_pool.spawn_blocking(move || {
            let block_id = block.unwrap_or(BlockId::Latest);
            let mut evm = self.prepare_evm(block_id, state_overrides)?;

            // 设置交易环境
            let tx_env = request.into_tx_env();
            evm.context.evm.env.tx = tx_env;

            // 执行
            let result = evm.transact()?;

            match result.result {
                ExecutionResult::Success { output, .. } => {
                    Ok(output.into_data())
                }
                ExecutionResult::Revert { output, .. } => {
                    Err(RpcError::Revert(output))
                }
                ExecutionResult::Halt { reason, .. } => {
                    Err(RpcError::Halt(reason))
                }
            }
        }).await??;

        Ok(result)
    }

    async fn send_raw_transaction(&self, bytes: Bytes) -> RpcResult<B256> {
        // 解码交易
        let tx = PooledTransaction::decode(&mut bytes.as_ref())
            .map_err(|_| RpcError::InvalidTransaction)?;

        let hash = tx.hash();

        // 添加到交易池
        self.pool.add_transaction(TransactionOrigin::Local, tx).await?;

        Ok(hash)
    }

    async fn estimate_gas(
        &self,
        request: TransactionRequest,
        block: Option<BlockId>,
    ) -> RpcResult<U256> {
        // 二分查找
        let mut lo = 21000u64;
        let mut hi = self.provider.chain_spec().max_gas_limit;

        while lo + 1 < hi {
            let mid = (lo + hi) / 2;

            let mut req = request.clone();
            req.gas = Some(U256::from(mid));

            match self.call(req, block, None).await {
                Ok(_) => hi = mid,
                Err(_) => lo = mid,
            }
        }

        Ok(U256::from(hi))
    }
}
```

### X.2 WebSocket 订阅

```rust
// WebSocket 订阅实现
use tokio::sync::broadcast;

#[rpc(server, namespace = "eth")]
pub trait EthPubSubApi {
    #[subscription(name = "subscribe", item = SubscriptionResult)]
    async fn subscribe(&self, kind: SubscriptionKind, params: Option<FilterParams>);
}

pub struct EthPubSubApiImpl<Provider> {
    provider: Provider,
    new_heads_tx: broadcast::Sender<Header>,
    pending_txs_tx: broadcast::Sender<B256>,
    logs_tx: broadcast::Sender<Log>,
}

#[async_trait]
impl<Provider: BlockProvider> EthPubSubApiServer for EthPubSubApiImpl<Provider> {
    async fn subscribe(
        &self,
        pending: PendingSubscriptionSink,
        kind: SubscriptionKind,
        params: Option<FilterParams>,
    ) -> SubscriptionResult {
        let sink = pending.accept().await?;

        match kind {
            SubscriptionKind::NewHeads => {
                let mut rx = self.new_heads_tx.subscribe();
                tokio::spawn(async move {
                    while let Ok(header) = rx.recv().await {
                        if sink.send(SubscriptionMessage::from(header)).await.is_err() {
                            break;
                        }
                    }
                });
            }
            SubscriptionKind::Logs => {
                let filter = params.unwrap_or_default().into_filter();
                let mut rx = self.logs_tx.subscribe();
                tokio::spawn(async move {
                    while let Ok(log) = rx.recv().await {
                        if filter.matches(&log) {
                            if sink.send(SubscriptionMessage::from(log)).await.is_err() {
                                break;
                            }
                        }
                    }
                });
            }
            SubscriptionKind::NewPendingTransactions => {
                let mut rx = self.pending_txs_tx.subscribe();
                tokio::spawn(async move {
                    while let Ok(hash) = rx.recv().await {
                        if sink.send(SubscriptionMessage::from(hash)).await.is_err() {
                            break;
                        }
                    }
                });
            }
        }

        Ok(())
    }
}
```

---

## 附录 Y：硬分叉管理

### Y.1 revm 硬分叉配置

```rust
// revm 中的硬分叉处理
use revm::primitives::SpecId;

/// 支持的 EVM 规范版本
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub enum SpecId {
    FRONTIER = 0,
    FRONTIER_THAWING = 1,
    HOMESTEAD = 2,
    DAO_FORK = 3,
    TANGERINE = 4,
    SPURIOUS_DRAGON = 5,
    BYZANTIUM = 6,
    CONSTANTINOPLE = 7,
    PETERSBURG = 8,
    ISTANBUL = 9,
    MUIR_GLACIER = 10,
    BERLIN = 11,
    LONDON = 12,
    ARROW_GLACIER = 13,
    GRAY_GLACIER = 14,
    MERGE = 15,
    SHANGHAI = 16,
    CANCUN = 17,
    PRAGUE = 18,
    LATEST = u8::MAX as usize,
}

impl SpecId {
    /// 检查是否启用特定功能
    pub const fn is_enabled_in(self, spec: Self) -> bool {
        self as usize <= spec as usize
    }

    /// 从区块号和时间戳获取规范
    pub fn from_block_and_timestamp(
        chain_spec: &ChainSpec,
        block: u64,
        timestamp: u64,
    ) -> Self {
        // 时间戳激活的分叉 (Post-Merge)
        if chain_spec.is_prague_active(timestamp) {
            return SpecId::PRAGUE;
        }
        if chain_spec.is_cancun_active(timestamp) {
            return SpecId::CANCUN;
        }
        if chain_spec.is_shanghai_active(timestamp) {
            return SpecId::SHANGHAI;
        }

        // 区块号激活的分叉
        if block >= chain_spec.paris_block.unwrap_or(u64::MAX) {
            return SpecId::MERGE;
        }
        if block >= chain_spec.london_block.unwrap_or(u64::MAX) {
            return SpecId::LONDON;
        }
        if block >= chain_spec.berlin_block.unwrap_or(u64::MAX) {
            return SpecId::BERLIN;
        }
        // ... 其他分叉检查

        SpecId::FRONTIER
    }
}

/// 链规范配置
#[derive(Clone, Debug)]
pub struct ChainSpec {
    pub chain_id: u64,

    // 区块号激活
    pub homestead_block: Option<u64>,
    pub dao_fork_block: Option<u64>,
    pub berlin_block: Option<u64>,
    pub london_block: Option<u64>,
    pub paris_block: Option<u64>,

    // 时间戳激活
    pub shanghai_time: Option<u64>,
    pub cancun_time: Option<u64>,
    pub prague_time: Option<u64>,

    // 特殊配置
    pub dao_fork_support: bool,
    pub eip1559_base_fee: Option<u64>,
}

impl ChainSpec {
    /// 以太坊主网配置
    pub fn mainnet() -> Self {
        Self {
            chain_id: 1,
            homestead_block: Some(1_150_000),
            dao_fork_block: Some(1_920_000),
            berlin_block: Some(12_244_000),
            london_block: Some(12_965_000),
            paris_block: Some(15_537_394),
            shanghai_time: Some(1681338455),
            cancun_time: Some(1710338135),
            prague_time: None,
            dao_fork_support: true,
            eip1559_base_fee: Some(1_000_000_000), // 1 gwei
        }
    }

    pub fn is_shanghai_active(&self, timestamp: u64) -> bool {
        self.shanghai_time.map(|t| timestamp >= t).unwrap_or(false)
    }

    pub fn is_cancun_active(&self, timestamp: u64) -> bool {
        self.cancun_time.map(|t| timestamp >= t).unwrap_or(false)
    }
}
```

### Y.2 指令集演进

```rust
// 根据规范版本选择指令实现
impl<H: Host + ?Sized> Interpreter<'_> {
    pub fn run<SPEC: Spec>(&mut self, host: &mut H) -> InstructionResult {
        loop {
            let opcode = self.current_opcode();

            // 使用编译时分发
            let result = match opcode {
                // 基础操作 (所有版本)
                opcode::STOP => control::stop(self),
                opcode::ADD => arithmetic::add(self),
                opcode::MUL => arithmetic::mul(self),

                // Berlin 新增 (EIP-2929)
                opcode::SLOAD if SPEC::enabled(SpecId::BERLIN) => {
                    host_env::sload_eip2929::<H, SPEC>(self, host)
                }
                opcode::SLOAD => host_env::sload::<H, SPEC>(self, host),

                // London 新增
                opcode::BASEFEE if SPEC::enabled(SpecId::LONDON) => {
                    system::basefee::<H, SPEC>(self, host)
                }
                opcode::BASEFEE => InstructionResult::NotActivated,

                // Shanghai 新增
                opcode::PUSH0 if SPEC::enabled(SpecId::SHANGHAI) => {
                    stack::push0(self)
                }
                opcode::PUSH0 => InstructionResult::NotActivated,

                // Cancun 新增
                opcode::TLOAD if SPEC::enabled(SpecId::CANCUN) => {
                    host_env::tload::<H, SPEC>(self, host)
                }
                opcode::TSTORE if SPEC::enabled(SpecId::CANCUN) => {
                    host_env::tstore::<H, SPEC>(self, host)
                }
                opcode::MCOPY if SPEC::enabled(SpecId::CANCUN) => {
                    memory::mcopy::<H, SPEC>(self, host)
                }
                opcode::BLOBHASH if SPEC::enabled(SpecId::CANCUN) => {
                    system::blobhash::<H, SPEC>(self, host)
                }
                opcode::BLOBBASEFEE if SPEC::enabled(SpecId::CANCUN) => {
                    system::blobbasefee::<H, SPEC>(self, host)
                }

                // 未知或未激活
                _ => InstructionResult::OpcodeNotFound,
            };

            if result != InstructionResult::Continue {
                return result;
            }

            self.instruction_pointer += 1;
        }
    }
}
```

---

## 附录 Z：网络层

### Z.1 libp2p 网络栈

```rust
// Reth 使用 libp2p 实现 P2P 网络
use libp2p::{
    identity::Keypair,
    swarm::{NetworkBehaviour, Swarm},
    PeerId, Multiaddr,
};

/// 以太坊网络行为
#[derive(NetworkBehaviour)]
pub struct EthNetworkBehaviour {
    /// Kademlia DHT 节点发现
    kademlia: Kademlia<MemoryStore>,
    /// 请求-响应协议
    request_response: RequestResponse<EthCodec>,
    /// Gossipsub 消息广播
    gossipsub: Gossipsub,
    /// 识别协议
    identify: Identify,
}

/// 网络服务
pub struct NetworkService {
    swarm: Swarm<EthNetworkBehaviour>,
    /// 已连接的对等节点
    peers: HashMap<PeerId, PeerInfo>,
    /// 本地节点 ID
    local_peer_id: PeerId,
}

impl NetworkService {
    pub fn new(config: NetworkConfig) -> Result<Self, NetworkError> {
        // 生成或加载密钥
        let keypair = config.keypair.unwrap_or_else(Keypair::generate_ed25519);
        let local_peer_id = PeerId::from(keypair.public());

        // 构建传输层
        let transport = libp2p::tcp::tokio::Transport::new(
            libp2p::tcp::Config::default().nodelay(true)
        )
        .upgrade(libp2p::core::upgrade::Version::V1)
        .authenticate(libp2p::noise::NoiseAuthenticated::xx(&keypair)?)
        .multiplex(libp2p::yamux::YamuxConfig::default())
        .boxed();

        // 构建网络行为
        let behaviour = EthNetworkBehaviour {
            kademlia: Kademlia::new(local_peer_id, MemoryStore::new(local_peer_id)),
            request_response: RequestResponse::new(
                EthCodec::default(),
                iter::once((EthProtocol::Eth68, ProtocolSupport::Full)),
                RequestResponseConfig::default(),
            ),
            gossipsub: GossipsubBuilder::default()
                .build()?,
            identify: Identify::new(IdentifyConfig::new(
                "/eth/1.0.0".into(),
                keypair.public(),
            )),
        };

        let swarm = Swarm::with_tokio_executor(transport, behaviour, local_peer_id);

        Ok(Self {
            swarm,
            peers: HashMap::new(),
            local_peer_id,
        })
    }

    /// 广播新区块
    pub fn broadcast_block(&mut self, block: Arc<Block>) {
        let message = EthMessage::NewBlock {
            block: block.clone(),
            td: block.total_difficulty(),
        };

        // 发送给 sqrt(peers) 个节点完整区块
        let peers: Vec<_> = self.peers.keys().cloned().collect();
        let full_send_count = (peers.len() as f64).sqrt() as usize;

        for peer_id in &peers[..full_send_count] {
            self.swarm.behaviour_mut().request_response.send_request(
                peer_id,
                EthRequest::NewBlock(message.clone()),
            );
        }

        // 发送哈希给其余节点
        let hash_message = EthMessage::NewBlockHashes(vec![block.hash()]);
        for peer_id in &peers[full_send_count..] {
            self.swarm.behaviour_mut().request_response.send_request(
                peer_id,
                EthRequest::NewBlockHashes(hash_message.clone()),
            );
        }
    }

    /// 请求区块头
    pub async fn get_block_headers(
        &mut self,
        peer: PeerId,
        start: BlockId,
        limit: u64,
    ) -> Result<Vec<Header>, RequestError> {
        let request = GetBlockHeaders {
            start,
            limit,
            skip: 0,
            reverse: false,
        };

        let response = self.swarm.behaviour_mut()
            .request_response
            .send_request(&peer, EthRequest::GetBlockHeaders(request))
            .await?;

        match response {
            EthResponse::BlockHeaders(headers) => Ok(headers),
            _ => Err(RequestError::UnexpectedResponse),
        }
    }
}
```

### Z.2 ENR 与节点发现

```rust
// Ethereum Node Records (ENR)
use enr::{Enr, EnrBuilder, CombinedKey};

/// 构建 ENR
pub fn build_enr(
    keypair: &CombinedKey,
    ip: IpAddr,
    tcp_port: u16,
    udp_port: u16,
    fork_id: ForkId,
) -> Enr<CombinedKey> {
    EnrBuilder::new("v4")
        .ip(ip)
        .tcp4(tcp_port)
        .udp4(udp_port)
        .add_value("eth", fork_id.encode())
        .build(keypair)
        .expect("valid ENR")
}

/// discv5 节点发现
pub struct Discovery {
    discv5: Discv5,
    /// Bootstrap 节点
    bootnodes: Vec<Enr<CombinedKey>>,
}

impl Discovery {
    /// 发现新节点
    pub async fn find_nodes(&mut self, target: NodeId) -> Vec<Enr<CombinedKey>> {
        self.discv5.find_node(target).await.unwrap_or_default()
    }

    /// 验证 ENR 的 fork ID
    pub fn validate_fork_id(&self, enr: &Enr<CombinedKey>, our_fork_id: &ForkId) -> bool {
        if let Some(eth_value) = enr.get("eth") {
            if let Ok(peer_fork_id) = ForkId::decode(eth_value) {
                return peer_fork_id.is_compatible(our_fork_id);
            }
        }
        false
    }
}
```

---

## 附录 AA：数据库后端

### AA.1 MDBX 集成

```rust
// Reth 使用 MDBX 作为数据库后端
use reth_db::mdbx::{Env, WriteMap, RO, RW};

/// MDBX 数据库环境
pub struct Database {
    env: Env<WriteMap>,
}

impl Database {
    pub fn open(path: &Path) -> Result<Self, DatabaseError> {
        let env = Env::builder()
            .set_max_dbs(100)
            .set_map_size(1024 * 1024 * 1024 * 1024) // 1 TB
            .set_geometry(Geometry {
                size: Some(10 * 1024 * 1024 * 1024..1024 * 1024 * 1024 * 1024),
                growth_step: Some(4 * 1024 * 1024 * 1024), // 4 GB 增长
                ..Default::default()
            })
            .open(path)?;

        Ok(Self { env })
    }

    /// 开始只读事务
    pub fn begin_ro_txn(&self) -> Result<Transaction<'_, RO>, DatabaseError> {
        Ok(self.env.begin_ro_txn()?)
    }

    /// 开始读写事务
    pub fn begin_rw_txn(&self) -> Result<Transaction<'_, RW>, DatabaseError> {
        Ok(self.env.begin_rw_txn()?)
    }
}

/// 表定义
pub trait Table: Sized {
    const NAME: &'static str;
    type Key: Encode + Decode;
    type Value: Compress + Decompress;
}

/// 定义具体表
macro_rules! tables {
    ($($table:ident : $key:ty => $value:ty),* $(,)?) => {
        $(
            pub struct $table;
            impl Table for $table {
                const NAME: &'static str = stringify!($table);
                type Key = $key;
                type Value = $value;
            }
        )*
    };
}

tables! {
    Headers: BlockNumber => Header,
    BlockBodies: BlockNumber => BlockBody,
    Transactions: TxNumber => TransactionSigned,
    Receipts: TxNumber => Receipt,
    AccountHashes: B256 => Account,
    StorageHashes: (B256, B256) => U256,
    Bytecodes: B256 => Bytecode,
    AccountHistory: (Address, BlockNumber) => AccountChanges,
    StorageHistory: (Address, B256, BlockNumber) => U256,
}
```

### AA.2 状态存储优化

```rust
// Reth 扁平状态存储
pub struct FlatState<DB> {
    db: DB,
}

impl<DB: Database> FlatState<DB> {
    /// 获取账户 (不需要遍历 trie)
    pub fn get_account(&self, address: Address) -> Result<Option<Account>, StateError> {
        let txn = self.db.begin_ro_txn()?;
        let hash = keccak256(address);

        txn.get::<AccountHashes>(hash)
            .map_err(|e| StateError::Database(e))
    }

    /// 获取存储值
    pub fn get_storage(
        &self,
        address: Address,
        slot: U256,
    ) -> Result<U256, StateError> {
        let txn = self.db.begin_ro_txn()?;
        let address_hash = keccak256(address);
        let slot_hash = keccak256(slot.to_be_bytes::<32>());

        txn.get::<StorageHashes>((address_hash, slot_hash))
            .map(|opt| opt.unwrap_or(U256::ZERO))
            .map_err(|e| StateError::Database(e))
    }

    /// 批量更新状态
    pub fn update_state(&self, changes: StateChangeset) -> Result<(), StateError> {
        let mut txn = self.db.begin_rw_txn()?;

        // 更新账户
        for (address, account_change) in changes.accounts {
            let hash = keccak256(address);
            match account_change {
                AccountChange::Create(account) | AccountChange::Update(account) => {
                    txn.put::<AccountHashes>(hash, account)?;
                }
                AccountChange::Delete => {
                    txn.delete::<AccountHashes>(hash)?;
                }
            }
        }

        // 更新存储
        for (address, storage_changes) in changes.storage {
            let address_hash = keccak256(address);
            for (slot, value) in storage_changes {
                let slot_hash = keccak256(slot.to_be_bytes::<32>());
                if value == U256::ZERO {
                    txn.delete::<StorageHashes>((address_hash, slot_hash))?;
                } else {
                    txn.put::<StorageHashes>((address_hash, slot_hash), value)?;
                }
            }
        }

        txn.commit()?;
        Ok(())
    }
}
```

### AA.3 数据库架构对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    Reth 数据库架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                      MDBX (默认)                           │ │
│   │  ┌─────────────────────────────────────────────────────┐  │ │
│   │  │  特点                                                │  │ │
│   │  │  - B+ Tree 结构                                     │  │ │
│   │  │  - ACID 事务 (MVCC)                                 │  │ │
│   │  │  - 内存映射 I/O                                     │  │ │
│   │  │  - 单写入者多读取者                                  │  │ │
│   │  │  - 自动压缩                                         │  │ │
│   │  └─────────────────────────────────────────────────────┘  │ │
│   │                                                           │ │
│   │  表结构                                                    │ │
│   │  ┌─────────────────────────────────────────────────────┐  │ │
│   │  │  PlainAccountState    : Address -> Account          │  │ │
│   │  │  PlainStorageState    : (Address, Slot) -> Value    │  │ │
│   │  │  Bytecodes            : CodeHash -> Code            │  │ │
│   │  │  Headers              : BlockNum -> Header          │  │ │
│   │  │  BlockBodies          : BlockNum -> Body            │  │ │
│   │  │  Transactions         : TxNum -> Tx                 │  │ │
│   │  │  Receipts             : TxNum -> Receipt            │  │ │
│   │  │  AccountHistory       : (Addr, Block) -> Changes    │  │ │
│   │  │  StorageHistory       : (Addr, Slot, Block) -> Val  │  │ │
│   │  └─────────────────────────────────────────────────────┘  │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│   存储优化                                                        │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                                                           │ │
│   │  1. 扁平状态 (无 trie 遍历)                                │ │
│   │     - 直接 key-value 查询                                 │ │
│   │     - O(1) 读取复杂度                                     │ │
│   │                                                           │ │
│   │  2. 历史状态索引                                           │ │
│   │     - 按区块索引变化                                       │ │
│   │     - 支持时间点查询                                       │ │
│   │                                                           │ │
│   │  3. 压缩                                                   │ │
│   │     - 键值压缩 (prefix/suffix)                            │ │
│   │     - 页面压缩                                            │ │
│   │                                                           │ │
│   │  空间对比 (主网):                                          │ │
│   │  - Geth (LevelDB): ~900 GB                               │ │
│   │  - Reth (MDBX):    ~500 GB                               │ │
│   │                                                           │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 AB：Witness 生成

### AB.1 无状态客户端架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    无状态客户端架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   传统客户端                         无状态客户端                 │
│   ┌─────────────────┐              ┌─────────────────┐          │
│   │                 │              │                 │          │
│   │  ┌───────────┐  │              │  ┌───────────┐  │          │
│   │  │  状态根    │  │              │  │  状态根    │  │          │
│   │  └─────┬─────┘  │              │  └─────┬─────┘  │          │
│   │        │        │              │        │        │          │
│   │  ┌─────▼─────┐  │              │        │        │          │
│   │  │ 完整状态   │  │              │        X        │          │
│   │  │  树存储    │  │              │    无本地存储    │          │
│   │  │ (500GB+)  │  │              │                 │          │
│   │  └───────────┘  │              │                 │          │
│   │                 │              │                 │          │
│   └─────────────────┘              └─────────────────┘          │
│                                                                  │
│   执行验证                                                        │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                                                           │ │
│   │   区块 + Witness ──────────────┐                          │ │
│   │                                │                          │ │
│   │                         ┌──────▼──────┐                   │ │
│   │                         │   验证器     │                   │ │
│   │                         │             │                   │ │
│   │                         │ 1. 解析witness│                  │ │
│   │                         │ 2. 构建子树  │                   │ │
│   │                         │ 3. 执行交易  │                   │ │
│   │                         │ 4. 验证根    │                   │ │
│   │                         └──────┬──────┘                   │ │
│   │                                │                          │ │
│   │                         ┌──────▼──────┐                   │ │
│   │                         │  新状态根    │                   │ │
│   │                         └─────────────┘                   │ │
│   │                                                           │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### AB.2 Witness 收集器实现

```rust
// Rust 实现的 Witness 收集器
use alloy_primitives::{Address, B256, U256};
use std::collections::{HashMap, HashSet};

/// 账户 witness 数据
#[derive(Debug, Clone)]
pub struct AccountWitness {
    /// 账户地址
    pub address: Address,
    /// 账户数据 (如果存在)
    pub account: Option<AccountData>,
    /// Merkle 证明路径
    pub proof: Vec<B256>,
}

#[derive(Debug, Clone)]
pub struct AccountData {
    pub nonce: u64,
    pub balance: U256,
    pub storage_root: B256,
    pub code_hash: B256,
}

/// 存储 witness 数据
#[derive(Debug, Clone)]
pub struct StorageWitness {
    /// 存储槽
    pub slot: U256,
    /// 存储值
    pub value: U256,
    /// Merkle 证明路径
    pub proof: Vec<B256>,
}

/// 完整的区块 witness
#[derive(Debug, Clone)]
pub struct BlockWitness {
    /// 状态根
    pub state_root: B256,
    /// 账户 witness
    pub accounts: HashMap<Address, AccountWitness>,
    /// 存储 witness (地址 -> 槽 -> witness)
    pub storage: HashMap<Address, HashMap<U256, StorageWitness>>,
    /// 合约代码
    pub codes: HashMap<B256, Vec<u8>>,
}

/// Witness 收集器 - 在 EVM 执行时收集访问的状态
pub struct WitnessCollector {
    /// 收集的 witness 数据
    witness: BlockWitness,
    /// 访问过的账户
    accessed_accounts: HashSet<Address>,
    /// 访问过的存储槽
    accessed_storage: HashMap<Address, HashSet<U256>>,
    /// 状态提供者
    state_provider: Box<dyn StateProvider>,
}

impl WitnessCollector {
    pub fn new(state_root: B256, state_provider: Box<dyn StateProvider>) -> Self {
        Self {
            witness: BlockWitness {
                state_root,
                accounts: HashMap::new(),
                storage: HashMap::new(),
                codes: HashMap::new(),
            },
            accessed_accounts: HashSet::new(),
            accessed_storage: HashMap::new(),
            state_provider,
        }
    }

    /// 记录账户访问
    pub fn access_account(&mut self, address: Address) -> Result<Option<AccountData>, WitnessError> {
        if self.accessed_accounts.insert(address) {
            // 首次访问，获取账户数据和证明
            let (account, proof) = self.state_provider.get_account_with_proof(address)?;

            self.witness.accounts.insert(address, AccountWitness {
                address,
                account: account.clone(),
                proof,
            });

            // 如果有代码，也收集代码
            if let Some(ref acc) = account {
                if acc.code_hash != EMPTY_CODE_HASH {
                    let code = self.state_provider.get_code(acc.code_hash)?;
                    self.witness.codes.insert(acc.code_hash, code);
                }
            }

            Ok(account)
        } else {
            // 已访问过，从缓存返回
            Ok(self.witness.accounts.get(&address).and_then(|w| w.account.clone()))
        }
    }

    /// 记录存储访问
    pub fn access_storage(&mut self, address: Address, slot: U256) -> Result<U256, WitnessError> {
        let slots = self.accessed_storage.entry(address).or_default();

        if slots.insert(slot) {
            // 首次访问此槽
            let (value, proof) = self.state_provider.get_storage_with_proof(address, slot)?;

            let storage_map = self.witness.storage.entry(address).or_default();
            storage_map.insert(slot, StorageWitness {
                slot,
                value,
                proof,
            });

            Ok(value)
        } else {
            // 已访问过
            Ok(self.witness.storage
                .get(&address)
                .and_then(|m| m.get(&slot))
                .map(|w| w.value)
                .unwrap_or(U256::ZERO))
        }
    }

    /// 完成收集，返回 witness
    pub fn finalize(self) -> BlockWitness {
        self.witness
    }
}
```

### AB.3 作为 Host 实现

```rust
use revm::{
    primitives::{Address, B256, U256, AccountInfo, Bytecode},
    Database, DatabaseRef,
};

/// 带 witness 收集的数据库包装器
pub struct WitnessDatabase<DB> {
    inner: DB,
    collector: WitnessCollector,
}

impl<DB: Database> WitnessDatabase<DB> {
    pub fn new(inner: DB, state_root: B256, state_provider: Box<dyn StateProvider>) -> Self {
        Self {
            inner,
            collector: WitnessCollector::new(state_root, state_provider),
        }
    }

    pub fn into_witness(self) -> BlockWitness {
        self.collector.finalize()
    }
}

impl<DB: Database> Database for WitnessDatabase<DB> {
    type Error = DB::Error;

    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error> {
        // 记录访问
        let _ = self.collector.access_account(address);
        // 委托给内部数据库
        self.inner.basic(address)
    }

    fn code_by_hash(&mut self, code_hash: B256) -> Result<Bytecode, Self::Error> {
        // 代码在 access_account 中已收集
        self.inner.code_by_hash(code_hash)
    }

    fn storage(&mut self, address: Address, slot: U256) -> Result<U256, Self::Error> {
        // 记录存储访问
        let _ = self.collector.access_storage(address, slot);
        // 委托给内部数据库
        self.inner.storage(address, slot)
    }

    fn block_hash(&mut self, number: u64) -> Result<B256, Self::Error> {
        self.inner.block_hash(number)
    }
}
```

### AB.4 Verkle Witness（未来方向）

```rust
// Verkle 树 witness 结构 (EIP-6800)
use banderwagon::{Element, Fr};

/// Verkle 承诺
#[derive(Debug, Clone)]
pub struct VerkleCommitment(pub Element);

/// Verkle 证明
#[derive(Debug, Clone)]
pub struct VerkleProof {
    /// 多点证明
    pub proof: IpaProof,
    /// 承诺路径
    pub commitments: Vec<VerkleCommitment>,
    /// 深度
    pub depths: Vec<u8>,
}

/// IPA 证明 (Inner Product Argument)
#[derive(Debug, Clone)]
pub struct IpaProof {
    pub cl: Vec<Element>,
    pub cr: Vec<Element>,
    pub final_evaluation: Fr,
}

/// Verkle witness 数据
#[derive(Debug, Clone)]
pub struct VerkleWitness {
    /// 父状态根
    pub parent_state_root: B256,
    /// 访问的键值对
    pub key_values: Vec<(B256, Option<Vec<u8>>)>,
    /// Verkle 证明
    pub proof: VerkleProof,
}

impl VerkleWitness {
    /// 验证 witness
    pub fn verify(&self, state_root: &VerkleCommitment) -> bool {
        // 1. 从键值对重建叶子节点
        let leaves = self.key_values.iter().map(|(key, value)| {
            verkle_hash(key, value.as_deref())
        }).collect::<Vec<_>>();

        // 2. 验证 IPA 证明
        verify_ipa_proof(
            state_root,
            &leaves,
            &self.proof,
        )
    }

    /// 序列化为 SSZ
    pub fn to_ssz(&self) -> Vec<u8> {
        ssz_encode(self)
    }
}

/// Verkle 树地址空间 (31 字节 stem + 1 字节 suffix)
pub fn verkle_tree_key(address: Address, tree_index: U256, sub_index: u8) -> B256 {
    let mut stem = [0u8; 31];

    // 地址哈希作为 stem 前缀
    let address_hash = pedersen_hash(&address.0);
    stem[..20].copy_from_slice(&address_hash[..20]);

    // tree_index 作为 stem 后缀
    let index_bytes = tree_index.to_be_bytes::<32>();
    stem[20..31].copy_from_slice(&index_bytes[21..32]);

    // 组合 stem 和 sub_index
    let mut key = [0u8; 32];
    key[..31].copy_from_slice(&stem);
    key[31] = sub_index;

    B256::from(key)
}

// Verkle 树中的账户布局
pub mod verkle_account_layout {
    pub const VERSION_LEAF_KEY: u8 = 0;
    pub const BALANCE_LEAF_KEY: u8 = 1;
    pub const NONCE_LEAF_KEY: u8 = 2;
    pub const CODE_HASH_LEAF_KEY: u8 = 3;
    pub const CODE_SIZE_LEAF_KEY: u8 = 4;
}
```

### AB.5 Witness 大小比较

```
┌─────────────────────────────────────────────────────────────────┐
│                    Witness 大小比较                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   MPT Witness (当前)                                             │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                                                           │ │
│   │   组成部分                    典型大小                      │ │
│   │   ─────────────────────────────────────────────          │ │
│   │   账户证明 (每个)              ~1.5 KB (8层)               │ │
│   │   存储证明 (每个)              ~1.5 KB (8层)               │ │
│   │   合约代码                     变化 (0-24KB)               │ │
│   │                                                           │ │
│   │   典型区块 witness                                         │ │
│   │   - 访问 ~100 账户                                        │ │
│   │   - 访问 ~500 存储槽                                       │ │
│   │   - 总计: ~1-2 MB                                         │ │
│   │                                                           │ │
│   │   问题:                                                    │ │
│   │   - 证明大小随树深度增长                                    │ │
│   │   - 重复的中间节点                                         │ │
│   │                                                           │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│   Verkle Witness (未来)                                          │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                                                           │ │
│   │   组成部分                    典型大小                      │ │
│   │   ─────────────────────────────────────────────          │ │
│   │   键值对 (每个)                ~64 字节                    │ │
│   │   IPA 证明 (聚合)              ~1 KB (固定)                │ │
│   │   承诺路径                     ~256 字节                   │ │
│   │                                                           │ │
│   │   同等区块 witness                                         │ │
│   │   - 访问 ~600 叶子                                        │ │
│   │   - IPA 证明: ~1 KB                                       │ │
│   │   - 总计: ~100-200 KB                                     │ │
│   │                                                           │ │
│   │   优势:                                                    │ │
│   │   - 证明可聚合 (IPA)                                       │ │
│   │   - 大小减少 ~10x                                         │ │
│   │   - 验证复杂度 O(n log n)                                  │ │
│   │                                                           │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 AC：Portal Network

### AC.1 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                    Portal Network 架构                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                     Portal 客户端                         │   │
│   │                                                         │   │
│   │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │   │
│   │  │   History   │ │   State     │ │   Beacon    │       │   │
│   │  │   Network   │ │   Network   │ │   Network   │       │   │
│   │  │             │ │             │ │             │       │   │
│   │  │  区块/交易   │ │  状态数据    │ │  共识数据    │       │   │
│   │  │  收据       │ │  (未来)     │ │  light sync │       │   │
│   │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘       │   │
│   │         │               │               │               │   │
│   │  ┌──────▼───────────────▼───────────────▼──────┐       │   │
│   │  │              Discovery v5                    │       │   │
│   │  │          (discv5 - UDP 协议)                 │       │   │
│   │  └──────────────────────────────────────────────┘       │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   数据分布 (DHT)                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │   内容键 = content_id = hash(content_key)               │   │
│   │   节点键 = node_id = hash(ENR public_key)               │   │
│   │                                                         │   │
│   │   距离 = XOR(content_id, node_id)                       │   │
│   │                                                         │   │
│   │   Node A           Node B           Node C              │   │
│   │   ┌─────┐          ┌─────┐          ┌─────┐            │   │
│   │   │0x1..│          │0x5..│          │0x9..│            │   │
│   │   └──┬──┘          └──┬──┘          └──┬──┘            │   │
│   │      │                │                │                │   │
│   │      │   Content 0x4..│                │                │   │
│   │      │   XOR=0x5      │   XOR=0x1      │   XOR=0xD      │   │
│   │      │                ▼                │                │   │
│   │      │           存储在此              │                │   │
│   │                  (距离最近)                              │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### AC.2 Rust Portal 客户端实现

```rust
// Trin - Rust 实现的 Portal Network 客户端
use discv5::{Discv5, Enr};
use tokio::sync::mpsc;

/// Portal Network 协议 ID
pub mod protocol_id {
    pub const HISTORY: &[u8] = b"portal-history";
    pub const STATE: &[u8] = b"portal-state";
    pub const BEACON: &[u8] = b"portal-beacon";
}

/// 内容键类型
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum ContentKey {
    /// 区块头
    BlockHeader(BlockHeaderKey),
    /// 区块体
    BlockBody(BlockBodyKey),
    /// 收据
    Receipts(ReceiptsKey),
    /// 区块头累加器证明
    EpochAccumulator(EpochAccumulatorKey),
}

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct BlockHeaderKey {
    pub block_hash: B256,
}

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct BlockBodyKey {
    pub block_hash: B256,
}

impl ContentKey {
    /// 计算内容 ID
    pub fn content_id(&self) -> B256 {
        match self {
            ContentKey::BlockHeader(key) => {
                let mut data = vec![0x00]; // selector
                data.extend_from_slice(key.block_hash.as_slice());
                keccak256(&data)
            }
            ContentKey::BlockBody(key) => {
                let mut data = vec![0x01];
                data.extend_from_slice(key.block_hash.as_slice());
                keccak256(&data)
            }
            ContentKey::Receipts(key) => {
                let mut data = vec![0x02];
                data.extend_from_slice(key.block_hash.as_slice());
                keccak256(&data)
            }
            ContentKey::EpochAccumulator(key) => {
                let mut data = vec![0x03];
                data.extend_from_slice(&key.epoch.to_be_bytes());
                keccak256(&data)
            }
        }
    }

    /// SSZ 编码
    pub fn encode(&self) -> Vec<u8> {
        match self {
            ContentKey::BlockHeader(key) => {
                let mut result = vec![0x00];
                result.extend_from_slice(key.block_hash.as_slice());
                result
            }
            ContentKey::BlockBody(key) => {
                let mut result = vec![0x01];
                result.extend_from_slice(key.block_hash.as_slice());
                result
            }
            _ => todo!(),
        }
    }
}

/// History Network 服务
pub struct HistoryNetwork {
    /// discv5 实例
    discv5: Discv5,
    /// 本地存储
    storage: ContentStorage,
    /// 存储配额 (字节)
    storage_quota: u64,
    /// 内容请求通道
    request_rx: mpsc::Receiver<ContentRequest>,
}

impl HistoryNetwork {
    pub async fn new(config: NetworkConfig) -> Result<Self, NetworkError> {
        let enr = config.enr;
        let listen_config = discv5::ListenConfig::default()
            .with_ipv4(config.listen_addr, config.listen_port);

        let discv5 = Discv5::new(
            enr,
            config.enr_key,
            discv5::ConfigBuilder::new(listen_config)
                .build(),
        )?;

        Ok(Self {
            discv5,
            storage: ContentStorage::new(config.data_dir)?,
            storage_quota: config.storage_quota,
            request_rx: config.request_rx,
        })
    }

    /// 查找内容
    pub async fn find_content(&mut self, key: ContentKey) -> Result<Option<Vec<u8>>, NetworkError> {
        let content_id = key.content_id();

        // 1. 先检查本地存储
        if let Some(content) = self.storage.get(&content_id)? {
            return Ok(Some(content));
        }

        // 2. 查找最近的节点
        let closest_nodes = self.discv5.find_node(content_id.into()).await?;

        // 3. 向最近节点发送 FindContent 请求
        for node in closest_nodes.iter().take(3) {
            match self.send_find_content(node, &key).await {
                Ok(FindContentResponse::Content(content)) => {
                    // 存储到本地
                    self.storage.put(&content_id, &content)?;
                    return Ok(Some(content));
                }
                Ok(FindContentResponse::Enrs(enrs)) => {
                    // 收到更近的节点列表，继续查找
                    for enr in enrs {
                        if let Ok(FindContentResponse::Content(content)) =
                            self.send_find_content(&enr, &key).await
                        {
                            self.storage.put(&content_id, &content)?;
                            return Ok(Some(content));
                        }
                    }
                }
                Err(_) => continue,
            }
        }

        Ok(None)
    }

    /// 存储并传播内容
    pub async fn store_content(&mut self, key: ContentKey, content: Vec<u8>) -> Result<(), NetworkError> {
        let content_id = key.content_id();

        // 本地存储
        self.storage.put(&content_id, &content)?;

        // 查找应该存储此内容的节点
        let target_nodes = self.find_nodes_for_content(&content_id).await?;

        // 向它们发送 Offer
        for node in target_nodes {
            let _ = self.send_offer(&node, &key, &content).await;
        }

        Ok(())
    }

    async fn send_find_content(&self, node: &Enr, key: &ContentKey) -> Result<FindContentResponse, NetworkError> {
        let request = PortalRequest::FindContent {
            content_key: key.encode(),
        };

        let response = self.discv5.talk_req(
            node.clone(),
            protocol_id::HISTORY.to_vec(),
            request.encode(),
        ).await?;

        FindContentResponse::decode(&response)
    }
}

/// 内容存储
pub struct ContentStorage {
    db: sled::Db,
    /// 当前使用的存储空间
    used_bytes: u64,
}

impl ContentStorage {
    pub fn new(path: PathBuf) -> Result<Self, StorageError> {
        let db = sled::open(path)?;
        let used_bytes = db.size_on_disk()?;
        Ok(Self { db, used_bytes })
    }

    pub fn get(&self, content_id: &B256) -> Result<Option<Vec<u8>>, StorageError> {
        Ok(self.db.get(content_id.as_slice())?.map(|v| v.to_vec()))
    }

    pub fn put(&mut self, content_id: &B256, content: &[u8]) -> Result<(), StorageError> {
        self.db.insert(content_id.as_slice(), content)?;
        self.used_bytes += content.len() as u64;
        Ok(())
    }

    /// 基于距离的内容驱逐
    pub fn evict_furthest(&mut self, node_id: &B256, target_size: u64) -> Result<(), StorageError> {
        while self.used_bytes > target_size {
            // 找到距离本节点最远的内容
            let mut furthest_key = None;
            let mut max_distance = U256::ZERO;

            for entry in self.db.iter() {
                let (key, value) = entry?;
                let content_id = B256::from_slice(&key);
                let distance = xor_distance(node_id, &content_id);

                if distance > max_distance {
                    max_distance = distance;
                    furthest_key = Some(content_id);
                }
            }

            if let Some(key) = furthest_key {
                if let Some(value) = self.db.remove(key.as_slice())? {
                    self.used_bytes -= value.len() as u64;
                }
            } else {
                break;
            }
        }
        Ok(())
    }
}

/// XOR 距离计算
pub fn xor_distance(a: &B256, b: &B256) -> U256 {
    let a_bytes = a.as_slice();
    let b_bytes = b.as_slice();
    let mut result = [0u8; 32];

    for i in 0..32 {
        result[i] = a_bytes[i] ^ b_bytes[i];
    }

    U256::from_be_bytes(result)
}
```

### AC.3 Portal Wire 协议

```rust
/// Portal 协议消息
#[derive(Debug, Clone)]
pub enum PortalMessage {
    /// Ping - 保活和状态同步
    Ping(PingMessage),
    /// Pong - Ping 响应
    Pong(PongMessage),
    /// FindNodes - 查找节点
    FindNodes(FindNodesMessage),
    /// Nodes - FindNodes 响应
    Nodes(NodesMessage),
    /// FindContent - 查找内容
    FindContent(FindContentMessage),
    /// Content - FindContent 响应
    Content(ContentMessage),
    /// Offer - 提供内容
    Offer(OfferMessage),
    /// Accept - Offer 响应
    Accept(AcceptMessage),
}

#[derive(Debug, Clone)]
pub struct PingMessage {
    pub enr_seq: u64,
    pub custom_payload: Vec<u8>, // 用于网络特定数据
}

#[derive(Debug, Clone)]
pub struct FindContentMessage {
    pub content_key: Vec<u8>,
}

#[derive(Debug, Clone)]
pub enum ContentMessage {
    /// 连接 ID (用于 uTP 传输大内容)
    ConnectionId(u16),
    /// 直接返回内容 (小内容)
    Content(Vec<u8>),
    /// 更近的节点列表
    Enrs(Vec<Vec<u8>>),
}

#[derive(Debug, Clone)]
pub struct OfferMessage {
    pub content_keys: Vec<Vec<u8>>,
}

#[derive(Debug, Clone)]
pub struct AcceptMessage {
    /// 连接 ID (用于 uTP)
    pub connection_id: u16,
    /// 接受的内容键位图
    pub content_keys_bitmap: Vec<u8>,
}

impl PortalMessage {
    pub fn encode(&self) -> Vec<u8> {
        let mut result = Vec::new();
        match self {
            PortalMessage::Ping(msg) => {
                result.push(0x00);
                result.extend(msg.enr_seq.to_be_bytes());
                result.extend(&msg.custom_payload);
            }
            PortalMessage::FindContent(msg) => {
                result.push(0x04);
                result.extend(&msg.content_key);
            }
            PortalMessage::Offer(msg) => {
                result.push(0x06);
                // SSZ 编码 content_keys 列表
                let encoded = ssz_encode_list(&msg.content_keys);
                result.extend(encoded);
            }
            _ => todo!(),
        }
        result
    }

    pub fn decode(data: &[u8]) -> Result<Self, DecodeError> {
        if data.is_empty() {
            return Err(DecodeError::EmptyData);
        }

        match data[0] {
            0x00 => {
                let enr_seq = u64::from_be_bytes(data[1..9].try_into()?);
                let custom_payload = data[9..].to_vec();
                Ok(PortalMessage::Ping(PingMessage { enr_seq, custom_payload }))
            }
            0x01 => {
                let enr_seq = u64::from_be_bytes(data[1..9].try_into()?);
                let custom_payload = data[9..].to_vec();
                Ok(PortalMessage::Pong(PongMessage { enr_seq, custom_payload }))
            }
            0x04 => {
                let content_key = data[1..].to_vec();
                Ok(PortalMessage::FindContent(FindContentMessage { content_key }))
            }
            0x05 => {
                match data[1] {
                    0x00 => {
                        let conn_id = u16::from_be_bytes(data[2..4].try_into()?);
                        Ok(PortalMessage::Content(ContentMessage::ConnectionId(conn_id)))
                    }
                    0x01 => {
                        let content = data[2..].to_vec();
                        Ok(PortalMessage::Content(ContentMessage::Content(content)))
                    }
                    0x02 => {
                        let enrs = ssz_decode_list(&data[2..])?;
                        Ok(PortalMessage::Content(ContentMessage::Enrs(enrs)))
                    }
                    _ => Err(DecodeError::InvalidSelector),
                }
            }
            _ => Err(DecodeError::UnknownMessageType),
        }
    }
}
```

### AC.4 区块头累加器

```rust
/// 历史区块头累加器 (Pre-Merge)
/// 用于验证历史区块头的有效性
#[derive(Debug, Clone)]
pub struct HeaderAccumulator {
    /// 历史 epochs (每个 epoch 8192 个区块头)
    pub historical_epochs: Vec<EpochAccumulator>,
    /// 当前 epoch 累加器
    pub current_epoch: EpochAccumulator,
}

/// Epoch 累加器 (Merkle Mountain Range)
#[derive(Debug, Clone)]
pub struct EpochAccumulator {
    pub header_records: Vec<HeaderRecord>,
}

#[derive(Debug, Clone)]
pub struct HeaderRecord {
    pub block_hash: B256,
    pub total_difficulty: U256,
}

impl HeaderAccumulator {
    /// 获取历史区块头的证明
    pub fn get_proof(&self, block_number: u64) -> Option<HeaderProof> {
        let epoch_index = (block_number / EPOCH_SIZE) as usize;
        let record_index = (block_number % EPOCH_SIZE) as usize;

        if epoch_index >= self.historical_epochs.len() {
            return None;
        }

        let epoch = &self.historical_epochs[epoch_index];
        let proof = epoch.generate_proof(record_index);

        Some(HeaderProof {
            epoch_index: epoch_index as u64,
            record_index: record_index as u64,
            proof,
        })
    }

    /// 验证区块头
    pub fn verify_header(&self, header: &Header, proof: &HeaderProof) -> bool {
        let epoch_index = proof.epoch_index as usize;

        if epoch_index >= self.historical_epochs.len() {
            return false;
        }

        let epoch = &self.historical_epochs[epoch_index];
        let expected_record = HeaderRecord {
            block_hash: header.hash(),
            total_difficulty: header.total_difficulty,
        };

        epoch.verify_proof(
            proof.record_index as usize,
            &expected_record,
            &proof.proof,
        )
    }
}

const EPOCH_SIZE: u64 = 8192;

impl EpochAccumulator {
    /// 生成 Merkle 证明
    pub fn generate_proof(&self, index: usize) -> Vec<B256> {
        let leaves: Vec<B256> = self.header_records.iter()
            .map(|r| keccak256(&rlp_encode(r)))
            .collect();

        generate_merkle_proof(&leaves, index)
    }

    /// 验证 Merkle 证明
    pub fn verify_proof(
        &self,
        index: usize,
        record: &HeaderRecord,
        proof: &[B256],
    ) -> bool {
        let leaf = keccak256(&rlp_encode(record));
        let root = self.root();

        verify_merkle_proof(&leaf, proof, index, &root)
    }

    /// 计算根哈希
    pub fn root(&self) -> B256 {
        let leaves: Vec<B256> = self.header_records.iter()
            .map(|r| keccak256(&rlp_encode(r)))
            .collect();

        compute_merkle_root(&leaves)
    }
}
```

---

## 附录 AD：SSZ 编码

### AD.1 SSZ 基础

```rust
// Simple Serialize (SSZ) - 以太坊共识层序列化格式
// Rust 实现: ssz_rs, ethereum_ssz

use ssz_rs::prelude::*;

/// 基本类型编码
pub mod basic_types {
    use super::*;

    /// 无符号整数 (小端序，固定大小)
    pub fn encode_u64(value: u64) -> Vec<u8> {
        value.to_le_bytes().to_vec()
    }

    pub fn decode_u64(data: &[u8]) -> u64 {
        u64::from_le_bytes(data[..8].try_into().unwrap())
    }

    /// 布尔值 (1 字节)
    pub fn encode_bool(value: bool) -> Vec<u8> {
        vec![if value { 1 } else { 0 }]
    }

    /// 定长字节数组
    pub fn encode_bytes32(value: &[u8; 32]) -> Vec<u8> {
        value.to_vec()
    }
}

/// SSZ 容器 - 使用 derive 宏
#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct BeaconBlockHeader {
    pub slot: u64,
    pub proposer_index: u64,
    pub parent_root: [u8; 32],
    pub state_root: [u8; 32],
    pub body_root: [u8; 32],
}

#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct ExecutionPayloadHeader {
    pub parent_hash: [u8; 32],
    pub fee_recipient: [u8; 20],
    pub state_root: [u8; 32],
    pub receipts_root: [u8; 32],
    pub logs_bloom: [u8; 256],
    pub prev_randao: [u8; 32],
    pub block_number: u64,
    pub gas_limit: u64,
    pub gas_used: u64,
    pub timestamp: u64,
    pub extra_data: List<u8, 32>,
    pub base_fee_per_gas: U256,
    pub block_hash: [u8; 32],
    pub transactions_root: [u8; 32],
    pub withdrawals_root: [u8; 32],
    pub blob_gas_used: u64,
    pub excess_blob_gas: u64,
}

/// 可变长度列表
#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct Transaction {
    // List<u8, MAX_BYTES_PER_TRANSACTION>
    pub data: List<u8, 1073741824>,
}

/// SSZ 联合类型 (Union)
#[derive(SimpleSerialize, Debug, Clone)]
pub enum WithdrawalCredentials {
    BLSWithdrawal([u8; 32]),
    ExecutionAddress([u8; 32]),
}

impl Default for WithdrawalCredentials {
    fn default() -> Self {
        Self::BLSWithdrawal([0u8; 32])
    }
}
```

### AD.2 Merkleization

```rust
use sha2::{Sha256, Digest};

/// SSZ Merkleization - 计算 hash_tree_root
pub struct Merkleizer {
    chunks: Vec<[u8; 32]>,
}

impl Merkleizer {
    pub fn new() -> Self {
        Self { chunks: Vec::new() }
    }

    /// 添加一个 chunk (32 字节)
    pub fn add_chunk(&mut self, chunk: [u8; 32]) {
        self.chunks.push(chunk);
    }

    /// 计算 hash_tree_root
    pub fn hash_tree_root(&self) -> [u8; 32] {
        if self.chunks.is_empty() {
            return [0u8; 32];
        }

        let mut chunks = self.chunks.clone();

        // 填充到 2 的幂
        let target_len = chunks.len().next_power_of_two();
        while chunks.len() < target_len {
            chunks.push([0u8; 32]); // 零填充
        }

        // 自底向上构建 Merkle 树
        while chunks.len() > 1 {
            let mut next_level = Vec::new();
            for pair in chunks.chunks(2) {
                let hash = hash_concat(&pair[0], &pair[1]);
                next_level.push(hash);
            }
            chunks = next_level;
        }

        chunks[0]
    }
}

/// 连接两个 32 字节值并哈希
fn hash_concat(a: &[u8; 32], b: &[u8; 32]) -> [u8; 32] {
    let mut hasher = Sha256::new();
    hasher.update(a);
    hasher.update(b);
    let result = hasher.finalize();
    result.into()
}

/// 计算基本类型的 hash_tree_root
pub fn hash_tree_root_u64(value: u64) -> [u8; 32] {
    let mut chunk = [0u8; 32];
    chunk[..8].copy_from_slice(&value.to_le_bytes());
    chunk
}

/// 计算容器的 hash_tree_root
pub fn hash_tree_root_container<T: SimpleSerialize>(value: &T) -> [u8; 32] {
    // 收集所有字段的 hash_tree_root
    let field_roots = value.hash_tree_root_fields();

    let mut merkleizer = Merkleizer::new();
    for root in field_roots {
        merkleizer.add_chunk(root);
    }

    merkleizer.hash_tree_root()
}

/// 计算列表的 hash_tree_root (需要 mix_in_length)
pub fn hash_tree_root_list<T: SimpleSerialize>(
    items: &[T],
    limit: usize,
) -> [u8; 32] {
    // 1. 计算元素的 Merkle 根
    let mut merkleizer = Merkleizer::new();
    for item in items {
        merkleizer.add_chunk(item.hash_tree_root());
    }

    // 填充到 limit 对应的容量
    let chunks_limit = (limit * T::SSZ_FIXED_LEN).div_ceil(32);
    let target_len = chunks_limit.next_power_of_two();

    let content_root = merkleizer.hash_tree_root();

    // 2. Mix in length
    mix_in_length(content_root, items.len())
}

/// 混入长度 (用于可变长度类型)
pub fn mix_in_length(root: [u8; 32], length: usize) -> [u8; 32] {
    let mut length_chunk = [0u8; 32];
    length_chunk[..8].copy_from_slice(&(length as u64).to_le_bytes());

    hash_concat(&root, &length_chunk)
}

/// SSZ trait 实现示例
pub trait SimpleSerialize: Sized + Default {
    const SSZ_FIXED_LEN: usize;
    const IS_VARIABLE_SIZE: bool;

    fn ssz_encode(&self) -> Vec<u8>;
    fn ssz_decode(data: &[u8]) -> Result<Self, DecodeError>;
    fn hash_tree_root(&self) -> [u8; 32];
    fn hash_tree_root_fields(&self) -> Vec<[u8; 32]>;
}

// 为 u64 实现
impl SimpleSerialize for u64 {
    const SSZ_FIXED_LEN: usize = 8;
    const IS_VARIABLE_SIZE: bool = false;

    fn ssz_encode(&self) -> Vec<u8> {
        self.to_le_bytes().to_vec()
    }

    fn ssz_decode(data: &[u8]) -> Result<Self, DecodeError> {
        if data.len() < 8 {
            return Err(DecodeError::InsufficientData);
        }
        Ok(u64::from_le_bytes(data[..8].try_into().unwrap()))
    }

    fn hash_tree_root(&self) -> [u8; 32] {
        hash_tree_root_u64(*self)
    }

    fn hash_tree_root_fields(&self) -> Vec<[u8; 32]> {
        vec![self.hash_tree_root()]
    }
}
```

### AD.3 执行层 SSZ（EIP-6493）

```rust
// 执行层交易的 SSZ 编码 (未来方向)
// 目前执行层使用 RLP，共识层使用 SSZ

/// SSZ 编码的交易 (EIP-6493 提案)
#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct SszTransaction {
    pub chain_id: u64,
    pub nonce: u64,
    pub max_priority_fee_per_gas: U256,
    pub max_fee_per_gas: U256,
    pub gas: u64,
    pub to: Option<[u8; 20]>,  // None 表示合约创建
    pub value: U256,
    pub input: List<u8, 16777216>,
    pub access_list: List<AccessTuple, 16777216>,
    pub max_fee_per_blob_gas: U256,
    pub blob_versioned_hashes: List<[u8; 32], 16777216>,
    pub authorization_list: List<Authorization, 16777216>,
}

#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct AccessTuple {
    pub address: [u8; 20],
    pub storage_keys: List<[u8; 32], 16777216>,
}

#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct Authorization {
    pub chain_id: u64,
    pub address: [u8; 20],
    pub nonce: u64,
    pub y_parity: u8,
    pub r: U256,
    pub s: U256,
}

/// SSZ 签名交易
#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct SignedSszTransaction {
    pub message: SszTransaction,
    pub signature: EcdsaSignature,
}

#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct EcdsaSignature {
    pub y_parity: bool,
    pub r: U256,
    pub s: U256,
}

/// SSZ 收据
#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct SszReceipt {
    pub status: bool,
    pub cumulative_gas_used: u64,
    pub logs_bloom: [u8; 256],
    pub logs: List<SszLog, 16777216>,
}

#[derive(SimpleSerialize, Default, Debug, Clone)]
pub struct SszLog {
    pub address: [u8; 20],
    pub topics: List<[u8; 32], 4>,
    pub data: List<u8, 16777216>,
}

/// RLP 与 SSZ 转换
pub mod conversion {
    use super::*;

    pub fn rlp_to_ssz_transaction(rlp_tx: &RlpTransaction) -> SszTransaction {
        SszTransaction {
            chain_id: rlp_tx.chain_id,
            nonce: rlp_tx.nonce,
            max_priority_fee_per_gas: rlp_tx.max_priority_fee_per_gas,
            max_fee_per_gas: rlp_tx.max_fee_per_gas,
            gas: rlp_tx.gas_limit,
            to: rlp_tx.to,
            value: rlp_tx.value,
            input: rlp_tx.data.clone().into(),
            access_list: rlp_tx.access_list.iter()
                .map(|t| AccessTuple {
                    address: t.address,
                    storage_keys: t.storage_keys.clone().into(),
                })
                .collect::<Vec<_>>()
                .into(),
            max_fee_per_blob_gas: rlp_tx.max_fee_per_blob_gas.unwrap_or_default(),
            blob_versioned_hashes: rlp_tx.blob_versioned_hashes.clone()
                .unwrap_or_default()
                .into(),
            authorization_list: List::default(),
        }
    }
}
```

### AD.4 SSZ 证明生成

```rust
/// 生成 SSZ Merkle 证明
pub struct SszProofGenerator;

impl SszProofGenerator {
    /// 生成容器字段的证明
    pub fn generate_field_proof<T: SimpleSerialize>(
        container: &T,
        field_index: usize,
    ) -> SszProof {
        let field_roots = container.hash_tree_root_fields();
        let leaf = field_roots[field_index];

        // 构建完整的 Merkle 树
        let tree = build_merkle_tree(&field_roots);

        // 提取证明路径
        let proof_path = extract_proof_path(&tree, field_index);

        SszProof {
            leaf,
            branch: proof_path,
            index: field_index as u64,
        }
    }

    /// 生成嵌套证明 (GeneralizedIndex)
    pub fn generate_proof_at_gindex(
        root: &[u8; 32],
        tree: &MerkleTree,
        gindex: u64,
    ) -> SszProof {
        let depth = 64 - gindex.leading_zeros() as usize - 1;
        let mut branch = Vec::with_capacity(depth);
        let mut current_gindex = gindex;

        for _ in 0..depth {
            let sibling_gindex = current_gindex ^ 1;
            let sibling = tree.get_node(sibling_gindex)
                .unwrap_or(&[0u8; 32]);
            branch.push(*sibling);
            current_gindex /= 2;
        }

        SszProof {
            leaf: *tree.get_node(gindex).unwrap_or(&[0u8; 32]),
            branch,
            index: gindex,
        }
    }
}

#[derive(Debug, Clone)]
pub struct SszProof {
    pub leaf: [u8; 32],
    pub branch: Vec<[u8; 32]>,
    pub index: u64,
}

impl SszProof {
    /// 验证证明
    pub fn verify(&self, root: &[u8; 32]) -> bool {
        let mut current = self.leaf;
        let mut index = self.index;

        for sibling in &self.branch {
            if index % 2 == 0 {
                current = hash_concat(&current, sibling);
            } else {
                current = hash_concat(sibling, &current);
            }
            index /= 2;
        }

        &current == root
    }
}

/// Generalized Index 计算
pub mod gindex {
    /// 计算容器字段的 generalized index
    pub fn container_field_gindex(field_index: usize, num_fields: usize) -> u64 {
        let depth = (num_fields as f64).log2().ceil() as u32;
        let base = 1u64 << depth;
        base + field_index as u64
    }

    /// 计算嵌套路径的 generalized index
    pub fn concat_gindices(parent: u64, child: u64) -> u64 {
        let child_depth = 64 - child.leading_zeros() as u64 - 1;
        (parent << child_depth) | (child & ((1 << child_depth) - 1))
    }
}
```

---

## 附录 AE：MEV-Boost 详解

### AE.1 完整架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MEV-Boost 完整架构                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   用户交易流                                                                  │
│   ┌─────┐                                                                    │
│   │User │───TX───┐                                                          │
│   └─────┘        │                                                          │
│                  ▼                                                          │
│   ┌──────────────────────────────────────────────────────────────┐         │
│   │                        Public Mempool                        │         │
│   └──────────────────────────────┬───────────────────────────────┘         │
│                                  │                                          │
│                                  ▼                                          │
│   ┌──────────────────────────────────────────────────────────────┐         │
│   │                         Searchers                            │         │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │         │
│   │  │ Arbitrage   │  │ Liquidation │  │ Sandwich    │          │         │
│   │  │   Bot       │  │    Bot      │  │   Bot       │          │         │
│   │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │         │
│   │         │                │                │                  │         │
│   │         └────────────────┼────────────────┘                  │         │
│   │                          │ Bundles                           │         │
│   └──────────────────────────┼───────────────────────────────────┘         │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────┐         │
│   │                         Builders                             │         │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │         │
│   │  │  Flashbots  │  │    Titan    │  │   BloXroute │          │         │
│   │  │   Builder   │  │   Builder   │  │   Builder   │          │         │
│   │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │         │
│   │         │                │                │                  │         │
│   │         │ Block Bids     │                │                  │         │
│   │         └────────────────┼────────────────┘                  │         │
│   └──────────────────────────┼───────────────────────────────────┘         │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────┐         │
│   │                          Relays                              │         │
│   │  ┌───────────────────┐  ┌───────────────────┐               │         │
│   │  │  Flashbots Relay  │  │  Ultra Sound Relay│               │         │
│   │  │                   │  │                   │               │         │
│   │  │ 1. 验证区块有效性  │  │ 1. 验证区块有效性 │               │         │
│   │  │ 2. 托管区块      │  │ 2. 托管区块       │               │         │
│   │  │ 3. 提交出价      │  │ 3. 提交出价       │               │         │
│   │  └─────────┬─────────┘  └─────────┬─────────┘               │         │
│   │            │                      │                          │         │
│   │            └──────────┬───────────┘                          │         │
│   └───────────────────────┼──────────────────────────────────────┘         │
│                           │ getHeader (bids)                                │
│                           ▼                                                 │
│   ┌──────────────────────────────────────────────────────────────┐         │
│   │                        MEV-Boost                             │         │
│   │                    (Sidecar Service)                         │         │
│   │                                                              │         │
│   │  1. 聚合多个 Relay 的出价                                     │         │
│   │  2. 选择最高出价                                              │         │
│   │  3. 签名盲区块头                                              │         │
│   │  4. 获取完整区块                                              │         │
│   │                                                              │         │
│   └───────────────────────┬──────────────────────────────────────┘         │
│                           │ Builder API                                     │
│                           ▼                                                 │
│   ┌──────────────────────────────────────────────────────────────┐         │
│   │                     Consensus Client                         │         │
│   │  ┌─────────────────────────────────────────────────────┐    │         │
│   │  │                   Validator                          │    │         │
│   │  │                                                      │    │         │
│   │  │  Slot N: 被选为 Proposer                             │    │         │
│   │  │  1. registerValidator (每 epoch)                     │    │         │
│   │  │  2. getHeader → 获取最高出价                          │    │         │
│   │  │  3. 签名盲区块头                                      │    │         │
│   │  │  4. getPayload → 获取完整区块                         │    │         │
│   │  │  5. 广播区块                                          │    │         │
│   │  │                                                      │    │         │
│   │  └─────────────────────────────────────────────────────┘    │         │
│   └──────────────────────────────────────────────────────────────┘         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### AE.2 Builder API（Rust 实现）

```rust
// MEV-Boost Builder API 的 Rust 实现
use axum::{Router, routing::post, Json, extract::State};
use alloy_primitives::{B256, U256, Address};

/// Builder API 路由
pub fn builder_api_router(state: AppState) -> Router {
    Router::new()
        .route("/eth/v1/builder/validators", post(register_validators))
        .route("/eth/v1/builder/header/:slot/:parent_hash/:pubkey", axum::routing::get(get_header))
        .route("/eth/v1/builder/blinded_blocks", post(submit_blinded_block))
        .route("/eth/v1/builder/status", axum::routing::get(status))
        .with_state(state)
}

/// 验证者注册
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SignedValidatorRegistration {
    pub message: ValidatorRegistration,
    pub signature: BlsSignature,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ValidatorRegistration {
    pub fee_recipient: Address,
    pub gas_limit: u64,
    pub timestamp: u64,
    pub pubkey: BlsPubkey,
}

async fn register_validators(
    State(state): State<AppState>,
    Json(registrations): Json<Vec<SignedValidatorRegistration>>,
) -> Result<(), ApiError> {
    for reg in registrations {
        // 验证 BLS 签名
        if !verify_registration_signature(&reg) {
            return Err(ApiError::InvalidSignature);
        }

        // 存储注册信息
        state.registrations.insert(reg.message.pubkey, reg.message);
    }
    Ok(())
}

/// 获取区块头 (出价)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SignedBuilderBid {
    pub message: BuilderBid,
    pub signature: BlsSignature,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BuilderBid {
    pub header: ExecutionPayloadHeader,
    pub blob_kzg_commitments: Vec<KzgCommitment>,
    pub value: U256,
    pub pubkey: BlsPubkey,
}

async fn get_header(
    State(state): State<AppState>,
    axum::extract::Path((slot, parent_hash, pubkey)): axum::extract::Path<(u64, B256, BlsPubkey)>,
) -> Result<Json<SignedBuilderBid>, ApiError> {
    // 从所有 relay 获取出价
    let bids = state.fetch_bids_from_relays(slot, parent_hash, &pubkey).await?;

    if bids.is_empty() {
        return Err(ApiError::NoBidsAvailable);
    }

    // 选择最高出价
    let best_bid = bids.into_iter()
        .max_by_key(|b| b.message.value)
        .unwrap();

    // 验证出价
    verify_bid(&best_bid, slot, &parent_hash)?;

    Ok(Json(best_bid))
}

/// 提交盲区块
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SignedBlindedBeaconBlock {
    pub message: BlindedBeaconBlock,
    pub signature: BlsSignature,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BlindedBeaconBlock {
    pub slot: u64,
    pub proposer_index: u64,
    pub parent_root: B256,
    pub state_root: B256,
    pub body: BlindedBeaconBlockBody,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BlindedBeaconBlockBody {
    pub execution_payload_header: ExecutionPayloadHeader,
    pub blob_kzg_commitments: Vec<KzgCommitment>,
    // ... 其他字段
}

async fn submit_blinded_block(
    State(state): State<AppState>,
    Json(block): Json<SignedBlindedBeaconBlock>,
) -> Result<Json<ExecutionPayload>, ApiError> {
    // 验证签名
    let proposer = state.get_proposer(block.message.slot)?;
    if !verify_block_signature(&block, &proposer.pubkey) {
        return Err(ApiError::InvalidSignature);
    }

    // 从 relay 获取完整 payload
    let payload = state.get_payload_from_relay(
        &block.message.body.execution_payload_header,
    ).await?;

    // 验证 payload 匹配 header
    if payload.hash() != block.message.body.execution_payload_header.block_hash {
        return Err(ApiError::PayloadMismatch);
    }

    Ok(Json(payload))
}
```

### AE.3 MEV-Boost Sidecar 实现

```rust
// MEV-Boost sidecar 服务
use tokio::sync::RwLock;
use std::sync::Arc;

pub struct MevBoost {
    /// 配置的 relay 列表
    relays: Vec<RelayClient>,
    /// 验证者注册缓存
    registrations: Arc<RwLock<HashMap<BlsPubkey, ValidatorRegistration>>>,
    /// 当前最佳出价缓存
    bid_cache: Arc<RwLock<HashMap<(u64, B256), SignedBuilderBid>>>,
    /// 配置
    config: MevBoostConfig,
}

#[derive(Debug, Clone)]
pub struct MevBoostConfig {
    /// 最小出价值 (wei)
    pub min_bid_value: U256,
    /// Relay 请求超时
    pub relay_timeout: Duration,
    /// 是否允许无 relay 出价时本地构建
    pub fallback_to_local: bool,
}

impl MevBoost {
    pub fn new(config: MevBoostConfig, relay_urls: Vec<String>) -> Self {
        let relays = relay_urls.into_iter()
            .map(|url| RelayClient::new(url))
            .collect();

        Self {
            relays,
            registrations: Arc::new(RwLock::new(HashMap::new())),
            bid_cache: Arc::new(RwLock::new(HashMap::new())),
            config,
        }
    }

    /// 注册验证者到所有 relay
    pub async fn register_validators(
        &self,
        registrations: Vec<SignedValidatorRegistration>,
    ) -> Result<(), MevBoostError> {
        // 并行发送到所有 relay
        let futures: Vec<_> = self.relays.iter()
            .map(|relay| relay.register_validators(&registrations))
            .collect();

        let results = futures::future::join_all(futures).await;

        // 至少一个成功即可
        let success_count = results.iter().filter(|r| r.is_ok()).count();
        if success_count == 0 {
            return Err(MevBoostError::AllRelaysFailed);
        }

        // 缓存注册信息
        let mut cache = self.registrations.write().await;
        for reg in registrations {
            cache.insert(reg.message.pubkey, reg.message);
        }

        Ok(())
    }

    /// 获取最佳出价
    pub async fn get_header(
        &self,
        slot: u64,
        parent_hash: B256,
        pubkey: &BlsPubkey,
    ) -> Result<SignedBuilderBid, MevBoostError> {
        // 检查缓存
        let cache_key = (slot, parent_hash);
        {
            let cache = self.bid_cache.read().await;
            if let Some(bid) = cache.get(&cache_key) {
                return Ok(bid.clone());
            }
        }

        // 并行从所有 relay 获取出价
        let futures: Vec<_> = self.relays.iter()
            .map(|relay| {
                let timeout = self.config.relay_timeout;
                async move {
                    tokio::time::timeout(
                        timeout,
                        relay.get_header(slot, parent_hash, pubkey),
                    ).await
                }
            })
            .collect();

        let results = futures::future::join_all(futures).await;

        // 收集有效出价
        let mut valid_bids: Vec<SignedBuilderBid> = Vec::new();
        for result in results {
            if let Ok(Ok(bid)) = result {
                if self.validate_bid(&bid, slot, &parent_hash).is_ok() {
                    valid_bids.push(bid);
                }
            }
        }

        if valid_bids.is_empty() {
            return Err(MevBoostError::NoBidsAvailable);
        }

        // 选择最高出价
        let best_bid = valid_bids.into_iter()
            .filter(|b| b.message.value >= self.config.min_bid_value)
            .max_by_key(|b| b.message.value)
            .ok_or(MevBoostError::NoBidsAboveMinimum)?;

        // 缓存
        {
            let mut cache = self.bid_cache.write().await;
            cache.insert(cache_key, best_bid.clone());
        }

        Ok(best_bid)
    }

    /// 获取 payload
    pub async fn get_payload(
        &self,
        signed_block: &SignedBlindedBeaconBlock,
    ) -> Result<ExecutionPayloadWithBlobs, MevBoostError> {
        let block_hash = signed_block.message.body.execution_payload_header.block_hash;

        // 尝试从每个 relay 获取
        for relay in &self.relays {
            match relay.get_payload(signed_block).await {
                Ok(payload) => {
                    // 验证 payload
                    if payload.execution_payload.block_hash == block_hash {
                        return Ok(payload);
                    }
                }
                Err(e) => {
                    tracing::warn!("Relay get_payload failed: {:?}", e);
                    continue;
                }
            }
        }

        Err(MevBoostError::PayloadNotFound)
    }

    fn validate_bid(
        &self,
        bid: &SignedBuilderBid,
        slot: u64,
        parent_hash: &B256,
    ) -> Result<(), MevBoostError> {
        // 验证 slot
        if bid.message.header.block_number != slot {
            return Err(MevBoostError::InvalidSlot);
        }

        // 验证 parent hash
        if bid.message.header.parent_hash != *parent_hash {
            return Err(MevBoostError::InvalidParentHash);
        }

        // 验证签名
        if !verify_builder_signature(bid) {
            return Err(MevBoostError::InvalidSignature);
        }

        Ok(())
    }
}

/// Relay 客户端
pub struct RelayClient {
    url: String,
    client: reqwest::Client,
}

impl RelayClient {
    pub fn new(url: String) -> Self {
        Self {
            url,
            client: reqwest::Client::new(),
        }
    }

    pub async fn register_validators(
        &self,
        registrations: &[SignedValidatorRegistration],
    ) -> Result<(), RelayError> {
        let url = format!("{}/eth/v1/builder/validators", self.url);
        let response = self.client
            .post(&url)
            .json(registrations)
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(RelayError::RegistrationFailed);
        }
        Ok(())
    }

    pub async fn get_header(
        &self,
        slot: u64,
        parent_hash: B256,
        pubkey: &BlsPubkey,
    ) -> Result<SignedBuilderBid, RelayError> {
        let url = format!(
            "{}/eth/v1/builder/header/{}/{}/{}",
            self.url, slot, parent_hash, pubkey
        );

        let response = self.client.get(&url).send().await?;

        if response.status() == 204 {
            return Err(RelayError::NoBid);
        }

        let bid: SignedBuilderBid = response.json().await?;
        Ok(bid)
    }

    pub async fn get_payload(
        &self,
        block: &SignedBlindedBeaconBlock,
    ) -> Result<ExecutionPayloadWithBlobs, RelayError> {
        let url = format!("{}/eth/v1/builder/blinded_blocks", self.url);

        let response = self.client
            .post(&url)
            .json(block)
            .send()
            .await?;

        let payload: ExecutionPayloadWithBlobs = response.json().await?;
        Ok(payload)
    }
}
```

### AE.4 收益分析

```rust
/// MEV 收益分析
pub struct MevAnalyzer;

impl MevAnalyzer {
    /// 分析区块的 MEV 收益
    pub fn analyze_block_mev(block: &Block, receipts: &[Receipt]) -> MevAnalysis {
        let mut analysis = MevAnalysis::default();

        // 1. 计算总 Gas 费
        analysis.total_gas_fees = receipts.iter()
            .map(|r| r.gas_used * r.effective_gas_price)
            .sum();

        // 2. 计算优先费 (给验证者)
        analysis.priority_fees = receipts.iter()
            .map(|r| {
                r.gas_used * (r.effective_gas_price.saturating_sub(block.base_fee_per_gas))
            })
            .sum();

        // 3. 检测 MEV 模式
        analysis.patterns = Self::detect_mev_patterns(&block.transactions, receipts);

        // 4. 估算 MEV 提取值
        analysis.extracted_mev = Self::estimate_extracted_mev(&analysis.patterns);

        analysis
    }

    fn detect_mev_patterns(txs: &[Transaction], receipts: &[Receipt]) -> Vec<MevPattern> {
        let mut patterns = Vec::new();

        // 检测三明治攻击
        for window in txs.windows(3) {
            if Self::is_sandwich(&window[0], &window[1], &window[2]) {
                patterns.push(MevPattern::Sandwich {
                    front_run: window[0].hash(),
                    victim: window[1].hash(),
                    back_run: window[2].hash(),
                });
            }
        }

        // 检测套利
        for (i, tx) in txs.iter().enumerate() {
            if let Some(arb) = Self::detect_arbitrage(tx, &receipts[i]) {
                patterns.push(arb);
            }
        }

        // 检测清算
        for (i, tx) in txs.iter().enumerate() {
            if let Some(liq) = Self::detect_liquidation(tx, &receipts[i]) {
                patterns.push(liq);
            }
        }

        patterns
    }

    fn is_sandwich(front: &Transaction, victim: &Transaction, back: &Transaction) -> bool {
        // 检查是否同一发送者的前后交易
        if front.from != back.from {
            return false;
        }

        // 检查是否涉及相同的 DEX 合约
        if front.to != victim.to || victim.to != back.to {
            return false;
        }

        // 检查交易方向 (买-卖 或 卖-买)
        // 简化检测：检查方法选择器
        let front_selector = &front.input[..4];
        let back_selector = &back.input[..4];

        // swap 方法选择器
        let swap_exact_tokens = [0x38, 0xed, 0x17, 0x39]; // swapExactTokensForTokens
        let swap_tokens_exact = [0x88, 0x03, 0xdb, 0xee]; // swapTokensForExactTokens

        (front_selector == swap_exact_tokens && back_selector == swap_tokens_exact) ||
        (front_selector == swap_tokens_exact && back_selector == swap_exact_tokens)
    }

    fn estimate_extracted_mev(patterns: &[MevPattern]) -> U256 {
        patterns.iter()
            .map(|p| match p {
                MevPattern::Sandwich { .. } => U256::from(1000) * U256::from(10).pow(U256::from(18)), // 估算
                MevPattern::Arbitrage { profit, .. } => *profit,
                MevPattern::Liquidation { bonus, .. } => *bonus,
            })
            .sum()
    }
}

#[derive(Debug, Default)]
pub struct MevAnalysis {
    pub total_gas_fees: U256,
    pub priority_fees: U256,
    pub patterns: Vec<MevPattern>,
    pub extracted_mev: U256,
}

#[derive(Debug)]
pub enum MevPattern {
    Sandwich {
        front_run: B256,
        victim: B256,
        back_run: B256,
    },
    Arbitrage {
        tx_hash: B256,
        profit: U256,
        path: Vec<Address>,
    },
    Liquidation {
        tx_hash: B256,
        protocol: String,
        bonus: U256,
    },
}
```

---

## 附录 AF：Hive 测试框架

### AF.1 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Hive 测试框架架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        Hive Controller                              │   │
│   │                                                                     │   │
│   │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │   │
│   │  │  Test Suites    │  │  Client Images  │  │   Result Store  │    │   │
│   │  │                 │  │                 │  │                 │    │   │
│   │  │  - sync         │  │  - geth         │  │  - JSON 结果    │    │   │
│   │  │  - rpc          │  │  - reth         │  │  - 日志收集     │    │   │
│   │  │  - devp2p       │  │  - erigon       │  │  - 性能指标     │    │   │
│   │  │  - engine       │  │  - nethermind   │  │                 │    │   │
│   │  │  - graphql      │  │  - besu         │  │                 │    │   │
│   │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │   │
│   │           │                    │                    │              │   │
│   └───────────┼────────────────────┼────────────────────┼──────────────┘   │
│               │                    │                    │                   │
│               ▼                    ▼                    ▼                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         Docker Engine                               │   │
│   │                                                                     │   │
│   │   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │   │
│   │   │   Client A    │  │   Client B    │  │  Test Runner  │          │   │
│   │   │   Container   │  │   Container   │  │   Container   │          │   │
│   │   │               │  │               │  │               │          │   │
│   │   │  ┌─────────┐  │  │  ┌─────────┐  │  │  执行测试用例  │          │   │
│   │   │  │  Reth   │  │  │  │  Geth   │  │  │               │          │   │
│   │   │  │ (EL)    │  │  │  │ (EL)    │  │  │  1. 启动客户端│          │   │
│   │   │  └─────────┘  │  │  └─────────┘  │  │  2. 执行操作  │          │   │
│   │   │               │  │               │  │  3. 验证结果  │          │   │
│   │   └───────┬───────┘  └───────┬───────┘  └───────┬───────┘          │   │
│   │           │                  │                  │                   │   │
│   │           └──────────────────┼──────────────────┘                   │   │
│   │                              │                                      │   │
│   │                    Docker Network                                   │   │
│   │                              │                                      │   │
│   └──────────────────────────────┼──────────────────────────────────────┘   │
│                                  │                                          │
│                                  ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       Test Execution Flow                           │   │
│   │                                                                     │   │
│   │   1. Hive 读取测试套件定义                                           │   │
│   │   2. 为每个测试构建/拉取客户端镜像                                    │   │
│   │   3. 创建 Docker 网络                                                │   │
│   │   4. 启动客户端容器 (带配置)                                          │   │
│   │   5. 运行测试用例                                                    │   │
│   │   6. 收集结果和日志                                                  │   │
│   │   7. 生成报告                                                        │   │
│   │                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### AF.2 Reth 客户端 Dockerfile

```dockerfile
# clients/reth/Dockerfile
FROM rust:1.75-bookworm as builder

# 构建参数
ARG RETH_VERSION=v0.1.0
ARG FEATURES=""

# 安装依赖
RUN apt-get update && apt-get install -y \
    cmake \
    libclang-dev \
    && rm -rf /var/lib/apt/lists/*

# 克隆并构建 reth
WORKDIR /build
RUN git clone --depth 1 --branch ${RETH_VERSION} https://github.com/paradigmxyz/reth.git
WORKDIR /build/reth
RUN cargo build --release --features "${FEATURES}"

# 运行时镜像
FROM debian:bookworm-slim

# 安装运行时依赖
RUN apt-get update && apt-get install -y \
    ca-certificates \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# 复制二进制
COPY --from=builder /build/reth/target/release/reth /usr/local/bin/

# 复制 hive 入口脚本
COPY reth.sh /reth.sh
RUN chmod +x /reth.sh

# 暴露端口
EXPOSE 8545 8546 8551 30303 30303/udp

# 入口点
ENTRYPOINT ["/reth.sh"]
```

### AF.3 Hive 入口脚本

```bash
#!/bin/bash
# clients/reth/reth.sh - Reth 的 Hive 入口脚本

set -e

# 从环境变量获取配置
HIVE_LOGLEVEL=${HIVE_LOGLEVEL:-3}
HIVE_NETWORK_ID=${HIVE_NETWORK_ID:-1}
HIVE_CHAIN_ID=${HIVE_CHAIN_ID:-1}
HIVE_BOOTNODE=${HIVE_BOOTNODE:-}
HIVE_GRAPHQL_ENABLED=${HIVE_GRAPHQL_ENABLED:-0}

# 数据目录
DATA_DIR="/data"
mkdir -p $DATA_DIR

# 初始化创世块
if [ -n "$HIVE_GENESIS" ]; then
    echo "$HIVE_GENESIS" > /tmp/genesis.json
fi

# 导入预分配账户密钥
if [ -n "$HIVE_CLIQUE_PRIVATEKEY" ]; then
    echo "$HIVE_CLIQUE_PRIVATEKEY" > $DATA_DIR/signer.key
fi

# 构建命令参数
RETH_ARGS=(
    --datadir "$DATA_DIR"
    --chain "/tmp/genesis.json"
    --http
    --http.addr "0.0.0.0"
    --http.port 8545
    --http.api "eth,net,web3,debug,trace,txpool,admin"
    --ws
    --ws.addr "0.0.0.0"
    --ws.port 8546
    --authrpc.addr "0.0.0.0"
    --authrpc.port 8551
)

# 日志级别
case $HIVE_LOGLEVEL in
    1) RETH_ARGS+=(--log.stdout.filter "error") ;;
    2) RETH_ARGS+=(--log.stdout.filter "warn") ;;
    3) RETH_ARGS+=(--log.stdout.filter "info") ;;
    4) RETH_ARGS+=(--log.stdout.filter "debug") ;;
    5) RETH_ARGS+=(--log.stdout.filter "trace") ;;
esac

# 网络配置
RETH_ARGS+=(--network.chain-id "$HIVE_CHAIN_ID")

# Bootnode
if [ -n "$HIVE_BOOTNODE" ]; then
    RETH_ARGS+=(--bootnodes "$HIVE_BOOTNODE")
fi

# Engine API JWT
if [ -n "$HIVE_ENGINE_JWT" ]; then
    echo "$HIVE_ENGINE_JWT" > $DATA_DIR/jwt.hex
    RETH_ARGS+=(--authrpc.jwtsecret "$DATA_DIR/jwt.hex")
fi

# 终端总难度 (for merge)
if [ -n "$HIVE_TERMINAL_TOTAL_DIFFICULTY" ]; then
    # Reth 从 genesis 读取 TTD
    :
fi

# 同步模式
if [ "$HIVE_SYNC_MODE" = "snap" ]; then
    RETH_ARGS+=(--full)
fi

# GraphQL
if [ "$HIVE_GRAPHQL_ENABLED" = "1" ]; then
    RETH_ARGS+=(--graphql --graphql.addr "0.0.0.0")
fi

# 导入区块 (用于同步测试)
if [ -n "$HIVE_INIT_BLOCKS" ]; then
    for block in $HIVE_INIT_BLOCKS; do
        reth import "$block" --datadir "$DATA_DIR" --chain "/tmp/genesis.json"
    done
fi

# 启动 reth
echo "Starting reth with args: ${RETH_ARGS[@]}"
exec reth node "${RETH_ARGS[@]}"
```

### AF.4 测试套件实现 (Rust)

```rust
// Hive 测试套件的 Rust 实现
use hive_rs::{Simulation, TestSuite, TestSpec, Client};

/// Engine API 测试套件
pub struct EngineApiTestSuite;

impl TestSuite for EngineApiTestSuite {
    fn name(&self) -> &'static str {
        "engine-api"
    }

    fn description(&self) -> &'static str {
        "Engine API conformance tests for execution clients"
    }

    fn tests(&self) -> Vec<Box<dyn TestSpec>> {
        vec![
            Box::new(TestForkchoiceUpdated),
            Box::new(TestNewPayload),
            Box::new(TestGetPayload),
            Box::new(TestExchangeCapabilities),
            Box::new(TestInvalidPayload),
        ]
    }
}

/// forkchoiceUpdated 测试
struct TestForkchoiceUpdated;

impl TestSpec for TestForkchoiceUpdated {
    fn name(&self) -> &'static str {
        "engine_forkchoiceUpdatedV3"
    }

    fn run(&self, sim: &Simulation, client: &Client) -> TestResult {
        // 构造 forkchoice 状态
        let forkchoice_state = ForkchoiceState {
            head_block_hash: client.genesis_hash(),
            safe_block_hash: client.genesis_hash(),
            finalized_block_hash: B256::ZERO,
        };

        // 构造 payload 属性
        let payload_attrs = PayloadAttributes {
            timestamp: 12,
            prev_randao: B256::random(),
            suggested_fee_recipient: Address::random(),
            withdrawals: Some(vec![]),
            parent_beacon_block_root: Some(B256::random()),
        };

        // 调用 engine_forkchoiceUpdatedV3
        let response = client.engine_forkchoice_updated_v3(
            forkchoice_state,
            Some(payload_attrs),
        )?;

        // 验证响应
        assert_eq!(response.payload_status.status, PayloadStatusEnum::Valid);
        assert!(response.payload_id.is_some());

        TestResult::Pass
    }
}

/// newPayload 测试
struct TestNewPayload;

impl TestSpec for TestNewPayload {
    fn name(&self) -> &'static str {
        "engine_newPayloadV3"
    }

    fn run(&self, sim: &Simulation, client: &Client) -> TestResult {
        // 首先获取 payload
        let payload_id = self.get_payload_id(client)?;
        let payload = client.engine_get_payload_v3(payload_id)?;

        // 提交新 payload
        let response = client.engine_new_payload_v3(
            payload.execution_payload,
            payload.blobs_bundle.commitments.clone(),
            payload.parent_beacon_block_root,
        )?;

        // 验证
        match response.status {
            PayloadStatusEnum::Valid | PayloadStatusEnum::Accepted => {
                TestResult::Pass
            }
            _ => TestResult::Fail(format!(
                "Unexpected status: {:?}",
                response.status
            )),
        }
    }
}

/// 无效 payload 测试
struct TestInvalidPayload;

impl TestSpec for TestInvalidPayload {
    fn name(&self) -> &'static str {
        "engine_newPayloadV3_invalid"
    }

    fn run(&self, sim: &Simulation, client: &Client) -> TestResult {
        // 构造无效 payload (错误的 state_root)
        let mut payload = self.build_valid_payload(client)?;
        payload.state_root = B256::random(); // 错误的状态根

        let response = client.engine_new_payload_v3(
            payload,
            vec![],
            B256::random(),
        )?;

        // 应该返回 INVALID
        match response.status {
            PayloadStatusEnum::Invalid => TestResult::Pass,
            _ => TestResult::Fail(format!(
                "Expected INVALID, got: {:?}",
                response.status
            )),
        }
    }
}

/// RPC 测试套件
pub struct RpcTestSuite;

impl TestSuite for RpcTestSuite {
    fn name(&self) -> &'static str {
        "rpc-compat"
    }

    fn tests(&self) -> Vec<Box<dyn TestSpec>> {
        vec![
            Box::new(TestEthBlockNumber),
            Box::new(TestEthGetBalance),
            Box::new(TestEthCall),
            Box::new(TestEthEstimateGas),
            Box::new(TestEthGetTransactionReceipt),
            Box::new(TestDebugTraceCall),
        ]
    }
}

struct TestEthCall;

impl TestSpec for TestEthCall {
    fn name(&self) -> &'static str {
        "eth_call"
    }

    fn run(&self, sim: &Simulation, client: &Client) -> TestResult {
        // 部署测试合约
        let contract_address = sim.deploy_contract(
            client,
            include_bytes!("../contracts/test.bin"),
        )?;

        // 调用合约
        let call = TransactionRequest {
            to: Some(contract_address),
            data: Some(encode_function_call("getValue()")),
            ..Default::default()
        };

        let result = client.eth_call(call, BlockId::Latest)?;

        // 验证返回值
        let expected = U256::from(42).to_be_bytes::<32>();
        if result.as_ref() == expected {
            TestResult::Pass
        } else {
            TestResult::Fail(format!(
                "Expected {:?}, got {:?}",
                expected, result
            ))
        }
    }
}
```

### AF.5 测试结果处理

```rust
use serde::{Serialize, Deserialize};

/// Hive 测试结果
#[derive(Debug, Serialize, Deserialize)]
pub struct HiveResults {
    pub name: String,
    pub description: String,
    #[serde(rename = "NClients")]
    pub n_clients: usize,
    #[serde(rename = "NTests")]
    pub n_tests: usize,
    #[serde(rename = "NPass")]
    pub n_pass: usize,
    #[serde(rename = "NFail")]
    pub n_fail: usize,
    pub clients: HashMap<String, ClientInfo>,
    pub test_cases: Vec<TestCaseResult>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ClientInfo {
    pub name: String,
    pub version: String,
    pub image: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct TestCaseResult {
    pub name: String,
    pub description: String,
    pub client: String,
    pub pass: bool,
    pub details: String,
    pub duration: f64,
    pub logs: Vec<String>,
}

/// 结果分析器
pub struct ResultAnalyzer {
    results: HiveResults,
}

impl ResultAnalyzer {
    pub fn load(path: &Path) -> Result<Self, Error> {
        let content = std::fs::read_to_string(path)?;
        let results: HiveResults = serde_json::from_str(&content)?;
        Ok(Self { results })
    }

    /// 按客户端统计结果
    pub fn client_summary(&self) -> HashMap<String, ClientSummary> {
        let mut summaries: HashMap<String, ClientSummary> = HashMap::new();

        for test in &self.results.test_cases {
            let summary = summaries.entry(test.client.clone()).or_default();
            summary.total += 1;
            if test.pass {
                summary.passed += 1;
            } else {
                summary.failed += 1;
                summary.failed_tests.push(test.name.clone());
            }
            summary.total_duration += test.duration;
        }

        summaries
    }

    /// 生成兼容性矩阵
    pub fn compatibility_matrix(&self) -> CompatibilityMatrix {
        let clients: Vec<String> = self.results.clients.keys().cloned().collect();
        let tests: Vec<String> = self.results.test_cases.iter()
            .map(|t| t.name.clone())
            .collect::<std::collections::HashSet<_>>()
            .into_iter()
            .collect();

        let mut matrix = vec![vec![TestStatus::Unknown; clients.len()]; tests.len()];

        for test in &self.results.test_cases {
            if let Some(test_idx) = tests.iter().position(|t| t == &test.name) {
                if let Some(client_idx) = clients.iter().position(|c| c == &test.client) {
                    matrix[test_idx][client_idx] = if test.pass {
                        TestStatus::Pass
                    } else {
                        TestStatus::Fail
                    };
                }
            }
        }

        CompatibilityMatrix { clients, tests, matrix }
    }

    /// 检测回归
    pub fn detect_regressions(&self, baseline: &HiveResults) -> Vec<Regression> {
        let mut regressions = Vec::new();

        let baseline_map: HashMap<(String, String), bool> = baseline.test_cases.iter()
            .map(|t| ((t.name.clone(), t.client.clone()), t.pass))
            .collect();

        for test in &self.results.test_cases {
            let key = (test.name.clone(), test.client.clone());
            if let Some(&baseline_pass) = baseline_map.get(&key) {
                if baseline_pass && !test.pass {
                    regressions.push(Regression {
                        test: test.name.clone(),
                        client: test.client.clone(),
                        details: test.details.clone(),
                    });
                }
            }
        }

        regressions
    }
}

#[derive(Debug, Default)]
pub struct ClientSummary {
    pub total: usize,
    pub passed: usize,
    pub failed: usize,
    pub failed_tests: Vec<String>,
    pub total_duration: f64,
}

#[derive(Debug)]
pub struct CompatibilityMatrix {
    pub clients: Vec<String>,
    pub tests: Vec<String>,
    pub matrix: Vec<Vec<TestStatus>>,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum TestStatus {
    Pass,
    Fail,
    Unknown,
}

#[derive(Debug)]
pub struct Regression {
    pub test: String,
    pub client: String,
    pub details: String,
}

impl CompatibilityMatrix {
    /// 渲染为 Markdown 表格
    pub fn to_markdown(&self) -> String {
        let mut md = String::new();

        // 表头
        md.push_str("| Test |");
        for client in &self.clients {
            md.push_str(&format!(" {} |", client));
        }
        md.push('\n');

        // 分隔符
        md.push_str("|------|");
        for _ in &self.clients {
            md.push_str("---|");
        }
        md.push('\n');

        // 数据行
        for (i, test) in self.tests.iter().enumerate() {
            md.push_str(&format!("| {} |", test));
            for (j, _) in self.clients.iter().enumerate() {
                let status = match self.matrix[i][j] {
                    TestStatus::Pass => "✅",
                    TestStatus::Fail => "❌",
                    TestStatus::Unknown => "❓",
                };
                md.push_str(&format!(" {} |", status));
            }
            md.push('\n');
        }

        md
    }
}
```

---

## 附录 AG: Solidity 代理合约与可升级合约

### AG.1 代理合约基本原理

代理合约（Proxy Contract）是实现合约可升级性的核心机制，其基础是 EVM 的 `DELEGATECALL` 操作码。

#### AG.1.1 DELEGATECALL 原理

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DELEGATECALL 执行模型                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   用户 ──────► Proxy (存储)                                          │
│                  │                                                   │
│                  │ DELEGATECALL                                      │
│                  ▼                                                   │
│             Implementation (逻辑)                                    │
│                  │                                                   │
│                  │ 在 Proxy 的上下文中执行                            │
│                  ▼                                                   │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  msg.sender = 原始调用者                                      │   │
│   │  msg.value  = 原始 value                                      │   │
│   │  address(this) = Proxy 地址                                   │   │
│   │  存储读写 → Proxy 的存储                                       │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 最小化代理合约示例
 * @notice 展示 DELEGATECALL 的基本工作原理
 */
contract MinimalProxy {
    // 实现合约地址存储在特定 slot
    address public implementation;

    constructor(address _implementation) {
        implementation = _implementation;
    }

    /**
     * @notice Fallback 函数 - 所有调用都会转发到实现合约
     * @dev 使用 DELEGATECALL 保持调用上下文
     */
    fallback() external payable {
        address impl = implementation;

        assembly {
            // 复制 calldata 到内存
            // calldatacopy(destOffset, offset, size)
            calldatacopy(0, 0, calldatasize())

            // 执行 delegatecall
            // delegatecall(gas, address, argsOffset, argsSize, retOffset, retSize)
            let result := delegatecall(
                gas(),           // 转发所有 gas
                impl,            // 目标实现合约
                0,               // 输入数据起始位置
                calldatasize(),  // 输入数据长度
                0,               // 输出数据起始位置（稍后复制）
                0                // 输出数据长度（未知）
            )

            // 获取返回数据大小
            let size := returndatasize()

            // 复制返回数据到内存
            // returndatacopy(destOffset, offset, size)
            returndatacopy(0, 0, size)

            // 根据结果返回或回滚
            switch result
            case 0 {
                // delegatecall 失败，回滚并返回错误数据
                revert(0, size)
            }
            default {
                // delegatecall 成功，返回数据
                return (0, size)
            }
        }
    }

    receive() external payable {}
}
```

#### AG.1.2 存储布局对齐

代理合约最关键的要求是存储布局必须兼容：

```solidity
/**
 * @title 存储布局示例
 * @notice 演示代理和实现合约的存储对齐要求
 */

// 实现合约 V1
contract ImplementationV1 {
    // slot 0: 被代理合约占用，必须预留
    address private _reserved;

    // slot 1: 实际业务数据开始
    uint256 public value;        // slot 1
    address public owner;        // slot 2
    mapping(address => uint256) public balances;  // slot 3 (mapping 基础位置)

    function setValue(uint256 _value) external {
        value = _value;
    }

    function setOwner(address _owner) external {
        owner = _owner;
    }
}

// 实现合约 V2 - 只能追加新变量
contract ImplementationV2 {
    // 必须保持与 V1 完全相同的布局
    address private _reserved;   // slot 0
    uint256 public value;        // slot 1
    address public owner;        // slot 2
    mapping(address => uint256) public balances;  // slot 3

    // V2 新增的变量只能追加在末尾
    uint256 public newValue;     // slot 4
    string public description;   // slot 5

    // ❌ 错误：不能在中间插入变量
    // uint256 public inserted;  // 这会破坏存储布局

    // ❌ 错误：不能修改现有变量类型
    // uint128 public value;     // 类型改变会导致数据损坏

    function setValue(uint256 _value) external {
        value = _value;
    }

    function setNewValue(uint256 _newValue) external {
        newValue = _newValue;
    }
}
```

### AG.2 透明代理模式 (Transparent Proxy)

OpenZeppelin 的透明代理是最早广泛使用的可升级模式：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      透明代理架构                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                    ┌──────────────────┐                              │
│                    │   ProxyAdmin     │                              │
│                    │  (管理员合约)     │                              │
│                    └────────┬─────────┘                              │
│                             │ upgrade/changeAdmin                    │
│                             ▼                                        │
│   用户 ──────────► TransparentProxy ◄───────── 管理员                │
│        │                   │                      │                  │
│        │                   │                      │                  │
│        │ 用户调用          │                      │ 管理调用          │
│        │ → delegatecall   │                      │ → 直接执行        │
│        │                   ▼                      │                  │
│        │           Implementation                 │                  │
│        │                                          │                  │
│        └──────────────────────────────────────────┘                  │
│                                                                      │
│   关键：管理员调用 proxy 函数，普通用户调用 implementation 函数        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 透明代理合约
 * @notice 管理员和用户的调用路径完全分离
 */
contract TransparentUpgradeableProxy {
    /**
     * @dev EIP-1967 定义的标准存储槽
     * bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)
     */
    bytes32 private constant IMPLEMENTATION_SLOT =
    0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    /**
     * @dev bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1)
     */
    bytes32 private constant ADMIN_SLOT =
    0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;

    /**
     * @dev 仅管理员可调用的修饰符
     */
    modifier ifAdmin() {
        if (msg.sender == _getAdmin()) {
            _;
        } else {
            _fallback();
        }
    }

    constructor(address _logic, address admin_, bytes memory _data) payable {
        _setAdmin(admin_);
        _setImplementation(_logic);
        if (_data.length > 0) {
            (bool success,) = _logic.delegatecall(_data);
            require(success, "Initialization failed");
        }
    }

    /**
     * @notice 获取当前实现地址
     * @dev 只有管理员可以调用此函数
     */
    function implementation() external ifAdmin returns (address) {
        return _getImplementation();
    }

    /**
     * @notice 获取当前管理员
     */
    function admin() external ifAdmin returns (address) {
        return _getAdmin();
    }

    /**
     * @notice 更换管理员
     * @param newAdmin 新管理员地址
     */
    function changeAdmin(address newAdmin) external ifAdmin {
        require(newAdmin != address(0), "New admin is zero address");
        emit AdminChanged(_getAdmin(), newAdmin);
        _setAdmin(newAdmin);
    }

    /**
     * @notice 升级到新实现
     * @param newImplementation 新实现合约地址
     */
    function upgradeTo(address newImplementation) external ifAdmin {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }

    /**
     * @notice 升级并调用初始化函数
     */
    function upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) external payable ifAdmin {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Upgrade call failed");
        }
    }

    // ============ 内部函数 ============

    function _getImplementation() internal view returns (address impl) {
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            impl := sload(slot)
        }
    }

    function _setImplementation(address newImplementation) internal {
        require(
            newImplementation.code.length > 0,
            "Implementation is not a contract"
        );
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            sstore(slot, newImplementation)
        }
    }

    function _getAdmin() internal view returns (address adm) {
        bytes32 slot = ADMIN_SLOT;
        assembly {
            adm := sload(slot)
        }
    }

    function _setAdmin(address newAdmin) internal {
        bytes32 slot = ADMIN_SLOT;
        assembly {
            sstore(slot, newAdmin)
        }
    }

    function _fallback() internal {
        address impl = _getImplementation();
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 {revert(0, returndatasize())}
            default {return (0, returndatasize())}
        }
    }

    fallback() external payable {
        _fallback();
    }

    receive() external payable {
        _fallback();
    }

    event Upgraded(address indexed implementation);
    event AdminChanged(address previousAdmin, address newAdmin);
}

/**
 * @title 代理管理合约
 * @notice 独立的管理合约，避免函数选择器冲突
 */
contract ProxyAdmin {
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function getProxyImplementation(
        TransparentUpgradeableProxy proxy
    ) external view returns (address) {
        // 静态调用获取实现地址
        (bool success, bytes memory returndata) = address(proxy).staticcall(
            abi.encodeWithSelector(proxy.implementation.selector)
        );
        require(success, "Call failed");
        return abi.decode(returndata, (address));
    }

    function getProxyAdmin(
        TransparentUpgradeableProxy proxy
    ) external view returns (address) {
        (bool success, bytes memory returndata) = address(proxy).staticcall(
            abi.encodeWithSelector(proxy.admin.selector)
        );
        require(success, "Call failed");
        return abi.decode(returndata, (address));
    }

    function upgrade(
        TransparentUpgradeableProxy proxy,
        address implementation
    ) external onlyOwner {
        proxy.upgradeTo(implementation);
    }

    function upgradeAndCall(
        TransparentUpgradeableProxy proxy,
        address implementation,
        bytes memory data
    ) external payable onlyOwner {
        proxy.upgradeToAndCall{value: msg.value}(implementation, data);
    }
}
```

### AG.3 UUPS 代理模式 (Universal Upgradeable Proxy Standard)

UUPS (EIP-1822) 将升级逻辑放在实现合约中，更加 gas 高效：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        UUPS 代理架构                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                     ERC1967Proxy                             │   │
│   │  ┌─────────────────────────────────────────────────────┐    │   │
│   │  │  storage slot 0x360894...bbc: implementation addr   │    │   │
│   │  │  (其他业务数据存储)                                   │    │   │
│   │  └─────────────────────────────────────────────────────┘    │   │
│   │                          │                                   │   │
│   │                          │ delegatecall                      │   │
│   │                          ▼                                   │   │
│   │  ┌─────────────────────────────────────────────────────┐    │   │
│   │  │         UUPSUpgradeable Implementation              │    │   │
│   │  │  ┌───────────────────────────────────────────────┐  │    │   │
│   │  │  │  upgradeTo()     ◄── 升级逻辑在实现合约中     │  │    │   │
│   │  │  │  upgradeToAndCall()                           │  │    │   │
│   │  │  │  _authorizeUpgrade() ◄── 权限检查             │  │    │   │
│   │  │  └───────────────────────────────────────────────┘  │    │   │
│   │  │  + 业务逻辑函数                                      │    │   │
│   │  └─────────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   优点：代理合约简单，部署 gas 低                                     │
│   缺点：升级逻辑错误可能导致合约永久锁定                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title ERC1967 代理合约
 * @notice 最简化的代理，只负责转发调用
 */
contract ERC1967Proxy {
    bytes32 private constant IMPLEMENTATION_SLOT =
    0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    constructor(address _logic, bytes memory _data) payable {
        require(_logic.code.length > 0, "Not a contract");
        assembly {
            sstore(IMPLEMENTATION_SLOT, _logic)
        }
        if (_data.length > 0) {
            (bool success,) = _logic.delegatecall(_data);
            require(success, "Initialization failed");
        }
    }

    fallback() external payable {
        assembly {
            let impl := sload(IMPLEMENTATION_SLOT)
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 {revert(0, returndatasize())}
            default {return (0, returndatasize())}
        }
    }

    receive() external payable {}
}

/**
 * @title UUPS 可升级基类
 * @notice 实现合约需要继承此合约
 */
abstract contract UUPSUpgradeable {
    bytes32 private constant IMPLEMENTATION_SLOT =
    0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    /**
     * @dev 用于防止实现合约被直接初始化
     */
    address private immutable __self = address(this);

    /**
     * @dev 确保只能通过代理调用
     */
    modifier onlyProxy() {
        require(address(this) != __self, "Must be called through proxy");
        require(_getImplementation() == __self, "Must be called through active proxy");
        _;
    }

    /**
     * @dev 确保不是通过代理调用（用于禁止某些操作）
     */
    modifier notDelegated() {
        require(address(this) == __self, "Must not be called through proxy");
        _;
    }

    /**
     * @notice 升级到新实现
     * @param newImplementation 新实现地址
     */
    function upgradeTo(address newImplementation) external virtual onlyProxy {
        _authorizeUpgrade(newImplementation);
        _upgradeToAndCallUUPS(newImplementation, new bytes(0));
    }

    /**
     * @notice 升级并调用
     */
    function upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) external payable virtual onlyProxy {
        _authorizeUpgrade(newImplementation);
        _upgradeToAndCallUUPS(newImplementation, data);
    }

    /**
     * @dev 子类必须实现的权限检查函数
     */
    function _authorizeUpgrade(address newImplementation) internal virtual;

    /**
     * @dev 返回实现地址
     */
    function _getImplementation() internal view returns (address impl) {
        assembly {
            impl := sload(IMPLEMENTATION_SLOT)
        }
    }

    /**
     * @dev 执行 UUPS 升级
     */
    function _upgradeToAndCallUUPS(
        address newImplementation,
        bytes memory data
    ) internal {
        // 验证新实现支持 UUPS
        try UUPSUpgradeable(newImplementation).proxiableUUID() returns (bytes32 slot) {
            require(slot == IMPLEMENTATION_SLOT, "Unsupported proxiableUUID");
        } catch {
            revert("New implementation is not UUPS");
        }

        // 更新实现地址
        assembly {
            sstore(IMPLEMENTATION_SLOT, newImplementation)
        }
        emit Upgraded(newImplementation);

        // 可选的初始化调用
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Upgrade call failed");
        }
    }

    /**
     * @dev UUPS 标识符，用于验证合约支持 UUPS
     */
    function proxiableUUID() external view virtual notDelegated returns (bytes32) {
        return IMPLEMENTATION_SLOT;
    }

    event Upgraded(address indexed implementation);
}

/**
 * @title 可初始化合约基类
 * @notice 替代 constructor，用于代理模式
 */
abstract contract Initializable {
    uint8 private _initialized;
    bool private _initializing;

    modifier initializer() {
        bool isTopLevelCall = !_initializing;
        require(
            (isTopLevelCall && _initialized < 1) ||
            (address(this).code.length == 0 && _initialized == 1),
            "Already initialized"
        );
        _initialized = 1;
        if (isTopLevelCall) {
            _initializing = true;
        }
        _;
        if (isTopLevelCall) {
            _initializing = false;
        }
    }

    modifier reinitializer(uint8 version) {
        require(!_initializing && _initialized < version, "Already initialized");
        _initialized = version;
        _initializing = true;
        _;
        _initializing = false;
    }

    /**
     * @dev 禁止再次初始化（用于实现合约的 constructor）
     */
    function _disableInitializers() internal virtual {
        require(!_initializing, "Currently initializing");
        if (_initialized < type(uint8).max) {
            _initialized = type(uint8).max;
        }
    }
}

/**
 * @title UUPS 实现合约示例
 */
contract MyContractV1 is UUPSUpgradeable, Initializable {
    address public owner;
    uint256 public value;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(address _owner) external initializer {
        owner = _owner;
    }

    function setValue(uint256 _value) external {
        require(msg.sender == owner, "Not owner");
        value = _value;
    }

    function _authorizeUpgrade(address) internal override view {
        require(msg.sender == owner, "Not owner");
    }

    function version() external pure virtual returns (string memory) {
        return "1.0.0";
    }
}

contract MyContractV2 is UUPSUpgradeable, Initializable {
    address public owner;
    uint256 public value;
    // 新增变量
    string public name;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(address _owner) external initializer {
        owner = _owner;
    }

    // V2 特有的重新初始化函数
    function initializeV2(string memory _name) external reinitializer(2) {
        name = _name;
    }

    function setValue(uint256 _value) external {
        require(msg.sender == owner, "Not owner");
        value = _value;
    }

    function _authorizeUpgrade(address) internal override view {
        require(msg.sender == owner, "Not owner");
    }

    function version() external pure virtual returns (string memory) {
        return "2.0.0";
    }
}
```

### AG.4 Beacon 代理模式

Beacon 代理适用于需要同时升级多个代理的场景：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Beacon 代理架构                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                    ┌──────────────────┐                              │
│                    │  UpgradeableBeacon│                             │
│                    │  implementation() │                             │
│                    │  upgradeTo()      │                             │
│                    └────────┬─────────┘                              │
│                             │                                        │
│         ┌───────────────────┼───────────────────┐                    │
│         │                   │                   │                    │
│         ▼                   ▼                   ▼                    │
│   ┌──────────┐       ┌──────────┐       ┌──────────┐                │
│   │ Beacon   │       │ Beacon   │       │ Beacon   │                │
│   │ Proxy 1  │       │ Proxy 2  │       │ Proxy 3  │                │
│   │(用户 A)  │       │(用户 B)  │       │(用户 C)  │                │
│   └──────────┘       └──────────┘       └──────────┘                │
│         │                   │                   │                    │
│         └───────────────────┼───────────────────┘                    │
│                             │                                        │
│                             ▼                                        │
│                    ┌──────────────────┐                              │
│                    │  Implementation  │                              │
│                    │     Contract     │                              │
│                    └──────────────────┘                              │
│                                                                      │
│   优点：一次升级 Beacon，所有代理自动更新                             │
│   场景：工厂模式创建的多个相同类型合约                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title Beacon 接口
 */
interface IBeacon {
    function implementation() external view returns (address);
}

/**
 * @title 可升级 Beacon
 * @notice 管理单个实现地址，供多个代理使用
 */
contract UpgradeableBeacon is IBeacon {
    address private _implementation;
    address public owner;

    event Upgraded(address indexed implementation);

    constructor(address implementation_, address owner_) {
        _setImplementation(implementation_);
        owner = owner_;
    }

    function implementation() public view override returns (address) {
        return _implementation;
    }

    function upgradeTo(address newImplementation) external {
        require(msg.sender == owner, "Not owner");
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }

    function _setImplementation(address newImplementation) private {
        require(newImplementation.code.length > 0, "Not a contract");
        _implementation = newImplementation;
    }
}

/**
 * @title Beacon 代理
 * @notice 从 Beacon 获取实现地址
 */
contract BeaconProxy {
    bytes32 private constant BEACON_SLOT =
    0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50;

    constructor(address beacon, bytes memory data) payable {
        require(beacon.code.length > 0, "Beacon is not a contract");
        assembly {
            sstore(BEACON_SLOT, beacon)
        }

        address impl = IBeacon(beacon).implementation();
        require(impl.code.length > 0, "Implementation is not a contract");

        if (data.length > 0) {
            (bool success,) = impl.delegatecall(data);
            require(success, "Initialization failed");
        }
    }

    function _beacon() internal view returns (address beacon) {
        assembly {
            beacon := sload(BEACON_SLOT)
        }
    }

    function _implementation() internal view returns (address) {
        return IBeacon(_beacon()).implementation();
    }

    fallback() external payable {
        address impl = _implementation();
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 {revert(0, returndatasize())}
            default {return (0, returndatasize())}
        }
    }

    receive() external payable {}
}

/**
 * @title Beacon 代理工厂
 * @notice 创建共享同一 Beacon 的多个代理
 */
contract BeaconProxyFactory {
    UpgradeableBeacon public immutable beacon;
    address[] public proxies;

    event ProxyCreated(address proxy, address indexed creator);

    constructor(address implementation) {
        beacon = new UpgradeableBeacon(implementation, msg.sender);
    }

    function createProxy(bytes memory initData) external returns (address) {
        BeaconProxy proxy = new BeaconProxy(address(beacon), initData);
        proxies.push(address(proxy));
        emit ProxyCreated(address(proxy), msg.sender);
        return address(proxy);
    }

    function upgradeImplementation(address newImplementation) external {
        // 这会自动升级所有使用此 Beacon 的代理
        beacon.upgradeTo(newImplementation);
    }

    function getProxyCount() external view returns (uint256) {
        return proxies.length;
    }
}
```

### AG.5 Diamond 代理模式 (EIP-2535)

Diamond 模式支持多个实现合约（facets），突破合约大小限制：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Diamond 代理架构                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                      Diamond Proxy                           │   │
│   │  ┌───────────────────────────────────────────────────────┐  │   │
│   │  │               Function Selector Mapping               │  │   │
│   │  │  ┌─────────────┬──────────────┬─────────────────┐    │  │   │
│   │  │  │  0xa9059cbb │ 0x23b872dd   │ 0x70a08231      │    │  │   │
│   │  │  │  transfer() │ transferFrom │ balanceOf()     │    │  │   │
│   │  │  │      │      │      │       │      │          │    │  │   │
│   │  │  │      ▼      │      ▼       │      ▼          │    │  │   │
│   │  │  │  Facet A    │  Facet A     │  Facet A        │    │  │   │
│   │  │  └─────────────┴──────────────┴─────────────────┘    │  │   │
│   │  │  ┌─────────────┬──────────────┐                      │  │   │
│   │  │  │  0x12345678 │ 0xabcdef12   │                      │  │   │
│   │  │  │  stake()    │ unstake()    │                      │  │   │
│   │  │  │      │      │      │       │                      │  │   │
│   │  │  │      ▼      │      ▼       │                      │  │   │
│   │  │  │  Facet B    │  Facet B     │                      │  │   │
│   │  │  └─────────────┴──────────────┘                      │  │   │
│   │  └───────────────────────────────────────────────────────┘  │   │
│   │                                                              │   │
│   │  ┌───────────────────────────────────────────────────────┐  │   │
│   │  │                 Shared Storage                        │  │   │
│   │  │  (所有 Facets 共享 Diamond 的存储空间)                 │  │   │
│   │  └───────────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐     │
│   │ Facet A  │    │ Facet B  │    │ Facet C  │    │ Facet D  │     │
│   │ (ERC20)  │    │(Staking) │    │(Governance)   │(Loupe)   │     │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title Diamond 存储库
 * @notice 定义 Diamond 的共享存储结构
 */
library LibDiamond {
    bytes32 constant DIAMOND_STORAGE_POSITION =
    keccak256("diamond.standard.diamond.storage");

    struct FacetAddressAndPosition {
        address facetAddress;
        uint96 functionSelectorPosition;  // 在 selectors 数组中的位置
    }

    struct FacetFunctionSelectors {
        bytes4[] functionSelectors;
        uint256 facetAddressPosition;  // 在 facetAddresses 数组中的位置
    }

    struct DiamondStorage {
        // selector => facet address and position
        mapping(bytes4 => FacetAddressAndPosition) selectorToFacetAndPosition;
        // facet address => selectors
        mapping(address => FacetFunctionSelectors) facetFunctionSelectors;
        // facet addresses
        address[] facetAddresses;
        // owner
        address contractOwner;
    }

    function diamondStorage() internal pure returns (DiamondStorage storage ds) {
        bytes32 position = DIAMOND_STORAGE_POSITION;
        assembly {
            ds.slot := position
        }
    }

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    event DiamondCut(FacetCut[] _diamondCut, address _init, bytes _calldata);

    enum FacetCutAction {Add, Replace, Remove}

    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }

    function setContractOwner(address _newOwner) internal {
        DiamondStorage storage ds = diamondStorage();
        address previousOwner = ds.contractOwner;
        ds.contractOwner = _newOwner;
        emit OwnershipTransferred(previousOwner, _newOwner);
    }

    function contractOwner() internal view returns (address) {
        return diamondStorage().contractOwner;
    }

    function enforceIsContractOwner() internal view {
        require(msg.sender == diamondStorage().contractOwner, "Not owner");
    }

    /**
     * @notice 执行 Diamond Cut
     * @param _diamondCut Facet 变更数组
     * @param _init 初始化合约地址
     * @param _calldata 初始化调用数据
     */
    function diamondCut(
        FacetCut[] memory _diamondCut,
        address _init,
        bytes memory _calldata
    ) internal {
        for (uint256 i; i < _diamondCut.length; i++) {
            FacetCutAction action = _diamondCut[i].action;
            if (action == FacetCutAction.Add) {
                addFunctions(_diamondCut[i].facetAddress, _diamondCut[i].functionSelectors);
            } else if (action == FacetCutAction.Replace) {
                replaceFunctions(_diamondCut[i].facetAddress, _diamondCut[i].functionSelectors);
            } else if (action == FacetCutAction.Remove) {
                removeFunctions(_diamondCut[i].facetAddress, _diamondCut[i].functionSelectors);
            }
        }
        emit DiamondCut(_diamondCut, _init, _calldata);
        initializeDiamondCut(_init, _calldata);
    }

    function addFunctions(address _facetAddress, bytes4[] memory _selectors) internal {
        require(_selectors.length > 0, "No selectors");
        DiamondStorage storage ds = diamondStorage();
        require(_facetAddress != address(0), "Zero address facet");

        uint96 selectorPosition = uint96(ds.facetFunctionSelectors[_facetAddress].functionSelectors.length);

        // 如果是新 facet，添加到列表
        if (selectorPosition == 0) {
            addFacet(ds, _facetAddress);
        }

        for (uint256 i; i < _selectors.length; i++) {
            bytes4 selector = _selectors[i];
            address oldFacet = ds.selectorToFacetAndPosition[selector].facetAddress;
            require(oldFacet == address(0), "Selector already exists");

            ds.selectorToFacetAndPosition[selector] = FacetAddressAndPosition({
                facetAddress: _facetAddress,
                functionSelectorPosition: selectorPosition
            });
            ds.facetFunctionSelectors[_facetAddress].functionSelectors.push(selector);
            selectorPosition++;
        }
    }

    function replaceFunctions(address _facetAddress, bytes4[] memory _selectors) internal {
        require(_selectors.length > 0, "No selectors");
        DiamondStorage storage ds = diamondStorage();
        require(_facetAddress != address(0), "Zero address facet");

        uint96 selectorPosition = uint96(ds.facetFunctionSelectors[_facetAddress].functionSelectors.length);

        if (selectorPosition == 0) {
            addFacet(ds, _facetAddress);
        }

        for (uint256 i; i < _selectors.length; i++) {
            bytes4 selector = _selectors[i];
            address oldFacet = ds.selectorToFacetAndPosition[selector].facetAddress;
            require(oldFacet != address(0), "Selector doesn't exist");
            require(oldFacet != _facetAddress, "Same facet");

            removeFunction(ds, oldFacet, selector);

            ds.selectorToFacetAndPosition[selector] = FacetAddressAndPosition({
                facetAddress: _facetAddress,
                functionSelectorPosition: selectorPosition
            });
            ds.facetFunctionSelectors[_facetAddress].functionSelectors.push(selector);
            selectorPosition++;
        }
    }

    function removeFunctions(address _facetAddress, bytes4[] memory _selectors) internal {
        require(_selectors.length > 0, "No selectors");
        DiamondStorage storage ds = diamondStorage();
        require(_facetAddress == address(0), "Remove facet must be zero");

        for (uint256 i; i < _selectors.length; i++) {
            bytes4 selector = _selectors[i];
            address oldFacet = ds.selectorToFacetAndPosition[selector].facetAddress;
            require(oldFacet != address(0), "Selector doesn't exist");
            removeFunction(ds, oldFacet, selector);
        }
    }

    function addFacet(DiamondStorage storage ds, address _facetAddress) internal {
        ds.facetFunctionSelectors[_facetAddress].facetAddressPosition = ds.facetAddresses.length;
        ds.facetAddresses.push(_facetAddress);
    }

    function removeFunction(
        DiamondStorage storage ds,
        address _facetAddress,
        bytes4 _selector
    ) internal {
        // 从 facet 的 selectors 数组中移除
        uint256 selectorPosition = ds.selectorToFacetAndPosition[_selector].functionSelectorPosition;
        uint256 lastSelectorPosition = ds.facetFunctionSelectors[_facetAddress].functionSelectors.length - 1;

        if (selectorPosition != lastSelectorPosition) {
            bytes4 lastSelector = ds.facetFunctionSelectors[_facetAddress].functionSelectors[lastSelectorPosition];
            ds.facetFunctionSelectors[_facetAddress].functionSelectors[selectorPosition] = lastSelector;
            ds.selectorToFacetAndPosition[lastSelector].functionSelectorPosition = uint96(selectorPosition);
        }
        ds.facetFunctionSelectors[_facetAddress].functionSelectors.pop();
        delete ds.selectorToFacetAndPosition[_selector];

        // 如果 facet 没有 selectors 了，移除它
        if (lastSelectorPosition == 0) {
            uint256 facetPosition = ds.facetFunctionSelectors[_facetAddress].facetAddressPosition;
            uint256 lastFacetPosition = ds.facetAddresses.length - 1;

            if (facetPosition != lastFacetPosition) {
                address lastFacet = ds.facetAddresses[lastFacetPosition];
                ds.facetAddresses[facetPosition] = lastFacet;
                ds.facetFunctionSelectors[lastFacet].facetAddressPosition = facetPosition;
            }
            ds.facetAddresses.pop();
            delete ds.facetFunctionSelectors[_facetAddress];
        }
    }

    function initializeDiamondCut(address _init, bytes memory _calldata) internal {
        if (_init == address(0)) return;
        require(_init.code.length > 0, "Init is not a contract");
        (bool success, bytes memory error) = _init.delegatecall(_calldata);
        if (!success) {
            if (error.length > 0) {
                assembly {
                    revert(add(32, error), mload(error))
                }
            } else {
                revert("Init failed");
            }
        }
    }
}

/**
 * @title Diamond 代理主合约
 */
contract Diamond {
    constructor(address _owner, address _diamondCutFacet) payable {
        LibDiamond.setContractOwner(_owner);

        // 添加 diamondCut 函数
        LibDiamond.FacetCut[] memory cut = new LibDiamond.FacetCut[](1);
        bytes4[] memory selectors = new bytes4[](1);
        selectors[0] = IDiamondCut.diamondCut.selector;
        cut[0] = LibDiamond.FacetCut({
            facetAddress: _diamondCutFacet,
            action: LibDiamond.FacetCutAction.Add,
            functionSelectors: selectors
        });
        LibDiamond.diamondCut(cut, address(0), "");
    }

    fallback() external payable {
        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;
        require(facet != address(0), "Function not found");

        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 {revert(0, returndatasize())}
            default {return (0, returndatasize())}
        }
    }

    receive() external payable {}
}

interface IDiamondCut {
    function diamondCut(
        LibDiamond.FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;
}

/**
 * @title DiamondCut Facet
 */
contract DiamondCutFacet is IDiamondCut {
    function diamondCut(
        LibDiamond.FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external override {
        LibDiamond.enforceIsContractOwner();
        LibDiamond.diamondCut(_diamondCut, _init, _calldata);
    }
}

/**
 * @title DiamondLoupe Facet
 * @notice 提供查询 Diamond 结构的标准接口
 */
contract DiamondLoupeFacet {
    function facets() external view returns (Facet[] memory facets_) {
        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        uint256 numFacets = ds.facetAddresses.length;
        facets_ = new Facet[](numFacets);
        for (uint256 i; i < numFacets; i++) {
            address facetAddr = ds.facetAddresses[i];
            facets_[i].facetAddress = facetAddr;
            facets_[i].functionSelectors = ds.facetFunctionSelectors[facetAddr].functionSelectors;
        }
    }

    function facetFunctionSelectors(address _facet) external view returns (bytes4[] memory) {
        return LibDiamond.diamondStorage().facetFunctionSelectors[_facet].functionSelectors;
    }

    function facetAddresses() external view returns (address[] memory) {
        return LibDiamond.diamondStorage().facetAddresses;
    }

    function facetAddress(bytes4 _selector) external view returns (address) {
        return LibDiamond.diamondStorage().selectorToFacetAndPosition[_selector].facetAddress;
    }

    struct Facet {
        address facetAddress;
        bytes4[] functionSelectors;
    }
}
```

### AG.6 存储冲突与解决方案

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 存储冲突演示
 * @notice 展示不正确的存储布局导致的问题
 */

// ❌ 错误示例：存储冲突
contract ProxyBad {
    address public implementation;  // slot 0
    address public admin;           // slot 1
}

contract ImplementationBad {
    uint256 public value;           // slot 0 - 与 proxy.implementation 冲突！
    address public owner;           // slot 1 - 与 proxy.admin 冲突！
}

// ✅ 正确方案 1：使用 EIP-1967 标准存储槽
contract ProxyGoodEIP1967 {
    // 使用 keccak256 哈希派生的槽位，避免冲突
    bytes32 private constant IMPL_SLOT =
    0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    bytes32 private constant ADMIN_SLOT =
    0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;

    function _getImpl() internal view returns (address impl) {
        assembly {impl := sload(IMPL_SLOT)}
    }
}

// ✅ 正确方案 2：非结构化存储 (Unstructured Storage)
contract UnstructuredStorage {
    /**
     * @dev 使用 keccak256("my.app.storage.owner") 作为基础
     * 然后减 1 确保不会与 mapping 冲突
     */
    bytes32 private constant OWNER_SLOT =
    keccak256("my.app.storage.owner") - 1;

    function _setOwner(address owner) internal {
        assembly {
            sstore(OWNER_SLOT, owner)
        }
    }

    function _getOwner() internal view returns (address owner) {
        assembly {
            owner := sload(OWNER_SLOT)
        }
    }
}

// ✅ 正确方案 3：Diamond Storage Pattern
library AppStorage {
    bytes32 constant POSITION = keccak256("my.app.storage");

    struct Layout {
        address owner;
        uint256 value;
        mapping(address => uint256) balances;
        // 新变量追加在这里
    }

    function layout() internal pure returns (Layout storage l) {
        bytes32 position = POSITION;
        assembly {
            l.slot := position
        }
    }
}

contract FacetUsingDiamondStorage {
    function setValue(uint256 _value) external {
        AppStorage.layout().value = _value;
    }

    function getValue() external view returns (uint256) {
        return AppStorage.layout().value;
    }

    function getOwner() external view returns (address) {
        return AppStorage.layout().owner;
    }
}

// ✅ 正确方案 4：存储间隙 (Storage Gap)
contract UpgradeableV1 {
    address public owner;      // slot 0
    uint256 public value;      // slot 1

    /**
     * @dev 保留 50 个存储槽给未来升级使用
     * 这样添加新变量时不会影响继承合约的存储布局
     */
    uint256[48] private __gap;  // slots 2-49
}

contract UpgradeableV2 {
    address public owner;       // slot 0
    uint256 public value;       // slot 1
    string public name;         // slot 2 (新增)

    uint256[47] private __gap;  // slots 3-49 (减少一个)
}

// 继承时的存储间隙使用
contract BaseContract {
    address public owner;
    uint256[49] private __gap;
}

contract ChildContract is BaseContract {
    // 即使 BaseContract 升级添加新变量，
    // ChildContract 的存储布局也不会被破坏
    uint256 public childValue;
    uint256[49] private __gap;
}
```

### AG.7 最小代理合约 (Clone/EIP-1167)

最小代理是一种极其 gas 高效的代理模式，用于创建大量相同逻辑的合约实例：

```
┌─────────────────────────────────────────────────────────────────────┐
│                   EIP-1167 最小代理架构                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   字节码（仅 45 bytes）:                                             │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ 3d602d80600a3d3981f3363d3d373d3d3d363d73                    │   │
│   │ <20 bytes: implementation address>                          │   │
│   │ 5af43d82803e903d91602b57fd5bf3                              │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐           │
│   │  Clone 1     │   │  Clone 2     │   │  Clone 3     │           │
│   │  45 bytes    │   │  45 bytes    │   │  45 bytes    │           │
│   │  独立存储    │   │  独立存储    │   │  独立存储    │           │
│   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘           │
│          │                  │                  │                    │
│          └──────────────────┼──────────────────┘                    │
│                             │ delegatecall                          │
│                             ▼                                        │
│                    ┌──────────────────┐                              │
│                    │  Implementation  │                              │
│                    │  (共享逻辑)      │                              │
│                    └──────────────────┘                              │
│                                                                      │
│   优点：部署成本极低（~800 gas vs ~32000 gas）                       │
│   缺点：不可升级，运行时多一次 delegatecall 开销                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title EIP-1167 Clone 工厂
 * @notice 创建最小代理合约
 */
library Clones {
    /**
     * @notice 使用 CREATE 部署最小代理
     * @param implementation 实现合约地址
     * @return instance 新创建的代理地址
     */
    function clone(address implementation) internal returns (address instance) {
        assembly {
            // 将实现地址编码到字节码中
            // 0x3d602d80600a3d3981f3: 部署代码前缀
            // implementation address (20 bytes)
            // 0x5af43d82803e903d91602b57fd5bf3: 运行时代码后缀

            // 存储创建代码到内存 0x00
            mstore(0x00, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73)
            // 在 0x14 (20) 位置存储实现地址
            mstore(0x14, shl(0x60, implementation))
            // 在 0x28 (40) 位置存储后缀
            mstore(0x28, 0x5af43d82803e903d91602b57fd5bf3)

            // 使用 CREATE 部署
            // create(value, offset, size)
            instance := create(0, 0x0c, 0x37)  // 55 bytes
        }
        require(instance != address(0), "Clone failed");
    }

    /**
     * @notice 使用 CREATE2 部署确定性最小代理
     * @param implementation 实现合约地址
     * @param salt 用于地址计算的盐值
     */
    function cloneDeterministic(
        address implementation,
        bytes32 salt
    ) internal returns (address instance) {
        assembly {
            mstore(0x00, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73)
            mstore(0x14, shl(0x60, implementation))
            mstore(0x28, 0x5af43d82803e903d91602b57fd5bf3)

            // create2(value, offset, size, salt)
            instance := create2(0, 0x0c, 0x37, salt)
        }
        require(instance != address(0), "Clone failed");
    }

    /**
     * @notice 预测 CREATE2 部署的地址
     */
    function predictDeterministicAddress(
        address implementation,
        bytes32 salt,
        address deployer
    ) internal pure returns (address predicted) {
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73)
            mstore(add(ptr, 0x14), shl(0x60, implementation))
            mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf3)

            // keccak256(0xff ++ deployer ++ salt ++ keccak256(initCode))
            mstore(add(ptr, 0x37), keccak256(add(ptr, 0x0c), 0x37))
            mstore(add(ptr, 0x17), salt)
            mstore(add(ptr, 0x03), deployer)
            mstore(ptr, 0xff)

            predicted := keccak256(ptr, 0x57)
        }
    }
}

/**
 * @title Clone 工厂示例
 */
contract CloneFactory {
    using Clones for address;

    address public immutable implementation;
    address[] public clones;

    event CloneCreated(address indexed clone, address indexed creator);

    constructor(address _implementation) {
        implementation = _implementation;
    }

    /**
     * @notice 创建新的 Clone 实例
     */
    function createClone(bytes memory initData) external returns (address clone_) {
        clone_ = implementation.clone();
        clones.push(clone_);

        // 初始化
        if (initData.length > 0) {
            (bool success,) = clone_.call(initData);
            require(success, "Init failed");
        }

        emit CloneCreated(clone_, msg.sender);
    }

    /**
     * @notice 创建确定性地址的 Clone
     */
    function createCloneDeterministic(
        bytes32 salt,
        bytes memory initData
    ) external returns (address clone_) {
        clone_ = implementation.cloneDeterministic(salt);
        clones.push(clone_);

        if (initData.length > 0) {
            (bool success,) = clone_.call(initData);
            require(success, "Init failed");
        }

        emit CloneCreated(clone_, msg.sender);
    }

    /**
     * @notice 预测确定性 Clone 地址
     */
    function predictCloneAddress(bytes32 salt) external view returns (address) {
        return implementation.predictDeterministicAddress(salt, address(this));
    }
}

/**
 * @title 可克隆的实现合约
 */
contract CloneableVault {
    address public owner;
    bool private initialized;

    function initialize(address _owner) external {
        require(!initialized, "Already initialized");
        initialized = true;
        owner = _owner;
    }

    function deposit() external payable {
        // 存款逻辑
    }

    function withdraw(uint256 amount) external {
        require(msg.sender == owner, "Not owner");
        payable(owner).transfer(amount);
    }

    receive() external payable {}
}
```

### AG.8 代理模式对比与选型

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         代理模式对比                                        │
├──────────────┬──────────┬──────────┬──────────┬──────────┬────────────────┤
│    特性       │ 透明代理  │  UUPS    │  Beacon  │ Diamond  │  Clone/1167   │
├──────────────┼──────────┼──────────┼──────────┼──────────┼────────────────┤
│ 部署成本      │   高      │   中     │   中     │   高     │    极低       │
├──────────────┼──────────┼──────────┼──────────┼──────────┼────────────────┤
│ 运行成本      │   中      │   低     │   中     │   中     │    低         │
├──────────────┼──────────┼──────────┼──────────┼──────────┼────────────────┤
│ 升级灵活性    │   高      │   高     │   高     │  最高    │    无         │
├──────────────┼──────────┼──────────┼──────────┼──────────┼────────────────┤
│ 批量升级      │   否      │   否     │   是     │   否     │    否         │
├──────────────┼──────────┼──────────┼──────────┼──────────┼────────────────┤
│ 合约大小限制  │  24KB    │  24KB    │  24KB    │  无限制  │    24KB       │
├──────────────┼──────────┼──────────┼──────────┼──────────┼────────────────┤
│ 复杂度        │   中      │   低     │   中     │   高     │    低         │
├──────────────┼──────────┼──────────┼──────────┼──────────┼────────────────┤
│ 升级风险      │   中      │   高     │   中     │   中     │    N/A        │
├──────────────┼──────────┼──────────┼──────────┼──────────┼────────────────┤
│ 适用场景      │ 通用     │ 简单升级 │ 工厂模式 │ 大型dApp │  大量实例     │
└──────────────┴──────────┴──────────┴──────────┴──────────┴────────────────┘

选型建议:
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  需要单独管理升级？                                                  │
│       │                                                              │
│       ├─── 是 ──► 需要节省 gas？                                    │
│       │              │                                               │
│       │              ├─── 是 ──► UUPS                               │
│       │              │                                               │
│       │              └─── 否 ──► 合约大小 > 24KB？                   │
│       │                              │                               │
│       │                              ├─── 是 ──► Diamond             │
│       │                              │                               │
│       │                              └─── 否 ──► Transparent Proxy   │
│       │                                                              │
│       └─── 否 ──► 需要升级功能？                                    │
│                      │                                               │
│                      ├─── 是 ──► Beacon Proxy                       │
│                      │                                               │
│                      └─── 否 ──► Clone (EIP-1167)                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 升级安全检查库
 * @notice 提供升级前的安全验证
 */
library UpgradeChecker {
    /**
     * @notice 验证存储布局兼容性
     * @dev 通过检查关键存储槽的值是否保持一致
     */
    function validateStorageLayout(
        address proxy,
        address newImpl,
        bytes32[] memory criticalSlots
    ) internal view returns (bool) {
        for (uint256 i = 0; i < criticalSlots.length; i++) {
            bytes32 slot = criticalSlots[i];
            bytes32 proxyValue;
            bytes32 implValue;

            assembly {
                proxyValue := sload(slot)
            }

            // 新实现应该能正确读取现有数据
            // 这里只是示例，实际需要更复杂的验证
        }
        return true;
    }

    /**
     * @notice 验证新实现是否为有效合约
     */
    function validateImplementation(address impl) internal view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(impl)
        }
        return size > 0;
    }

    /**
     * @notice 验证 UUPS 兼容性
     */
    function validateUUPS(address impl) internal view returns (bool) {
        // 检查是否实现了 proxiableUUID
        (bool success, bytes memory data) = impl.staticcall(
            abi.encodeWithSignature("proxiableUUID()")
        );
        if (!success || data.length != 32) return false;

        bytes32 uuid = abi.decode(data, (bytes32));
        return uuid == 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    }
}
```

### AG.9 EVM 中代理调用的底层实现

```rust
// revm 中 DELEGATECALL 的实现原理

use revm::{
    primitives::{Address, Bytes, U256},
    interpreter::{CallContext, CallScheme, Gas, InstructionResult},
};

/// DELEGATECALL 指令处理
pub fn delegatecall_handler(
    interpreter: &mut Interpreter,
    host: &mut dyn Host,
) -> InstructionResult {
    // 从栈中弹出参数
    let gas_limit = interpreter.stack.pop()?;
    let target_address = Address::from_word(interpreter.stack.pop()?.into());
    let args_offset = interpreter.stack.pop()?;
    let args_size = interpreter.stack.pop()?;
    let ret_offset = interpreter.stack.pop()?;
    let ret_size = interpreter.stack.pop()?;

    // 获取输入数据
    let input = interpreter.memory.get_slice(
        args_offset.as_usize(),
        args_size.as_usize(),
    );

    // 构建调用上下文
    // 关键：delegatecall 保持原始的 caller 和 value
    let call_context = CallContext {
        // 代码从目标地址加载
        code_address: target_address,
        // 但执行上下文保持当前合约
        address: interpreter.contract.address,        // 保持当前地址
        caller: interpreter.contract.caller,          // 保持原始调用者
        apparent_value: interpreter.contract.value,   // 保持原始 value
    };

    // 执行调用
    let (result, gas_used, output) = host.call(
        CallScheme::DelegateCall,
        call_context,
        Gas::new(gas_limit.as_u64()),
        input.to_vec().into(),
    );

    // 将结果写入内存
    if let Some(output) = output {
        interpreter.memory.set_slice(
            ret_offset.as_usize(),
            &output[..ret_size.as_usize().min(output.len())],
        );
    }

    // 返回状态压栈
    interpreter.stack.push(if result.is_ok() { U256::ONE } else { U256::ZERO })?;

    // 扣除 gas
    interpreter.gas.consume(gas_used);

    InstructionResult::Continue
}

/// 代理合约存储访问示例
/// 展示 delegatecall 如何访问代理合约的存储
pub struct ProxyStorageExample {
    /// 代理合约地址
    proxy: Address,
    /// 实现合约地址
    implementation: Address,
}

impl ProxyStorageExample {
    /// 模拟 delegatecall 的存储访问
    pub fn simulate_storage_access(&self, host: &mut dyn Host, slot: U256) -> U256 {
        // 在 delegatecall 中：
        // - 代码来自 implementation
        // - 存储操作作用于 proxy

        // SLOAD 指令：从 proxy 的存储读取
        let value = host.sload(self.proxy, slot);

        // SSTORE 指令：写入 proxy 的存储
        // host.sstore(self.proxy, slot, new_value);

        value
    }
}

/// EIP-1967 存储槽计算
pub mod eip1967 {
    use revm::primitives::{keccak256, B256, U256};

    /// 计算 EIP-1967 实现槽
    pub fn implementation_slot() -> U256 {
        // keccak256("eip1967.proxy.implementation") - 1
        let hash = keccak256(b"eip1967.proxy.implementation");
        U256::from_be_bytes(hash.0) - U256::from(1)
    }

    /// 计算 EIP-1967 管理员槽
    pub fn admin_slot() -> U256 {
        // keccak256("eip1967.proxy.admin") - 1
        let hash = keccak256(b"eip1967.proxy.admin");
        U256::from_be_bytes(hash.0) - U256::from(1)
    }

    /// 计算 EIP-1967 beacon 槽
    pub fn beacon_slot() -> U256 {
        // keccak256("eip1967.proxy.beacon") - 1
        let hash = keccak256(b"eip1967.proxy.beacon");
        U256::from_be_bytes(hash.0) - U256::from(1)
    }

    /// 验证槽值（十六进制）
    /// implementation: 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
    /// admin:          0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103
    /// beacon:         0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50
}
```

### AG.10 安全最佳实践

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 代理合约安全最佳实践
 */
contract ProxySecurityBestPractices {

    // ============ 1. 初始化保护 ============

    /**
     * @dev 防止实现合约被直接初始化
     * 在实现合约的 constructor 中调用
     */
    bool private _initialized;
    bool private _initializing;

    modifier initializer() {
        require(!_initialized || _initializing, "Already initialized");
        bool isTopLevel = !_initializing;
        if (isTopLevel) {
            _initializing = true;
            _initialized = true;
        }
        _;
        if (isTopLevel) {
            _initializing = false;
        }
    }

    // ============ 2. 函数选择器冲突检查 ============

    /**
     * @notice 检查两个函数选择器是否冲突
     * @dev 在透明代理中，admin 函数和 implementation 函数不能有相同的选择器
     */
    function checkSelectorCollision(
        bytes4 selector1,
        bytes4 selector2
    ) external pure returns (bool) {
        return selector1 == selector2;
    }

    // 常见冲突示例:
    // "owner()" (0x8da5cb5b) 和某些实现函数可能冲突
    // 使用工具如 slither 或 foundry 检查冲突

    // ============ 3. 升级安全检查清单 ============

    /**
     * 升级前检查清单:
     *
     * □ 存储布局兼容性
     *   - 不能删除现有变量
     *   - 不能改变现有变量类型
     *   - 不能在中间插入新变量
     *   - 新变量只能追加在末尾
     *
     * □ 初始化函数
     *   - 新增变量需要 reinitializer
     *   - 不能重复调用 initializer
     *
     * □ 函数签名
     *   - 保持关键函数签名不变
     *   - 检查选择器冲突
     *
     * □ 权限控制
     *   - 升级函数有适当的访问控制
     *   - 多签/时间锁保护
     *
     * □ 测试
     *   - 升级脚本测试
     *   - 存储迁移测试
     *   - 回滚计划
     */

    // ============ 4. 时间锁保护升级 ============

    struct UpgradeProposal {
        address newImplementation;
        uint256 proposedAt;
        bool executed;
    }

    uint256 public constant UPGRADE_DELAY = 2 days;
    UpgradeProposal public pendingUpgrade;

    event UpgradeProposed(address indexed newImplementation, uint256 executeAfter);
    event UpgradeExecuted(address indexed newImplementation);
    event UpgradeCancelled();

    function proposeUpgrade(address newImplementation) external {
        // 只有 owner 可以提议
        require(newImplementation.code.length > 0, "Not a contract");

        pendingUpgrade = UpgradeProposal({
            newImplementation: newImplementation,
            proposedAt: block.timestamp,
            executed: false
        });

        emit UpgradeProposed(newImplementation, block.timestamp + UPGRADE_DELAY);
    }

    function executeUpgrade() external {
        require(pendingUpgrade.newImplementation != address(0), "No pending upgrade");
        require(!pendingUpgrade.executed, "Already executed");
        require(
            block.timestamp >= pendingUpgrade.proposedAt + UPGRADE_DELAY,
            "Delay not passed"
        );

        pendingUpgrade.executed = true;

        // 执行升级
        // _upgradeTo(pendingUpgrade.newImplementation);

        emit UpgradeExecuted(pendingUpgrade.newImplementation);
    }

    function cancelUpgrade() external {
        delete pendingUpgrade;
        emit UpgradeCancelled();
    }

    // ============ 5. 紧急暂停机制 ============

    bool public paused;

    modifier whenNotPaused() {
        require(!paused, "Contract paused");
        _;
    }

    function pause() external {
        paused = true;
    }

    function unpause() external {
        paused = false;
    }
}

/**
 * @title 代理漏洞示例与修复
 */
contract ProxyVulnerabilities {

// ============ 漏洞 1: 未初始化的实现合约 ============

// ❌ 错误: 任何人都可以初始化实现合约并获得控制权
contract VulnerableImpl {
    address public owner;

    function initialize(address _owner) external {
        owner = _owner;
    }
}

// ✅ 修复: 在 constructor 中禁用初始化
contract SecureImpl {
    address public owner;
    bool private _disabledInitializer;

    constructor() {
        _disabledInitializer = true;
    }

    function initialize(address _owner) external {
        require(!_disabledInitializer, "Disabled");
        owner = _owner;
    }
}

// ============ 漏洞 2: UUPS 升级函数未保护 ============

// ❌ 错误: 任何人都可以升级
contract VulnerableUUPS {
    function upgradeTo(address newImpl) external {
        // 没有权限检查!
        _upgradeTo(newImpl);
    }
}

// ✅ 修复: 添加权限检查
contract SecureUUPS {
    address public owner;

    function upgradeTo(address newImpl) external {
        require(msg.sender == owner, "Not owner");
        _upgradeTo(newImpl);
    }
}

// ============ 漏洞 3: selfdestruct 攻击 ============

// ❌ 危险: 实现合约可以被 selfdestruct
// 如果实现合约被销毁，所有代理都会失效

// ✅ 缓解措施:
// 1. 不要在实现合约中使用 selfdestruct
// 2. 使用多签钱包作为 owner
// 3. 实现合约应该验证只能通过代理调用

function _upgradeTo(address newImpl) internal virtual {}
}
```

---

## 附录 AH: ABI 编码与解码

### AH.1 ABI 编码基础

ABI（Application Binary Interface）是以太坊合约交互的标准编码格式。

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ABI 编码结构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   函数调用数据 (Calldata):                                           │
│   ┌──────────────┬──────────────────────────────────────────────┐   │
│   │ 4 bytes      │           N * 32 bytes                        │   │
│   │ 函数选择器    │           编码后的参数                         │   │
│   │ (selector)   │           (encoded arguments)                 │   │
│   └──────────────┴──────────────────────────────────────────────┘   │
│                                                                      │
│   函数选择器 = keccak256("functionName(type1,type2,...)")[0:4]      │
│                                                                      │
│   示例: transfer(address,uint256)                                    │
│   选择器 = keccak256("transfer(address,uint256)")[0:4]              │
│          = 0xa9059cbb                                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title ABI 编码示例
 * @notice 展示各种类型的 ABI 编码规则
 */
contract ABIEncodingExamples {

    // ============ 1. 静态类型编码 ============

    /**
     * @notice 静态类型占用固定 32 字节
     */
    function encodeStaticTypes() external pure returns (bytes memory) {
        uint256 a = 0x1234;
        address b = 0x1234567890123456789012345678901234567890;
        bool c = true;
        bytes32 d = bytes32(uint256(0xabcd));

        // 每个参数占用 32 字节，左填充 0
        return abi.encode(a, b, c, d);
        /*
         返回:
         0x0000000000000000000000000000000000000000000000000000000000001234  // uint256
         0x0000000000000000000000001234567890123456789012345678901234567890  // address
         0x0000000000000000000000000000000000000000000000000000000000000001  // bool
         0x000000000000000000000000000000000000000000000000000000000000abcd  // bytes32
        */
    }

    /**
     * @notice 小于 32 字节的类型
     */
    function encodeSmallTypes() external pure returns (bytes memory) {
        uint8 a = 0xff;
        int16 b = - 1;
        bytes4 c = 0x12345678;

        return abi.encode(a, b, c);
        /*
         返回:
         0x00000000000000000000000000000000000000000000000000000000000000ff  // uint8 左填充
         0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff  // int16 符号扩展
         0x1234567800000000000000000000000000000000000000000000000000000000  // bytes4 右填充
        */
    }

    // ============ 2. 动态类型编码 ============

    /**
     * @notice 动态类型使用 offset + data 结构
     */
    function encodeDynamicTypes() external pure returns (bytes memory) {
        bytes memory a = hex"1234";
        string memory b = "hello";

        return abi.encode(a, b);
        /*
         返回结构:
         0x0000000000000000000000000000000000000000000000000000000000000040  // offset of 'a' (64)
         0x0000000000000000000000000000000000000000000000000000000000000080  // offset of 'b' (128)
         0x0000000000000000000000000000000000000000000000000000000000000002  // length of 'a' (2)
         0x1234000000000000000000000000000000000000000000000000000000000000  // data of 'a'
         0x0000000000000000000000000000000000000000000000000000000000000005  // length of 'b' (5)
         0x68656c6c6f000000000000000000000000000000000000000000000000000000  // data of 'b' ("hello")
        */
    }

    /**
     * @notice 动态数组编码
     */
    function encodeDynamicArray() external pure returns (bytes memory) {
        uint256[] memory arr = new uint256[](3);
        arr[0] = 1;
        arr[1] = 2;
        arr[2] = 3;

        return abi.encode(arr);
        /*
         返回:
         0x0000000000000000000000000000000000000000000000000000000000000020  // offset (32)
         0x0000000000000000000000000000000000000000000000000000000000000003  // length (3)
         0x0000000000000000000000000000000000000000000000000000000000000001  // arr[0]
         0x0000000000000000000000000000000000000000000000000000000000000002  // arr[1]
         0x0000000000000000000000000000000000000000000000000000000000000003  // arr[2]
        */
    }

    /**
     * @notice 静态数组编码（无 offset，直接编码）
     */
    function encodeStaticArray() external pure returns (bytes memory) {
        uint256[3] memory arr = [uint256(1), 2, 3];

        return abi.encode(arr);
        /*
         返回（无 offset 和 length）:
         0x0000000000000000000000000000000000000000000000000000000000000001
         0x0000000000000000000000000000000000000000000000000000000000000002
         0x0000000000000000000000000000000000000000000000000000000000000003
        */
    }

    // ============ 3. 混合类型编码 ============

    /**
     * @notice 混合静态和动态类型
     */
    function encodeMixed(
        uint256 a,
        bytes memory b,
        uint256 c,
        string memory d
    ) external pure returns (bytes memory) {
        return abi.encode(a, b, c, d);
        /*
         结构:
         [0x00] a (直接值)
         [0x20] offset of b
         [0x40] c (直接值)
         [0x60] offset of d
         [0x80] length of b
         [0xa0] data of b (padded)
         [...] length of d
         [...] data of d (padded)
        */
    }

    // ============ 4. 函数选择器计算 ============

    /**
     * @notice 计算函数选择器
     */
    function getSelector(string calldata signature) external pure returns (bytes4) {
        return bytes4(keccak256(bytes(signature)));
    }

    /**
     * @notice 常见函数选择器
     */
    function commonSelectors() external pure returns (
        bytes4 transfer,
        bytes4 approve,
        bytes4 transferFrom,
        bytes4 balanceOf
    ) {
        transfer = bytes4(keccak256("transfer(address,uint256)"));         // 0xa9059cbb
        approve = bytes4(keccak256("approve(address,uint256)"));           // 0x095ea7b3
        transferFrom = bytes4(keccak256("transferFrom(address,address,uint256)")); // 0x23b872dd
        balanceOf = bytes4(keccak256("balanceOf(address)"));               // 0x70a08231
    }

    // ============ 5. abi.encodePacked vs abi.encode ============

    /**
     * @notice abi.encodePacked 紧凑编码（无填充）
     * @dev 警告：可能导致哈希碰撞
     */
    function compareEncoding() external pure returns (
        bytes memory encoded,
        bytes memory packed
    ) {
        uint8 a = 1;
        uint8 b = 2;

        encoded = abi.encode(a, b);
        // 0x0000000000000000000000000000000000000000000000000000000000000001
        // 0x0000000000000000000000000000000000000000000000000000000000000002
        // 共 64 字节

        packed = abi.encodePacked(a, b);
        // 0x0102
        // 共 2 字节
    }

    /**
     * @notice 哈希碰撞示例
     * @dev encodePacked 的危险性
     */
    function hashCollision() external pure returns (bool) {
        // 这两个会产生相同的 encodePacked 结果！
        bytes memory a = abi.encodePacked("ab", "c");   // "abc"
        bytes memory b = abi.encodePacked("a", "bc");   // "abc"

        return keccak256(a) == keccak256(b);  // true - 碰撞!
    }

    // ============ 6. abi.encodeWithSelector / encodeWithSignature ============

    /**
     * @notice 构造完整的函数调用数据
     */
    function encodeCall() external pure returns (
        bytes memory withSelector,
        bytes memory withSignature,
        bytes memory withCall
    ) {
        address to = address(0x1234);
        uint256 amount = 100;

        // 方式 1: encodeWithSelector
        withSelector = abi.encodeWithSelector(
            bytes4(keccak256("transfer(address,uint256)")),
            to,
            amount
        );

        // 方式 2: encodeWithSignature
        withSignature = abi.encodeWithSignature(
            "transfer(address,uint256)",
            to,
            amount
        );

        // 方式 3: encodeCall (类型安全，推荐)
        // withCall = abi.encodeCall(IERC20.transfer, (to, amount));
    }

    // ============ 7. ABI 解码 ============

    /**
     * @notice 解码 ABI 数据
     */
    function decodeData(bytes calldata data) external pure returns (
        address to,
        uint256 amount
    ) {
        // 跳过 4 字节选择器
        (to, amount) = abi.decode(data[4 :], (address, uint256));
    }

    /**
     * @notice 解码返回值
     */
    function decodeReturnData(bytes memory returnData) external pure returns (bool success) {
        if (returnData.length == 0) {
            // 无返回值，假设成功（旧版 ERC20）
            return true;
        }
        // 解码 bool
        return abi.decode(returnData, (bool));
    }
}
```

### AH.2 Rust/revm 中的 ABI 编解码

```rust
use alloy_primitives::{Address, Bytes, U256, FixedBytes, keccak256};
use alloy_sol_types::{sol, SolCall, SolValue};

// 使用 alloy 的 sol! 宏定义接口
sol! {
    interface IERC20 {
        function transfer(address to, uint256 amount) external returns (bool);
        function approve(address spender, uint256 amount) external returns (bool);
        function balanceOf(address account) external view returns (uint256);
        event Transfer(address indexed from, address indexed to, uint256 value);
    }
}

/// ABI 编码器
pub struct AbiEncoder;

impl AbiEncoder {
    /// 计算函数选择器
    pub fn selector(signature: &str) -> [u8; 4] {
        let hash = keccak256(signature.as_bytes());
        [hash[0], hash[1], hash[2], hash[3]]
    }

    /// 编码 transfer 调用
    pub fn encode_transfer(to: Address, amount: U256) -> Bytes {
        let call = IERC20::transferCall { to, amount };
        call.abi_encode().into()
    }

    /// 编码 balanceOf 调用
    pub fn encode_balance_of(account: Address) -> Bytes {
        let call = IERC20::balanceOfCall { account };
        call.abi_encode().into()
    }

    /// 手动编码静态类型
    pub fn encode_static(values: &[U256]) -> Vec<u8> {
        let mut result = Vec::with_capacity(values.len() * 32);
        for value in values {
            result.extend_from_slice(&value.to_be_bytes::<32>());
        }
        result
    }

    /// 手动编码动态 bytes
    pub fn encode_bytes(data: &[u8]) -> Vec<u8> {
        let mut result = Vec::new();

        // offset (指向数据开始位置)
        let offset = U256::from(32);
        result.extend_from_slice(&offset.to_be_bytes::<32>());

        // length
        let length = U256::from(data.len());
        result.extend_from_slice(&length.to_be_bytes::<32>());

        // data (32 字节对齐)
        result.extend_from_slice(data);
        let padding = (32 - (data.len() % 32)) % 32;
        result.extend(vec![0u8; padding]);

        result
    }

    /// 编码动态数组
    pub fn encode_uint_array(values: &[U256]) -> Vec<u8> {
        let mut result = Vec::new();

        // offset
        result.extend_from_slice(&U256::from(32).to_be_bytes::<32>());

        // length
        result.extend_from_slice(&U256::from(values.len()).to_be_bytes::<32>());

        // elements
        for value in values {
            result.extend_from_slice(&value.to_be_bytes::<32>());
        }

        result
    }
}

/// ABI 解码器
pub struct AbiDecoder;

impl AbiDecoder {
    /// 解码 transfer 调用
    pub fn decode_transfer(data: &[u8]) -> Result<(Address, U256), String> {
        if data.len() < 4 {
            return Err("Data too short".into());
        }

        let call = IERC20::transferCall::abi_decode(&data[4..], true)
            .map_err(|e| e.to_string())?;

        Ok((call.to, call.amount))
    }

    /// 解码 balanceOf 返回值
    pub fn decode_balance(data: &[u8]) -> Result<U256, String> {
        let returns = IERC20::balanceOfCall::abi_decode_returns(data, true)
            .map_err(|e| e.to_string())?;
        Ok(returns._0)
    }

    /// 手动解码静态类型
    pub fn decode_uint256(data: &[u8], offset: usize) -> Result<U256, String> {
        if data.len() < offset + 32 {
            return Err("Insufficient data".into());
        }
        let bytes: [u8; 32] = data[offset..offset + 32]
            .try_into()
            .map_err(|_| "Invalid slice")?;
        Ok(U256::from_be_bytes(bytes))
    }

    /// 解码地址
    pub fn decode_address(data: &[u8], offset: usize) -> Result<Address, String> {
        if data.len() < offset + 32 {
            return Err("Insufficient data".into());
        }
        // 地址在 32 字节的后 20 字节
        let bytes: [u8; 20] = data[offset + 12..offset + 32]
            .try_into()
            .map_err(|_| "Invalid slice")?;
        Ok(Address::from(bytes))
    }

    /// 解码动态 bytes
    pub fn decode_bytes(data: &[u8], head_offset: usize) -> Result<Vec<u8>, String> {
        // 读取数据偏移
        let data_offset = Self::decode_uint256(data, head_offset)?.to::<usize>();

        // 读取长度
        let length = Self::decode_uint256(data, data_offset)?.to::<usize>();

        // 读取数据
        let start = data_offset + 32;
        let end = start + length;

        if data.len() < end {
            return Err("Insufficient data for bytes".into());
        }

        Ok(data[start..end].to_vec())
    }
}

/// 事件日志解码
pub struct EventDecoder;

impl EventDecoder {
    /// 解码 Transfer 事件
    pub fn decode_transfer_event(
        topics: &[FixedBytes<32>],
        data: &[u8],
    ) -> Result<(Address, Address, U256), String> {
        // topic[0] = event signature hash
        // topic[1] = indexed from
        // topic[2] = indexed to
        // data = value

        if topics.len() < 3 {
            return Err("Insufficient topics".into());
        }

        // 验证事件签名
        let expected_sig = keccak256(b"Transfer(address,address,uint256)");
        if topics[0] != expected_sig {
            return Err("Invalid event signature".into());
        }

        // 解码 indexed 参数（从 topic）
        let from = Address::from_slice(&topics[1][12..]);
        let to = Address::from_slice(&topics[2][12..]);

        // 解码非 indexed 参数（从 data）
        let value = AbiDecoder::decode_uint256(data, 0)?;

        Ok((from, to, value))
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_selector() {
        assert_eq!(
            AbiEncoder::selector("transfer(address,uint256)"),
            [0xa9, 0x05, 0x9c, 0xbb]
        );
        assert_eq!(
            AbiEncoder::selector("balanceOf(address)"),
            [0x70, 0xa0, 0x82, 0x31]
        );
    }

    #[test]
    fn test_encode_decode_transfer() {
        let to = Address::repeat_byte(0x12);
        let amount = U256::from(1000);

        let encoded = AbiEncoder::encode_transfer(to, amount);
        let (decoded_to, decoded_amount) = AbiDecoder::decode_transfer(&encoded).unwrap();

        assert_eq!(to, decoded_to);
        assert_eq!(amount, decoded_amount);
    }
}
```

### AH.3 复杂类型编码示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 复杂 ABI 编码示例
 */
contract ComplexABIEncoding {

    // 结构体定义
    struct Order {
        address maker;
        address taker;
        uint256 makerAmount;
        uint256 takerAmount;
        uint256 expiry;
        bytes signature;
    }

    // 嵌套结构体
    struct Batch {
        Order[] orders;
        uint256 totalValue;
    }

    /**
     * @notice 结构体编码
     */
    function encodeOrder(Order calldata order) external pure returns (bytes memory) {
        return abi.encode(order);
        /*
         结构体按字段顺序编码：
         - maker (静态)
         - taker (静态)
         - makerAmount (静态)
         - takerAmount (静态)
         - expiry (静态)
         - signature offset (指向动态数据)
         - signature length
         - signature data
        */
    }

    /**
     * @notice 嵌套结构体数组编码
     */
    function encodeBatch(Batch calldata batch) external pure returns (bytes memory) {
        return abi.encode(batch);
    }

    /**
     * @notice 元组编码
     */
    function encodeTuple() external pure returns (bytes memory) {
        // 元组 (uint256, (address, uint256), bytes)
        return abi.encode(
            uint256(1),
            (address(0x1234), uint256(100)),
            hex"abcd"
        );
    }

    /**
     * @notice 解码结构体
     */
    function decodeOrder(bytes calldata data) external pure returns (Order memory) {
        return abi.decode(data, (Order));
    }

    /**
     * @notice 多返回值编码
     */
    function multipleReturns() external pure returns (
        uint256 a,
        address b,
        bytes memory c
    ) {
        return (1, address(0x1234), hex"abcd");
        // 返回值编码与参数编码相同
    }

    /**
     * @notice 错误编码 (Solidity 0.8.4+)
     */
    error InsufficientBalance(address account, uint256 required, uint256 available);

    function revertWithError() external pure {
        // 错误编码: selector + abi.encode(params)
        // selector = keccak256("InsufficientBalance(address,uint256,uint256)")[0:4]
        revert InsufficientBalance(address(0x1234), 100, 50);
    }

    /**
     * @notice 解码错误
     */
    function decodeError(bytes calldata data) external pure returns (
        address account,
        uint256 required,
        uint256 available
    ) {
        // 跳过 4 字节选择器
        (account, required, available) = abi.decode(
            data[4 :],
            (address, uint256, uint256)
        );
    }
}
```

---

## 附录 AI: CREATE 与 CREATE2 合约部署

### AI.1 CREATE 部署机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CREATE 地址计算                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   新合约地址 = keccak256(rlp([sender, nonce]))[12:]                 │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  sender: 部署者地址 (20 bytes)                               │   │
│   │  nonce:  部署者的交易计数 (可变长度 RLP)                      │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   特点:                                                              │
│   - 地址依赖于 nonce，每次部署递增                                   │
│   - 无法预先确定地址（除非知道确切的 nonce）                         │
│   - 合约被销毁后，同一地址可能被其他代码占用                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title CREATE 部署示例
 */
contract CreateDeployment {

    event Deployed(address addr, uint256 salt);

    /**
     * @notice 使用 CREATE 部署合约
     */
    function deploy(bytes memory bytecode) external returns (address) {
        address addr;
        assembly {
            // create(value, offset, size)
            // bytecode 在内存中: 前 32 字节是长度，之后是实际代码
            addr := create(
                0,                      // 发送的 ETH 数量
                add(bytecode, 0x20),    // 跳过长度前缀
                mload(bytecode)         // 字节码长度
            )

            // 检查部署是否成功
            if iszero(extcodesize(addr)) {
                revert(0, 0)
            }
        }
        return addr;
    }

    /**
     * @notice 使用 new 关键字部署（编译器生成 CREATE）
     */
    function deployWithNew() external returns (address) {
        SimpleContract c = new SimpleContract();
        return address(c);
    }

    /**
     * @notice 带构造函数参数的部署
     */
    function deployWithArgs(uint256 initialValue) external returns (address) {
        SimpleContract c = new SimpleContract{value: 0}();
        return address(c);
    }

    /**
     * @notice 预测 CREATE 地址
     */
    function predictAddress(address deployer, uint256 nonce) external pure returns (address) {
        bytes memory data;

        if (nonce == 0x00) {
            data = abi.encodePacked(bytes1(0xd6), bytes1(0x94), deployer, bytes1(0x80));
        } else if (nonce <= 0x7f) {
            data = abi.encodePacked(bytes1(0xd6), bytes1(0x94), deployer, uint8(nonce));
        } else if (nonce <= 0xff) {
            data = abi.encodePacked(bytes1(0xd7), bytes1(0x94), deployer, bytes1(0x81), uint8(nonce));
        } else if (nonce <= 0xffff) {
            data = abi.encodePacked(bytes1(0xd8), bytes1(0x94), deployer, bytes1(0x82), uint16(nonce));
        } else if (nonce <= 0xffffff) {
            data = abi.encodePacked(bytes1(0xd9), bytes1(0x94), deployer, bytes1(0x83), uint24(nonce));
        } else {
            data = abi.encodePacked(bytes1(0xda), bytes1(0x94), deployer, bytes1(0x84), uint32(nonce));
        }

        return address(uint160(uint256(keccak256(data))));
    }
}

contract SimpleContract {
    uint256 public value;

    constructor() {
        value = 100;
    }
}
```

### AI.2 CREATE2 确定性部署

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CREATE2 地址计算                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   新合约地址 = keccak256(0xff ++ sender ++ salt ++ keccak256(init_code))[12:]
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  0xff:      固定前缀 (1 byte)                                │   │
│   │  sender:    工厂合约地址 (20 bytes)                          │   │
│   │  salt:      用户提供的盐值 (32 bytes)                        │   │
│   │  init_code: 合约初始化代码的哈希 (32 bytes)                  │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   特点:                                                              │
│   - 地址完全确定，可提前计算                                         │
│   - 适用于反事实部署（Counterfactual Deployment）                   │
│   - 同一 salt + init_code 只能部署一次                              │
│   - 合约销毁后可以用相同参数重新部署                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title CREATE2 工厂合约
 */
contract Create2Factory {

    event Deployed(address addr, bytes32 salt);

    /**
     * @notice 使用 CREATE2 部署
     */
    function deploy(bytes memory bytecode, bytes32 salt) external payable returns (address) {
        address addr;
        assembly {
            // create2(value, offset, size, salt)
            addr := create2(
                callvalue(),            // 转发的 ETH
                add(bytecode, 0x20),    // 字节码起始位置
                mload(bytecode),        // 字节码长度
                salt                    // 盐值
            )

            if iszero(extcodesize(addr)) {
                revert(0, 0)
            }
        }

        emit Deployed(addr, salt);
        return addr;
    }

    /**
     * @notice 使用 new 关键字 + salt（Solidity 0.8.0+）
     */
    function deployWithNew(bytes32 salt, uint256 arg) external returns (address) {
        // 编译器自动生成 CREATE2
        Create2Target c = new Create2Target{salt: salt}(arg);
        return address(c);
    }

    /**
     * @notice 预测 CREATE2 地址
     */
    function predictAddress(
        bytes memory bytecode,
        bytes32 salt
    ) external view returns (address) {
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(bytecode)
            )
        );
        return address(uint160(uint256(hash)));
    }

    /**
     * @notice 带构造函数参数的地址预测
     */
    function predictAddressWithArgs(
        bytes32 salt,
        uint256 arg
    ) external view returns (address) {
        // 完整 init_code = 合约创建代码 + 构造函数参数
        bytes memory bytecode = abi.encodePacked(
            type(Create2Target).creationCode,
            abi.encode(arg)
        );

        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(bytecode)
            )
        );
        return address(uint160(uint256(hash)));
    }

    /**
     * @notice 检查地址是否已部署
     */
    function isDeployed(address addr) external view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(addr)
        }
        return size > 0;
    }
}

contract Create2Target {
    uint256 public immutable value;
    address public immutable factory;
    address public immutable deployer;

    constructor(uint256 _value) {
        value = _value;
        factory = msg.sender;
        deployer = tx.origin;
    }
}

/**
 * @title 反事实钱包工厂
 * @notice 展示 CREATE2 在账户抽象中的应用
 */
contract CounterfactualWalletFactory {
    /**
     * @notice 计算钱包地址（无需实际部署）
     */
    function getWalletAddress(
        address owner,
        uint256 salt
    ) external view returns (address) {
        bytes memory bytecode = abi.encodePacked(
            type(SimpleWallet).creationCode,
            abi.encode(owner)
        );

        return address(uint160(uint256(keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                bytes32(salt),
                keccak256(bytecode)
            )
        ))));
    }

    /**
     * @notice 部署钱包（可以在收到资金后再部署）
     */
    function deployWallet(
        address owner,
        uint256 salt
    ) external returns (address wallet) {
        wallet = address(new SimpleWallet{salt: bytes32(salt)}(owner));
    }
}

contract SimpleWallet {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function execute(
        address target,
        uint256 value,
        bytes calldata data
    ) external returns (bytes memory) {
        require(msg.sender == owner, "Not owner");
        (bool success, bytes memory result) = target.call{value: value}(data);
        require(success, "Call failed");
        return result;
    }

    receive() external payable {}
}
```

### AI.3 Rust/revm 中的合约部署

```rust
use alloy_primitives::{Address, Bytes, B256, U256, keccak256};
use revm::{
    primitives::{CreateScheme, TransactTo, TxEnv},
    Evm,
};

/// 合约部署工具
pub struct ContractDeployer;

impl ContractDeployer {
    /// 预测 CREATE 地址
    pub fn predict_create_address(deployer: Address, nonce: u64) -> Address {
        use alloy_rlp::Encodable;

        // RLP 编码 [sender, nonce]
        let mut rlp_data = Vec::new();

        // 计算 RLP 列表长度
        let sender_len = 21; // 0x94 + 20 bytes
        let nonce_rlp_len = if nonce == 0 {
            1 // 0x80
        } else if nonce < 0x80 {
            1
        } else if nonce < 0x100 {
            2
        } else if nonce < 0x10000 {
            3
        } else if nonce < 0x1000000 {
            4
        } else {
            5
        };

        let list_len = sender_len + nonce_rlp_len;

        // RLP 列表前缀
        if list_len < 56 {
            rlp_data.push(0xc0 + list_len as u8);
        } else {
            // 长列表（实际场景中很少见）
            unimplemented!("Long RLP list");
        }

        // 编码地址
        rlp_data.push(0x94); // 20 字节字符串前缀
        rlp_data.extend_from_slice(deployer.as_slice());

        // 编码 nonce
        if nonce == 0 {
            rlp_data.push(0x80);
        } else if nonce < 0x80 {
            rlp_data.push(nonce as u8);
        } else {
            let nonce_bytes = nonce.to_be_bytes();
            let leading_zeros = nonce_bytes.iter().take_while(|&&b| b == 0).count();
            let significant_bytes = &nonce_bytes[leading_zeros..];
            rlp_data.push(0x80 + significant_bytes.len() as u8);
            rlp_data.extend_from_slice(significant_bytes);
        }

        // keccak256 并取后 20 字节
        let hash = keccak256(&rlp_data);
        Address::from_slice(&hash[12..])
    }

    /// 预测 CREATE2 地址
    pub fn predict_create2_address(
        deployer: Address,
        salt: B256,
        init_code: &[u8],
    ) -> Address {
        let init_code_hash = keccak256(init_code);

        let mut data = Vec::with_capacity(1 + 20 + 32 + 32);
        data.push(0xff);
        data.extend_from_slice(deployer.as_slice());
        data.extend_from_slice(salt.as_slice());
        data.extend_from_slice(init_code_hash.as_slice());

        let hash = keccak256(&data);
        Address::from_slice(&hash[12..])
    }

    /// 使用 revm 执行 CREATE
    pub fn deploy_create(
        evm: &mut Evm<'_, (), impl revm::Database>,
        deployer: Address,
        bytecode: Bytes,
        value: U256,
    ) -> Result<Address, String> {
        evm.tx_mut().caller = deployer;
        evm.tx_mut().transact_to = TransactTo::Create;
        evm.tx_mut().data = bytecode;
        evm.tx_mut().value = value;

        let result = evm.transact().map_err(|e| e.to_string())?;

        match result.result {
            revm::primitives::ExecutionResult::Success { output, .. } => {
                match output {
                    revm::primitives::Output::Create(_, Some(addr)) => Ok(addr),
                    _ => Err("No address returned".into()),
                }
            }
            revm::primitives::ExecutionResult::Revert { output, .. } => {
                Err(format!("Revert: {:?}", output))
            }
            revm::primitives::ExecutionResult::Halt { reason, .. } => {
                Err(format!("Halt: {:?}", reason))
            }
        }
    }

    /// 使用 revm 执行 CREATE2
    pub fn deploy_create2(
        evm: &mut Evm<'_, (), impl revm::Database>,
        factory: Address,
        salt: B256,
        bytecode: Bytes,
        value: U256,
    ) -> Result<Address, String> {
        // CREATE2 通过工厂合约执行
        // 这里展示的是概念，实际需要调用工厂合约

        let predicted = Self::predict_create2_address(factory, salt, &bytecode);

        // 设置交易参数调用工厂
        evm.tx_mut().caller = factory;
        evm.tx_mut().transact_to = TransactTo::Create;
        evm.tx_mut().data = bytecode;
        evm.tx_mut().value = value;

        // 实际的 CREATE2 需要通过 CreateScheme::Create2 { salt }
        // 在 revm 内部处理

        Ok(predicted)
    }
}

/// 字节码构建器
pub struct BytecodeBuilder;

impl BytecodeBuilder {
    /// 构建带构造函数参数的部署代码
    pub fn with_constructor_args(
        creation_code: &[u8],
        args: &[u8],
    ) -> Vec<u8> {
        let mut bytecode = creation_code.to_vec();
        bytecode.extend_from_slice(args);
        bytecode
    }

    /// 构建最小代理 (EIP-1167) 字节码
    pub fn minimal_proxy(implementation: Address) -> Vec<u8> {
        let mut bytecode = Vec::with_capacity(55);

        // 部署代码
        bytecode.extend_from_slice(&[
            0x3d, 0x60, 0x2d, 0x80, 0x60, 0x0a, 0x3d, 0x39, 0x81, 0xf3,
        ]);

        // 运行时代码
        bytecode.extend_from_slice(&[
            0x36, 0x3d, 0x3d, 0x37, 0x3d, 0x3d, 0x3d, 0x36, 0x3d, 0x73,
        ]);
        bytecode.extend_from_slice(implementation.as_slice());
        bytecode.extend_from_slice(&[
            0x5a, 0xf4, 0x3d, 0x82, 0x80, 0x3e, 0x90, 0x3d, 0x91, 0x60,
            0x2b, 0x57, 0xfd, 0x5b, 0xf3,
        ]);

        bytecode
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_predict_create_address() {
        let deployer = Address::repeat_byte(0x12);

        // nonce = 0
        let addr0 = ContractDeployer::predict_create_address(deployer, 0);
        println!("nonce 0: {:?}", addr0);

        // nonce = 1
        let addr1 = ContractDeployer::predict_create_address(deployer, 1);
        println!("nonce 1: {:?}", addr1);

        // 地址应该不同
        assert_ne!(addr0, addr1);
    }

    #[test]
    fn test_predict_create2_address() {
        let deployer = Address::repeat_byte(0x12);
        let salt = B256::repeat_byte(0x00);
        let init_code = vec![0x60, 0x00]; // PUSH1 0x00

        let addr = ContractDeployer::predict_create2_address(deployer, salt, &init_code);
        println!("CREATE2 address: {:?}", addr);

        // 相同参数应该产生相同地址
        let addr2 = ContractDeployer::predict_create2_address(deployer, salt, &init_code);
        assert_eq!(addr, addr2);
    }
}
```

---

## 附录 AJ: Solidity 安全漏洞模式

### AJ.1 重入攻击 (Reentrancy)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      重入攻击流程                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   攻击者合约                      受害者合约                          │
│       │                              │                               │
│       │ 1. withdraw(1 ETH)           │                               │
│       │─────────────────────────────►│                               │
│       │                              │ 2. 检查余额 ✓                 │
│       │                              │                               │
│       │ 3. 发送 1 ETH                │                               │
│       │◄─────────────────────────────│                               │
│       │                              │                               │
│       │ 4. receive() 触发            │                               │
│       │    再次调用 withdraw()       │                               │
│       │─────────────────────────────►│                               │
│       │                              │ 5. 余额仍未更新！             │
│       │                              │    检查通过 ✓                 │
│       │ 6. 再次发送 1 ETH            │                               │
│       │◄─────────────────────────────│                               │
│       │      ... 循环直到耗尽 ...    │                               │
│       │                              │ 7. 最后更新余额               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 重入攻击示例与防护
 */

// ❌ 易受攻击的合约
contract VulnerableBank {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        // 危险：先发送 ETH，后更新状态
        (bool success,) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");

        balances[msg.sender] = 0;  // 太晚了！
    }
}

// 攻击者合约
contract Attacker {
    VulnerableBank public bank;
    uint256 public attackCount;

    constructor(address _bank) {
        bank = VulnerableBank(_bank);
    }

    function attack() external payable {
        bank.deposit{value: msg.value}();
        bank.withdraw();
    }

    receive() external payable {
        if (address(bank).balance >= 1 ether && attackCount < 10) {
            attackCount++;
            bank.withdraw();  // 重入调用
        }
    }
}

// ✅ 修复方案 1: Checks-Effects-Interactions 模式
contract SecureBankCEI {
    mapping(address => uint256) public balances;

    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        // 先更新状态（Effects）
        balances[msg.sender] = 0;

        // 最后进行外部调用（Interactions）
        (bool success,) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");
    }
}

// ✅ 修复方案 2: 重入锁
contract SecureBankLock {
    mapping(address => uint256) public balances;
    bool private locked;

    modifier nonReentrant() {
        require(!locked, "Reentrant call");
        locked = true;
        _;
        locked = false;
    }

    function withdraw() external nonReentrant {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        (bool success,) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");

        balances[msg.sender] = 0;
    }
}

// ✅ 修复方案 3: Pull 模式（推荐）
contract SecureBankPull {
    mapping(address => uint256) public balances;
    mapping(address => uint256) public pendingWithdrawals;

    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        balances[msg.sender] = 0;
        pendingWithdrawals[msg.sender] += balance;
    }

    function claim() external {
        uint256 amount = pendingWithdrawals[msg.sender];
        require(amount > 0, "Nothing to claim");

        pendingWithdrawals[msg.sender] = 0;

        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### AJ.2 闪电贷攻击

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 闪电贷攻击模式
 */

interface IFlashLoanProvider {
    function flashLoan(uint256 amount, bytes calldata data) external;
}

interface IPriceOracle {
    function getPrice(address token) external view returns (uint256);
}

// ❌ 易受攻击的预言机（使用现货价格）
contract VulnerableOracle is IPriceOracle {
    address public pair;  // Uniswap pair

    function getPrice(address token) external view override returns (uint256) {
        // 危险：直接使用 AMM 现货价格
        (uint112 reserve0, uint112 reserve1,) = IUniswapPair(pair).getReserves();
        return uint256(reserve1) * 1e18 / uint256(reserve0);
    }
}

// ❌ 易受攻击的借贷协议
contract VulnerableLending {
    IPriceOracle public oracle;
    mapping(address => uint256) public collateral;
    mapping(address => uint256) public debt;

    function borrow(uint256 amount) external {
        uint256 price = oracle.getPrice(address(0));  // 获取价格
        uint256 collateralValue = collateral[msg.sender] * price / 1e18;

        // 要求 150% 抵押率
        require(collateralValue >= (debt[msg.sender] + amount) * 150 / 100, "Undercollateralized");

        debt[msg.sender] += amount;
        // 转移借款...
    }
}

// 闪电贷攻击合约
contract FlashLoanAttacker {
    IFlashLoanProvider public flashLoanProvider;
    VulnerableLending public lending;
    address public pair;

    function attack() external {
        // 1. 借入大量代币
        flashLoanProvider.flashLoan(1000000 ether, "");
    }

    function executeOperation(uint256 amount, uint256 fee, bytes calldata) external {
        // 2. 操纵 AMM 价格（大额交换）
        // swap(amount) -> 价格暴涨

        // 3. 以操纵后的高价借出资产
        lending.borrow(/* 大量资产 */);

        // 4. 反向交换恢复价格
        // swap back

        // 5. 归还闪电贷
        // repay(amount + fee)

        // 利润 = 借出资产 - 闪电贷费用
    }
}

// ✅ 防护：使用 TWAP 预言机
contract SecureOracle is IPriceOracle {
    address public pair;
    uint256 public constant PERIOD = 30 minutes;

    uint256 public price0CumulativeLast;
    uint256 public price1CumulativeLast;
    uint32 public blockTimestampLast;
    uint256 public price0Average;
    uint256 public price1Average;

    function update() external {
        (uint256 price0Cumulative, uint256 price1Cumulative, uint32 blockTimestamp) =
                        currentCumulativePrices();

        uint32 timeElapsed = blockTimestamp - blockTimestampLast;

        if (timeElapsed >= PERIOD) {
            // 计算时间加权平均价格
            price0Average = (price0Cumulative - price0CumulativeLast) / timeElapsed;
            price1Average = (price1Cumulative - price1CumulativeLast) / timeElapsed;

            price0CumulativeLast = price0Cumulative;
            price1CumulativeLast = price1Cumulative;
            blockTimestampLast = blockTimestamp;
        }
    }

    function getPrice(address) external view override returns (uint256) {
        return price0Average;  // 返回 TWAP 价格
    }

    function currentCumulativePrices() internal view returns (uint256, uint256, uint32) {
        // 实现细节...
        return (0, 0, 0);
    }
}

interface IUniswapPair {
    function getReserves() external view returns (uint112, uint112, uint32);
}
```

### AJ.3 访问控制漏洞

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 访问控制漏洞示例
 */

// ❌ 漏洞 1: 缺少访问控制
contract MissingAccessControl {
    address public owner;
    bool public paused;

    constructor() {
        owner = msg.sender;
    }

    // 危险：任何人都可以调用！
    function setOwner(address newOwner) external {
        owner = newOwner;
    }

    // 危险：任何人都可以暂停！
    function pause() external {
        paused = true;
    }
}

// ❌ 漏洞 2: tx.origin 认证
contract TxOriginVulnerable {
    address public owner;

    constructor() {
        owner = tx.origin;
    }

    function withdraw() external {
        // 危险：使用 tx.origin 进行认证
        require(tx.origin == owner, "Not owner");
        payable(owner).transfer(address(this).balance);
    }
}

// tx.origin 攻击合约
contract TxOriginAttacker {
    TxOriginVulnerable public target;

    constructor(address _target) {
        target = TxOriginVulnerable(_target);
    }

    // 诱骗 owner 调用此函数
    function attack() external {
        // tx.origin = owner, msg.sender = this
        // target 会认为是 owner 在调用
        target.withdraw();
    }
}

// ❌ 漏洞 3: 默认可见性
contract DefaultVisibility {
    uint256 public balance;

    // 危险：Solidity 0.8 前默认是 public
    function _internalUpdate(uint256 amount) internal {
        balance = amount;
    }

    // 本意是 internal，但忘记加修饰符
    function sensitiveOperation() public {  // 应该是 internal 或 private
        // 敏感操作
    }
}

// ✅ 正确的访问控制实现
contract SecureAccessControl {
    address public owner;
    mapping(address => bool) public admins;
    mapping(address => mapping(bytes4 => bool)) public permissions;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    event AdminAdded(address indexed admin);
    event AdminRemoved(address indexed admin);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    modifier onlyAdmin() {
        require(admins[msg.sender] || msg.sender == owner, "Not admin");
        _;
    }

    modifier hasPermission(bytes4 selector) {
        require(
            msg.sender == owner ||
            admins[msg.sender] ||
            permissions[msg.sender][selector],
            "No permission"
        );
        _;
    }

    constructor() {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), msg.sender);
    }

    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Zero address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }

    function addAdmin(address admin) external onlyOwner {
        require(admin != address(0), "Zero address");
        admins[admin] = true;
        emit AdminAdded(admin);
    }

    function removeAdmin(address admin) external onlyOwner {
        admins[admin] = false;
        emit AdminRemoved(admin);
    }

    function grantPermission(address user, bytes4 selector) external onlyAdmin {
        permissions[user][selector] = true;
    }

    function revokePermission(address user, bytes4 selector) external onlyAdmin {
        permissions[user][selector] = false;
    }

    // 使用权限检查的敏感函数
    function sensitiveOperation() external hasPermission(this.sensitiveOperation.selector) {
        // 敏感操作
    }
}
```

### AJ.4 整数溢出与精度损失

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 数值安全问题
 */

// Solidity 0.8+ 内置溢出检查，但仍需注意 unchecked 块

contract NumericSafety {
    // ❌ 危险：unchecked 块中的溢出
    function unsafeAdd(uint256 a, uint256 b) external pure returns (uint256) {
        unchecked {
            return a + b;  // 可能溢出！
        }
    }

    // ❌ 危险：类型转换导致的截断
    function unsafeCast(uint256 amount) external pure returns (uint128) {
        return uint128(amount);  // 如果 amount > type(uint128).max，会截断
    }

    // ✅ 安全的类型转换
    function safeCast(uint256 amount) external pure returns (uint128) {
        require(amount <= type(uint128).max, "Overflow");
        return uint128(amount);
    }

    // ❌ 危险：除法精度损失
    function calculateShare(
        uint256 amount,
        uint256 totalSupply,
        uint256 totalAssets
    ) external pure returns (uint256) {
        // 如果 amount * totalAssets 小于 totalSupply，结果为 0
        return amount * totalAssets / totalSupply;
    }

    // ✅ 改进：先乘后除，使用更高精度
    function calculateShareSafe(
        uint256 amount,
        uint256 totalSupply,
        uint256 totalAssets
    ) external pure returns (uint256) {
        if (totalSupply == 0) return amount;

        // 使用 mulDiv 避免溢出同时保持精度
        return mulDiv(amount, totalAssets, totalSupply);
    }

    // 来自 OpenZeppelin 的 mulDiv
    function mulDiv(uint256 x, uint256 y, uint256 denominator) internal pure returns (uint256 result) {
        unchecked {
            uint256 prod0;
            uint256 prod1;
            assembly {
                let mm := mulmod(x, y, not(0))
                prod0 := mul(x, y)
                prod1 := sub(sub(mm, prod0), lt(mm, prod0))
            }

            if (prod1 == 0) {
                return prod0 / denominator;
            }

            require(denominator > prod1, "Math: mulDiv overflow");

            uint256 remainder;
            assembly {
                remainder := mulmod(x, y, denominator)
                prod1 := sub(prod1, gt(remainder, prod0))
                prod0 := sub(prod0, remainder)
            }

            uint256 twos = denominator & (~denominator + 1);
            assembly {
                denominator := div(denominator, twos)
                prod0 := div(prod0, twos)
                twos := add(div(sub(0, twos), twos), 1)
            }

            prod0 |= prod1 * twos;

            uint256 inverse = (3 * denominator) ^ 2;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;

            result = prod0 * inverse;
        }
    }

    // ❌ 危险：舍入方向不一致
    function vulnerableRounding(uint256 assets, uint256 shares, uint256 totalAssets, uint256 totalShares)
    external pure returns (uint256)
    {
        // 存款时向下舍入对用户有利
        // 取款时向下舍入对协议有利
        // 需要根据场景选择正确的舍入方向
        return assets * totalShares / totalAssets;
    }

    // ✅ 显式控制舍入方向
    enum Rounding {Down, Up}

    function divRound(uint256 a, uint256 b, Rounding rounding) internal pure returns (uint256) {
        uint256 result = a / b;
        if (rounding == Rounding.Up && a % b != 0) {
            result += 1;
        }
        return result;
    }
}
```

### AJ.5 签名重放攻击

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 签名安全问题
 */

// ❌ 易受重放攻击的签名验证
contract VulnerableSignature {
    mapping(address => uint256) public balances;

    function withdraw(uint256 amount, bytes memory signature) external {
        bytes32 messageHash = keccak256(abi.encodePacked(msg.sender, amount));
        address signer = recoverSigner(messageHash, signature);

        require(signer == msg.sender, "Invalid signature");

        // 危险：同一签名可以重复使用！
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }

    function recoverSigner(bytes32 hash, bytes memory sig) internal pure returns (address) {
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }
        return ecrecover(hash, v, r, s);
    }
}

// ✅ 安全的签名验证
contract SecureSignature {
    mapping(address => uint256) public balances;
    mapping(address => uint256) public nonces;
    mapping(bytes32 => bool) public usedSignatures;

    // 域分隔符防止跨链/跨合约重放
    bytes32 public immutable DOMAIN_SEPARATOR;

    bytes32 public constant WITHDRAW_TYPEHASH =
    keccak256("Withdraw(address owner,uint256 amount,uint256 nonce,uint256 deadline)");

    constructor() {
        DOMAIN_SEPARATOR = keccak256(
            abi.encode(
                keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
                keccak256(bytes("SecureSignature")),
                keccak256(bytes("1")),
                block.chainid,
                address(this)
            )
        );
    }

    function withdraw(
        uint256 amount,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        require(block.timestamp <= deadline, "Signature expired");

        // 使用 nonce 防止重放
        uint256 nonce = nonces[msg.sender]++;

        bytes32 structHash = keccak256(
            abi.encode(WITHDRAW_TYPEHASH, msg.sender, amount, nonce, deadline)
        );

        bytes32 digest = keccak256(
            abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, structHash)
        );

        // 防止签名重放
        require(!usedSignatures[digest], "Signature already used");
        usedSignatures[digest] = true;

        address signer = ecrecover(digest, v, r, s);
        require(signer == msg.sender && signer != address(0), "Invalid signature");

        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }

    // 防止签名延展性攻击
    function isValidSignature(bytes32 s) internal pure returns (bool) {
        // s 必须在曲线的下半部分
        return uint256(s) <= 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0;
    }
}
```

### AJ.6 常见漏洞检查清单

```solidity
/**
 * 智能合约安全检查清单
 *
 * □ 重入攻击
 *   - 使用 Checks-Effects-Interactions 模式
 *   - 或使用 ReentrancyGuard
 *   - 警惕 ERC-777, ERC-1155 的回调
 *
 * □ 访问控制
 *   - 所有敏感函数有适当的访问修饰符
 *   - 不使用 tx.origin 进行授权
 *   - 使用 OpenZeppelin AccessControl
 *
 * □ 数值安全
 *   - 注意 unchecked 块中的操作
 *   - 安全的类型转换
 *   - 正确的舍入方向
 *   - 防止除零
 *
 * □ 外部调用
 *   - 检查返回值
 *   - 使用 try-catch
 *   - 限制 gas
 *   - 防止 DoS
 *
 * □ 预言机
 *   - 使用 TWAP 而非现货价格
 *   - 多预言机聚合
 *   - 价格有效性检查
 *
 * □ 签名
 *   - 使用 EIP-712 类型化数据
 *   - 包含 nonce 或 deadline
 *   - 域分隔符包含 chainId
 *   - 检查签名延展性
 *
 * □ 时间依赖
 *   - 不依赖 block.timestamp 做精确计算
 *   - 考虑矿工操纵可能
 *
 * □ 前端运行 (Frontrunning)
 *   - 使用 commit-reveal 模式
 *   - 或使用 Flashbots 私有交易
 *
 * □ 升级安全
 *   - 存储布局兼容
 *   - 初始化函数保护
 *   - 时间锁
 */
```

---

## 附录 AK: 事件与日志系统

### AK.1 EVM 日志机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                      EVM 日志结构                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   LOG0: 无 topic，只有 data                                          │
│   LOG1: 1 个 topic + data                                           │
│   LOG2: 2 个 topics + data                                          │
│   LOG3: 3 个 topics + data                                          │
│   LOG4: 4 个 topics + data（最大）                                   │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  Log Entry                                                   │   │
│   │  ├── address: 发出日志的合约地址                             │   │
│   │  ├── topics[]: 32 字节的索引数据（最多 4 个）                │   │
│   │  │   └── topics[0]: 通常是事件签名的 keccak256              │   │
│   │  └── data: 非索引的 ABI 编码数据                             │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   Gas 成本:                                                          │
│   - 基础成本: 375 gas                                                │
│   - 每个 topic: 375 gas                                             │
│   - 每字节 data: 8 gas                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 事件与日志示例
 */
contract EventExamples {

    // ============ 1. 基本事件定义 ============

    // 简单事件
    event SimpleEvent(uint256 value);

    // 带索引的事件（可被高效过滤）
    event Transfer(
        address indexed from,    // topic[1]
        address indexed to,      // topic[2]
        uint256 value            // data
    );

    // 多索引事件
    event Approval(
        address indexed owner,   // topic[1]
        address indexed spender, // topic[2]
        uint256 value            // data
    );

    // 匿名事件（无签名 topic）
    event Anonymous(uint256 indexed id, bytes data) anonymous;

    // ============ 2. 事件触发 ============

    function emitEvents() external {
        // 普通触发
        emit SimpleEvent(100);

        // 触发 Transfer
        emit Transfer(msg.sender, address(0x1234), 1000);

        // 匿名事件节省一个 topic 的 gas
        emit Anonymous(1, "test");
    }

    // ============ 3. 低级日志操作 ============

    function lowLevelLog() external {
        // LOG0: 无 topic
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, 0x1234)
            log0(ptr, 32)
        }

        // LOG1: 1 个 topic
        bytes32 topic = keccak256("CustomEvent(uint256)");
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, 100)
            log1(ptr, 32, topic)
        }

        // LOG2: 2 个 topics
        bytes32 topic1 = keccak256("Transfer(address,address,uint256)");
        bytes32 topic2 = bytes32(uint256(uint160(msg.sender)));
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, 1000)
            log2(ptr, 32, topic1, topic2)
        }
    }

    // ============ 4. 大数据日志优化 ============

    // 存储大数据到日志而非状态（节省 gas）
    event DataStored(bytes32 indexed key, bytes data);

    function storeDataToLogs(bytes32 key, bytes calldata data) external {
        // 将数据存储到日志，比状态存储便宜得多
        // 但无法在合约中读取
        emit DataStored(key, data);
    }

    // ============ 5. 事件签名计算 ============

    function getEventSignatures() external pure returns (
        bytes32 transferSig,
        bytes32 approvalSig
    ) {
        transferSig = keccak256("Transfer(address,address,uint256)");
        // = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef

        approvalSig = keccak256("Approval(address,address,uint256)");
        // = 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925
    }
}
```

### AK.2 Bloom Filter 与日志索引

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Bloom Filter 原理                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   区块头包含 logsBloom (2048 bits)                                   │
│                                                                      │
│   添加日志到 Bloom:                                                  │
│   1. 对 address 和每个 topic 计算 keccak256                         │
│   2. 取哈希的 [0:2], [2:4], [4:6] 字节                              │
│   3. 对每对字节 mod 2048 得到 3 个位置                              │
│   4. 在 Bloom filter 中设置这 3 个位                                │
│                                                                      │
│   查询时：                                                           │
│   - 如果查询的位都被设置 → 可能包含该日志                            │
│   - 如果任意位未设置 → 一定不包含该日志                              │
│   - 假阳性率约 1/10000                                              │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  Bloom Filter (256 bytes = 2048 bits)                       │   │
│   │  [0000...0100...0010...0001...0000]                         │   │
│   │         ↑         ↑         ↑                               │   │
│   │         │         │         │                               │   │
│   │    keccak(topic)[0:2] mod 2048                              │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```rust
use alloy_primitives::{Address, Bloom, B256, keccak256};

/// Bloom Filter 实现
pub struct BloomFilter {
    bits: [u8; 256],
}

impl BloomFilter {
    pub fn new() -> Self {
        Self { bits: [0u8; 256] }
    }

    /// 添加数据到 Bloom filter
    pub fn add(&mut self, data: &[u8]) {
        let hash = keccak256(data);

        // 从哈希中提取 3 个位置
        for i in 0..3 {
            let bit_index = self.get_bit_index(&hash, i);
            self.set_bit(bit_index);
        }
    }

    /// 添加日志地址
    pub fn add_address(&mut self, address: Address) {
        self.add(address.as_slice());
    }

    /// 添加 topic
    pub fn add_topic(&mut self, topic: B256) {
        self.add(topic.as_slice());
    }

    /// 检查数据是否可能存在
    pub fn may_contain(&self, data: &[u8]) -> bool {
        let hash = keccak256(data);

        for i in 0..3 {
            let bit_index = self.get_bit_index(&hash, i);
            if !self.get_bit(bit_index) {
                return false;
            }
        }
        true
    }

    /// 合并两个 Bloom filter
    pub fn merge(&mut self, other: &BloomFilter) {
        for i in 0..256 {
            self.bits[i] |= other.bits[i];
        }
    }

    fn get_bit_index(&self, hash: &B256, index: usize) -> usize {
        let offset = index * 2;
        let byte1 = hash[offset] as usize;
        let byte2 = hash[offset + 1] as usize;
        ((byte1 << 8) | byte2) % 2048
    }

    fn set_bit(&mut self, bit_index: usize) {
        let byte_index = bit_index / 8;
        let bit_offset = bit_index % 8;
        self.bits[byte_index] |= 1 << (7 - bit_offset);
    }

    fn get_bit(&self, bit_index: usize) -> bool {
        let byte_index = bit_index / 8;
        let bit_offset = bit_index % 8;
        (self.bits[byte_index] & (1 << (7 - bit_offset))) != 0
    }

    /// 转换为 alloy Bloom 类型
    pub fn to_bloom(&self) -> Bloom {
        Bloom::from_slice(&self.bits)
    }
}

/// 日志过滤器
pub struct LogFilter {
    /// 地址过滤（OR 关系）
    pub addresses: Vec<Address>,
    /// Topic 过滤（AND 关系，每个位置内部是 OR 关系）
    pub topics: [Vec<B256>; 4],
    /// 区块范围
    pub from_block: Option<u64>,
    pub to_block: Option<u64>,
}

impl LogFilter {
    /// 检查日志是否匹配过滤器
    pub fn matches(&self, log: &Log) -> bool {
        // 检查地址
        if !self.addresses.is_empty() && !self.addresses.contains(&log.address) {
            return false;
        }

        // 检查 topics
        for (i, filter_topics) in self.topics.iter().enumerate() {
            if filter_topics.is_empty() {
                continue;
            }

            match log.topics.get(i) {
                Some(log_topic) => {
                    if !filter_topics.contains(log_topic) {
                        return false;
                    }
                }
                None => return false,
            }
        }

        true
    }

    /// 构建查询的 Bloom filter
    pub fn to_bloom(&self) -> BloomFilter {
        let mut bloom = BloomFilter::new();

        for address in &self.addresses {
            bloom.add_address(*address);
        }

        for topic_filter in &self.topics {
            for topic in topic_filter {
                bloom.add_topic(*topic);
            }
        }

        bloom
    }
}

/// 日志结构
pub struct Log {
    pub address: Address,
    pub topics: Vec<B256>,
    pub data: Vec<u8>,
    pub block_number: u64,
    pub transaction_index: u32,
    pub log_index: u32,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_bloom_filter() {
        let mut bloom = BloomFilter::new();

        let address = Address::repeat_byte(0x12);
        let topic = B256::repeat_byte(0xab);

        bloom.add_address(address);
        bloom.add_topic(topic);

        assert!(bloom.may_contain(address.as_slice()));
        assert!(bloom.may_contain(topic.as_slice()));
        assert!(!bloom.may_contain(&[0x00; 32])); // 可能假阳性
    }
}
```

### AK.3 事件监听与解析

```rust
use alloy_primitives::{Address, B256, U256, FixedBytes};
use alloy_sol_types::{sol, SolEvent};

// 定义事件
sol! {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Swap(
        address indexed sender,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out,
        address indexed to
    );
}

/// 事件解析器
pub struct EventParser;

impl EventParser {
    /// 获取事件签名
    pub fn transfer_signature() -> B256 {
        Transfer::SIGNATURE_HASH
    }

    pub fn approval_signature() -> B256 {
        Approval::SIGNATURE_HASH
    }

    /// 解析 Transfer 事件
    pub fn parse_transfer(log: &Log) -> Result<TransferEvent, String> {
        if log.topics.is_empty() {
            return Err("No topics".into());
        }

        if log.topics[0] != Transfer::SIGNATURE_HASH {
            return Err("Not a Transfer event".into());
        }

        let event = Transfer::decode_log_data(&log.data, true)
            .map_err(|e| e.to_string())?;

        // 解析 indexed 参数
        let from = Address::from_slice(&log.topics[1][12..]);
        let to = Address::from_slice(&log.topics[2][12..]);

        Ok(TransferEvent {
            from,
            to,
            value: event.value,
        })
    }

    /// 解析 Swap 事件（Uniswap V2）
    pub fn parse_swap(log: &Log) -> Result<SwapEvent, String> {
        if log.topics.len() < 3 {
            return Err("Insufficient topics".into());
        }

        // 解码非 indexed 数据
        if log.data.len() < 128 {
            return Err("Insufficient data".into());
        }

        let amount0_in = U256::from_be_slice(&log.data[0..32]);
        let amount1_in = U256::from_be_slice(&log.data[32..64]);
        let amount0_out = U256::from_be_slice(&log.data[64..96]);
        let amount1_out = U256::from_be_slice(&log.data[96..128]);

        Ok(SwapEvent {
            sender: Address::from_slice(&log.topics[1][12..]),
            to: Address::from_slice(&log.topics[2][12..]),
            amount0_in,
            amount1_in,
            amount0_out,
            amount1_out,
        })
    }
}

pub struct TransferEvent {
    pub from: Address,
    pub to: Address,
    pub value: U256,
}

pub struct SwapEvent {
    pub sender: Address,
    pub to: Address,
    pub amount0_in: U256,
    pub amount1_in: U256,
    pub amount0_out: U256,
    pub amount1_out: U256,
}

/// 事件订阅器（WebSocket）
pub struct EventSubscriber {
    filter: LogFilter,
}

impl EventSubscriber {
    pub fn new() -> Self {
        Self {
            filter: LogFilter {
                addresses: vec![],
                topics: [vec![], vec![], vec![], vec![]],
                from_block: None,
                to_block: None,
            },
        }
    }

    /// 添加 ERC20 Transfer 事件过滤
    pub fn filter_erc20_transfers(mut self, token: Address) -> Self {
        self.filter.addresses.push(token);
        self.filter.topics[0].push(Transfer::SIGNATURE_HASH);
        self
    }

    /// 添加特定发送者的过滤
    pub fn from_address(mut self, from: Address) -> Self {
        let topic = B256::left_padding_from(from.as_slice());
        self.filter.topics[1].push(topic);
        self
    }

    /// 添加特定接收者的过滤
    pub fn to_address(mut self, to: Address) -> Self {
        let topic = B256::left_padding_from(to.as_slice());
        self.filter.topics[2].push(topic);
        self
    }

    /// 构建 eth_getLogs 请求参数
    pub fn build_filter_params(&self) -> serde_json::Value {
        let mut params = serde_json::Map::new();

        if !self.filter.addresses.is_empty() {
            let addresses: Vec<String> = self.filter.addresses
                .iter()
                .map(|a| format!("{:?}", a))
                .collect();
            params.insert("address".into(), addresses.into());
        }

        let topics: Vec<Option<Vec<String>>> = self.filter.topics
            .iter()
            .map(|t| {
                if t.is_empty() {
                    None
                } else {
                    Some(t.iter().map(|h| format!("{:?}", h)).collect())
                }
            })
            .collect();
        params.insert("topics".into(), serde_json::json!(topics));

        if let Some(from) = self.filter.from_block {
            params.insert("fromBlock".into(), format!("0x{:x}", from).into());
        }

        if let Some(to) = self.filter.to_block {
            params.insert("toBlock".into(), format!("0x{:x}", to).into());
        }

        params.into()
    }
}

struct Log {
    address: Address,
    topics: Vec<B256>,
    data: Vec<u8>,
}
```

---

## 附录 AL: Yul 与内联汇编

### AL.1 Yul 基础语法

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Yul 语言概述                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Yul 是 Solidity 的中间语言，也可直接用于内联汇编                   │
│                                                                      │
│   特点：                                                             │
│   - 类型化：只有一种类型（256 位字）                                 │
│   - 函数式：所有操作都是函数调用                                     │
│   - 低级：直接映射到 EVM 操作码                                     │
│   - 优化友好：编译器可以进行激进优化                                │
│                                                                      │
│   语法元素：                                                         │
│   - 字面量: 0x123, 123, "hello"                                     │
│   - 变量: let x := 0                                                │
│   - 赋值: x := add(x, 1)                                            │
│   - 函数: function f(a, b) -> c { c := add(a, b) }                  │
│   - 条件: if condition { ... }                                      │
│   - 循环: for { init } condition { post } { body }                  │
│   - Switch: switch value case 0 { } default { }                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title Yul 内联汇编示例
 */
contract YulExamples {

    // ============ 1. 基本操作 ============

    function basicOperations() external pure returns (uint256 result) {
        assembly {
            // 变量声明和赋值
            let a := 10
            let b := 20

            // 算术运算
            let sum := add(a, b)         // 30
            let diff := sub(b, a)        // 10
            let prod := mul(a, b)        // 200
            let quot := div(prod, a)     // 20

            // 比较运算
            let isLt := lt(a, b)         // 1 (true)
            let isGt := gt(a, b)         // 0 (false)
            let isEq := eq(a, a)         // 1

            // 位运算
            let andResult := and(0xff, 0x0f)    // 0x0f
            let orResult := or(0xf0, 0x0f)      // 0xff
            let xorResult := xor(0xff, 0x0f)    // 0xf0
            let notResult := not(0)              // 0xfff...fff
            let shl := shl(4, 1)                // 16 (1 << 4)
            let shr := shr(4, 16)               // 1 (16 >> 4)

            result := sum
        }
    }

    // ============ 2. 内存操作 ============

    function memoryOperations() external pure returns (bytes32 result) {
        assembly {
            // 获取空闲内存指针
            let freePtr := mload(0x40)

            // 写入内存
            mstore(freePtr, 0x1234)         // 写入 32 字节
            mstore8(add(freePtr, 32), 0xff) // 写入 1 字节

            // 读取内存
            result := mload(freePtr)

            // 更新空闲内存指针
            mstore(0x40, add(freePtr, 64))
        }
    }

    // ============ 3. 存储操作 ============

    uint256 public storedValue;  // slot 0

    function storageOperations(uint256 newValue) external {
        assembly {
            // 读取存储
            let oldValue := sload(0)

            // 写入存储
            sstore(0, newValue)

            // 复杂存储槽计算（mapping）
            // mapping(address => uint256) balances 的 slot
            // slot = keccak256(key . baseSlot)
            let key := caller()
            mstore(0, key)
            mstore(32, 1)  // baseSlot = 1
            let balanceSlot := keccak256(0, 64)

            // 读写 mapping
            let balance := sload(balanceSlot)
            sstore(balanceSlot, add(balance, 100))
        }
    }

    // ============ 4. Calldata 操作 ============

    function calldataOperations(bytes calldata data) external pure returns (bytes4 selector, uint256 firstArg) {
        assembly {
            // 读取函数选择器
            selector := calldataload(0)  // 前 32 字节（包含选择器）
            selector := shr(224, selector)  // 右移保留前 4 字节

            // 读取第一个参数（跳过 4 字节选择器）
            firstArg := calldataload(4)

            // calldata 大小
            let size := calldatasize()

            // 复制 calldata 到内存
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, size)
        }
    }

    // ============ 5. 控制流 ============

    function controlFlow(uint256 n) external pure returns (uint256 result) {
        assembly {
            // if 语句
            if gt(n, 10) {
                result := 1
            }

            // switch 语句
            switch n
            case 0 {
                result := 100
            }
            case 1 {
                result := 200
            }
            default {
                result := 300
            }

            // for 循环
            let sum := 0
            for {let i := 0} lt(i, n) {i := add(i, 1)} {
                sum := add(sum, i)
            }
            result := sum
        }
    }

    // ============ 6. 函数定义 ============

    function functionExample(uint256 a, uint256 b) external pure returns (uint256) {
        assembly {
            // 定义内部函数
            function safeAdd(x, y) -> z {
                z := add(x, y)
                if lt(z, x) {
                    revert(0, 0)  // 溢出
                }
            }

            function max(x, y) -> m {
                m := x
                if gt(y, x) {
                    m := y
                }
            }

            // 调用函数
            let sum := safeAdd(a, b)
            let maximum := max(a, b)

            mstore(0, add(sum, maximum))
            return (0, 32)
        }
    }

    // ============ 7. 外部调用 ============

    function externalCallExample(address target, bytes calldata data) external returns (bool success, bytes memory returnData) {
        assembly {
            // 准备 calldata
            let ptr := mload(0x40)
            calldatacopy(ptr, data.offset, data.length)

            // call(gas, addr, value, argsOffset, argsSize, retOffset, retSize)
            success := call(
                gas(),          // 转发所有 gas
                target,         // 目标地址
                0,              // 发送的 ETH
                ptr,            // 输入数据位置
                data.length,    // 输入数据长度
                0,              // 输出位置（稍后复制）
                0               // 输出长度（未知）
            )

            // 获取返回数据
            let retSize := returndatasize()

            // 分配内存给返回数据
            returnData := mload(0x40)
            mstore(returnData, retSize)  // 长度
            returndatacopy(add(returnData, 32), 0, retSize)

            // 更新空闲指针
            mstore(0x40, add(add(returnData, 32), retSize))
        }
    }

    // ============ 8. 错误处理 ============

    function errorHandling() external pure {
        assembly {
            // revert 无数据
            // revert(0, 0)

            // revert 带错误消息
            let ptr := mload(0x40)
            // Error(string) 选择器
            mstore(ptr, 0x08c379a000000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 4), 32)  // offset
            mstore(add(ptr, 36), 11) // length
            mstore(add(ptr, 68), "Custom error")
            // revert(ptr, 100)

            // 自定义错误
            // error MyError(uint256 code)
            // mstore(ptr, 0xaabbccdd)  // 错误选择器
            // mstore(add(ptr, 4), 123) // 参数
            // revert(ptr, 36)

            // invalid 操作（消耗所有 gas）
            // invalid()
        }
    }
}
```

### AL.2 高级 Yul 模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title 高级 Yul 优化模式
 */
contract AdvancedYul {

    // ============ 1. 高效批量操作 ============

    /**
     * @notice 批量转账优化
     */
    function batchTransfer(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external {
        assembly {
            // 验证数组长度
            let len := recipients.length
            if iszero(eq(len, amounts.length)) {
                revert(0, 0)
            }

            // 准备 transfer 调用数据
            let ptr := mload(0x40)
            // transfer(address,uint256) selector
            mstore(ptr, 0xa9059cbb00000000000000000000000000000000000000000000000000000000)

            // 遍历执行
            for {let i := 0} lt(i, len) {i := add(i, 1)} {
                // 加载接收者地址
                let recipientOffset := add(recipients.offset, mul(i, 32))
                let recipient := calldataload(recipientOffset)

                // 加载金额
                let amountOffset := add(amounts.offset, mul(i, 32))
                let amount := calldataload(amountOffset)

                // 填充参数
                mstore(add(ptr, 4), recipient)
                mstore(add(ptr, 36), amount)

                // 调用（假设是 ERC20 代币）
                // let success := call(gas(), token, 0, ptr, 68, 0, 32)
            }
        }
    }

    // ============ 2. 内存高效的字符串操作 ============

    function efficientConcat(
        bytes memory a,
        bytes memory b
    ) external pure returns (bytes memory result) {
        assembly {
            // 获取长度
            let aLen := mload(a)
            let bLen := mload(b)
            let totalLen := add(aLen, bLen)

            // 分配结果内存
            result := mload(0x40)
            mstore(result, totalLen)

            // 复制 a 的数据
            let aData := add(a, 32)
            let resultData := add(result, 32)

            // 使用 mcopy 或手动复制
            for {let i := 0} lt(i, aLen) {i := add(i, 32)} {
                mstore(add(resultData, i), mload(add(aData, i)))
            }

            // 复制 b 的数据
            let bData := add(b, 32)
            let bStart := add(resultData, aLen)

            for {let i := 0} lt(i, bLen) {i := add(i, 32)} {
                mstore(add(bStart, i), mload(add(bData, i)))
            }

            // 更新空闲指针（32 字节对齐）
            mstore(0x40, add(add(result, 32), and(add(totalLen, 31), not(31))))
        }
    }

    // ============ 3. Gas 优化的数学运算 ============

    /**
     * @notice 快速幂运算
     */
    function fastPow(uint256 base, uint256 exp) external pure returns (uint256 result) {
        assembly {
            result := 1

            for {} gt(exp, 0) {} {
                // 如果 exp 是奇数
                if and(exp, 1) {
                    result := mul(result, base)
                }

                base := mul(base, base)
                exp := shr(1, exp)  // exp /= 2
            }
        }
    }

    /**
     * @notice 查找最高有效位
     */
    function mostSignificantBit(uint256 x) external pure returns (uint256 r) {
        assembly {
            r := 0

            // 二分查找
            if gt(x, 0xffffffffffffffffffffffffffffffff) {
                x := shr(128, x)
                r := 128
            }
            if gt(x, 0xffffffffffffffff) {
                x := shr(64, x)
                r := add(r, 64)
            }
            if gt(x, 0xffffffff) {
                x := shr(32, x)
                r := add(r, 32)
            }
            if gt(x, 0xffff) {
                x := shr(16, x)
                r := add(r, 16)
            }
            if gt(x, 0xff) {
                x := shr(8, x)
                r := add(r, 8)
            }
            if gt(x, 0xf) {
                x := shr(4, x)
                r := add(r, 4)
            }
            if gt(x, 0x3) {
                x := shr(2, x)
                r := add(r, 2)
            }
            if gt(x, 0x1) {
                r := add(r, 1)
            }
        }
    }

    // ============ 4. 紧凑存储打包 ============

    // 将多个小值打包到一个 slot
    // [  8 bits  ][  8 bits  ][ 160 bits ][ 80 bits ]
    // [  status  ][  flags   ][  address ][ amount  ]

    function packData(
        uint8 status,
        uint8 flags,
        address addr,
        uint80 amount
    ) external pure returns (uint256 packed) {
        assembly {
            packed := or(
                or(
                    shl(248, status),
                    shl(240, flags)
                ),
                or(
                    shl(80, addr),
                    amount
                )
            )
        }
    }

    function unpackData(uint256 packed) external pure returns (
        uint8 status,
        uint8 flags,
        address addr,
        uint80 amount
    ) {
        assembly {
            status := shr(248, packed)
            flags := and(shr(240, packed), 0xff)
            addr := and(shr(80, packed), 0xffffffffffffffffffffffffffffffffffffffff)
            amount := and(packed, 0xffffffffffffffffffff)
        }
    }

    // ============ 5. 高效的 keccak256 ============

    function efficientHash(bytes32 a, bytes32 b) external pure returns (bytes32 result) {
        assembly {
            // 使用 scratch space (0x00-0x40)
            mstore(0x00, a)
            mstore(0x20, b)
            result := keccak256(0x00, 0x40)
        }
    }

    function hashAddress(address a) external pure returns (bytes32 result) {
        assembly {
            mstore(0x00, a)
            result := keccak256(0x0c, 0x14)  // 只哈希 20 字节
        }
    }

    // ============ 6. 低级创建合约 ============

    function deployContract(bytes memory bytecode) external returns (address deployed) {
        assembly {
            deployed := create(
                0,                        // value
                add(bytecode, 0x20),     // 跳过长度前缀
                mload(bytecode)          // bytecode 长度
            )

            if iszero(deployed) {
                revert(0, 0)
            }
        }
    }

    function deployCreate2(
        bytes memory bytecode,
        bytes32 salt
    ) external returns (address deployed) {
        assembly {
            deployed := create2(
                0,                        // value
                add(bytecode, 0x20),
                mload(bytecode),
                salt
            )

            if iszero(deployed) {
                revert(0, 0)
            }
        }
    }
}
```

### AL.3 Yul 安全注意事项

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title Yul 安全最佳实践
 */
contract YulSecurity {

    // ❌ 危险：内存覆盖
    function unsafeMemory() external pure returns (uint256) {
        uint256 result;
        assembly {
            // 危险：可能覆盖 Solidity 管理的内存
            mstore(0x00, 123)  // 覆盖 scratch space 可能影响其他计算
            result := mload(0x00)
        }
        return result;
    }

    // ✅ 安全：使用空闲内存指针
    function safeMemory() external pure returns (uint256) {
        uint256 result;
        assembly {
            let ptr := mload(0x40)  // 获取空闲指针
            mstore(ptr, 123)
            result := mload(ptr)
            mstore(0x40, add(ptr, 32))  // 更新空闲指针
        }
        return result;
    }

    // ❌ 危险：未检查返回值
    function unsafeCall(address target) external {
        assembly {
            let success := call(gas(), target, 0, 0, 0, 0, 0)
            // 危险：忽略了 success！
        }
    }

    // ✅ 安全：检查返回值
    function safeCall(address target) external {
        assembly {
            let success := call(gas(), target, 0, 0, 0, 0, 0)
            if iszero(success) {
                // 复制错误数据并 revert
                returndatacopy(0, 0, returndatasize())
                revert(0, returndatasize())
            }
        }
    }

    // ❌ 危险：整数溢出
    function unsafeAdd(uint256 a, uint256 b) external pure returns (uint256) {
        assembly {
            mstore(0, add(a, b))  // 可能溢出！
            return (0, 32)
        }
    }

    // ✅ 安全：检查溢出
    function safeAdd(uint256 a, uint256 b) external pure returns (uint256 result) {
        assembly {
            result := add(a, b)
            if lt(result, a) {
                // 自定义错误 Overflow()
                mstore(0, 0x35278d12)  // selector
                revert(0, 4)
            }
        }
    }

    // ❌ 危险：存储槽计算错误
    function unsafeSlot() external {
        assembly {
            // 危险：硬编码的槽位可能与其他变量冲突
            sstore(100, 123)
        }
    }

    // ✅ 安全：使用唯一的存储槽
    function safeSlot() external {
        assembly {
            // 使用 keccak256 生成唯一槽位
            mstore(0, "my.unique.storage.slot")
            let slot := keccak256(0, 22)
            sstore(slot, 123)
        }
    }

    /**
     * Yul 安全检查清单：
     *
     * □ 内存安全
     *   - 使用 mload(0x40) 获取空闲指针
     *   - 使用后更新空闲指针
     *   - 注意 scratch space (0x00-0x40) 的临时性
     *
     * □ 存储安全
     *   - 正确计算 mapping/array 槽位
     *   - 避免硬编码槽位号
     *   - 使用 EIP-1967 风格的槽位
     *
     * □ 外部调用
     *   - 始终检查 call/delegatecall 返回值
     *   - 正确处理 returndatasize
     *   - 考虑 gas 限制
     *
     * □ 数值安全
     *   - 检查算术溢出
     *   - 注意有符号/无符号运算的差异
     *   - 使用 sdiv/smod 处理有符号数
     *
     * □ 类型安全
     *   - Yul 中所有值都是 256 位
     *   - 手动进行类型边界检查
     *   - 小心地址的高位清零
     */
}
```

---

## 附录 AM: 重要 EIP 详解

### AM.1 EIP-712: 类型化结构数据签名

EIP-712 定义了一种类型化结构数据的哈希和签名标准，解决了原始消息签名的可读性和安全性问题。

#### AM.1.1 问题背景

```solidity
// 传统签名方式 - 用户看到的是不可读的哈希
bytes32 messageHash = keccak256(abi.encodePacked(from, to, amount, nonce));
// 用户在钱包中看到: 0x7f83b166...（无法理解签名内容）

// EIP-712 - 用户看到结构化数据
// "Transfer 100 USDC from 0x123... to 0x456..."
```

#### AM.1.2 域分隔符 (Domain Separator)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EIP712Example {
    bytes32 public constant DOMAIN_TYPEHASH = keccak256(
        "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
    );

    bytes32 public immutable DOMAIN_SEPARATOR;

    constructor() {
        DOMAIN_SEPARATOR = keccak256(
            abi.encode(
                DOMAIN_TYPEHASH,
                keccak256(bytes("MyDApp")),           // name
                keccak256(bytes("1")),                // version
                block.chainid,                        // chainId
                address(this)                         // verifyingContract
            )
        );
    }
}
```

#### AM.1.3 结构化消息签名

```solidity
contract Permit {
    bytes32 public constant PERMIT_TYPEHASH = keccak256(
        "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
    );

    mapping(address => uint256) public nonces;

    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        require(block.timestamp <= deadline, "Permit expired");

        bytes32 structHash = keccak256(
            abi.encode(
                PERMIT_TYPEHASH,
                owner,
                spender,
                value,
                nonces[owner]++,
                deadline
            )
        );

        bytes32 digest = keccak256(
            abi.encodePacked(
                "\x19\x01",
                DOMAIN_SEPARATOR,
                structHash
            )
        );

        address recoveredAddress = ecrecover(digest, v, r, s);
        require(recoveredAddress != address(0) && recoveredAddress == owner, "Invalid signature");

        // 执行授权...
    }
}
```

#### AM.1.4 嵌套结构签名

```solidity
contract NestedEIP712 {
    // 嵌套类型需要按字母顺序排列
    bytes32 public constant ORDER_TYPEHASH = keccak256(
        "Order(address maker,Asset makerAsset,Asset takerAsset,uint256 expiry)"
        "Asset(address token,uint256 amount)"  // 嵌套类型定义
    );

    bytes32 public constant ASSET_TYPEHASH = keccak256(
        "Asset(address token,uint256 amount)"
    );

    struct Asset {
        address token;
        uint256 amount;
    }

    struct Order {
        address maker;
        Asset makerAsset;
        Asset takerAsset;
        uint256 expiry;
    }

    function hashAsset(Asset memory asset) internal pure returns (bytes32) {
        return keccak256(abi.encode(
            ASSET_TYPEHASH,
            asset.token,
            asset.amount
        ));
    }

    function hashOrder(Order memory order) internal pure returns (bytes32) {
        return keccak256(abi.encode(
            ORDER_TYPEHASH,
            order.maker,
            hashAsset(order.makerAsset),
            hashAsset(order.takerAsset),
            order.expiry
        ));
    }
}
```

### AM.2 EIP-1559: 费用市场改进

EIP-1559 引入了基础费用燃烧机制和优先费用小费模型，使 Gas 费用更可预测。

#### AM.2.1 费用结构

```
传统模式 (Pre-1559):
┌─────────────────────────────────┐
│  Gas Price × Gas Used = 费用    │
│  全部支付给矿工                  │
└─────────────────────────────────┘

EIP-1559 模式:
┌─────────────────────────────────┐
│  (Base Fee + Priority Fee) × Gas│
│                                 │
│  Base Fee → 燃烧 (销毁)         │
│  Priority Fee → 验证者          │
└─────────────────────────────────┘
```

#### AM.2.2 基础费用调整算法

```go
// go-ethereum 中的基础费用计算
func CalcBaseFee(config *params.ChainConfig, parent *types.Header) *big.Int {
// 目标 Gas: 区块限制的 50%
parentGasTarget := parent.GasLimit / 2

// 弹性因子
baseFeeChangeDenominator := uint64(8)

if parent.GasUsed == parentGasTarget {
// 使用量 = 目标，费用不变
return new(big.Int).Set(parent.BaseFee)
}

if parent.GasUsed > parentGasTarget {
// 使用量 > 目标，费用上涨
gasUsedDelta := parent.GasUsed - parentGasTarget
baseFeeChange := new(big.Int).Mul(parent.BaseFee, new(big.Int).SetUint64(gasUsedDelta))
baseFeeChange.Div(baseFeeChange, new(big.Int).SetUint64(parentGasTarget))
baseFeeChange.Div(baseFeeChange, new(big.Int).SetUint64(baseFeeChangeDenominator))

// 最小增加 1 wei
if baseFeeChange.Sign() == 0 {
baseFeeChange.SetUint64(1)
}

return new(big.Int).Add(parent.BaseFee, baseFeeChange)
} else {
// 使用量 < 目标，费用下降
gasUsedDelta := parentGasTarget - parent.GasUsed
baseFeeChange := new(big.Int).Mul(parent.BaseFee, new(big.Int).SetUint64(gasUsedDelta))
baseFeeChange.Div(baseFeeChange, new(big.Int).SetUint64(parentGasTarget))
baseFeeChange.Div(baseFeeChange, new(big.Int).SetUint64(baseFeeChangeDenominator))

return new(big.Int).Sub(parent.BaseFee, baseFeeChange)
}
}
```

#### AM.2.3 交易类型

```solidity
// Type 0: Legacy 交易
// gasPrice = 用户指定

// Type 1: EIP-2930 访问列表交易
// gasPrice = 用户指定
// accessList = [(address, [storageKeys])]

// Type 2: EIP-1559 交易
// maxFeePerGas = 用户愿意支付的最高费用
// maxPriorityFeePerGas = 给验证者的小费
// 实际费用 = min(maxFeePerGas, baseFee + maxPriorityFeePerGas)

contract GasEstimator {
    function estimateGasCost() external view returns (uint256) {
        // 获取当前基础费用
        uint256 baseFee = block.basefee;

        // 建议的优先费用 (通常 1-2 gwei)
        uint256 priorityFee = 2 gwei;

        // 估算的 Gas 使用量
        uint256 estimatedGas = 21000;

        // 总费用估算
        return (baseFee + priorityFee) * estimatedGas;
    }
}
```

### AM.3 EIP-4337: 账户抽象

EIP-4337 在不修改共识层的情况下实现账户抽象，允许使用智能合约钱包。

#### AM.3.1 核心架构

```
用户操作流程:
┌──────────┐    ┌─────────────┐    ┌────────────┐    ┌──────────┐
│  User    │───►│  Bundler    │───►│ EntryPoint │───►│ Account  │
│Operation │    │ (聚合者)    │    │ (入口点)   │    │ Contract │
└──────────┘    └─────────────┘    └────────────┘    └──────────┘
                      │                   │
                      │                   ▼
                      │            ┌────────────┐
                      │            │ Paymaster  │
                      │            │ (代付 Gas) │
                      │            └────────────┘
                      │
                      ▼
                ┌─────────────┐
                │  Mempool    │
                │ (专用池)    │
                └─────────────┘
```

#### AM.3.2 UserOperation 结构

```solidity
struct UserOperation {
    address sender;              // 智能合约钱包地址
    uint256 nonce;              // 防重放
    bytes initCode;             // 钱包创建代码 (首次使用)
    bytes callData;             // 要执行的调用数据
    uint256 callGasLimit;       // 执行调用的 Gas 限制
    uint256 verificationGasLimit; // 验证阶段 Gas 限制
    uint256 preVerificationGas; // 额外 Gas (补偿 bundler)
    uint256 maxFeePerGas;       // EIP-1559 最高费用
    uint256 maxPriorityFeePerGas; // EIP-1559 优先费用
    bytes paymasterAndData;     // Paymaster 地址和数据
    bytes signature;            // 签名
}
```

#### AM.3.3 智能合约钱包实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@account-abstraction/contracts/core/BaseAccount.sol";

contract SimpleAccount is BaseAccount {
    address public owner;
    IEntryPoint private immutable _entryPoint;

    constructor(IEntryPoint anEntryPoint, address anOwner) {
        _entryPoint = anEntryPoint;
        owner = anOwner;
    }

    function entryPoint() public view override returns (IEntryPoint) {
        return _entryPoint;
    }

    // 验证签名
    function _validateSignature(
        UserOperation calldata userOp,
        bytes32 userOpHash
    ) internal view override returns (uint256 validationData) {
        bytes32 hash = MessageHashUtils.toEthSignedMessageHash(userOpHash);

        if (owner != ECDSA.recover(hash, userOp.signature)) {
            return SIG_VALIDATION_FAILED;
        }
        return 0;
    }

    // 执行调用
    function execute(
        address dest,
        uint256 value,
        bytes calldata func
    ) external {
        _requireFromEntryPoint();
        (bool success, bytes memory result) = dest.call{value: value}(func);
        if (!success) {
            assembly {
                revert(add(result, 32), mload(result))
            }
        }
    }

    // 批量执行
    function executeBatch(
        address[] calldata dest,
        uint256[] calldata value,
        bytes[] calldata func
    ) external {
        _requireFromEntryPoint();
        require(dest.length == func.length && dest.length == value.length, "Length mismatch");

        for (uint256 i = 0; i < dest.length; i++) {
            (bool success, bytes memory result) = dest[i].call{value: value[i]}(func[i]);
            if (!success) {
                assembly {
                    revert(add(result, 32), mload(result))
                }
            }
        }
    }
}
```

#### AM.3.4 Paymaster 代付 Gas

```solidity
contract TokenPaymaster is BasePaymaster {
    IERC20 public token;
    uint256 public priceMarkup = 110; // 10% 加价

    constructor(IEntryPoint _entryPoint, IERC20 _token) BasePaymaster(_entryPoint) {
        token = _token;
    }

    // 验证是否愿意代付
    function _validatePaymasterUserOp(
        UserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 maxCost
    ) internal override returns (bytes memory context, uint256 validationData) {
        // 计算需要的代币数量
        uint256 tokenAmount = (maxCost * priceMarkup) / 100;

        // 检查用户代币余额
        require(
            token.balanceOf(userOp.sender) >= tokenAmount,
            "Insufficient token balance"
        );

        // 返回上下文供 postOp 使用
        return (abi.encode(userOp.sender, tokenAmount), 0);
    }

    // 操作后处理 - 收取代币
    function _postOp(
        PostOpMode mode,
        bytes calldata context,
        uint256 actualGasCost
    ) internal override {
        (address sender, uint256 maxTokenAmount) = abi.decode(context, (address, uint256));

        // 计算实际代币费用
        uint256 actualTokenCost = (actualGasCost * priceMarkup) / 100;

        // 从用户转移代币
        token.transferFrom(sender, address(this), actualTokenCost);
    }
}
```

### AM.4 EIP-4844: Blob 交易 (Proto-Danksharding)

EIP-4844 引入了一种新的交易类型，携带大量数据 blob，为 Layer 2 Rollup 提供廉价数据可用性。

#### AM.4.1 Blob 结构

```
Blob 规格:
┌─────────────────────────────────────┐
│  每个 Blob: 4096 × 32 bytes = 128KB │
│  每个区块最多: 6 个 blobs           │
│  每区块数据: 最多 768KB             │
│  KZG 承诺: 48 bytes                 │
│  KZG 证明: 48 bytes                 │
└─────────────────────────────────────┘

费用模型:
┌─────────────────────────────────────┐
│  Blob Gas 与 EIP-1559 类似          │
│  目标: 3 blobs/block               │
│  最大: 6 blobs/block               │
│  独立的 base fee 调整               │
└─────────────────────────────────────┘
```

#### AM.4.2 Blob 交易结构

```go
// Type 3 交易结构
type BlobTx struct {
ChainID              *uint256.Int
Nonce                uint64
GasTipCap            *uint256.Int // maxPriorityFeePerGas
GasFeeCap            *uint256.Int  // maxFeePerGas
Gas                  uint64
To                   common.Address
Value                *uint256.Int
Data                 []byte
AccessList           AccessList
BlobFeeCap           *uint256.Int // maxFeePerBlobGas
BlobHashes           []common.Hash // blob 版本化哈希

// Sidecar (不进入执行层)
Sidecar *BlobTxSidecar
}

type BlobTxSidecar struct {
Blobs       []kzg4844.Blob // 实际 blob 数据
Commitments []kzg4844.Commitment // KZG 承诺
Proofs      []kzg4844.Proof      // KZG 证明
}
```

#### AM.4.3 KZG 承诺验证

```go
// KZG 承诺方案
// 用于验证 blob 数据的正确性，无需下载完整数据

func VerifyBlobProof(blob Blob, commitment Commitment, proof Proof) bool {
// 1. 计算 blob 的 KZG 承诺
computedCommitment := ComputeBlobCommitment(blob)

// 2. 验证承诺匹配
if computedCommitment != commitment {
return false
}

// 3. 验证证明
return VerifyKZGProof(commitment, proof)
}

// 版本化哈希计算
func BlobHash(commitment Commitment) common.Hash {
// VERSIONED_HASH_VERSION_KZG = 0x01
hash := sha256.Sum256(commitment[:])
hash[0] = 0x01 // 设置版本字节
return common.Hash(hash)
}
```

#### AM.4.4 Rollup 使用 Blob 数据

```solidity
contract RollupBlobConsumer {
    // 新的操作码 BLOBHASH (0x49)
    // 返回当前交易中第 i 个 blob 的版本化哈希

    function verifyBlobData(
        uint256 blobIndex,
        bytes32 expectedHash
    ) external view returns (bool) {
        bytes32 blobHash;
        assembly {
            // BLOBHASH 操作码
            blobHash := blobhash(blobIndex)
        }
        return blobHash == expectedHash;
    }

    // 新的预编译: Point Evaluation (0x0A)
    // 验证 blob 在特定点的值
    function verifyBlobPoint(
        bytes32 versionedHash,
        bytes32 z,           // 评估点
        bytes32 y,           // 期望值
        bytes calldata commitment,
        bytes calldata proof
    ) external view returns (bool) {
        bytes memory input = abi.encodePacked(
            versionedHash,
            z,
            y,
            commitment,
            proof
        );

        (bool success, bytes memory result) = address(0x0A).staticcall(input);
        return success && abi.decode(result, (bool));
    }
}
```

#### AM.4.5 Blob 费用市场

```go
// Blob 基础费用计算 (类似 EIP-1559)
func CalcBlobFee(excessBlobGas uint64) *big.Int {
// 每个 blob 消耗 131072 (2^17) gas
// 目标: 3 blobs = 393216 blob gas
// 最大: 6 blobs = 786432 blob gas

return fakeExponential(
big.NewInt(1),          // 最小费用: 1 wei
big.NewInt(int64(excessBlobGas)),
big.NewInt(3338477), // blob 费用更新分数
)
}

// 指数函数近似
func fakeExponential(factor, numerator, denominator *big.Int) *big.Int {
// 返回 factor * e^(numerator/denominator)
// 使用泰勒展开近似
}
```

### AM.5 EIP-7702: 设置 EOA 代码

EIP-7702 允许 EOA（外部拥有账户）临时设置代码，实现账户抽象功能。

#### AM.5.1 授权机制

```go
// 授权元组结构
type SetCodeAuthorization struct {
ChainId uint64         // 链 ID (0 表示任意链)
Address common.Address // 要委托的合约地址
Nonce   uint64         // 签名者当前 nonce
V, R, S                // ECDSA 签名
}

// 授权签名消息
// MAGIC = 0x05
// message = keccak256(MAGIC || rlp([chain_id, address, nonce]))
```

#### AM.5.2 Type 4 交易

```solidity
// Type 4 交易允许 EOA 临时拥有代码
// 这使得 EOA 可以:
// 1. 批量执行多个操作
// 2. 使用 Paymaster 代付 Gas
// 3. 实现社交恢复等高级功能

contract DelegateImplementation {
    // EOA 可以将代码设置为此合约
    // 然后 EOA 就拥有了这些能力

    function batchExecute(
        address[] calldata targets,
        bytes[] calldata datas
    ) external {
        for (uint i = 0; i < targets.length; i++) {
            (bool success,) = targets[i].call(datas[i]);
            require(success);
        }
    }
}
```

---

## 附录 AN: 预编译合约详解

预编译合约是在 EVM 中以原生代码实现的特殊合约，提供高效的密码学和工具函数。

### AN.1 预编译合约列表

| 地址   | 名称                   | 功能            | Gas 成本             |
|------|----------------------|---------------|--------------------|
| 0x01 | ECRECOVER            | ECDSA 签名恢复    | 3000               |
| 0x02 | SHA256               | SHA-256 哈希    | 60 + 12/word       |
| 0x03 | RIPEMD160            | RIPEMD-160 哈希 | 600 + 120/word     |
| 0x04 | IDENTITY             | 数据复制          | 15 + 3/word        |
| 0x05 | MODEXP               | 大数模幂运算        | 动态计算               |
| 0x06 | ECADD                | BN254 椭圆曲线点加  | 150                |
| 0x07 | ECMUL                | BN254 椭圆曲线标量乘 | 6000               |
| 0x08 | ECPAIRING            | BN254 配对检查    | 45000 + 34000/pair |
| 0x09 | BLAKE2F              | BLAKE2 压缩函数   | 1/round            |
| 0x0A | KZG Point Evaluation | KZG 承诺点验证     | 50000              |

### AN.2 ECRECOVER (0x01)

```solidity
contract ECRecoverExample {
    // 从签名恢复签名者地址
    function recoverSigner(
        bytes32 messageHash,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public pure returns (address) {
        // 直接调用预编译
        return ecrecover(messageHash, v, r, s);
    }

    // 使用 assembly 调用
    function recoverSignerAssembly(
        bytes32 messageHash,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public view returns (address signer) {
        assembly {
            // 准备输入数据
            let ptr := mload(0x40)
            mstore(ptr, messageHash)
            mstore(add(ptr, 0x20), v)
            mstore(add(ptr, 0x40), r)
            mstore(add(ptr, 0x60), s)

            // 调用预编译 0x01
            let success := staticcall(
                gas(),
                0x01,
                ptr,
                0x80,      // 输入长度: 128 bytes
                ptr,
                0x20       // 输出长度: 32 bytes
            )

            if success {
                signer := mload(ptr)
            }
        }
    }

    // EIP-712 签名验证完整示例
    function verifyTypedSignature(
        bytes32 domainSeparator,
        bytes32 structHash,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public pure returns (address) {
        bytes32 digest = keccak256(
            abi.encodePacked(
                "\x19\x01",
                domainSeparator,
                structHash
            )
        );
        return ecrecover(digest, v, r, s);
    }
}
```

### AN.3 SHA256 与 RIPEMD160 (0x02, 0x03)

```solidity
contract HashPrecompiles {
    // SHA256 - 用于比特币兼容性
    function sha256Hash(bytes memory data) public view returns (bytes32 result) {
        assembly {
            let len := mload(data)
            let success := staticcall(
                gas(),
                0x02,
                add(data, 0x20),
                len,
                result,
                0x20
            )
            if iszero(success) {revert(0, 0)}
        }
    }

    // RIPEMD160 - 比特币地址生成
    function ripemd160Hash(bytes memory data) public view returns (bytes20 result) {
        bytes32 temp;
        assembly {
            let len := mload(data)
            let success := staticcall(
                gas(),
                0x03,
                add(data, 0x20),
                len,
                temp,
                0x20
            )
            if iszero(success) {revert(0, 0)}
        }
        result = bytes20(temp << 96); // RIPEMD160 输出右对齐
    }

    // 比特币地址计算: RIPEMD160(SHA256(pubkey))
    function bitcoinAddress(bytes memory pubkey) public view returns (bytes20) {
        bytes32 sha = sha256(pubkey);
        return ripemd160Hash(abi.encodePacked(sha));
    }
}
```

### AN.4 MODEXP (0x05) - 大数模幂运算

```solidity
contract ModExpExample {
    // RSA 验证、Miller-Rabin 素性测试等
    function modExp(
        bytes memory base,
        bytes memory exponent,
        bytes memory modulus
    ) public view returns (bytes memory result) {
        uint256 baseLen = base.length;
        uint256 expLen = exponent.length;
        uint256 modLen = modulus.length;

        // 构建输入
        bytes memory input = abi.encodePacked(
            uint256(baseLen),
            uint256(expLen),
            uint256(modLen),
            base,
            exponent,
            modulus
        );

        result = new bytes(modLen);

        assembly {
            let success := staticcall(
                gas(),
                0x05,
                add(input, 0x20),
                mload(input),
                add(result, 0x20),
                modLen
            )
            if iszero(success) {revert(0, 0)}
        }
    }

    // RSA 签名验证示例
    function verifyRSA(
        bytes memory signature,
        bytes memory exponent,  // 通常是 65537 = 0x010001
        bytes memory modulus,
        bytes32 messageHash
    ) public view returns (bool) {
        bytes memory decrypted = modExp(signature, exponent, modulus);

        // 验证 PKCS#1 v1.5 填充格式
        // 0x00 0x01 [padding] 0x00 [DigestInfo] [hash]
        // 简化版本：直接比较哈希
        bytes32 recoveredHash;
        assembly {
            recoveredHash := mload(add(decrypted, mload(decrypted)))
        }

        return recoveredHash == messageHash;
    }
}
```

### AN.5 BN254 椭圆曲线操作 (0x06, 0x07, 0x08)

```solidity
contract BN254Precompiles {
    // G1 点结构
    struct G1Point {
        uint256 x;
        uint256 y;
    }

    // G2 点结构 (在扩展域上)
    struct G2Point {
        uint256[2] x;  // x = x0 + x1 * u
        uint256[2] y;  // y = y0 + y1 * u
    }

    // 椭圆曲线点加 (0x06)
    function ecAdd(G1Point memory p1, G1Point memory p2)
    internal view returns (G1Point memory r)
    {
        uint256[4] memory input;
        input[0] = p1.x;
        input[1] = p1.y;
        input[2] = p2.x;
        input[3] = p2.y;

        uint256[2] memory output;

        assembly {
            if iszero(staticcall(gas(), 0x06, input, 0x80, output, 0x40)) {
                revert(0, 0)
            }
        }

        r.x = output[0];
        r.y = output[1];
    }

    // 标量乘法 (0x07)
    function ecMul(G1Point memory p, uint256 scalar)
    internal view returns (G1Point memory r)
    {
        uint256[3] memory input;
        input[0] = p.x;
        input[1] = p.y;
        input[2] = scalar;

        uint256[2] memory output;

        assembly {
            if iszero(staticcall(gas(), 0x07, input, 0x60, output, 0x40)) {
                revert(0, 0)
            }
        }

        r.x = output[0];
        r.y = output[1];
    }

    // 配对检查 (0x08) - 用于 zkSNARK 验证
    // 验证: e(p1[0], p2[0]) * e(p1[1], p2[1]) * ... == 1
    function pairing(
        G1Point[] memory p1,
        G2Point[] memory p2
    ) internal view returns (bool) {
        require(p1.length == p2.length, "Length mismatch");

        uint256 elements = p1.length;
        uint256 inputSize = elements * 6 * 32;
        uint256[] memory input = new uint256[](elements * 6);

        for (uint256 i = 0; i < elements; i++) {
            input[i * 6 + 0] = p1[i].x;
            input[i * 6 + 1] = p1[i].y;
            input[i * 6 + 2] = p2[i].x[1];  // 注意顺序
            input[i * 6 + 3] = p2[i].x[0];
            input[i * 6 + 4] = p2[i].y[1];
            input[i * 6 + 5] = p2[i].y[0];
        }

        uint256[1] memory output;

        assembly {
            if iszero(staticcall(gas(), 0x08, add(input, 0x20), inputSize, output, 0x20)) {
                revert(0, 0)
            }
        }

        return output[0] == 1;
    }
}
```

### AN.6 Groth16 zkSNARK 验证器

```solidity
contract Groth16Verifier {
    using BN254Precompiles for *;

    // 验证密钥 (由可信设置生成)
    struct VerifyingKey {
        G1Point alpha;
        G2Point beta;
        G2Point gamma;
        G2Point delta;
        G1Point[] ic;  // 公共输入承诺
    }

    // 证明结构
    struct Proof {
        G1Point a;
        G2Point b;
        G1Point c;
    }

    function verify(
        VerifyingKey memory vk,
        Proof memory proof,
        uint256[] memory publicInputs
    ) public view returns (bool) {
        require(publicInputs.length + 1 == vk.ic.length, "Invalid inputs");

        // 计算 vk_x = ic[0] + sum(publicInputs[i] * ic[i+1])
        G1Point memory vk_x = vk.ic[0];
        for (uint256 i = 0; i < publicInputs.length; i++) {
            vk_x = ecAdd(vk_x, ecMul(vk.ic[i + 1], publicInputs[i]));
        }

        // 配对检查
        // e(proof.a, proof.b) == e(vk.alpha, vk.beta) * e(vk_x, vk.gamma) * e(proof.c, vk.delta)
        // 等价于: e(-proof.a, proof.b) * e(vk.alpha, vk.beta) * e(vk_x, vk.gamma) * e(proof.c, vk.delta) == 1

        G1Point[] memory p1 = new G1Point[](4);
        G2Point[] memory p2 = new G2Point[](4);

        p1[0] = negate(proof.a);
        p2[0] = proof.b;
        p1[1] = vk.alpha;
        p2[1] = vk.beta;
        p1[2] = vk_x;
        p2[2] = vk.gamma;
        p1[3] = proof.c;
        p2[3] = vk.delta;

        return pairing(p1, p2);
    }

    // 点取反
    function negate(G1Point memory p) internal pure returns (G1Point memory) {
        uint256 q = 21888242871839275222246405745257275088696311157297823662689037894645226208583;
        if (p.x == 0 && p.y == 0) {
            return G1Point(0, 0);
        }
        return G1Point(p.x, q - (p.y % q));
    }
}
```

### AN.7 BLAKE2F (0x09)

```solidity
contract Blake2FExample {
    // BLAKE2 压缩函数 - 用于 Zcash 等
    function blake2f(
        uint32 rounds,
        bytes32[2] memory h,      // 状态向量
        bytes32[4] memory m,      // 消息块
        bytes8[2] memory t,       // 偏移计数器
        bool lastBlock
    ) public view returns (bytes32[2] memory) {
        bytes memory input = abi.encodePacked(
            rounds,
            h[0], h[1],
            m[0], m[1], m[2], m[3],
            t[0], t[1],
            lastBlock ? bytes1(0x01) : bytes1(0x00)
        );

        bytes32[2] memory output;

        assembly {
            if iszero(staticcall(gas(), 0x09, add(input, 0x20), 213, output, 0x40)) {
                revert(0, 0)
            }
        }

        return output;
    }
}
```

### AN.8 KZG Point Evaluation (0x0A) - EIP-4844

```solidity
contract KZGVerifier {
    address constant POINT_EVALUATION_PRECOMPILE = address(0x0A);

    // 验证 blob 在点 z 的值为 y
    function verifyKZGProof(
        bytes32 versionedHash,  // blob 版本化哈希
        bytes32 z,              // 评估点
        bytes32 y,              // 期望值
        bytes memory commitment, // 48 bytes KZG 承诺
        bytes memory proof       // 48 bytes KZG 证明
    ) public view returns (bool) {
        bytes memory input = abi.encodePacked(
            versionedHash,
            z,
            y,
            commitment,
            proof
        );

        (bool success, bytes memory result) = POINT_EVALUATION_PRECOMPILE.staticcall(input);

        if (!success || result.length != 64) {
            return false;
        }

        // 返回字段元素和 BLS 模数
        return true;
    }
}
```

### AN.9 Go-ethereum 预编译实现

```go
// core/vm/contracts.go
var PrecompiledContractsBerlin = map[common.Address]PrecompiledContract{
common.BytesToAddress([]byte{1}): &ecrecover{},
common.BytesToAddress([]byte{2}): &sha256hash{},
common.BytesToAddress([]byte{3}): &ripemd160hash{},
common.BytesToAddress([]byte{4}): &dataCopy{},
common.BytesToAddress([]byte{5}): &bigModExp{eip2565: true},
common.BytesToAddress([]byte{6}): &bn256AddIstanbul{},
common.BytesToAddress([]byte{7}): &bn256ScalarMulIstanbul{},
common.BytesToAddress([]byte{8}): &bn256PairingIstanbul{},
common.BytesToAddress([]byte{9}): &blake2F{},
}

// ecrecover 实现
type ecrecover struct{}

func (c *ecrecover) Run(input []byte) ([]byte, error) {
const ecRecoverInputLength = 128

input = common.RightPadBytes(input, ecRecoverInputLength)

// 解析输入
r := new(big.Int).SetBytes(input[64:96])
s := new(big.Int).SetBytes(input[96:128])
v := input[63] - 27

// 验证签名分量
if !allZero(input[32:63]) || !crypto.ValidateSignatureValues(v, r, s, false) {
return nil, nil
}

// 构造签名
sig := make([]byte, 65)
copy(sig[32-len(r.Bytes()):32], r.Bytes())
copy(sig[64-len(s.Bytes()):64], s.Bytes())
sig[64] = v

// 恢复公钥
pubKey, err := crypto.Ecrecover(input[:32], sig)
if err != nil {
return nil, nil
}

// 返回地址
return common.LeftPadBytes(crypto.Keccak256(pubKey[1:])[12:], 32), nil
}
```

---

## 附录 AO: MEV 与 Flashbots

### AO.1 MEV 概述

MEV (Maximal Extractable Value) 是指区块生产者通过重新排序、插入或审查交易可以提取的最大价值。

#### AO.1.1 MEV 类型

```
MEV 分类:
┌─────────────────────────────────────────────────────────┐
│  套利 (Arbitrage)                                       │
│  ├── DEX 套利: 不同 DEX 间的价格差异                    │
│  ├── CEX-DEX 套利: 中心化与去中心化交易所间套利         │
│  └── 三角套利: A→B→C→A 循环套利                        │
├─────────────────────────────────────────────────────────┤
│  三明治攻击 (Sandwich Attack)                           │
│  ├── 前置交易: 在目标交易前执行                         │
│  ├── 目标交易: 受害者的交易                             │
│  └── 后置交易: 在目标交易后执行获利                     │
├─────────────────────────────────────────────────────────┤
│  清算 (Liquidation)                                     │
│  └── 监控借贷协议，抢先清算抵押不足的头寸               │
├─────────────────────────────────────────────────────────┤
│  NFT 抢购                                               │
│  └── 监控热门 NFT mint，抢先购买                        │
└─────────────────────────────────────────────────────────┘
```

#### AO.1.2 MEV 提取流程

```
                    Mempool 监控
                         │
                         ▼
┌─────────────────────────────────────────┐
│           交易发现与分析                │
│  • 解码交易数据                         │
│  • 模拟交易结果                         │
│  • 计算潜在利润                         │
└─────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────┐
│           策略执行                      │
│  • 构建 MEV 交易                        │
│  • 计算最优 Gas 价格                    │
│  • 发送到 Flashbots/区块构建者          │
└─────────────────────────────────────────┘
```

### AO.2 三明治攻击详解

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);

    function getAmountsOut(uint amountIn, address[] calldata path)
    external view returns (uint[] memory amounts);
}

contract SandwichAnalysis {
    // 三明治攻击原理:
    //
    // 1. 受害者交易: swap 100 ETH → USDC (滑点 1%)
    //    amountOutMin = 预期输出 * 0.99
    //
    // 2. 攻击者前置交易:
    //    买入大量 USDC，推高价格
    //
    // 3. 受害者交易执行:
    //    以更高价格买入，获得更少 USDC
    //
    // 4. 攻击者后置交易:
    //    卖出 USDC，获取利润

    function calculateSandwichProfit(
        uint256 victimAmountIn,
        uint256 attackerAmountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) public pure returns (uint256 profit) {
        // 使用恒定乘积公式 x * y = k

        // 攻击者前置交易
        uint256 attackerOut = getAmountOut(attackerAmountIn, reserveIn, reserveOut);
        uint256 newReserveIn = reserveIn + attackerAmountIn;
        uint256 newReserveOut = reserveOut - attackerOut;

        // 受害者交易
        uint256 victimOut = getAmountOut(victimAmountIn, newReserveIn, newReserveOut);
        newReserveIn = newReserveIn + victimAmountIn;
        newReserveOut = newReserveOut - victimOut;

        // 攻击者后置交易 (卖出获得的代币)
        uint256 attackerReturn = getAmountOut(attackerOut, newReserveOut, newReserveIn);

        // 利润 = 返回值 - 投入值 - Gas 成本
        if (attackerReturn > attackerAmountIn) {
            profit = attackerReturn - attackerAmountIn;
        }
    }

    function getAmountOut(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) internal pure returns (uint256) {
        uint256 amountInWithFee = amountIn * 997;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = reserveIn * 1000 + amountInWithFee;
        return numerator / denominator;
    }
}
```

### AO.3 Flashbots 架构

```
Flashbots 生态系统:
┌─────────────────────────────────────────────────────────────┐
│                      Searcher (搜索者)                       │
│  • 寻找 MEV 机会                                             │
│  • 构建交易包 (Bundle)                                       │
│  • 发送到 Flashbots Relay                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Flashbots Relay                           │
│  • 接收 Bundle                                               │
│  • 模拟执行                                                  │
│  • 转发给区块构建者                                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Block Builder (区块构建者)                 │
│  • 整合多个 Bundle                                           │
│  • 最大化区块价值                                            │
│  • 提交给 Proposer                                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Proposer (提议者/验证者)                   │
│  • 选择最高价值的区块                                        │
│  • 提议并验证区块                                            │
└─────────────────────────────────────────────────────────────┘
```

### AO.4 Flashbots Bundle 发送

```go
// 使用 Go 发送 Flashbots Bundle
package main

import (
	"context"
	"crypto/ecdsa"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/metachris/flashbotsrpc"
)

func sendFlashbotsBundle() error {
	// Flashbots RPC 端点
	rpc := flashbotsrpc.New("https://relay.flashbots.net")

	// 签名密钥 (用于身份验证，不需要持有 ETH)
	privateKey, _ := crypto.GenerateKey()

	// 构建交易
	tx1 := buildFrontrunTx() // 前置交易
	tx2 := buildBackrunTx()  // 后置交易

	// 创建 Bundle
	bundle := flashbotsrpc.FlashbotsSendBundleRequest{
		Txs:         []string{rawTxHex(tx1), rawTxHex(tx2)},
		BlockNumber: targetBlockNumber,
	}

	// 发送 Bundle
	result, err := rpc.FlashbotsSendBundle(
		privateKey,
		bundle,
	)
	if err != nil {
		return err
	}

	// 检查模拟结果
	if result.Error != nil {
		return result.Error
	}

	return nil
}

// Bundle 模拟
func simulateBundle(rpc *flashbotsrpc.FlashbotsRPC, privateKey *ecdsa.PrivateKey) {
	simRequest := flashbotsrpc.FlashbotsCallBundleParam{
		Txs:              []string{rawTxHex(tx1), rawTxHex(tx2)},
		BlockNumber:      targetBlockNumber,
		StateBlockNumber: "latest",
	}

	result, err := rpc.FlashbotsCallBundle(privateKey, simRequest)
	if err != nil {
		panic(err)
	}

	// 分析模拟结果
	for _, txResult := range result.Results {
		fmt.Printf("Tx: %s\n", txResult.TxHash)
		fmt.Printf("Gas Used: %d\n", txResult.GasUsed)
		fmt.Printf("Gas Price: %s\n", txResult.GasPrice)
		fmt.Printf("Coinbase Transfer: %s\n", txResult.CoinbaseTransfer)
	}
}
```

### AO.5 MEV-Boost (PBS)

```
Proposer-Builder Separation (PBS):

┌──────────┐     ┌───────────┐     ┌───────────┐     ┌──────────┐
│ Searcher │────►│  Builder  │────►│   Relay   │────►│ Proposer │
└──────────┘     └───────────┘     └───────────┘     └──────────┘
     │                 │                 │                 │
     │   发送 Bundle   │   构建区块      │   传递区块头   │  选择最高出价
     │                 │                 │                 │
     └─────────────────┴─────────────────┴─────────────────┘

工作流程:
1. Searcher 发送 Bundle 给 Builder
2. Builder 整合 Bundle，构建最优区块
3. Builder 向 Relay 提交区块和出价
4. Relay 验证区块有效性
5. Proposer 从 Relay 获取最高出价的区块头
6. Proposer 签名区块头
7. Relay 公开完整区块
```

```go
// MEV-Boost 配置示例
type MEVBoostConfig struct {
Relays []string // Relay 端点列表
}

var defaultRelays = []string{
"https://0xac6e77dfe25ecd6110b8e780608cce0dab71fdd5ebea22a16c0205200f2f8e2e3ad3b71d3499c54ad14d6c21b41a37ae@boost-relay.flashbots.net",
"https://0xa1559ace749633b997cb3fdacffb890aeebdb0f5a3b6aaa7eeeaf1a38af0a8fe88b9e4b1f61f236d2e64d95733327a62@relay.ultrasound.money",
"https://0x8b5d2e73e2a3a55c6c87b8b6eb92e0149a125c852751db1422fa951e42a09b82c142c3ea98d0d9930b056a3bc9896b8f@bloxroute.max-profit.blxrbdn.com",
}
```

### AO.6 MEV 保护策略

```solidity
contract MEVProtection {
    // 1. 提交-揭示方案
    mapping(bytes32 => uint256) public commitments;

    function commit(bytes32 hash) external {
        commitments[hash] = block.number;
    }

    function reveal(uint256 amount, bytes32 salt) external {
        bytes32 hash = keccak256(abi.encodePacked(msg.sender, amount, salt));
        require(commitments[hash] != 0, "No commitment");
        require(block.number > commitments[hash], "Too early");
        require(block.number <= commitments[hash] + 256, "Too late");

        // 执行交易...
        delete commitments[hash];
    }

    // 2. 时间锁定
    uint256 public constant DELAY = 2; // 2 个区块

    mapping(bytes32 => uint256) public pendingTxs;

    function queueTx(bytes32 txHash) external {
        pendingTxs[txHash] = block.number + DELAY;
    }

    function executeTx(bytes32 txHash) external {
        require(block.number >= pendingTxs[txHash], "Not ready");
        // 执行...
    }

    // 3. 私有交易池 (通过 Flashbots Protect)
    // 用户通过 https://protect.flashbots.net 发送交易
    // 交易不会进入公共 mempool

    // 4. 滑点保护
    function swapWithProtection(
        uint256 amountIn,
        uint256 minAmountOut,
        uint256 maxSlippageBps // 基点
    ) external {
        uint256 expectedOut = getExpectedOutput(amountIn);
        uint256 minAllowed = expectedOut * (10000 - maxSlippageBps) / 10000;

        require(minAmountOut >= minAllowed, "Slippage too high");

        // 执行 swap...
    }
}
```

### AO.7 套利机器人示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@uniswap/v2-periphery/contracts/interfaces/IUniswapV2Router02.sol";
import "@aave/v3-core/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";

contract ArbitrageBot is FlashLoanSimpleReceiverBase {
    address public owner;

    constructor(IPoolAddressesProvider provider)
    FlashLoanSimpleReceiverBase(provider)
    {
        owner = msg.sender;
    }

    // 执行套利
    function executeArbitrage(
        address token,
        uint256 amount,
        bytes calldata params
    ) external {
        require(msg.sender == owner, "Not owner");

        // 请求闪电贷
        POOL.flashLoanSimple(
            address(this),
            token,
            amount,
            params,
            0 // referral code
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
        require(msg.sender == address(POOL), "Invalid caller");
        require(initiator == address(this), "Invalid initiator");

        // 解码套利参数
        (
            address router1,
            address router2,
            address[] memory path1,
            address[] memory path2
        ) = abi.decode(params, (address, address, address[], address[]));

        // 在 DEX1 买入
        IERC20(asset).approve(router1, amount);
        uint256[] memory amounts1 = IUniswapV2Router02(router1)
            .swapExactTokensForTokens(
            amount,
            0,
            path1,
            address(this),
            block.timestamp
        );

        // 在 DEX2 卖出
        uint256 intermediateAmount = amounts1[amounts1.length - 1];
        address intermediateToken = path1[path1.length - 1];
        IERC20(intermediateToken).approve(router2, intermediateAmount);
        uint256[] memory amounts2 = IUniswapV2Router02(router2)
            .swapExactTokensForTokens(
            intermediateAmount,
            0,
            path2,
            address(this),
            block.timestamp
        );

        // 验证利润
        uint256 finalAmount = amounts2[amounts2.length - 1];
        uint256 amountOwed = amount + premium;
        require(finalAmount > amountOwed, "No profit");

        // 还款
        IERC20(asset).approve(address(POOL), amountOwed);

        return true;
    }

    // 提取利润
    function withdraw(address token) external {
        require(msg.sender == owner, "Not owner");
        uint256 balance = IERC20(token).balanceOf(address(this));
        IERC20(token).transfer(owner, balance);
    }
}
```

---

## 附录 AP: Layer 2 扩容方案

### AP.1 Layer 2 概述

```
Layer 2 分类:
┌─────────────────────────────────────────────────────────────┐
│                    State Channels                            │
│  • 链下交易，仅在开启/关闭时上链                            │
│  • 示例: Lightning Network, Raiden                          │
├─────────────────────────────────────────────────────────────┤
│                    Plasma                                    │
│  • 子链提交根哈希到主链                                      │
│  • 退出需要挑战期                                            │
├─────────────────────────────────────────────────────────────┤
│                    Optimistic Rollup                         │
│  • 乐观假设交易有效                                          │
│  • 欺诈证明机制                                              │
│  • 7天挑战期                                                 │
│  • 示例: Optimism, Arbitrum                                  │
├─────────────────────────────────────────────────────────────┤
│                    ZK Rollup                                 │
│  • 零知识证明验证                                            │
│  • 即时最终性                                                │
│  • 示例: zkSync, StarkNet, Polygon zkEVM                    │
├─────────────────────────────────────────────────────────────┤
│                    Validium                                  │
│  • ZK 证明 + 链下数据可用性                                  │
│  • 更高吞吐量，较弱安全保证                                  │
└─────────────────────────────────────────────────────────────┘
```

### AP.2 Optimistic Rollup 原理

```
Optimistic Rollup 工作流程:

1. 交易提交
┌─────────┐     ┌───────────┐     ┌──────────┐
│  用户   │────►│ Sequencer │────►│ L1 合约  │
└─────────┘     └───────────┘     └──────────┘
                     │
                     ▼
              批量打包交易
              提交到 L1

2. 状态更新
┌──────────────────────────────────────────┐
│  Sequencer 提交:                         │
│  • 交易数据 (calldata)                   │
│  • 状态根                                │
│  • 签名                                  │
└──────────────────────────────────────────┘

3. 欺诈证明
┌──────────────────────────────────────────┐
│  挑战期 (通常 7 天):                     │
│  • 任何人可以提交欺诈证明                │
│  • 证明状态转换无效                      │
│  • 成功挑战获得奖励                      │
│  • Sequencer 被惩罚                      │
└──────────────────────────────────────────┘
```

```solidity
// Optimistic Rollup 核心合约 (简化版)
contract OptimisticRollup {
    struct StateCommitment {
        bytes32 stateRoot;
        uint256 blockNumber;
        uint256 timestamp;
        address proposer;
    }

    StateCommitment[] public stateCommitments;
    uint256 public constant CHALLENGE_PERIOD = 7 days;

    mapping(uint256 => bool) public challenged;
    mapping(uint256 => bool) public finalized;

    // 提交状态
    function proposeState(
        bytes32 stateRoot,
        bytes calldata txBatch
    ) external {
        // 验证 Sequencer 身份
        require(isSequencer(msg.sender), "Not sequencer");

        stateCommitments.push(StateCommitment({
            stateRoot: stateRoot,
            blockNumber: block.number,
            timestamp: block.timestamp,
            proposer: msg.sender
        }));

        // 存储交易数据 (用于数据可用性)
        emit StateBatchAppended(stateCommitments.length - 1, stateRoot, txBatch);
    }

    // 欺诈证明
    function challengeState(
        uint256 stateIndex,
        bytes calldata fraudProof
    ) external {
        StateCommitment storage commitment = stateCommitments[stateIndex];

        require(!finalized[stateIndex], "Already finalized");
        require(
            block.timestamp < commitment.timestamp + CHALLENGE_PERIOD,
            "Challenge period ended"
        );

        // 验证欺诈证明
        require(verifyFraudProof(stateIndex, fraudProof), "Invalid proof");

        // 回滚状态
        challenged[stateIndex] = true;

        // 惩罚 Sequencer，奖励挑战者
        slashSequencer(commitment.proposer);
        rewardChallenger(msg.sender);
    }

    // 确认状态
    function finalizeState(uint256 stateIndex) external {
        StateCommitment storage commitment = stateCommitments[stateIndex];

        require(!challenged[stateIndex], "State was challenged");
        require(
            block.timestamp >= commitment.timestamp + CHALLENGE_PERIOD,
            "Challenge period not ended"
        );

        finalized[stateIndex] = true;
    }
}
```

### AP.3 ZK Rollup 原理

```
ZK Rollup 工作流程:

1. 交易收集
┌─────────┐     ┌───────────┐
│  用户   │────►│ Sequencer │
└─────────┘     └───────────┘
                     │
                     ▼
              收集交易批次

2. 证明生成
┌──────────────────────────────────────────┐
│  ZK Prover:                              │
│  • 执行所有交易                          │
│  • 计算新状态根                          │
│  • 生成有效性证明 (SNARK/STARK)          │
└──────────────────────────────────────────┘
                     │
                     ▼

3. 链上验证
┌──────────────────────────────────────────┐
│  L1 Verifier 合约:                       │
│  • 验证 ZK 证明                          │
│  • 更新状态根                            │
│  • 即时最终性                            │
└──────────────────────────────────────────┘
```

```solidity
// ZK Rollup 验证合约 (简化版)
contract ZKRollup {
    bytes32 public stateRoot;

    IVerifier public verifier;  // ZK 证明验证器

    struct BatchHeader {
        bytes32 newStateRoot;
        bytes32 txDataHash;
        uint256 batchIndex;
    }

    // 提交批次
    function submitBatch(
        BatchHeader calldata header,
        bytes calldata proof,
        bytes calldata publicInputs
    ) external {
        // 构建公共输入
        bytes32 inputHash = keccak256(abi.encode(
            stateRoot,           // 旧状态根
            header.newStateRoot, // 新状态根
            header.txDataHash    // 交易数据哈希
        ));

        // 验证 ZK 证明
        require(
            verifier.verify(proof, publicInputs),
            "Invalid proof"
        );

        // 验证公共输入
        require(
            keccak256(publicInputs) == inputHash,
            "Invalid public inputs"
        );

        // 更新状态 (即时最终性)
        stateRoot = header.newStateRoot;

        emit BatchSubmitted(header.batchIndex, header.newStateRoot);
    }
}

// Groth16 验证器接口
interface IVerifier {
    function verify(
        bytes calldata proof,
        bytes calldata publicInputs
    ) external view returns (bool);
}
```

### AP.4 数据可用性

```
数据可用性方案:

┌─────────────────────────────────────────────────────────────┐
│  On-chain DA (Rollup)                                       │
│  • 交易数据存储在 L1 calldata                               │
│  • 最高安全性                                               │
│  • 成本: ~16 gas/byte (calldata)                            │
│  • EIP-4844 后: blob 数据更便宜                             │
├─────────────────────────────────────────────────────────────┤
│  Off-chain DA (Validium)                                    │
│  • 数据存储在链下 (DAC 委员会)                              │
│  • 较低成本                                                 │
│  • 信任假设: DAC 诚实                                       │
├─────────────────────────────────────────────────────────────┤
│  数据可用性委员会 (DAC)                                     │
│  • 多签名方案                                               │
│  • 委员会成员签名确认数据可用                               │
├─────────────────────────────────────────────────────────────┤
│  数据可用性采样 (DAS)                                       │
│  • 验证者只需下载部分数据                                   │
│  • 使用纠错码确保数据可恢复                                 │
│  • Danksharding 的核心技术                                  │
└─────────────────────────────────────────────────────────────┘
```

### AP.5 跨层消息传递

```solidity
// L1 -> L2 消息传递
contract L1CrossDomainMessenger {
    mapping(bytes32 => bool) public sentMessages;

    event SentMessage(
        address indexed target,
        address sender,
        bytes message,
        uint256 messageNonce,
        uint256 gasLimit
    );

    function sendMessage(
        address target,
        bytes calldata message,
        uint32 gasLimit
    ) external {
        bytes32 messageHash = keccak256(abi.encode(
            msg.sender,
            target,
            message,
            messageNonce++
        ));

        sentMessages[messageHash] = true;

        emit SentMessage(target, msg.sender, message, messageNonce, gasLimit);
    }
}

// L2 -> L1 消息传递 (需要等待挑战期)
contract L2CrossDomainMessenger {
    IL1CrossDomainMessenger public l1Messenger;

    mapping(bytes32 => bool) public sentMessages;
    mapping(bytes32 => uint256) public messageTimestamps;

    uint256 public constant FINALIZATION_PERIOD = 7 days;

    // L2 发送消息
    function sendMessage(
        address target,
        bytes calldata message
    ) external {
        bytes32 messageHash = keccak256(abi.encode(
            msg.sender,
            target,
            message,
            block.number
        ));

        sentMessages[messageHash] = true;
        messageTimestamps[messageHash] = block.timestamp;

        emit SentMessage(target, msg.sender, message);
    }

    // L1 上最终确认消息
    function finalizeMessage(
        address sender,
        address target,
        bytes calldata message,
        uint256 l2BlockNumber,
        bytes calldata proof
    ) external {
        bytes32 messageHash = keccak256(abi.encode(
            sender,
            target,
            message,
            l2BlockNumber
        ));

        // 验证消息在 L2 上存在
        require(verifyMessageInclusion(messageHash, proof), "Invalid proof");

        // 验证挑战期已过
        require(
            block.timestamp >= getMessageTimestamp(messageHash) + FINALIZATION_PERIOD,
            "Not finalized"
        );

        // 执行消息
        (bool success,) = target.call(message);
        require(success, "Message execution failed");
    }
}
```

### AP.6 Optimism 与 Arbitrum 对比

| 特性        | Optimism      | Arbitrum     |
|-----------|---------------|--------------|
| 欺诈证明      | 单轮交互式         | 多轮交互式        |
| 证明方式      | 重新执行全部交易      | 二分查找定位分歧     |
| EVM 兼容性   | OVM (修改版 EVM) | AVM (自定义 VM) |
| 挑战期       | 7 天           | 7 天          |
| 数据压缩      | 基础压缩          | 更激进的压缩       |
| Sequencer | 中心化           | 中心化 (计划去中心化) |

---

## 附录 AQ: 跨链桥机制

### AQ.1 跨链桥分类

```
跨链桥类型:
┌─────────────────────────────────────────────────────────────┐
│  锁定-铸造 (Lock-Mint)                                      │
│  • 源链锁定资产                                             │
│  • 目标链铸造包装资产                                       │
│  • 示例: WBTC, Wrapped ETH                                  │
├─────────────────────────────────────────────────────────────┤
│  销毁-铸造 (Burn-Mint)                                      │
│  • 源链销毁代币                                             │
│  • 目标链铸造等量代币                                       │
│  • 原生跨链代币                                             │
├─────────────────────────────────────────────────────────────┤
│  原子交换 (Atomic Swap)                                     │
│  • 哈希时间锁定合约 (HTLC)                                  │
│  • 无需信任第三方                                           │
│  • 需要两链都有流动性                                       │
├─────────────────────────────────────────────────────────────┤
│  流动性网络                                                 │
│  • 流动性提供者预付资金                                     │
│  • 后续结算                                                 │
│  • 示例: Hop Protocol, Connext                              │
└─────────────────────────────────────────────────────────────┘
```

### AQ.2 多签桥实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MultisigBridge {
    address[] public validators;
    uint256 public threshold;  // 需要的签名数

    mapping(bytes32 => bool) public processedDeposits;
    mapping(bytes32 => uint256) public withdrawalSignatures;

    event Deposit(
        address indexed from,
        address indexed to,
        uint256 amount,
        uint256 destinationChainId,
        bytes32 depositId
    );

    event Withdrawal(
        address indexed to,
        uint256 amount,
        bytes32 depositId
    );

    constructor(address[] memory _validators, uint256 _threshold) {
        validators = _validators;
        threshold = _threshold;
    }

    // 源链: 锁定资产
    function deposit(
        address to,
        uint256 destinationChainId
    ) external payable {
        bytes32 depositId = keccak256(abi.encode(
            block.chainid,
            destinationChainId,
            msg.sender,
            to,
            msg.value,
            block.number,
            block.timestamp
        ));

        emit Deposit(msg.sender, to, msg.value, destinationChainId, depositId);
    }

    // 目标链: 释放/铸造资产
    function withdraw(
        address to,
        uint256 amount,
        bytes32 depositId,
        bytes[] calldata signatures
    ) external {
        require(!processedDeposits[depositId], "Already processed");
        require(signatures.length >= threshold, "Not enough signatures");

        // 验证签名
        bytes32 messageHash = keccak256(abi.encode(
            block.chainid,
            to,
            amount,
            depositId
        ));
        bytes32 ethSignedHash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            messageHash
        ));

        address[] memory signers = new address[](signatures.length);
        for (uint256 i = 0; i < signatures.length; i++) {
            address signer = recoverSigner(ethSignedHash, signatures[i]);
            require(isValidator(signer), "Invalid signer");

            // 检查重复签名
            for (uint256 j = 0; j < i; j++) {
                require(signers[j] != signer, "Duplicate signer");
            }
            signers[i] = signer;
        }

        processedDeposits[depositId] = true;

        // 释放资产
        (bool success,) = to.call{value: amount}("");
        require(success, "Transfer failed");

        emit Withdrawal(to, amount, depositId);
    }

    function isValidator(address addr) public view returns (bool) {
        for (uint256 i = 0; i < validators.length; i++) {
            if (validators[i] == addr) return true;
        }
        return false;
    }

    function recoverSigner(bytes32 hash, bytes memory sig) internal pure returns (address) {
        require(sig.length == 65, "Invalid signature length");

        bytes32 r;
        bytes32 s;
        uint8 v;

        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }

        return ecrecover(hash, v, r, s);
    }
}
```

### AQ.3 哈希时间锁定合约 (HTLC)

```solidity
contract HTLC {
    struct Lock {
        address sender;
        address receiver;
        uint256 amount;
        bytes32 hashlock;
        uint256 timelock;
        bool withdrawn;
        bool refunded;
    }

    mapping(bytes32 => Lock) public locks;

    event Locked(
        bytes32 indexed lockId,
        address indexed sender,
        address indexed receiver,
        uint256 amount,
        bytes32 hashlock,
        uint256 timelock
    );

    event Withdrawn(bytes32 indexed lockId, bytes32 preimage);
    event Refunded(bytes32 indexed lockId);

    // 创建锁定
    function lock(
        address receiver,
        bytes32 hashlock,
        uint256 timelock
    ) external payable returns (bytes32 lockId) {
        require(msg.value > 0, "No value");
        require(timelock > block.timestamp, "Invalid timelock");

        lockId = keccak256(abi.encode(
            msg.sender,
            receiver,
            msg.value,
            hashlock,
            timelock
        ));

        require(locks[lockId].sender == address(0), "Lock exists");

        locks[lockId] = Lock({
            sender: msg.sender,
            receiver: receiver,
            amount: msg.value,
            hashlock: hashlock,
            timelock: timelock,
            withdrawn: false,
            refunded: false
        });

        emit Locked(lockId, msg.sender, receiver, msg.value, hashlock, timelock);
    }

    // 使用原像提取
    function withdraw(bytes32 lockId, bytes32 preimage) external {
        Lock storage l = locks[lockId];

        require(l.receiver == msg.sender, "Not receiver");
        require(!l.withdrawn, "Already withdrawn");
        require(!l.refunded, "Already refunded");
        require(keccak256(abi.encodePacked(preimage)) == l.hashlock, "Invalid preimage");

        l.withdrawn = true;

        (bool success,) = l.receiver.call{value: l.amount}("");
        require(success, "Transfer failed");

        emit Withdrawn(lockId, preimage);
    }

    // 超时退款
    function refund(bytes32 lockId) external {
        Lock storage l = locks[lockId];

        require(l.sender == msg.sender, "Not sender");
        require(!l.withdrawn, "Already withdrawn");
        require(!l.refunded, "Already refunded");
        require(block.timestamp >= l.timelock, "Not expired");

        l.refunded = true;

        (bool success,) = l.sender.call{value: l.amount}("");
        require(success, "Transfer failed");

        emit Refunded(lockId);
    }
}
```

### AQ.4 轻客户端桥

```solidity
// 验证源链区块头的桥
contract LightClientBridge {
    // 源链区块头
    struct BlockHeader {
        bytes32 parentHash;
        bytes32 stateRoot;
        bytes32 transactionsRoot;
        bytes32 receiptsRoot;
        uint256 blockNumber;
        uint256 timestamp;
    }

    // 验证者集合
    address[] public validators;
    mapping(uint256 => BlockHeader) public verifiedHeaders;

    // 提交区块头
    function submitBlockHeader(
        BlockHeader calldata header,
        bytes[] calldata signatures
    ) external {
        bytes32 headerHash = keccak256(abi.encode(header));

        // 验证签名
        uint256 validSignatures = 0;
        for (uint256 i = 0; i < signatures.length; i++) {
            address signer = recoverSigner(headerHash, signatures[i]);
            if (isValidator(signer)) {
                validSignatures++;
            }
        }

        // 需要 2/3 验证者签名
        require(validSignatures * 3 > validators.length * 2, "Not enough signatures");

        verifiedHeaders[header.blockNumber] = header;
    }

    // 验证交易包含证明
    function verifyTransaction(
        uint256 blockNumber,
        bytes calldata txData,
        bytes32[] calldata proof,
        uint256 txIndex
    ) external view returns (bool) {
        BlockHeader storage header = verifiedHeaders[blockNumber];
        require(header.blockNumber != 0, "Block not verified");

        // 验证 Merkle 证明
        bytes32 txHash = keccak256(txData);
        bytes32 computedRoot = computeMerkleRoot(txHash, proof, txIndex);

        return computedRoot == header.transactionsRoot;
    }

    function computeMerkleRoot(
        bytes32 leaf,
        bytes32[] calldata proof,
        uint256 index
    ) internal pure returns (bytes32) {
        bytes32 hash = leaf;

        for (uint256 i = 0; i < proof.length; i++) {
            if (index % 2 == 0) {
                hash = keccak256(abi.encodePacked(hash, proof[i]));
            } else {
                hash = keccak256(abi.encodePacked(proof[i], hash));
            }
            index = index / 2;
        }

        return hash;
    }
}
```

### AQ.5 跨链桥安全考虑

```
常见跨链桥攻击:
┌─────────────────────────────────────────────────────────────┐
│  1. 虚假存款证明                                            │
│     • 伪造源链存款事件                                      │
│     • 防护: 完整验证源链状态                                │
├─────────────────────────────────────────────────────────────┤
│  2. 验证者密钥泄露                                          │
│     • 攻击者获取足够验证者私钥                              │
│     • 防护: MPC, HSM, 阈值签名                              │
├─────────────────────────────────────────────────────────────┤
│  3. 重放攻击                                                │
│     • 在不同链上重复使用同一签名                            │
│     • 防护: 包含 chainId 在签名消息中                       │
├─────────────────────────────────────────────────────────────┤
│  4. 重入攻击                                                │
│     • 在提款过程中重入                                      │
│     • 防护: CEI 模式, ReentrancyGuard                       │
├─────────────────────────────────────────────────────────────┤
│  5. 预言机操纵                                              │
│     • 操纵价格预言机影响跨链交换                            │
│     • 防护: TWAP, 多预言机                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 附录 AR: 形式化验证

### AR.1 形式化验证概述

```
形式化验证方法:
┌─────────────────────────────────────────────────────────────┐
│  模型检测 (Model Checking)                                  │
│  • 穷举所有可能状态                                         │
│  • 验证属性在所有状态下成立                                 │
│  • 工具: SPIN, TLA+                                         │
├─────────────────────────────────────────────────────────────┤
│  定理证明 (Theorem Proving)                                 │
│  • 使用数学证明验证程序属性                                 │
│  • 可处理无限状态空间                                       │
│  • 工具: Coq, Isabelle, Lean                                │
├─────────────────────────────────────────────────────────────┤
│  符号执行 (Symbolic Execution)                              │
│  • 使用符号值代替具体值执行                                 │
│  • 探索所有执行路径                                         │
│  • 工具: Mythril, Manticore                                 │
├─────────────────────────────────────────────────────────────┤
│  SMT 求解 (SMT Solving)                                     │
│  • 可满足性模理论                                           │
│  • 验证约束是否可满足                                       │
│  • 工具: Z3, CVC4                                           │
└─────────────────────────────────────────────────────────────┘
```

### AR.2 Certora Prover

```solidity
// 使用 Certora 规范语言 (CVL)
// spec/ERC20.spec

methods {
function totalSupply() external returns (uint256) envfree;
function balanceOf(address) external returns (uint256) envfree;
function transfer(address, uint256) external returns (bool);
}

// 不变量: 总供应量等于所有余额之和
invariant totalSupplyIsSumOfBalances()
totalSupply() == sum(balanceOf)

// 规则: transfer 保持总供应量不变
rule transferPreservesTotalSupply(address to, uint256 amount) {
env e;

uint256 totalBefore = totalSupply();

transfer(e, to, amount);

uint256 totalAfter = totalSupply();

assert totalBefore == totalAfter;
}

// 规则: transfer 正确更新余额
rule transferUpdatesBalances(address to, uint256 amount) {
env e;
address from = e.msg.sender;

require from != to;

uint256 fromBalanceBefore = balanceOf(from);
uint256 toBalanceBefore = balanceOf(to);

bool success = transfer(e, to, amount);

uint256 fromBalanceAfter = balanceOf(from);
uint256 toBalanceAfter = balanceOf(to);

assert success => (
fromBalanceAfter == fromBalanceBefore - amount &&
toBalanceAfter == toBalanceBefore + amount
);

assert !success => (
fromBalanceAfter == fromBalanceBefore &&
toBalanceAfter == toBalanceBefore
);
}

// 规则: 没有凭空创建代币
rule noTokenCreation(method f) {
env e;
calldataarg args;

uint256 totalBefore = totalSupply();

f(e, args);

uint256 totalAfter = totalSupply();

assert totalAfter <= totalBefore || f.selector == sig : mint(address, uint256).selector;
}
```

### AR.3 Echidna 模糊测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./ERC20.sol";

contract ERC20Echidna is ERC20 {
    address echidna_caller = msg.sender;

    constructor() ERC20("Test", "TST") {
        _mint(echidna_caller, 1000000e18);
    }

    // 不变量: 用户余额不应超过总供应量
    function echidna_balance_under_total() public view returns (bool) {
        return balanceOf(echidna_caller) <= totalSupply();
    }

    // 不变量: transfer 后余额变化正确
    function echidna_transfer_correct(address to, uint256 amount) public returns (bool) {
        if (to == address(0) || to == address(this)) return true;
        if (amount > balanceOf(msg.sender)) return true;

        uint256 senderBefore = balanceOf(msg.sender);
        uint256 receiverBefore = balanceOf(to);

        transfer(to, amount);

        if (msg.sender == to) {
            return balanceOf(msg.sender) == senderBefore;
        }

        return balanceOf(msg.sender) == senderBefore - amount &&
            balanceOf(to) == receiverBefore + amount;
    }

    // 不变量: 总供应量不能为负
    function echidna_total_supply_positive() public view returns (bool) {
        return totalSupply() >= 0;
    }
}

// echidna.yaml 配置
/*
testMode: "assertion"
testLimit: 100000
seqLen: 100
contractAddr: "0x00a329c0648769a73afac7f9381e08fb43dbea72"
deployer: "0x00a329c0648769a73afac7f9381e08fb43dbea72"
sender: ["0x00a329c0648769a73afac7f9381e08fb43dbea72"]
*/
```

### AR.4 Foundry Invariant Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/Vault.sol";

contract VaultHandler is Test {
    Vault public vault;
    IERC20 public token;

    uint256 public ghost_depositSum;
    uint256 public ghost_withdrawSum;

    constructor(Vault _vault, IERC20 _token) {
        vault = _vault;
        token = _token;
    }

    function deposit(uint256 amount) public {
        amount = bound(amount, 1, token.balanceOf(address(this)));

        token.approve(address(vault), amount);
        vault.deposit(amount);

        ghost_depositSum += amount;
    }

    function withdraw(uint256 amount) public {
        amount = bound(amount, 0, vault.balanceOf(address(this)));
        if (amount == 0) return;

        vault.withdraw(amount);

        ghost_withdrawSum += amount;
    }
}

contract VaultInvariantTest is Test {
    Vault public vault;
    MockERC20 public token;
    VaultHandler public handler;

    function setUp() public {
        token = new MockERC20();
        vault = new Vault(address(token));
        handler = new VaultHandler(vault, token);

        token.mint(address(handler), 1000000e18);

        // 设置目标合约
        targetContract(address(handler));
    }

    // 不变量: Vault 持有的代币 >= 所有用户份额
    function invariant_solvency() public {
        assertGe(
            token.balanceOf(address(vault)),
            vault.totalSupply()
        );
    }

    // 不变量: 存款总和 - 取款总和 = Vault 余额
    function invariant_accounting() public {
        assertEq(
            handler.ghost_depositSum() - handler.ghost_withdrawSum(),
            token.balanceOf(address(vault))
        );
    }

    // 不变量: 没有免费代币
    function invariant_noFreeTokens() public {
        assertEq(
            vault.totalSupply(),
            token.balanceOf(address(vault))
        );
    }
}
```

### AR.5 符号执行 (Mythril)

```python
# 使用 Mythril 进行符号执行分析

# 安装
# pip install mythril

# 分析单个合约
# myth analyze contracts/Vault.sol

# 输出示例:
"""
==== Integer Arithmetic Bugs ====
SWC ID: 101
Severity: High
Contract: Vault
Function name: withdraw(uint256)
PC address: 1234
Estimated Gas Usage: 5000 - 6000
The arithmetic operator can underflow.
...

==== Reentrancy ====
SWC ID: 107
Severity: High
Contract: Vault
Function name: withdraw(uint256)
PC address: 5678
External call at address 5678 is followed by state changes.
...
"""

# Mythril 自定义检测规则
from mythril.analysis.modules.base import DetectionModule
from mythril.analysis.report import Issue

class CustomChecker(DetectionModule):
    name = "Custom vulnerability checker"
    swc_id = "SWC-XXX"

    def execute(self, state):
        issues = []

        # 检测特定模式
        instruction = state.get_current_instruction()
        if instruction['opcode'] == 'SSTORE':
            # 检查是否在外部调用后写入存储
            if self.has_external_call_before(state):
                issues.append(Issue(
                    contract=state.environment.active_account.contract_name,
                    function_name=state.environment.active_function_name,
                    address=instruction['address'],
                    title="State change after external call",
                    severity="High",
                    description="Storage write after external call detected"
                ))

        return issues
```

### AR.6 K Framework 语义验证

```k
// KEVM: K 语义化的 EVM

// 定义 EVM 配置
configuration
    <k> $PGM:EthereumSimulation </k>
    <ethereum>
        <evm>
            <callStack> .List </callStack>
            <program> .Map </program>
            <pc> 0 </pc>
            <gas> 0 </gas>
            <memory> .Map </memory>
            <storage> .Map </storage>
        </evm>
        <network>
            <accounts>
                <account multiplicity="*">
                    <id> 0 </id>
                    <balance> 0 </balance>
                    <code> .ByteArray </code>
                    <storage> .Map </storage>
                    <nonce> 0 </nonce>
                </account>
            </accounts>
        </network>
    </ethereum>

// SSTORE 语义规则
rule <k> SSTORE => . ... </k>
     <pc> PCOUNT => PCOUNT +Int 1 </pc>
     <gas> G => G -Gas Csstore(SCHED, NEW, CURR, ORIG) </gas>
     <wordStack> INDEX : NEW : WS => WS </wordStack>
     <storage> STORAGE => STORAGE [ INDEX <- NEW ] </storage>
     <id> ACCT </id>
    requires G >=Gas Csstore(SCHED, NEW, CURR, ORIG)
      andBool INDEX in_keys(STORAGE)
      andBool CURR ==Int STORAGE [ INDEX ]

// 验证 ERC20 transfer 规范
claim <k> #execute ... </k>
      <callData> #abiCallData("transfer", #address(TO), #uint256(VALUE)) </callData>
      <storage>
          BALANCES [ FROM ] |-> (BAL_FROM => BAL_FROM -Int VALUE)
          BALANCES [ TO   ] |-> (BAL_TO   => BAL_TO   +Int VALUE)
      </storage>
    requires FROM =/=Int TO
     andBool VALUE <=Int BAL_FROM
```

---

## 附录 AS: Gas 深度优化

### AS.1 存储优化

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract StorageOptimization {
    // 未优化: 每个变量占用一个 slot
    // uint256 a;  // slot 0
    // uint256 b;  // slot 1
    // uint256 c;  // slot 2
    // 读取 3 个值: 3 × 2100 = 6300 gas (cold)

    // 优化: 打包到一个 slot
    uint96 a;   // slot 0 [0:96)
    uint96 b;   // slot 0 [96:192)
    uint64 c;   // slot 0 [192:256)
    // 读取 3 个值: 1 × 2100 = 2100 gas (cold)

    // 结构体打包
    struct BadPacking {
        uint256 id;      // slot n
        uint8 status;    // slot n+1 (浪费 31 bytes)
        uint256 amount;  // slot n+2
        uint8 flag;      // slot n+3 (浪费 31 bytes)
    }
    // 4 个 slots

    struct GoodPacking {
        uint256 id;      // slot n
        uint256 amount;  // slot n+1
        uint8 status;    // slot n+2 [0:8)
        uint8 flag;      // slot n+2 [8:16)
    }
    // 3 个 slots

    // 使用 uint256 代替小类型进行计算
    function badLoop(uint8[] memory arr) external pure returns (uint8 sum) {
        for (uint8 i = 0; i < arr.length; i++) {  // uint8 循环变量
            sum += arr[i];
        }
    }
    // 每次迭代都有额外的 masking 操作

    function goodLoop(uint8[] memory arr) external pure returns (uint256 sum) {
        uint256 len = arr.length;  // 缓存 length
        for (uint256 i = 0; i < len; i++) {  // uint256 循环变量
            sum += arr[i];
        }
    }

    // 使用 immutable 代替 storage
    address public owner;  // SLOAD: 2100 gas (cold)
    address public immutable OWNER;  // 直接内联: 3 gas

    constructor() {
        OWNER = msg.sender;
    }

    // 使用 constant 代替 storage
    uint256 public constant MAX_SUPPLY = 1000000;  // 编译时内联
}
```

### AS.2 函数优化

```solidity
contract FunctionOptimization {
    uint256[] public data;

    // 使用 calldata 代替 memory
    function badInput(uint256[] memory arr) external pure returns (uint256) {
        // 复制到 memory: 每个元素 ~100 gas
        return arr[0];
    }

    function goodInput(uint256[] calldata arr) external pure returns (uint256) {
        // 直接从 calldata 读取
        return arr[0];
    }

    // 缓存 storage 变量
    function badLoop() external view returns (uint256 sum) {
        for (uint256 i = 0; i < data.length; i++) {
            // 每次都 SLOAD data.length
            sum += data[i];
        }
    }

    function goodLoop() external view returns (uint256 sum) {
        uint256[] storage _data = data;
        uint256 len = _data.length;  // 缓存 length
        for (uint256 i = 0; i < len; ++i) {  // ++i 比 i++ 便宜
            sum += _data[i];
        }
    }

    // 使用 unchecked 跳过溢出检查
    function safeAdd(uint256 a, uint256 b) external pure returns (uint256) {
        return a + b;  // 包含溢出检查
    }

    function uncheckedAdd(uint256 a, uint256 b) external pure returns (uint256 result) {
        unchecked {
            result = a + b;  // 无溢出检查，节省 ~20 gas
        }
    }

    // 短路求值
    function badCondition(uint256 x) external view returns (bool) {
        return expensiveCheck() && x > 0;  // 总是先执行 expensiveCheck
    }

    function goodCondition(uint256 x) external view returns (bool) {
        return x > 0 && expensiveCheck();  // 如果 x <= 0，不执行 expensiveCheck
    }
}
```

### AS.3 内联汇编优化

```solidity
contract AssemblyOptimization {
    // 高效的数组求和
    function sumArray(uint256[] calldata arr) external pure returns (uint256 sum) {
        assembly {
            let len := arr.length
            let dataPtr := arr.offset

            for {let i := 0} lt(i, len) {i := add(i, 1)} {
                sum := add(sum, calldataload(add(dataPtr, mul(i, 0x20))))
            }
        }
    }

    // 高效的地址数组检查
    function containsAddress(
        address[] calldata addresses,
        address target
    ) external pure returns (bool found) {
        assembly {
            let len := addresses.length
            let ptr := addresses.offset

            for {let i := 0} lt(i, len) {i := add(i, 1)} {
                if eq(calldataload(add(ptr, mul(i, 0x20))), target) {
                    found := 1
                    break
                }
            }
        }
    }

    // 高效的 bytes32 比较
    function compareBytes32(bytes32 a, bytes32 b) external pure returns (bool) {
        assembly {
            mstore(0x00, eq(a, b))
            return (0x00, 0x20)
        }
    }

    // 批量转账优化
    function batchTransfer(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external payable {
        assembly {
            let recipientsLen := recipients.length

            // 验证长度匹配
            if iszero(eq(recipientsLen, amounts.length)) {
                revert(0, 0)
            }

            let recipientsPtr := recipients.offset
            let amountsPtr := amounts.offset

            for {let i := 0} lt(i, recipientsLen) {i := add(i, 1)} {
                let recipient := calldataload(add(recipientsPtr, mul(i, 0x20)))
                let amount := calldataload(add(amountsPtr, mul(i, 0x20)))

                // 转账
                let success := call(gas(), recipient, amount, 0, 0, 0, 0)
                if iszero(success) {
                    revert(0, 0)
                }
            }
        }
    }
}
```

### AS.4 编译器优化

```solidity
// foundry.toml 配置
/*
[profile.default]
optimizer = true
optimizer_runs = 200  # 部署成本 vs 运行成本的权衡

# 高频调用合约
[profile.high-frequency]
optimizer_runs = 1000000

# 一次性部署合约
[profile.one-time]
optimizer_runs = 1

# 启用 via-ir (新优化管道)
via_ir = true
*/

// 使用 solc 优化标志
// solc --optimize --optimize-runs 200 Contract.sol

contract CompilerOptimizationTips {
    // 1. 避免动态数组在 memory 中增长
    function badDynamic() external pure returns (uint256[] memory) {
        uint256[] memory result = new uint256[](0);
        for (uint256 i = 0; i < 10; i++) {
            // 每次 push 都要重新分配内存
            // result.push(i);  // 不支持
        }
        return result;
    }

    function goodDynamic() external pure returns (uint256[] memory) {
        uint256[] memory result = new uint256[](10);  // 预分配
        for (uint256 i = 0; i < 10; i++) {
            result[i] = i;
        }
        return result;
    }

    // 2. 使用 bytes32 代替 string
    string public badString = "Hello";  // 动态类型，更贵
    bytes32 public goodString = "Hello";  // 固定大小，更便宜

    // 3. 使用 error 代替 require string
    error InsufficientBalance(uint256 available, uint256 required);

    function badRequire(uint256 amount) external pure {
        require(amount > 0, "Amount must be greater than 0");  // 存储字符串
    }

    function goodRequire(uint256 amount) external pure {
        if (amount == 0) revert InsufficientBalance(0, amount);
    }
}
```

### AS.5 Gas 对比表

| 操作             | Gas 成本      | 优化建议        |
|----------------|-------------|-------------|
| SSTORE (0→非0)  | 22,100      | 避免频繁写入      |
| SSTORE (非0→非0) | 5,000       | 批量更新        |
| SSTORE (非0→0)  | 退还 4,800    | 清理无用存储      |
| SLOAD (cold)   | 2,100       | 缓存到 memory  |
| SLOAD (warm)   | 100         | -           |
| MSTORE         | 3           | 优先使用        |
| MLOAD          | 3           | 优先使用        |
| CALLDATALOAD   | 3           | 使用 calldata |
| CALL           | 100 + 动态    | 避免不必要调用     |
| CREATE         | 32,000      | 使用 clone    |
| CREATE2        | 32,000      | 使用 clone    |
| LOG0           | 375         | 减少事件数据      |
| LOG1           | 750         | -           |
| KECCAK256      | 30 + 6/word | 缓存哈希结果      |

---

## 附录 AT: 调试与测试工具

### AT.1 Foundry 测试框架

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/Token.sol";

contract TokenTest is Test {
    Token public token;
    address public alice = makeAddr("alice");
    address public bob = makeAddr("bob");

    function setUp() public {
        token = new Token("Test", "TST");
        token.mint(alice, 1000e18);
    }

    // 基础测试
    function test_Transfer() public {
        vm.prank(alice);
        token.transfer(bob, 100e18);

        assertEq(token.balanceOf(alice), 900e18);
        assertEq(token.balanceOf(bob), 100e18);
    }

    // 模糊测试
    function testFuzz_Transfer(uint256 amount) public {
        amount = bound(amount, 0, token.balanceOf(alice));

        vm.prank(alice);
        token.transfer(bob, amount);

        assertEq(token.balanceOf(alice), 1000e18 - amount);
        assertEq(token.balanceOf(bob), amount);
    }

    // 预期 revert
    function test_RevertWhen_InsufficientBalance() public {
        vm.prank(alice);
        vm.expectRevert("Insufficient balance");
        token.transfer(bob, 2000e18);
    }

    // 预期事件
    function test_EmitTransferEvent() public {
        vm.prank(alice);

        vm.expectEmit(true, true, false, true);
        emit Transfer(alice, bob, 100e18);

        token.transfer(bob, 100e18);
    }

    // 时间操作
    function test_TimeDependent() public {
        vm.warp(block.timestamp + 1 days);
        // 测试时间相关逻辑
    }

    // 区块操作
    function test_BlockDependent() public {
        vm.roll(block.number + 100);
        // 测试区块相关逻辑
    }

    // Fork 测试
    function test_ForkMainnet() public {
        // 在 foundry.toml 中配置 RPC URL
        uint256 mainnetFork = vm.createFork("mainnet");
        vm.selectFork(mainnetFork);

        // 与主网合约交互
        IERC20 usdc = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
        uint256 balance = usdc.balanceOf(alice);
        console.log("USDC balance:", balance);
    }

    // 快照和回滚
    function test_Snapshot() public {
        uint256 snapshot = vm.snapshot();

        vm.prank(alice);
        token.transfer(bob, 100e18);

        vm.revertTo(snapshot);

        assertEq(token.balanceOf(alice), 1000e18);
    }

    // Gas 报告
    function test_GasUsage() public {
        uint256 gasBefore = gasleft();

        vm.prank(alice);
        token.transfer(bob, 100e18);

        uint256 gasUsed = gasBefore - gasleft();
        console.log("Gas used:", gasUsed);
    }

    // 使用 console.log 调试
    function test_Debug() public {
        console.log("Alice balance before:", token.balanceOf(alice));
        console.log("Bob address:", bob);
        console.logBytes32(keccak256("test"));
    }
}
```

### AT.2 Hardhat 调试

```javascript
// hardhat.config.js
module.exports = {
    solidity: {
        version: "0.8.20",
        settings: {
            optimizer: {
                enabled: true,
                runs: 200
            }
        }
    },
    networks: {
        hardhat: {
            forking: {
                url: process.env.MAINNET_RPC,
                blockNumber: 18000000
            }
        }
    }
};

// 测试文件
const {expect} = require("chai");
const {ethers} = require("hardhat");

describe("Token", function () {
    let token, owner, alice, bob;

    beforeEach(async function () {
        [owner, alice, bob] = await ethers.getSigners();

        const Token = await ethers.getContractFactory("Token");
        token = await Token.deploy("Test", "TST");

        await token.mint(alice.address, ethers.parseEther("1000"));
    });

    it("Should transfer tokens", async function () {
        await token.connect(alice).transfer(bob.address, ethers.parseEther("100"));

        expect(await token.balanceOf(alice.address)).to.equal(
            ethers.parseEther("900")
        );
        expect(await token.balanceOf(bob.address)).to.equal(
            ethers.parseEther("100")
        );
    });

    // 使用 console.log 调试 (需要在合约中 import hardhat/console.sol)
    it("Should log debug info", async function () {
        // 合约中的 console.log 会输出到控制台
        await token.connect(alice).transfer(bob.address, ethers.parseEther("100"));
    });

    // 追踪交易
    it("Should trace transaction", async function () {
        const tx = await token.connect(alice).transfer(
            bob.address,
            ethers.parseEther("100")
        );

        // 获取交易回执
        const receipt = await tx.wait();
        console.log("Gas used:", receipt.gasUsed.toString());
        console.log("Events:", receipt.logs);
    });
});
```

### AT.3 Tenderly 调试

```javascript
// 使用 Tenderly 进行交易调试
const {TenderlyFork} = require("@tenderly/hardhat-tenderly");

// 创建 Fork
const fork = await TenderlyFork.create({
    network_id: "1",
    block_number: 18000000
});

// 模拟交易
const simulation = await fork.simulate({
    from: "0x...",
    to: "0x...",
    input: "0x...",
    value: "0",
    gas: 1000000
});

// 分析结果
console.log("Success:", simulation.success);
console.log("Gas used:", simulation.gas_used);
console.log("State diff:", simulation.state_diff);
console.log("Call trace:", simulation.call_trace);

// 在 Tenderly 仪表板中查看详细 trace
// https://dashboard.tenderly.co/tx/{txHash}
```

### AT.4 Trace 分析

```go
// 使用 go-ethereum debug API
package main

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/rpc"
)

func traceTransaction(txHash string) {
	client, _ := rpc.Dial("http://localhost:8545")

	var result json.RawMessage

	// 调用 debug_traceTransaction
	err := client.Call(&result, "debug_traceTransaction", txHash, map[string]string{
		"tracer": "callTracer",
	})
	if err != nil {
		panic(err)
	}

	fmt.Println(string(result))
}

// 自定义 JavaScript tracer
const customTracer = `{
    data: [],
    fault: function(log) {},
    step: function(log) {
        if (log.op.toString() === "SSTORE") {
            this.data.push({
                op: "SSTORE",
                depth: log.getDepth(),
                gas: log.getGas(),
                key: log.stack.peek(0).toString(16),
                value: log.stack.peek(1).toString(16)
            });
        }
    },
    result: function() { return this.data; }
}`;

func traceWithCustomTracer(txHash string) {
	client, _ := rpc.Dial("http://localhost:8545")

	var result json.RawMessage

	err := client.Call(&result, "debug_traceTransaction", txHash, map[string]interface{}{
		"tracer": customTracer,
	})
	if err != nil {
		panic(err)
	}

	fmt.Println(string(result))
}
```

### AT.5 静态分析工具

```bash
# Slither - 静态分析
pip install slither-analyzer
slither contracts/

# 输出示例:
# Contract Token (contracts/Token.sol)
#   - Function transfer(address,uint256) (contracts/Token.sol#25-35)
#     - Reentrancy vulnerability detected
#     - State variable written after external call

# Mythril - 符号执行
pip install mythril
myth analyze contracts/Token.sol

# Securify - 静态分析
docker run -v $(pwd):/contracts securify/securify /contracts/Token.sol

# 4naly3er - Gas 优化建议
npx 4naly3er contracts/

# sol2uml - 生成 UML 图
npm install -g sol2uml
sol2uml class contracts/ -o diagrams/
sol2uml storage contracts/Token.sol -o storage.svg
```

### AT.6 本地节点调试

```bash
# Anvil (Foundry)
anvil --fork-url $MAINNET_RPC --fork-block-number 18000000

# 常用参数
anvil \
  --port 8545 \
  --accounts 10 \
  --balance 10000 \
  --block-time 1 \
  --gas-limit 30000000 \
  --chain-id 31337

# Hardhat Node
npx hardhat node --fork $MAINNET_RPC

# Ganache
ganache --fork.url $MAINNET_RPC

# 在本地节点上调试
cast call $CONTRACT "balanceOf(address)" $ADDRESS --rpc-url http://localhost:8545
cast send $CONTRACT "transfer(address,uint256)" $TO $AMOUNT --private-key $KEY --rpc-url http://localhost:8545

# 查看存储
cast storage $CONTRACT 0 --rpc-url http://localhost:8545

# 追踪调用
cast run $TX_HASH --rpc-url http://localhost:8545 --trace
```

---

## 参考资料

- [revm GitHub](https://github.com/bluealloy/revm)
- [revm 文档](https://docs.rs/revm)
- [op-revm GitHub](https://github.com/op-rs/op-revm)
- [PEVM GitHub](https://github.com/risechain/pevm)
- [Grevm GitHub](https://github.com/paradigmxyz/grevm)
- [Reth GitHub](https://github.com/paradigmxyz/reth)
- [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts)
- [Solidity ABI Spec](https://docs.soliditylang.org/en/latest/abi-spec.html)
- [EIP-1167 Minimal Proxy](https://eips.ethereum.org/EIPS/eip-1167)
- [EIP-712 Typed Data](https://eips.ethereum.org/EIPS/eip-712)
- [Yul 文档](https://docs.soliditylang.org/en/latest/yul.html)
- [SWC Registry](https://swcregistry.io/)
- [Trin Portal Network](https://github.com/ethereum/trin)
- [ethereum-hive](https://github.com/ethereum/hive)
- [ssz_rs](https://github.com/ralexstokes/ssz-rs)
- [Flashbots MEV-Boost](https://github.com/flashbots/mev-boost)
- [Reth GitHub](https://github.com/paradigmxyz/reth)
- [Benchmark Issue #7](https://github.com/bluealloy/revm/issues/7)
- [REVM Is All You Need](https://medium.com/@solidquant/revm-is-all-you-need-e01b5b0421e4)
- [Reth 介绍](https://www.paradigm.xyz/2022/12/reth)
- [Optimism 交易费用文档](https://docs.optimism.io/stack/transactions/transaction-fees)
- [EOF 官网](https://evmobjectformat.xyz/)
- [EIP-7692 EOF Meta](https://eips.ethereum.org/EIPS/eip-7692)
- [execution-spec-tests](https://github.com/ethereum/execution-spec-tests)
- [EVMFuzz Paper](https://arxiv.org/abs/2307.09290)
- [Block-STM Paper](https://arxiv.org/abs/2203.06871)
- [Verkle Trees EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)
- [zkEVM 类型分类](https://vitalik.eth.limo/general/2022/08/04/zkevm.html)
- [Arbitrum Stylus](https://docs.arbitrum.io/stylus/stylus-gentle-introduction)
- [OP Stack Specs](https://specs.optimism.io/)
- [EIP-1153 Transient Storage](https://eips.ethereum.org/EIPS/eip-1153)
- [EIP-5656 MCOPY](https://eips.ethereum.org/EIPS/eip-5656)
- [EIP-3860 Limit Initcode](https://eips.ethereum.org/EIPS/eip-3860)
- [ERC-4337 Account Abstraction](https://eips.ethereum.org/EIPS/eip-4337)
- [EIP-7702 Set Code](https://eips.ethereum.org/EIPS/eip-7702)
- [KEVM GitHub](https://github.com/runtimeverification/evm-semantics)
- [K Framework](https://kframework.org/)
- [Act Language](https://github.com/ethereum/act)
- [proptest](https://github.com/proptest-rs/proptest)
- [Criterion.rs](https://github.com/bheisler/criterion.rs)
- [pprof-rs](https://github.com/tikv/pprof-rs)
- [DHAT](https://docs.rs/dhat)
- [Reth Snap Sync](https://paradigmxyz.github.io/reth/run/sync.html)
- [libp2p](https://libp2p.io/)
- [MDBX](https://github.com/erthink/libmdbx)
- [jsonrpsee](https://github.com/paritytech/jsonrpsee)
- [discv5](https://github.com/sigp/discv5)
- [ENR](https://eips.ethereum.org/EIPS/eip-778)
- [Flashbots MEV](https://docs.flashbots.net/)
- [EIP-4844 Blob Transactions](https://eips.ethereum.org/EIPS/eip-4844)
