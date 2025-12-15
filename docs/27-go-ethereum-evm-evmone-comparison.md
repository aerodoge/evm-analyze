# Go-Ethereum EVM 深入解析：evmone 对比分析

## 概述

[evmone](https://github.com/ipsilon/evmone) 是由以太坊基金会 Ipsilon 团队开发的高性能 C++ EVM 实现。本文将 evmone 与
go-ethereum 的 EVM 实现进行全方位对比，分析其架构设计、性能优化和技术创新。

## 1. 项目概览

### 1.1 基本信息对比

| 特性       | go-ethereum EVM     | evmone       |
|----------|---------------------|--------------|
| **语言**   | Go                  | C++ (C++20)  |
| **团队**   | Ethereum Foundation | Ipsilon Team |
| **协议**   | LGPL-3.0            | Apache-2.0   |
| **接口**   | 内部 API              | EVMC 标准      |
| **最新版本** | v1.14.x             | v0.18.0      |
| **设计目标** | 完整客户端               | 独立执行引擎       |

### 1.2 架构定位

```
┌─────────────────────────────────────────────────────────────┐
│                    架构定位对比                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  go-ethereum EVM:                                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │   go-ethereum Client                                │   │
│  │   ┌─────────────────────────────────────────────┐   │   │
│  │   │  P2P │ Consensus │ State │ TxPool │ RPC    │   │   │
│  │   └─────────────────────────────────────────────┘   │   │
│  │                      │                              │   │
│  │                      ▼                              │   │
│  │   ┌─────────────────────────────────────────────┐   │   │
│  │   │            内置 EVM 解释器                   │   │   │
│  │   │      紧密耦合，为 geth 优化                  │   │   │
│  │   └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  evmone:                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │   Any EVMC-Compatible Client                        │   │
│  │   ┌─────────────────────────────────────────────┐   │   │
│  │   │  geth │ nethermind │ besu │ erigon │ ...   │   │   │
│  │   └─────────────────────────────────────────────┘   │   │
│  │                      │ EVMC API                     │   │
│  │                      ▼                              │   │
│  │   ┌─────────────────────────────────────────────┐   │   │
│  │   │            evmone Library                    │   │   │
│  │   │      模块化设计，可插拔替换                   │   │   │
│  │   └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 核心架构对比

### 2.1 解释器设计

#### go-ethereum: 单一解释器

```go
// go-ethereum 的解释器结构
type EVMInterpreter struct {
evm   *EVM
table *JumpTable // 操作码跳转表
}

// 主执行循环
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) ([]byte, error) {
// 简单的 for 循环 + switch 分发
for {
op := contract.GetOp(pc)
operation := in.table[op]

// 每条指令检查 gas
if !contract.UseGas(operation.constantGas) {
return nil, ErrOutOfGas
}

// 执行指令
res, err := operation.execute(&pc, in, callContext)
// ...
}
}
```

#### evmone: 双解释器架构

```cpp
// evmone 提供两种解释器

// 1. Baseline 解释器 - 简单直接
namespace evmone::baseline {
    // 最小化的 JUMPDEST 分析
    // 简单的指令分发循环
    evmc_result execute(evmc_vm* vm, const evmc_host_interface* host,
                        evmc_host_context* ctx, evmc_revision rev,
                        const evmc_message* msg, const uint8_t* code,
                        size_t code_size) noexcept;
}

// 2. Advanced 解释器 - 高度优化
namespace evmone::advanced {
    // 间接调用线程化 (Indirect Call Threading)
    // 预计算 gas 成本和栈需求
    // 基本块优化
    evmc_result execute(evmc_vm* vm, const evmc_host_interface* host,
                        evmc_host_context* ctx, evmc_revision rev,
                        const evmc_message* msg, const uint8_t* code,
                        size_t code_size) noexcept;
}
```

### 2.2 指令分发机制

#### go-ethereum: Switch 分发

```go
// go-ethereum 使用函数表 + switch
type JumpTable [256]*operation

type operation struct {
execute     executionFunc
constantGas uint64
dynamicGas  gasFunc
minStack    int
maxStack    int
memorySize  memorySizeFunc
}

// 每条指令独立执行和检查
func opAdd(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
x, y := scope.Stack.pop(), scope.Stack.peek()
y.Add(&x, y)
return nil, nil
}
```

#### evmone: 间接调用线程化 (Indirect Call Threading)

```cpp
// evmone Advanced 使用间接调用线程化
// 程序被编译为指向指令实现函数的指针表

struct AdvancedCodeAnalysis {
    // 预编译的指令序列
    std::vector<Instruction> instructions;

    // 基本块信息
    std::vector<BlockInfo> blocks;
};

struct Instruction {
    // 直接指向实现函数的指针
    InstrFn fn;

    // 预计算的参数
    int16_t gas_block_index;
    int16_t stack_req;
    int16_t stack_change;
};

// 执行循环 - 无 switch 开销
while (true) {
    // 直接通过函数指针调用
    const auto result = instr->fn(instr, state);
    if (result.status != EVMC_SUCCESS)
        return result;
    ++instr;
}
```

### 2.3 基本块优化

evmone 的核心优化之一是基本块（Basic Block）预计算：

```cpp
// evmone 的基本块分析
struct BasicBlock {
    // 累积的基础 gas 成本
    int64_t gas_cost;

    // 执行此块所需的最小栈高度
    int16_t stack_required;

    // 栈高度的最大增长
    int16_t stack_max_growth;
};

// 基本块边界
// - 开始: JUMPDEST 或代码起始
// - 结束: JUMP, JUMPI, STOP, RETURN, REVERT, INVALID, SELFDESTRUCT

// 执行时只在块边界检查一次
void check_requirements(const BasicBlock& block, ExecutionState& state) {
    // 一次性检查整个块的需求
    if (state.gas_left < block.gas_cost)
        throw OutOfGas();
    if (state.stack.size() < block.stack_required)
        throw StackUnderflow();
    if (state.stack.size() + block.stack_max_growth > 1024)
        throw StackOverflow();
}
```

对比 go-ethereum 的逐指令检查：

```go
// go-ethereum 每条指令都检查
func (in *EVMInterpreter) Run(...) {
for {
op := contract.GetOp(pc)
operation := in.table[op]

// 每条指令检查栈
if sLen := stack.len(); sLen < operation.minStack {
return nil, &ErrStackUnderflow{}
} else if sLen > operation.maxStack {
return nil, &ErrStackOverflow{}
}

// 每条指令检查 gas
cost := operation.constantGas
if !contract.UseGas(cost) {
return nil, ErrOutOfGas
}
// ...
}
}
```

## 3. 性能对比

### 3.1 基准测试数据

根据 [官方性能报告](https://notes.ethereum.org/@ipsilon/evm-performance-report-geth-vs-evmone)：

| 测试项          | geth | evmone/Baseline | evmone/Advanced | geth 相对慢 |
|--------------|------|-----------------|-----------------|----------|
| **几何平均**     | 100% | 31%             | 35%             | **3.2x** |
| blake2b_huff | 100% | 18%             | 17%             | 5.5x     |
| sha1_divs    | 100% | 22%             | 18%             | 4.5x     |
| weierstrudel | 100% | 28%             | 24%             | 3.6x     |
| MSTORE 密集    | 100% | 14%             | 16%             | **6.0x** |

```
┌─────────────────────────────────────────────────────────────┐
│                   性能对比图 (相对执行时间)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  geth            ████████████████████████████████████ 100%  │
│                                                             │
│  evmone/Baseline ███████████ 31%                            │
│                                                             │
│  evmone/Advanced ████████████ 35%                           │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  Gas 处理效率 (Gas/秒):                                      │
│                                                             │
│  evmone/Baseline ████████████████████████████████████ 100%  │
│  evmone/Advanced ████████████████████████████████ 88%       │
│  geth            ████████ 20%                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 性能差异原因分析

```
┌─────────────────────────────────────────────────────────────┐
│                    性能差异原因                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 语言级别差异                                             │
│     ┌─────────────────────────────────────────────────┐    │
│     │  Go:                                            │    │
│     │  • 垃圾回收 (GC) 开销                           │    │
│     │  • 运行时调度开销                               │    │
│     │  • 接口调用动态分发                             │    │
│     │                                                 │    │
│     │  C++:                                           │    │
│     │  • 零成本抽象                                   │    │
│     │  • 编译时优化                                   │    │
│     │  • 内联函数展开                                 │    │
│     │  • 无 GC 开销                                   │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
│  2. 架构设计差异                                             │
│     ┌─────────────────────────────────────────────────┐    │
│     │  go-ethereum:                                   │    │
│     │  • 逐指令 gas/栈检查                           │    │
│     │  • 函数表间接调用                               │    │
│     │  • big.Int (已优化为 uint256)                  │    │
│     │                                                 │    │
│     │  evmone:                                        │    │
│     │  • 基本块批量检查                               │    │
│     │  • 间接调用线程化                               │    │
│     │  • intx 原生 256-bit                           │    │
│     │  • 预计算优化                                   │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
│  3. 整数运算                                                 │
│     ┌─────────────────────────────────────────────────┐    │
│     │  go-ethereum (旧):                              │    │
│     │  • big.Int 动态分配                            │    │
│     │  • 每次运算需要 modulo 2^256                   │    │
│     │                                                 │    │
│     │  go-ethereum (新 uint256):                      │    │
│     │  • 定长 [4]uint64                              │    │
│     │  • 提升 22-47% 性能                            │    │
│     │                                                 │    │
│     │  evmone (intx):                                 │    │
│     │  • 编译时优化的 uint256                        │    │
│     │  • SIMD 指令加速                               │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. 256-bit 整数实现对比

### 4.1 go-ethereum: uint256 库

```go
// holiman/uint256 - 专为 EVM 优化
type Int struct {
// 固定 4 个 uint64，避免动态分配
arr [4]uint64
}

// 加法实现
func (z *Int) Add(x, y *Int) *Int {
var carry uint64
z.arr[0], carry = bits.Add64(x.arr[0], y.arr[0], 0)
z.arr[1], carry = bits.Add64(x.arr[1], y.arr[1], carry)
z.arr[2], carry = bits.Add64(x.arr[2], y.arr[2], carry)
z.arr[3], _ = bits.Add64(x.arr[3], y.arr[3], carry)
return z
}

// 性能提升 (相比 big.Int)
// - 区块链同步: 快 0.73%
// - 执行阶段: 快 ~10%
// - 计算密集: 快 22-47%
```

### 4.2 evmone: intx 库

```cpp
// chfast/intx - C++ 模板库
template <unsigned N>
struct uint {
    static_assert(N >= 2);
    using word_type = uint64_t;
    static constexpr auto num_words = N;

    word_type words_[N]{};

    // 编译时展开的加法
    constexpr uint& operator+=(const uint& y) noexcept {
        uint64_t carry = 0;
        for (size_t i = 0; i < N; ++i) {
            const auto s = words_[i] + y.words_[i];
            const auto c = s < words_[i];
            words_[i] = s + carry;
            carry = (words_[i] < s) | c;
        }
        return *this;
    }
};

using uint256 = uint<4>;

// 优势:
// - constexpr 编译时计算
// - 自动内联和循环展开
// - 无运行时开销
```

## 5. EVMC 接口标准

evmone 实现了 [EVMC](https://github.com/ethereum/evmc) 标准接口，这是其模块化的关键：

### 5.1 EVMC 架构

```cpp
// EVMC Host 接口 - 客户端实现
struct evmc_host_interface {
    // 账户操作
    evmc_account_exists_fn account_exists;
    evmc_get_storage_fn get_storage;
    evmc_set_storage_fn set_storage;
    evmc_get_balance_fn get_balance;

    // 环境信息
    evmc_get_tx_context_fn get_tx_context;
    evmc_get_block_hash_fn get_block_hash;

    // 执行
    evmc_call_fn call;
    evmc_emit_log_fn emit_log;

    // EIP-1153 临时存储
    evmc_get_transient_storage_fn get_transient_storage;
    evmc_set_transient_storage_fn set_transient_storage;
};

// EVMC VM 接口 - evmone 实现
struct evmc_vm {
    int abi_version;
    const char* name;
    const char* version;

    evmc_destroy_fn destroy;
    evmc_execute_fn execute;
    evmc_get_capabilities_fn get_capabilities;
    evmc_set_option_fn set_option;
};

// 执行消息
struct evmc_message {
    evmc_call_kind kind;        // CALL, DELEGATECALL, CREATE 等
    uint32_t flags;
    int32_t depth;
    int64_t gas;
    evmc_address recipient;
    evmc_address sender;
    const uint8_t* input_data;
    size_t input_size;
    evmc_uint256be value;
    evmc_bytes32 create2_salt;
    evmc_address code_address;
};

// 执行结果
struct evmc_result {
    evmc_status_code status_code;
    int64_t gas_left;
    int64_t gas_refund;
    const uint8_t* output_data;
    size_t output_size;
    evmc_address create_address;
};
```

### 5.2 与 geth 集成

```bash
# 使用 evmone 作为 geth 的 EVM 插件
geth --vm.evm=./libevmone.so

# 需要修改版的 geth (支持 EVMC)
# 或使用 EVMC 绑定
```

```go
// Go EVMC 绑定示例
package evmc

/*
#cgo LDFLAGS: -levmone
#include <evmc/evmc.h>
*/
import "C"

type VM struct {
	handle *C.struct_evmc_vm
}

func Load(path string) (*VM, error) {
	handle := C.evmc_load_and_create()
	return &VM{handle: handle}, nil
}

func (vm *VM) Execute(host Host, rev Revision, msg *Message, code []byte) (*Result, error) {
	// 调用 evmone 执行
	result := C.evmc_execute(
		vm.handle,
		host.cInterface(),
		host.cContext(),
		C.enum_evmc_revision(rev),
		msg.cMessage(),
		(*C.uint8_t)(&code[0]),
		C.size_t(len(code)),
	)
	return wrapResult(result), nil
}
```

## 6. EOF 支持对比

### 6.1 evmone EOF 实现

evmone 曾是 EOF (EVM Object Format) 的参考实现：

```cpp
// EOF 容器结构
struct EOFContainer {
    uint8_t magic[2] = {0xEF, 0x00};
    uint8_t version = 1;

    // 类型部分 - 每个代码部分的元数据
    struct TypeSection {
        uint8_t inputs;      // 输入栈项数
        uint8_t outputs;     // 输出栈项数
        uint16_t max_stack;  // 最大栈高度
    };

    // 代码部分 - 实际字节码
    std::vector<bytes> code_sections;

    // 数据部分
    bytes data_section;
};

// EOF 验证
enum class EOFValidationError {
    success,
    invalid_prefix,
    unknown_version,
    incomplete_section_size,
    // ...
};

EOFValidationError validate_eof(bytes_view code, evmc_revision rev);
```

### 6.2 EOF 新指令

```cpp
// EOF 特有指令 (evmone 实现)

// 数据访问指令
DATALOAD      // 从数据部分加载 32 字节
DATALOADN     // 从数据部分加载 32 字节 (立即数偏移)
DATASIZE      // 获取数据部分大小
DATACOPY      // 复制数据到内存

// 子程序指令
CALLF         // 调用代码部分
RETF          // 从代码部分返回
JUMPF         // 跳转到代码部分

// 创建指令
EOFCREATE     // 创建 EOF 合约
RETURNCONTRACT // 返回合约代码

// 扩展栈操作
DUPN          // DUP1-DUP256
SWAPN         // SWAP1-SWAP256
EXCHANGE      // 任意栈位置交换
```

### 6.3 go-ethereum EOF 状态

go-ethereum 的 EOF 实现进度较慢，但也在跟进：

```go
// go-ethereum EOF 支持 (开发中)
type EOFContainer struct {
Version      byte
TypeSection  []FunctionType
CodeSections [][]byte
DataSection  []byte
}

// EOF 验证
func ValidateEOF(code []byte) error {
// 验证魔数
if len(code) < 2 || code[0] != 0xEF || code[1] != 0x00 {
return ErrInvalidMagic
}
// ...
}
```

## 7. 预编译合约对比

### 7.1 evmone 预编译实现

```cpp
// evmone 预编译合约
namespace evmone::state {

// 椭圆曲线优化 - 使用 Jacobian 坐标
struct JacobianPoint {
    intx::uint256 x;
    intx::uint256 y;
    intx::uint256 z;
};

// ecrecover - 签名恢复
ExecutionResult ecrecover(const uint8_t* input, size_t size) noexcept;

// bn256 曲线操作
ExecutionResult bn256_add(const uint8_t* input, size_t size) noexcept;
ExecutionResult bn256_mul(const uint8_t* input, size_t size) noexcept;
ExecutionResult bn256_pairing(const uint8_t* input, size_t size) noexcept;

// blake2f 压缩函数
ExecutionResult blake2f(const uint8_t* input, size_t size) noexcept;

// EIP-2537: BLS12-381 曲线
ExecutionResult bls12_g1_add(const uint8_t* input, size_t size) noexcept;
ExecutionResult bls12_g1_mul(const uint8_t* input, size_t size) noexcept;
ExecutionResult bls12_g1_msm(const uint8_t* input, size_t size) noexcept;
ExecutionResult bls12_g2_add(const uint8_t* input, size_t size) noexcept;
ExecutionResult bls12_pairing(const uint8_t* input, size_t size) noexcept;
ExecutionResult bls12_map_fp_to_g1(const uint8_t* input, size_t size) noexcept;

// EIP-7212: secp256r1/P-256 验证
ExecutionResult p256verify(const uint8_t* input, size_t size) noexcept;

}
```

### 7.2 go-ethereum 预编译

```go
// go-ethereum 预编译合约
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

// 预编译接口
type PrecompiledContract interface {
RequiredGas(input []byte) uint64
Run(input []byte) ([]byte, error)
}
```

## 8. 代码分析工具对比

### 8.1 evmone 工具

```bash
# evmone-eofparse - EOF 验证工具
evmone-eofparse bytecode.hex

# evmc run - 字节码执行
evmc run --vm ./libevmone.so bytecode.hex --input calldata.hex

# evmc-vmtester - VM 兼容性测试
evmc-vmtester ./libevmone.so
```

### 8.2 go-ethereum 工具

```bash
# evm - 字节码执行和调试
evm --code 6001600101 run

# evm disasm - 反汇编
evm disasm 6001600101

# evm statetest - 状态测试执行
evm statetest test.json
```

## 9. 内存管理对比

### 9.1 evmone 内存管理

```cpp
// evmone 使用固定大小栈和线性内存
class ExecutionState {
public:
    // 固定大小栈 - 无动态分配
    intx::uint256 stack[1024];
    int stack_top = 0;

    // 线性内存 - 按需增长
    bytes memory;

    // Gas 追踪
    int64_t gas_left;
    int64_t gas_refund = 0;

    // 返回数据
    bytes return_data;

    // 执行状态
    evmc_status_code status = EVMC_SUCCESS;
};

// 内存扩展 - 精确计算
size_t memory_cost(size_t new_size) noexcept {
    const auto w = (new_size + 31) / 32;  // 字数
    return 3 * w + w * w / 512;            // gas 成本
}
```

### 9.2 go-ethereum 内存管理

```go
// go-ethereum 内存管理
type Memory struct {
store       []byte
lastGasCost uint64
}

// 栈使用切片
type Stack struct {
data []uint256.Int
}

// 需要边界检查和可能的重新分配
func (m *Memory) Resize(size uint64) {
if uint64(len(m.store)) < size {
m.store = append(m.store, make([]byte, size-uint64(len(m.store)))...)
}
}
```

## 10. 测试和验证

### 10.1 测试覆盖对比

| 测试类型    | go-ethereum | evmone            |
|---------|-------------|-------------------|
| 单元测试    | ✅ 完整        | ✅ 完整              |
| 状态测试    | ✅ 官方测试套件    | ✅ 官方测试套件          |
| 模糊测试    | ✅           | ✅ evmone-fuzzer   |
| 基准测试    | ✅           | ✅ 专用基准套件          |
| EOF 测试  | 🔄 开发中      | ✅ evmone-eofparse |
| EVMC 测试 | ❌ 不适用       | ✅ evmc-vmtester   |

### 10.2 Fuzzing 支持

```cpp
// evmone fuzzing 入口
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // EOF 验证 fuzzing
    evmone::validate_eof({data, size}, EVMC_PRAGUE);

    // 执行 fuzzing
    evmone::execute(host, EVMC_PRAGUE, msg, data, size);

    return 0;
}
```

## 11. 硬分叉支持

### 11.1 版本支持对比

| 硬分叉            | go-ethereum | evmone    |
|----------------|-------------|-----------|
| Homestead      | ✅           | ✅         |
| Byzantium      | ✅           | ✅         |
| Constantinople | ✅           | ✅         |
| Istanbul       | ✅           | ✅         |
| Berlin         | ✅           | ✅         |
| London         | ✅           | ✅         |
| Paris (Merge)  | ✅           | ✅         |
| Shanghai       | ✅           | ✅         |
| Cancun         | ✅           | ✅         |
| Prague         | ✅           | ✅         |
| Osaka          | 🔄          | ✅ v0.17.0 |

### 11.2 EIP 实现示例

```cpp
// evmone: EIP-1153 临时存储
void op_tload(ExecutionState& state) noexcept {
    const auto key = state.stack.pop();
    const auto value = state.host.get_transient_storage(
        state.msg->recipient,
        intx::be::store<evmc_bytes32>(key)
    );
    state.stack.push(intx::be::load<uint256>(value));
}

void op_tstore(ExecutionState& state) noexcept {
    const auto key = state.stack.pop();
    const auto value = state.stack.pop();
    state.host.set_transient_storage(
        state.msg->recipient,
        intx::be::store<evmc_bytes32>(key),
        intx::be::store<evmc_bytes32>(value)
    );
}
```

## 12. 使用建议

### 12.1 何时使用 evmone

```
┌─────────────────────────────────────────────────────────────┐
│                    evmone 适用场景                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ 推荐使用:                                                │
│  • 高性能执行环境（3-6x 性能提升）                          │
│  • 需要 EVMC 标准接口的项目                                 │
│  • 智能合约测试和模拟                                       │
│  • EVM 研究和实验                                           │
│  • 非 Go 技术栈的集成                                       │
│  • EOF 新特性测试                                           │
│                                                             │
│  ⚠️ 需要注意:                                               │
│  • 需要 EVMC 兼容的客户端                                   │
│  • C++ 依赖（编译/运行时）                                  │
│  • 与 geth 内部 API 不兼容                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 何时使用 go-ethereum EVM

```
┌─────────────────────────────────────────────────────────────┐
│                 go-ethereum EVM 适用场景                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ 推荐使用:                                                │
│  • 完整以太坊客户端部署                                     │
│  • Go 技术栈项目                                            │
│  • 需要与 geth 深度集成                                     │
│  • 需要调试追踪功能                                         │
│  • 主网验证节点                                             │
│                                                             │
│  优势:                                                       │
│  • 与 geth 生态完全兼容                                     │
│  • 丰富的调试工具                                           │
│  • 活跃的社区支持                                           │
│  • 成熟稳定的实现                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 13. Go 集成 evmone 示例

### 13.1 CGO 绑定

```go
package evmone

/*
#cgo CFLAGS: -I${SRCDIR}/include
#cgo LDFLAGS: -L${SRCDIR}/lib -levmone -lstdc++

#include <evmc/evmc.h>
#include <evmc/loader.h>

extern struct evmc_vm* evmc_create_evmone(void);
*/
import "C"
import (
	"unsafe"
)

// EvmoneVM evmone 虚拟机
type EvmoneVM struct {
	vm *C.struct_evmc_vm
}

// NewEvmoneVM 创建 evmone 实例
func NewEvmoneVM() *EvmoneVM {
	vm := C.evmc_create_evmone()
	return &EvmoneVM{vm: vm}
}

// Execute 执行字节码
func (e *EvmoneVM) Execute(
	host *Host,
	rev Revision,
	msg *Message,
	code []byte,
) (*Result, error) {
	var codePtr *C.uint8_t
	if len(code) > 0 {
		codePtr = (*C.uint8_t)(unsafe.Pointer(&code[0]))
	}

	result := C.evmc_execute(
		e.vm,
		host.interface_,
		host.context,
		C.enum_evmc_revision(rev),
		(*C.struct_evmc_message)(unsafe.Pointer(msg)),
		codePtr,
		C.size_t(len(code)),
	)

	return wrapResult(&result), nil
}

// Revision EVM 版本
type Revision int

const (
	Frontier Revision = iota
	Homestead
	TangerineWhistle
	SpuriousDragon
	Byzantium
	Constantinople
	Petersburg
	Istanbul
	Berlin
	London
	Paris
	Shanghai
	Cancun
	Prague
)

// Message 执行消息
type Message struct {
	Kind        CallKind
	Flags       uint32
	Depth       int32
	Gas         int64
	Recipient   Address
	Sender      Address
	InputData   []byte
	Value       [32]byte
	Create2Salt [32]byte
	CodeAddress Address
}

// Result 执行结果
type Result struct {
	StatusCode    StatusCode
	GasLeft       int64
	GasRefund     int64
	Output        []byte
	CreateAddress Address
}
```

### 13.2 Host 接口实现

```go
package evmone

/*
#include <evmc/evmc.h>

// Host 回调函数声明
bool account_exists_cb(struct evmc_host_context*, const evmc_address*);
evmc_bytes32 get_storage_cb(struct evmc_host_context*, const evmc_address*, const evmc_bytes32*);
// ... 其他回调
*/
import "C"

// Host EVMC Host 实现
type Host struct {
	interface_ *C.struct_evmc_host_interface
	context    *C.struct_evmc_host_context

	// Go 状态
	state StateDB
	env   *ExecutionEnv
}

// NewHost 创建 Host
func NewHost(state StateDB, env *ExecutionEnv) *Host {
	h := &Host{
		state: state,
		env:   env,
	}

	// 设置回调函数
	h.interface_ = &C.struct_evmc_host_interface{
		account_exists: C.evmc_account_exists_fn(C.account_exists_cb),
		get_storage:    C.evmc_get_storage_fn(C.get_storage_cb),
		set_storage:    C.evmc_set_storage_fn(C.set_storage_cb),
		get_balance:    C.evmc_get_balance_fn(C.get_balance_cb),
		// ... 其他回调
	}

	return h
}

//export account_exists_cb
func account_exists_cb(ctx *C.struct_evmc_host_context, addr *C.evmc_address) C.bool {
	host := getHost(ctx)
	address := addressFromC(addr)
	return C.bool(host.state.Exist(address))
}

//export get_storage_cb
func get_storage_cb(ctx *C.struct_evmc_host_context, addr *C.evmc_address, key *C.evmc_bytes32) C.evmc_bytes32 {
	host := getHost(ctx)
	address := addressFromC(addr)
	k := hashFromC(key)
	value := host.state.GetState(address, k)
	return hashToC(value)
}
```

## 14. 总结

### 14.1 核心优势对比

| 方面         | go-ethereum EVM | evmone        |
|------------|-----------------|---------------|
| **性能**     | 基准 (100%)       | 3-6x 更快 (31%) |
| **语言生态**   | Go 原生           | C++ (需要绑定)    |
| **集成度**    | geth 深度集成       | EVMC 标准接口     |
| **EIP 支持** | 紧跟主网            | 略微领先          |
| **EOF 支持** | 开发中             | 已实现           |
| **调试工具**   | 丰富              | 基础            |
| **社区**     | 最大              | 较小但活跃         |

### 14.2 evmone 的创新点

1. **双解释器架构** - Baseline (简单) + Advanced (优化)
2. **间接调用线程化** - 消除 switch 分发开销
3. **基本块优化** - 批量 gas/栈检查
4. **EVMC 标准** - 模块化可替换
5. **EOF 先行** - 新格式参考实现
6. **高效整数库** - intx 编译时优化

### 14.3 选择建议

```
┌─────────────────────────────────────────────────────────────┐
│                      选择决策树                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  需要完整以太坊客户端?                                       │
│  ├─ 是 → go-ethereum (geth)                                │
│  │       └─ 可选: 使用 evmone 作为 EVM 插件提升性能         │
│  │                                                         │
│  └─ 否 → 继续评估                                          │
│          │                                                 │
│          ├─ 需要最高执行性能?                               │
│          │  └─ 是 → evmone                                 │
│          │                                                 │
│          ├─ Go 技术栈?                                     │
│          │  └─ 是 → go-ethereum EVM                        │
│          │                                                 │
│          ├─ 需要 EVMC 兼容?                                │
│          │  └─ 是 → evmone                                 │
│          │                                                 │
│          └─ 研究/测试用途?                                  │
│             └─ evmone (更快的反馈循环)                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 附录 A：evmone 核心源码分析

### A.1 执行状态 (ExecutionState)

evmone 的执行状态设计非常精简高效：

```cpp
// evmone/lib/evmone/execution_state.hpp

namespace evmone {

/// 栈空间 - 固定大小，无动态分配
class StackSpace {
public:
    /// 最大 1024 个元素，每个 256-bit 对齐
    alignas(sizeof(intx::uint256)) intx::uint256 m_data[1024];

    /// 获取栈底指针
    [[nodiscard]] intx::uint256* bottom() noexcept {
        return m_data;
    }
};

/// 内存管理 - 初始 4KB，2倍增长
class Memory {
    static constexpr size_t initial_capacity = 4096;
    std::vector<uint8_t> m_data;

public:
    Memory() noexcept {
        m_data.reserve(initial_capacity);
    }

    /// 扩展内存 - 只在需要时分配
    void grow(size_t new_size) noexcept {
        if (new_size > m_data.size())
            m_data.resize(new_size);
    }
};

/// 执行状态 - EVM 指令实现的通用状态
class ExecutionState {
public:
    int64_t gas_refund = 0;
    Memory memory;
    const evmc_message* msg = nullptr;
    evmc::HostContext host;
    evmc_revision rev = {};
    bytes return_data;

    /// 当前指令的输出
    bytes_view output;
    evmc_status_code status = EVMC_SUCCESS;

    /// 代码和分析
    bytes_view original_code;
    const EOF1Header* eof_header = nullptr;

    /// 栈空间和指针
    StackSpace stack_space;
    intx::uint256* stack_top;
    int64_t gas_left = 0;

    /// 初始化执行状态
    ExecutionState(const evmc_message& message,
                   evmc_revision revision,
                   const evmc_host_interface& host_interface,
                   evmc_host_context* host_ctx,
                   bytes_view code) noexcept
        : msg{&message},
          host{host_interface, host_ctx},
          rev{revision},
          original_code{code},
          stack_top{stack_space.bottom()},
          gas_left{message.gas} {}
};

}  // namespace evmone
```

### A.2 指令实现 (Instructions)

evmone 的指令实现采用"核心+包装"模式：

```cpp
// evmone/lib/evmone/instructions.hpp

namespace evmone::instr::core {

/// 栈顶管理类
class StackTop {
    intx::uint256* m_top;

public:
    explicit StackTop(intx::uint256* top) noexcept : m_top{top} {}

    /// 弹出栈顶
    [[nodiscard]] intx::uint256 pop() noexcept {
        return *m_top--;
    }

    /// 压入栈顶
    void push(const intx::uint256& value) noexcept {
        *++m_top = value;
    }

    /// 查看栈顶（不弹出）
    [[nodiscard]] intx::uint256& top() noexcept {
        return *m_top;
    }

    /// 索引访问
    [[nodiscard]] intx::uint256& operator[](size_t index) noexcept {
        return *(m_top - index);
    }
};

/// ADD 指令 - 简洁高效
inline void add(StackTop stack) noexcept {
    stack.top() += stack.pop();
}

/// MUL 指令
inline void mul(StackTop stack) noexcept {
    stack.top() *= stack.pop();
}

/// SUB 指令
inline void sub(StackTop stack) noexcept {
    stack[1] = stack[0] - stack[1];
    stack.pop();
}

/// DIV 指令 - 处理除零
inline void div(StackTop stack) noexcept {
    auto& x = stack[1];
    x = x != 0 ? stack[0] / x : 0;
    stack.pop();
}

/// SDIV 指令 - 有符号除法
inline void sdiv(StackTop stack) noexcept {
    auto& x = stack[1];
    const auto y = stack[0];
    // 处理特殊情况: MIN_INT256 / -1 = MIN_INT256
    x = (x != 0 && !(x == intx::uint256{1} << 255 && y == ~intx::uint256{0}))
        ? intx::sdivrem(y, x).quot : 0;
    stack.pop();
}

/// EXP 指令 - 指数运算
inline void exp(StackTop stack) noexcept {
    const auto base = stack.pop();
    auto& exponent = stack.top();
    exponent = intx::exp(base, exponent);
}

/// KECCAK256 指令
inline Result keccak256(StackTop stack, ExecutionState& state) noexcept {
    const auto offset = stack.pop();
    auto& size = stack.top();

    // 检查内存扩展
    if (!check_memory(state, offset, size))
        return {EVMC_OUT_OF_GAS, state.gas_left};

    // 计算哈希
    const auto data = state.memory.data() + static_cast<size_t>(offset);
    size = intx::be::load<intx::uint256>(
        ethash::keccak256(data, static_cast<size_t>(size)));

    return {EVMC_SUCCESS, state.gas_left};
}

/// MLOAD 指令
inline Result mload(StackTop stack, ExecutionState& state) noexcept {
    auto& offset = stack.top();

    if (!check_memory(state, offset, 32))
        return {EVMC_OUT_OF_GAS, state.gas_left};

    offset = intx::be::unsafe::load<intx::uint256>(
        state.memory.data() + static_cast<size_t>(offset));

    return {EVMC_SUCCESS, state.gas_left};
}

/// MSTORE 指令
inline Result mstore(StackTop stack, ExecutionState& state) noexcept {
    const auto offset = stack.pop();
    const auto value = stack.pop();

    if (!check_memory(state, offset, 32))
        return {EVMC_OUT_OF_GAS, state.gas_left};

    intx::be::unsafe::store(
        state.memory.data() + static_cast<size_t>(offset), value);

    return {EVMC_SUCCESS, state.gas_left};
}

/// PUSH<N> 模板指令
template <size_t N>
inline void push(StackTop stack, const uint8_t* code, size_t code_size, size_t pc) noexcept {
    static_assert(N >= 1 && N <= 32);

    intx::uint256 value;
    const auto pos = pc + 1;
    const auto num_bytes = std::min(N, code_size - pos);

    if (num_bytes == N) {
        // 完整数据
        value = intx::be::unsafe::load<intx::uint256>(code + pos);
        if constexpr (N < 32)
            value >>= (32 - N) * 8;
    } else {
        // 部分数据（代码末尾）
        value = 0;
        for (size_t i = 0; i < num_bytes; ++i)
            value = (value << 8) | code[pos + i];
        value <<= (N - num_bytes) * 8;
    }

    stack.push(value);
}

/// DUP<N> 模板指令
template <size_t N>
inline void dup(StackTop stack) noexcept {
    static_assert(N >= 1 && N <= 16);
    stack.push(stack[N - 1]);
}

/// SWAP<N> 模板指令
template <size_t N>
inline void swap(StackTop stack) noexcept {
    static_assert(N >= 1 && N <= 16);
    std::swap(stack.top(), stack[N]);
}

}  // namespace evmone::instr::core
```

### A.3 基本块分析 (Advanced Analysis)

evmone 的核心优化在于基本块预分析：

```cpp
// evmone/lib/evmone/advanced_analysis.cpp

namespace evmone::advanced {

/// 块分析状态
struct BlockAnalysis {
    int64_t gas_cost = 0;        // 累积 gas 成本
    int stack_req = 0;           // 最小栈需求
    int stack_max_growth = 0;    // 最大栈增长
    int stack_change = 0;        // 当前栈变化

    /// 关闭块，生成压缩信息
    BlockInfo close() const noexcept {
        return {
            static_cast<int64_t>(gas_cost),
            static_cast<int16_t>(stack_req),
            static_cast<int16_t>(stack_max_growth)
        };
    }
};

/// 分析字节码
AdvancedCodeAnalysis analyze(
    evmc_revision rev,
    bytes_view code) noexcept
{
    AdvancedCodeAnalysis analysis;

    // 预分配空间避免重新分配
    analysis.instructions.reserve(code.size() + 1);
    analysis.push_values.reserve(code.size() / 2);

    const auto& op_table = get_op_table(rev);
    BlockAnalysis block;

    for (size_t pc = 0; pc < code.size(); ++pc) {
        const auto op = code[pc];
        const auto& op_info = op_table[op];

        // 更新栈需求
        // stack_req = max(所有指令需求 - 之前的净变化)
        block.stack_req = std::max(
            block.stack_req,
            op_info.stack_req - block.stack_change);

        // 更新栈变化
        block.stack_change += op_info.stack_change;

        // 更新最大增长
        block.stack_max_growth = std::max(
            block.stack_max_growth,
            block.stack_change);

        // 累积 gas 成本
        block.gas_cost += op_info.gas_cost;

        // 构建指令
        Instruction instr;
        instr.fn = op_info.fn;

        // 处理 PUSH 指令
        if (op >= OP_PUSH1 && op <= OP_PUSH32) {
            const auto n = static_cast<size_t>(op - OP_PUSH0);
            const auto push_data_offset = pc + 1;
            const auto push_data_size = std::min(n, code.size() - push_data_offset);

            // 存储 push 值
            if (n <= 8) {
                // 小值内联存储
                uint64_t value = 0;
                for (size_t i = 0; i < push_data_size; ++i)
                    value = (value << 8) | code[push_data_offset + i];
                instr.small_push_value = value << ((8 - n) * 8);
            } else {
                // 大值外部存储
                const auto idx = analysis.push_values.size();
                analysis.push_values.resize(idx + 32);
                std::copy_n(code.data() + push_data_offset,
                           push_data_size,
                           analysis.push_values.data() + idx + (32 - n));
                instr.push_value_offset = idx;
            }

            pc += n;  // 跳过 push 数据
        }

        // 处理块边界
        if (op == OP_JUMPDEST) {
            // JUMPDEST 开始新块
            analysis.jumpdest_map[pc] = analysis.instructions.size();
            block = BlockAnalysis{};
        } else if (is_terminator(op)) {
            // 终止指令结束当前块
            analysis.blocks.push_back(block.close());
            block = BlockAnalysis{};

            // 跳过死代码
            while (pc + 1 < code.size() && code[pc + 1] != OP_JUMPDEST)
                ++pc;
        }

        analysis.instructions.push_back(instr);
    }

    // 添加隐式 STOP
    analysis.instructions.push_back({op_stop});
    analysis.blocks.push_back(block.close());

    return analysis;
}

}  // namespace evmone::advanced
```

### A.4 Baseline vs Advanced 执行循环

```cpp
// Baseline 执行 - 简单但高效
namespace evmone::baseline {

template <evmc_revision Rev>
evmc_result execute(ExecutionState& state, const CodeAnalysis& analysis) noexcept {
    const auto* code = state.original_code.data();
    const auto code_size = state.original_code.size();

    auto pc = size_t{0};
    while (true) {
        const auto op = code[pc];

        // 逐指令检查
        if (!check_requirements<Rev>(state, op))
            return make_result(state.status, state.gas_left, {});

        // 分发执行
        switch (op) {
            case OP_STOP:
                return make_result(EVMC_SUCCESS, state.gas_left, {});

            case OP_ADD:
                instr::core::add(StackTop{state.stack_top});
                break;

            case OP_MUL:
                instr::core::mul(StackTop{state.stack_top});
                break;

            // ... 所有 256 个操作码
        }

        ++pc;
    }
}

}  // namespace evmone::baseline

// Advanced 执行 - 高度优化
namespace evmone::advanced {

evmc_result execute(ExecutionState& state, const AdvancedCodeAnalysis& analysis) noexcept {
    const auto* instr = analysis.instructions.data();

    // 主执行循环 - 间接调用线程化
    while (instr != nullptr) {
        // 直接通过函数指针调用，无 switch 开销
        instr = instr->fn(instr, state);
    }

    return make_result(state.status, state.gas_left, state.output);
}

// 指令函数签名
using InstrFn = const Instruction* (*)(const Instruction*, ExecutionState&) noexcept;

// ADD 指令实现
const Instruction* op_add(const Instruction* instr, ExecutionState& state) noexcept {
    instr::core::add(StackTop{state.stack_top});
    return instr + 1;  // 返回下一条指令
}

// 块开始 - 一次性检查整个块
const Instruction* op_block_begin(const Instruction* instr, ExecutionState& state) noexcept {
    const auto& block = *reinterpret_cast<const BlockInfo*>(instr->arg);

    // 一次性检查：gas、栈下溢、栈溢出
    if (state.gas_left < block.gas_cost) {
        state.status = EVMC_OUT_OF_GAS;
        return nullptr;
    }
    if (state.stack_top - state.stack_space.bottom() < block.stack_req) {
        state.status = EVMC_STACK_UNDERFLOW;
        return nullptr;
    }
    if (state.stack_top - state.stack_space.bottom() + block.stack_max_growth > 1024) {
        state.status = EVMC_STACK_OVERFLOW;
        return nullptr;
    }

    state.gas_left -= block.gas_cost;
    return instr + 1;
}

}  // namespace evmone::advanced
```

### A.5 与 go-ethereum 的代码对比

```go
// go-ethereum/core/vm/interpreter.go

// Run 方法 - 逐指令执行
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
// 初始化
in.evm.depth++
defer func () { in.evm.depth-- }()

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
)

// 主循环
for {
// 获取操作码
op = contract.GetOp(pc)
operation := in.table[op]

// 栈验证 - 每条指令都检查
if sLen := stack.len(); sLen < operation.minStack {
return nil, &ErrStackUnderflow{stackLen: sLen, required: operation.minStack}
} else if sLen > operation.maxStack {
return nil, &ErrStackOverflow{stackLen: sLen, limit: operation.maxStack}
}

// Gas 检查 - 每条指令都检查
cost = operation.constantGas
if !contract.UseGas(cost) {
return nil, ErrOutOfGas
}

// 动态 gas
if operation.dynamicGas != nil {
var memorySize uint64
if operation.memorySize != nil {
memSize, overflow := operation.memorySize(stack)
if overflow {
return nil, ErrGasUintOverflow
}
if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
return nil, ErrGasUintOverflow
}
}

var dynamicCost uint64
dynamicCost, err = operation.dynamicGas(in.evm, contract, stack, mem, memorySize)
if err != nil || !contract.UseGas(dynamicCost) {
return nil, ErrOutOfGas
}

if memorySize > 0 {
mem.Resize(memorySize)
}
}

// 执行
res, err := operation.execute(&pc, in, callContext)

if err != nil {
break
}
pc++
}

return nil, err
}
```

**关键差异总结：**

| 方面       | go-ethereum | evmone      |
|----------|-------------|-------------|
| **检查频率** | 每条指令        | 每个基本块       |
| **分发方式** | 函数指针表       | 间接调用线程化     |
| **预分析**  | 仅 JUMPDEST  | 完整基本块分析     |
| **栈实现**  | Go slice    | 固定数组        |
| **内存分配** | 动态          | 预分配 + 2x 增长 |

---

## 附录 B：性能优化技术详解

### B.1 间接调用线程化 (Indirect Call Threading)

```
传统 Switch 分发:
┌──────────────────────────────────────────┐
│  for {                                   │
│      op = code[pc]                       │
│      switch op {        ← 分支预测失败     │
│          case ADD: ...                   │
│          case MUL: ...                   │
│          case SUB: ...                   │
│          ...                             │
│      }                                   │
│      pc++                                │
│  }                                       │
└──────────────────────────────────────────┘

间接调用线程化:
┌──────────────────────────────────────────┐
│  // 预编译为函数指针序列                    │
│  instructions = [&add, &mul, &sub, ...]  │
│                                          │
│  instr = instructions[0]                 │
│  while (instr) {                         │
│      instr = instr->fn(instr, state)     │
│             ↑                            │
│      直接调用，无分支                       │
│  }                                       │
└──────────────────────────────────────────┘

性能提升: 消除 switch 带来的分支预测失败
```

### B.2 基本块批量检查

```
逐指令检查 (go-ethereum):
┌─────────────────────────────────────────────┐
│ PUSH1 0x60   → 检查 gas ✓ 检查栈 ✓           │
│ PUSH1 0x40   → 检查 gas ✓ 检查栈 ✓           │
│ MSTORE       → 检查 gas ✓ 检查栈 ✓           │
│ CALLVALUE    → 检查 gas ✓ 检查栈 ✓           │
│ DUP1         → 检查 gas ✓ 检查栈 ✓           │
│ ISZERO       → 检查 gas ✓ 检查栈 ✓           │
│ PUSH2 0x0010 → 检查 gas ✓ 检查栈 ✓           │
│ JUMPI        → 检查 gas ✓ 检查栈 ✓           │
│                                            │
│ 总计: 8 次 gas 检查 + 8 次栈检查 = 16 次       │
└─────────────────────────────────────────────┘

基本块检查 (evmone):
┌─────────────────────────────────────────────┐
│ [块开始] → 检查 gas=24 ✓ 栈需求=0 ✓            │
│ PUSH1 0x60                                  │
│ PUSH1 0x40                                  │
│ MSTORE                                      │
│ CALLVALUE                                   │
│ DUP1                                        │
│ ISZERO                                      │
│ PUSH2 0x0010                                │
│ JUMPI [块结束]                               │
│                                             │
│ 总计: 1 次 gas 检查 + 1 次栈检查 = 2 次        │
└─────────────────────────────────────────────┘

性能提升: 减少 87.5% 的检查开销
```

### B.3 编译器优化

```cpp
// evmone 利用 C++ 编译器优化

// 1. constexpr 编译时计算
constexpr uint64_t gas_costs[] = {
    [OP_STOP] = 0,
    [OP_ADD] = 3,
    [OP_MUL] = 5,
    // ...
};

// 2. 强制内联
#define EVMONE_ALWAYS_INLINE [[gnu::always_inline]] inline

EVMONE_ALWAYS_INLINE
void add(StackTop stack) noexcept {
    stack.top() += stack.pop();
}

// 3. noexcept 优化
// 告诉编译器不会抛出异常，优化异常处理代码

// 4. [[nodiscard]] 属性
// 强制检查返回值，避免错误

// 5. 分支预测提示
#define EVMONE_UNLIKELY(x) __builtin_expect(!!(x), 0)

if (EVMONE_UNLIKELY(gas_left < 0)) {
    return EVMC_OUT_OF_GAS;
}
```

---

## Appendix C: EOF (EVM Object Format) 支持对比

### C.1 EOF 概述

[EOF](https://evmobjectformat.xyz/) (EVM Object Format) 是 EVM
字节码的新容器格式，通过 [EIP-7692](https://eips.ethereum.org/EIPS/eip-7692) 定义，计划在 Fusaka 硬分叉中启用。

```
┌────────────────────────────────────────────────────────────────┐
│                    EOF 结构                                     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Magic (0xEF00) │ Version │ Header │ Code │ Data          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  Legacy Bytecode:                                              │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Code + Data 混合，无结构，需要 JUMPDEST 分析              │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  EOF Bytecode:                                                 │
│  ┌────────┬────────┬──────────┬──────────┬──────────────────┐ │
│  │ Header │ Types  │ Code[0]  │ Code[n]  │ Data Section     │ │
│  │        │ Section│ Section  │ Section  │                  │ │
│  └────────┴────────┴──────────┴──────────┴──────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### C.2 EOF 核心改进

| 特性          | Legacy EVM        | EOF                   |
|-------------|-------------------|-----------------------|
| **代码/数据分离** | 混合存储              | 明确分离                  |
| **跳转验证**    | 运行时 JUMPDEST 分析   | 部署时一次性验证              |
| **跳转类型**    | 动态跳转 (JUMP/JUMPI) | 静态相对跳转 (RJUMP/RJUMPI) |
| **函数支持**    | 无原生支持             | 一等公民函数 (CALLF/RETF)   |
| **Gas 观察**  | 可通过 GAS 指令        | 移除 GAS 指令             |
| **Code 观察** | CODECOPY/CODESIZE | 移除，改用 DATALOAD        |
| **版本控制**    | 无                 | 支持版本化升级               |

### C.3 evmone EOF 实现

evmone 是 EOF 规范的参考实现之一，由 Ipsilon 团队开发：

```cpp
// evmone EOF 验证器
namespace evmone::eof {

/// EOF 容器结构
struct Container {
    bytes_view header;           // 头部
    std::vector<TypeSection> types;  // 类型段
    std::vector<bytes_view> code_sections;  // 代码段
    bytes_view data;             // 数据段

    // EOF 版本
    uint8_t version;
};

/// 验证 EOF 字节码（部署时调用）
[[nodiscard]]
ValidationResult validate(bytes_view code, evmc_revision rev) noexcept {
    // 1. 检查 magic bytes (0xEF00)
    if (code.size() < 2 || code[0] != 0xEF || code[1] != 0x00)
        return ValidationResult::invalid_prefix;

    // 2. 解析头部
    auto header = parse_header(code);
    if (!header)
        return header.error();

    // 3. 验证类型段（函数签名）
    for (const auto& type : header->types) {
        if (!validate_type_section(type))
            return ValidationResult::invalid_type_section;
    }

    // 4. 验证代码段
    for (size_t i = 0; i < header->code_sections.size(); ++i) {
        auto result = validate_code_section(
            header->code_sections[i],
            header->types[i],
            rev
        );
        if (result != ValidationResult::success)
            return result;
    }

    // 5. 验证控制流（静态分析）
    return validate_control_flow(*header);
}

/// EOF 特有指令
// RJUMP: 静态相对跳转（替代 JUMP）
// RJUMPI: 条件相对跳转（替代 JUMPI）
// RJUMPV: 跳转表（switch-case 优化）
// CALLF: 调用函数
// RETF: 函数返回
// JUMPF: 尾调用优化
// DATALOAD: 从 data section 加载
// DATALOADN: 加载固定偏移数据
// DATACOPY: 复制数据到内存
// EOFCREATE: 创建 EOF 合约
// RETURNCONTRACT: 从 initcode 返回

}
```

### C.4 go-ethereum EOF 实现

go-ethereum 的 EOF 实现在 `core/vm/eof.go`：

```go
// go-ethereum EOF 结构
type EOF struct {
TypeSections []TypeSection
CodeSections [][]byte
DataSection  []byte
Container    []byte
}

// EOF 验证
func (eof *EOF) Validate() error {
// 类似 evmone 的验证流程
// 但使用 Go 风格的错误处理

if err := eof.validateHeader(); err != nil {
return err
}

for i, code := range eof.CodeSections {
if err := validateCode(code, eof.TypeSections[i]); err != nil {
return fmt.Errorf("code section %d: %w", i, err)
}
}

return eof.validateControlFlow()
}

// EOF 执行器 - 需要特殊处理
type EOFInterpreter struct {
*EVMInterpreter
container *EOF
}

func (in *EOFInterpreter) Run(contract *Contract, input []byte) ([]byte, error) {
// EOF 代码无需 JUMPDEST 分析
// 使用静态跳转表
// 支持 CALLF/RETF 函数调用
}
```

### C.5 EOF 对解释器的影响

```
┌────────────────────────────────────────────────────────────────┐
│              EOF 对性能的提升                                   │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Legacy EVM 执行流程:                                          │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 1. 加载字节码                                             │ │
│  │ 2. JUMPDEST 分析（扫描整个代码）← 开销大                  │ │
│  │ 3. 执行循环                                               │ │
│  │    - 每次 JUMP 查找目标是否有效                           │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  EOF 执行流程:                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 1. 加载字节码（已在部署时验证）                           │ │
│  │ 2. 无需 JUMPDEST 分析！                                   │ │
│  │ 3. 执行循环                                               │ │
│  │    - RJUMP 直接跳转（目标已验证）                         │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  性能提升（evmone 测试）:                                      │
│  - 启动时间: 减少 ~10-20%（无需分析）                          │
│  - 跳转指令: 减少 ~5-10%（静态跳转更快）                       │
│  - 总体提升: ~7% (根据合约不同)                                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### C.6 evmone EOF 版本历史

| 版本      | EOF 状态  | 说明                               |
|---------|---------|----------------------------------|
| v0.10.0 | 初始实现    | EIP-3540, 3670, 4200, 4750, 5450 |
| v0.11.0 | 增强      | DATALOAD, DATACOPY, JUMPF        |
| v0.12.0 | 完整 v1.0 | EOFCREATE, RETURNCONTRACT        |
| v0.18.0 | **移除**  | EOF 移至 Osaka，从 Prague 移除         |

> 注：由于 EOF 在以太坊主网的时间表变化，evmone v0.18.0 暂时移除了 EOF 支持，等待 Osaka 硬分叉确定后重新加入。

---

## Appendix D: EVMC 接口标准详解

### D.1 EVMC 概述

[EVMC](https://github.com/ethereum/evmc) (Ethereum VM Connector API) 是一个 C 语言接口标准，允许不同的 EVM 实现与以太坊客户端集成。

```
┌────────────────────────────────────────────────────────────────┐
│                    EVMC 架构                                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                   以太坊客户端                            │ │
│  │  ┌────────────┐  ┌────────────┐  ┌──────────────────┐   │ │
│  │  │ 共识层     │  │ P2P 网络   │  │ 状态管理          │   │ │
│  │  └────────────┘  └────────────┘  └──────────────────┘   │ │
│  └────────────────────────┬─────────────────────────────────┘ │
│                           │ EVMC Host Interface               │
│                           ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              EVMC C ABI (版本 12)                         │ │
│  │  ┌────────────────────────────────────────────────────┐  │ │
│  │  │ evmc_vm      │ evmc_host_interface │ evmc_message  │  │ │
│  │  │ evmc_result  │ evmc_tx_context     │ evmc_revision │  │ │
│  │  └────────────────────────────────────────────────────┘  │ │
│  └────────────────────────┬─────────────────────────────────┘ │
│                           │ EVMC VM Interface                  │
│                           ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────────────────┐  │ │
│  │  │ evmone  │    │ Aleth   │    │  其他 EVM 实现      │  │ │
│  │  │ (C++)   │    │ (C++)   │    │  (任何语言)         │  │ │
│  │  └─────────┘    └─────────┘    └─────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### D.2 核心数据结构

```c
// EVMC 基本类型
typedef struct evmc_bytes32 {
    uint8_t bytes[32];  // 256 位整数/哈希
} evmc_bytes32;

typedef struct evmc_address {
    uint8_t bytes[20];  // 以太坊地址
} evmc_address;

// 调用消息
struct evmc_message {
    enum evmc_call_kind kind;      // CALL, DELEGATECALL, CREATE 等
    uint32_t flags;                 // 标志位
    int32_t depth;                  // 调用深度
    int64_t gas;                    // 可用 gas
    evmc_address recipient;         // 接收方
    evmc_address sender;            // 发送方
    const uint8_t* input_data;      // 输入数据
    size_t input_size;
    evmc_uint256be value;           // 转账金额
    evmc_bytes32 create2_salt;      // CREATE2 盐
    evmc_address code_address;      // 代码地址
    const uint8_t* code;            // 可选：代码
    size_t code_size;
};

// 执行结果
struct evmc_result {
    enum evmc_status_code status_code;  // 状态码
    int64_t gas_left;                    // 剩余 gas
    int64_t gas_refund;                  // 退款
    const uint8_t* output_data;          // 输出数据
    size_t output_size;
    evmc_release_result_fn release;      // 释放回调
    evmc_address create_address;         // CREATE 返回地址
    uint8_t padding[4];
};

// 交易上下文
struct evmc_tx_context {
    evmc_uint256be tx_gas_price;
    evmc_address tx_origin;
    evmc_address block_coinbase;
    int64_t block_number;
    int64_t block_timestamp;
    int64_t block_gas_limit;
    evmc_uint256be block_prev_randao;
    evmc_uint256be chain_id;
    evmc_uint256be block_base_fee;
    evmc_uint256be blob_base_fee;
    const evmc_bytes32* blob_hashes;
    size_t blob_hashes_count;
    uint8_t initcodes_count;
};
```

### D.3 Host Interface（客户端实现）

```c
// Host 回调函数表
struct evmc_host_interface {
    // 账户查询
    bool (*account_exists)(struct evmc_host_context* context,
                          const evmc_address* address);

    // 存储操作
    evmc_bytes32 (*get_storage)(struct evmc_host_context* context,
                                const evmc_address* address,
                                const evmc_bytes32* key);

    enum evmc_storage_status (*set_storage)(
        struct evmc_host_context* context,
        const evmc_address* address,
        const evmc_bytes32* key,
        const evmc_bytes32* value);

    // 余额查询
    evmc_uint256be (*get_balance)(struct evmc_host_context* context,
                                  const evmc_address* address);

    // 代码操作
    size_t (*get_code_size)(struct evmc_host_context* context,
                            const evmc_address* address);

    evmc_bytes32 (*get_code_hash)(struct evmc_host_context* context,
                                  const evmc_address* address);

    size_t (*copy_code)(struct evmc_host_context* context,
                        const evmc_address* address,
                        size_t code_offset,
                        uint8_t* buffer_data,
                        size_t buffer_size);

    // 自毁
    bool (*selfdestruct)(struct evmc_host_context* context,
                         const evmc_address* address,
                         const evmc_address* beneficiary);

    // 嵌套调用
    struct evmc_result (*call)(struct evmc_host_context* context,
                               const struct evmc_message* msg);

    // 获取交易上下文
    struct evmc_tx_context (*get_tx_context)(
        struct evmc_host_context* context);

    // 区块哈希
    evmc_bytes32 (*get_block_hash)(struct evmc_host_context* context,
                                   int64_t number);

    // 日志
    void (*emit_log)(struct evmc_host_context* context,
                     const evmc_address* address,
                     const uint8_t* data, size_t data_size,
                     const evmc_bytes32 topics[], size_t topics_count);

    // EIP-2929 访问追踪
    enum evmc_access_status (*access_account)(
        struct evmc_host_context* context,
        const evmc_address* address);

    enum evmc_access_status (*access_storage)(
        struct evmc_host_context* context,
        const evmc_address* address,
        const evmc_bytes32* key);

    // EIP-1153 瞬态存储
    evmc_bytes32 (*get_transient_storage)(
        struct evmc_host_context* context,
        const evmc_address* address,
        const evmc_bytes32* key);

    void (*set_transient_storage)(
        struct evmc_host_context* context,
        const evmc_address* address,
        const evmc_bytes32* key,
        const evmc_bytes32* value);
};
```

### D.4 VM Interface（evmone 实现）

```c
// VM 结构体
struct evmc_vm {
    int abi_version;              // EVMC ABI 版本
    const char* name;             // VM 名称
    const char* version;          // VM 版本
    evmc_destroy_fn destroy;      // 销毁函数
    evmc_execute_fn execute;      // 执行函数
    evmc_get_capabilities_fn get_capabilities;  // 能力查询
    evmc_set_option_fn set_option;  // 设置选项
};

// evmone 的入口点
EVMC_EXPORT struct evmc_vm* evmc_create_evmone(void) {
    static struct evmc_vm vm = {
        .abi_version = EVMC_ABI_VERSION,
        .name = "evmone",
        .version = PROJECT_VERSION,
        .destroy = evmone_destroy,
        .execute = evmone_execute,
        .get_capabilities = evmone_get_capabilities,
        .set_option = evmone_set_option,
    };
    return &vm;
}

// 执行函数
struct evmc_result evmone_execute(
    struct evmc_vm* vm,
    const struct evmc_host_interface* host,
    struct evmc_host_context* ctx,
    enum evmc_revision rev,
    const struct evmc_message* msg,
    const uint8_t* code,
    size_t code_size
) {
    // 选择解释器
    if (vm->use_advanced) {
        return advanced::execute(vm, host, ctx, rev, msg, code, code_size);
    } else {
        return baseline::execute(vm, host, ctx, rev, msg, code, code_size);
    }
}
```

### D.5 EVMC vs go-ethereum 内部 API

| 方面      | EVMC     | go-ethereum 内部 API |
|---------|----------|--------------------|
| **语言**  | C ABI    | Go interfaces      |
| **跨语言** | ✅ 任何语言   | ❌ 仅 Go/CGo         |
| **版本化** | ABI 版本号  | 无正式版本              |
| **标准化** | 是        | 否                  |
| **开销**  | C 调用约定   | Go 方法调用            |
| **灵活性** | 高（可替换实现） | 低（紧耦合）             |
| **调试**  | 较难（跨边界）  | 较易                 |

### D.6 EVMC 的优势与局限

**优势：**

```cpp
// 1. 实现可替换
// 客户端可以选择不同的 EVM 实现
evm = evmc_create_evmone();  // 使用 evmone
// 或
evm = evmc_create_hera();    // 使用 Hera (WebAssembly)

// 2. 语言无关
// C ABI 允许任何语言实现
// Rust, Go, Python 都可以通过 FFI 调用

// 3. 测试隔离
// EVM 实现可以独立于客户端测试
```

**局限：**

```cpp
// 1. FFI 开销
// 每次 Host 回调都是函数指针调用
// 无法内联优化

// 2. 数据复制
// 跨边界传递数据可能需要复制
evmc_bytes32 storage_value = host->get_storage(ctx, &addr, &key);
// 返回的是值，而非引用

// 3. 状态管理复杂
// 需要通过 context 指针管理状态
struct evmc_host_context {
    StateDB* state;      // 客户端状态
    Transaction* tx;     // 当前交易
    // ...
};
```

---

## Appendix E: evmone-compiler (AOT 编译器)

### E.1 项目概述

[evmone-compiler](https://github.com/megaeth-labs/evmone-compiler) 是基于 evmone 的 AOT (Ahead-of-Time) 编译器，由
MegaETH Labs 开发。

```
┌────────────────────────────────────────────────────────────────┐
│                解释器 vs AOT 编译器                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  解释器执行 (evmone):                                          │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ EVM 字节码 → [分析] → 指令序列 → [逐条解释执行]            │ │
│  │                                                          │ │
│  │ 每次执行都要：解码 → 分发 → 执行                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  AOT 编译执行 (evmone-compiler):                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ EVM 字节码 → [LLVM 编译] → 原生机器码 → [直接执行]         │ │
│  │                                                          │ │
│  │ 编译一次，之后无解释开销                                   │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  性能对比:                                                     │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 解释器:    ████████████████████ 100%                     │ │
│  │ AOT:       ████████ 40-60%（取决于合约）                  │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### E.2 编译流程

```cpp
// evmone-compiler 编译流程

// 1. 解析 EVM 字节码
AdvancedCodeAnalysis analysis = analyze(bytecode, rev);

// 2. 生成 LLVM IR
llvm::Module* module = generateIR(analysis);

// 生成的 IR 示例（ADD 指令）：
/*
define void @evm_add(%StackTop* %stack) {
entry:
    %a = call i256 @stack_pop(%stack)
    %b = call i256 @stack_top(%stack)
    %result = add i256 %a, %b
    call void @stack_set_top(%stack, %result)
    ret void
}
*/

// 3. LLVM 优化
llvm::PassManager PM;
PM.add(createInstructionCombiningPass());
PM.add(createReassociatePass());
PM.add(createGVNPass());
PM.add(createCFGSimplificationPass());
PM.run(*module);

// 4. 生成机器码
llvm::TargetMachine* TM = getTargetMachine();
std::string obj_code;
TM->emitObjectFile(*module, obj_code);

// 5. 链接并缓存
CompiledContract* compiled = link_and_cache(obj_code, code_hash);
```

### E.3 AOT vs JIT vs 解释器

| 方面         | 解释器                 | JIT  | AOT                |
|------------|---------------------|------|--------------------|
| **启动时间**   | 快                   | 中等   | 慢（需预编译）            |
| **执行速度**   | 慢                   | 快    | 最快                 |
| **内存使用**   | 低                   | 中等   | 高（缓存机器码）           |
| **适用场景**   | 一次性调用               | 热点合约 | 已知热点合约             |
| **evmone** | ✅ Baseline/Advanced | ❌    | ⚠️ evmone-compiler |

### E.4 与 go-ethereum 的对比

```
go-ethereum:
┌─────────────────────────────────────────────────────────┐
│ 纯解释器 + Go 编译器优化                                 │
│ - 无 JIT/AOT                                            │
│ - 依赖 Go 编译器的内联优化                               │
│ - 适合通用场景                                          │
└─────────────────────────────────────────────────────────┘

evmone + evmone-compiler:
┌─────────────────────────────────────────────────────────┐
│ 分层执行策略                                            │
│                                                         │
│ 冷合约 → Baseline 解释器（快速启动）                     │
│    ↓ 调用次数增加                                       │
│ 温合约 → Advanced 解释器（优化分析）                     │
│    ↓ 识别为热点                                         │
│ 热合约 → AOT 编译（LLVM 机器码）                         │
│                                                         │
│ 类似 JVM 的分层编译策略                                 │
└─────────────────────────────────────────────────────────┘
```

### E.5 限制与挑战

```cpp
// AOT 编译的挑战

// 1. 动态跳转
// EVM 的 JUMP/JUMPI 目标是运行时确定的
// AOT 需要特殊处理
switch (jump_target) {
    case 0x0010: goto label_0010;
    case 0x0020: goto label_0020;
    // ... 所有可能的 JUMPDEST
    default: return EVMC_BAD_JUMP_DESTINATION;
}

// 2. Gas 计算
// 必须在编译后代码中保留 gas 检查
// 不能完全消除

// 3. 状态访问
// SLOAD/SSTORE 仍需通过 Host 接口
// 无法被编译优化

// 4. 代码大小
// 编译后的机器码可能很大
// 需要缓存管理策略

// 5. 安全性
// 生成的代码必须是沙箱安全的
// 不能访问任意内存
```

---

## Appendix F: 测试基础设施

### F.1 evmone 测试工具

evmone 提供了完整的测试基础设施：

```
┌────────────────────────────────────────────────────────────────┐
│                 evmone 测试工具链                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    evmone-t8n                             │ │
│  │  状态转换工具（Transition Tool）                          │ │
│  │  - 执行单个交易并输出状态变化                              │ │
│  │  - 与 execution-spec-tests 配合使用                       │ │
│  │  - 支持所有硬分叉版本                                     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                   evmone-eofparse                         │ │
│  │  EOF 验证工具                                             │ │
│  │  - 解析和验证 EOF 字节码                                  │ │
│  │  - 输出详细的验证错误信息                                 │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                 evmone-eofparsefuzz                       │ │
│  │  EOF 模糊测试工具                                         │ │
│  │  - 生成随机 EOF 字节码                                    │ │
│  │  - 发现边界情况和漏洞                                     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    evmone-bench                           │ │
│  │  性能基准测试                                             │ │
│  │  - 测量指令执行时间                                       │ │
│  │  - 对比不同解释器性能                                     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### F.2 execution-spec-tests 集成

evmone 与以太坊官方的 [execution-spec-tests](https://github.com/ethereum/execution-spec-tests) 深度集成：

```python
# execution-spec-tests 测试流程

# 1. Python 定义测试用例
@pytest.mark.valid_from("London")
def test_simple_transfer():
    """测试简单 ETH 转账"""
    pre = {
        sender: Account(balance=1_000_000),
        receiver: Account(balance=0),
    }

    tx = Transaction(
        to=receiver,
        value=100,
        gas_limit=21000,
    )

    post = {
        sender: Account(balance=1_000_000 - 100 - 21000 * gas_price),
        receiver: Account(balance=100),
    }

    yield StateTest(pre=pre, tx=tx, post=post)

# 2. 使用 evmone-t8n 执行
# $ fill -k test_simple_transfer --evm-bin evmone-t8n

# 3. 生成 JSON fixture
# ./fixtures/state_tests/test_simple_transfer.json
```

### F.3 模糊测试（Fuzzing）

```cpp
// evmone 的 fuzzing 配置
// CMakeLists.txt

if(EVMONE_FUZZING)
    # 启用 fuzzing 相关编译选项
    add_compile_options(
        -fsanitize=fuzzer-no-link,address,undefined
        -fno-omit-frame-pointer
    )

    # EOF 解析 fuzzer
    add_executable(evmone-eofparsefuzz
        test/fuzzer/eofparsefuzz.cpp
    )
    target_link_libraries(evmone-eofparsefuzz
        evmone
        -fsanitize=fuzzer,address,undefined
    )

    # 执行 fuzzer
    add_executable(evmone-execfuzz
        test/fuzzer/execfuzz.cpp
    )
endif()
```

**EVMFuzz 差分测试：**

```
┌────────────────────────────────────────────────────────────────┐
│                  差分模糊测试架构                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  种子合约生成器                                                 │
│       │                                                        │
│       ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              随机 EVM 字节码                              │  │
│  └─────────────────────────────────────────────────────────┘  │
│       │                                                        │
│       ├──────────────┬──────────────┬──────────────┐          │
│       ▼              ▼              ▼              ▼          │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐       │
│  │ evmone  │   │  geth   │   │  besu   │   │ netherm │       │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘       │
│       │              │              │              │          │
│       └──────────────┴──────────────┴──────────────┘          │
│                      │                                         │
│                      ▼                                         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              结果交叉比对                                 │  │
│  │  - 状态根是否一致                                        │  │
│  │  - Gas 消耗是否相同                                      │  │
│  │  - 返回值是否相等                                        │  │
│  └─────────────────────────────────────────────────────────┘  │
│                      │                                         │
│              不一致 → 发现潜在漏洞                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘

EVMFuzz 研究成果：
- 发现 5 个未知安全漏洞
- 全部被收录到 CVE 数据库
- 覆盖 4 个主流 EVM 实现
```

### F.4 与 go-ethereum 测试对比

| 方面         | evmone              | go-ethereum         |
|------------|---------------------|---------------------|
| **单元测试框架** | GoogleTest → Catch2 | Go testing          |
| **状态测试**   | evmone-t8n          | evm t8n             |
| **模糊测试**   | libFuzzer           | go-fuzz             |
| **性能测试**   | evmone-bench        | internal benchmarks |
| **CI/CD**  | GitHub Actions      | GitHub Actions      |
| **代码覆盖**   | gcov/llvm-cov       | go test -cover      |

---

## Appendix G: 集成示例

### G.1 通过 EVMC 集成 evmone

```cpp
// 在以太坊客户端中集成 evmone

#include <evmc/evmc.hpp>
#include <evmc/loader.h>

class EthereumClient {
    evmc_vm* evm = nullptr;

public:
    // 初始化 evmone
    void init_evm() {
        // 动态加载 evmone 库
        evmc_loader_error_code error;
        evm = evmc_load_and_create("libevmone.so", &error);

        if (error != EVMC_LOADER_SUCCESS) {
            throw std::runtime_error("Failed to load evmone");
        }

        // 设置选项：使用 Advanced 解释器
        evmc_set_option(evm, "advanced", "");

        // 验证 ABI 版本
        if (!evmc_is_abi_compatible(evm)) {
            throw std::runtime_error("ABI version mismatch");
        }
    }

    // 执行合约
    evmc_result execute_contract(
        const evmc_message& msg,
        const uint8_t* code,
        size_t code_size,
        evmc_revision rev
    ) {
        // 设置 Host 回调
        static const evmc_host_interface host_interface = {
            .account_exists = my_account_exists,
            .get_storage = my_get_storage,
            .set_storage = my_set_storage,
            .get_balance = my_get_balance,
            .get_code_size = my_get_code_size,
            .get_code_hash = my_get_code_hash,
            .copy_code = my_copy_code,
            .selfdestruct = my_selfdestruct,
            .call = my_call,
            .get_tx_context = my_get_tx_context,
            .get_block_hash = my_get_block_hash,
            .emit_log = my_emit_log,
            .access_account = my_access_account,
            .access_storage = my_access_storage,
            .get_transient_storage = my_get_transient_storage,
            .set_transient_storage = my_set_transient_storage,
        };

        // 执行
        return evm->execute(
            evm,
            &host_interface,
            &my_host_context,
            rev,
            &msg,
            code,
            code_size
        );
    }

    ~EthereumClient() {
        if (evm) {
            evm->destroy(evm);
        }
    }
};

// Host 回调实现示例
bool my_account_exists(
    evmc_host_context* ctx,
    const evmc_address* addr
) {
    auto* state = static_cast<StateDB*>(ctx);
    return state->account_exists(*addr);
}

evmc_bytes32 my_get_storage(
    evmc_host_context* ctx,
    const evmc_address* addr,
    const evmc_bytes32* key
) {
    auto* state = static_cast<StateDB*>(ctx);
    return state->get_storage(*addr, *key);
}
```

### G.2 在 Rust 中使用 evmone（通过 FFI）

```rust
// evmc-sys crate 提供 Rust 绑定

use evmc_sys::*;
use std::ffi::CString;

pub struct Evmone {
    vm: *mut evmc_vm,
}

impl Evmone {
    pub fn new() -> Result<Self, &'static str> {
        unsafe {
            // 加载 evmone
            let path = CString::new("libevmone.so").unwrap();
            let mut error: evmc_loader_error_code = 0;
            let vm = evmc_load_and_create(path.as_ptr(), &mut error);

            if error != EVMC_LOADER_SUCCESS || vm.is_null() {
                return Err("Failed to load evmone");
            }

            Ok(Self { vm })
        }
    }

    pub fn execute(
        &self,
        host: &evmc_host_interface,
        ctx: *mut evmc_host_context,
        rev: evmc_revision,
        msg: &evmc_message,
        code: &[u8],
    ) -> evmc_result {
        unsafe {
            let execute_fn = (*self.vm).execute.unwrap();
            execute_fn(
                self.vm,
                host,
                ctx,
                rev,
                msg,
                code.as_ptr(),
                code.len(),
            )
        }
    }
}

impl Drop for Evmone {
    fn drop(&mut self) {
        unsafe {
            if !self.vm.is_null() {
                let destroy_fn = (*self.vm).destroy.unwrap();
                destroy_fn(self.vm);
            }
        }
    }
}
```

### G.3 Python 绑定（pyevmone）

```python
# pyevmone - Python 绑定

from pyevmone import EVM, Message, Revision

# 创建 EVM 实例
evm = EVM()

# 配置
evm.set_option("advanced", "true")

# 准备消息
msg = Message(
    kind=Message.CALL,
    sender=bytes.fromhex("0" * 40),
    recipient=bytes.fromhex("0" * 40),
    value=0,
    input_data=b"",
    gas=100000,
)

# 执行
code = bytes.fromhex("6080604052...")  # 合约字节码
result = evm.execute(
    host=my_host,
    revision=Revision.CANCUN,
    message=msg,
    code=code,
)

print(f"Status: {result.status}")
print(f"Gas left: {result.gas_left}")
print(f"Output: {result.output.hex()}")
```

---

## 附录 H: EIP 兼容性矩阵

### H.1 硬分叉支持状态

```
EVM 实现硬分叉支持对比 (截至 2024 年底):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
硬分叉          │ go-ethereum │ evmone │ 关键变化
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Frontier        │     ✓       │   ✓    │ 初始 EVM
Homestead       │     ✓       │   ✓    │ DELEGATECALL
Tangerine       │     ✓       │   ✓    │ Gas 重定价
Spurious Dragon │     ✓       │   ✓    │ 状态清理
Byzantium       │     ✓       │   ✓    │ REVERT, STATICCALL
Constantinople  │     ✓       │   ✓    │ CREATE2, EXTCODEHASH
Petersburg      │     ✓       │   ✓    │ 移除 EIP-1283
Istanbul        │     ✓       │   ✓    │ CHAINID, SELFBALANCE
Berlin          │     ✓       │   ✓    │ 访问列表 (EIP-2929)
London          │     ✓       │   ✓    │ BASEFEE, EIP-1559
Paris (Merge)   │     ✓       │   ✓    │ PREVRANDAO
Shanghai        │     ✓       │   ✓    │ PUSH0, Warm COINBASE
Cancun          │     ✓       │   ✓    │ TLOAD/TSTORE, MCOPY
Prague          │   开发中    │ 开发中  │ EOF, BLOCKHASH 变更
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### H.2 关键 EIP 支持详情

```
Cancun (Dencun) 升级 EIP 支持:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EIP      │ 名称                    │ geth │ evmone │ 说明
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EIP-1153 │ Transient Storage       │  ✓   │   ✓    │ TLOAD/TSTORE
EIP-4788 │ Beacon Root in EVM      │  ✓   │   ✓    │ 系统合约
EIP-4844 │ Blob Transactions       │  ✓   │   ✓    │ BLOBHASH, BLOBBASEFEE
EIP-5656 │ MCOPY                   │  ✓   │   ✓    │ 内存复制
EIP-6780 │ SELFDESTRUCT 限制       │  ✓   │   ✓    │ 仅同交易创建可销毁
EIP-7516 │ BLOBBASEFEE 操作码      │  ✓   │   ✓    │ 获取 blob gas 价格
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Prague 升级 EIP 支持 (计划中):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EIP      │ 名称                    │ geth │ evmone │ 说明
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EIP-2537 │ BLS12-381 预编译        │ 开发 │  开发   │ BLS 签名验证
EIP-6110 │ 验证者存款供应          │ 开发 │  开发   │ 存款合约变更
EIP-7002 │ EL 触发提款             │ 开发 │  开发   │ 执行层提款
EIP-7251 │ 验证者最大有效余额      │ 开发 │  开发   │ MaxEB 增加
EIP-7549 │ 委员会索引移出证明      │ 开发 │  开发   │ 证明格式变更
EIP-7692 │ EOF Meta                │ 开发 │  开发   │ EOF 完整支持
EIP-7702 │ EOA 代码设置            │ 开发 │  开发   │ AA 相关
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### H.3 预编译合约支持

```cpp
// evmone 预编译合约映射
static const std::map<evmc_revision, std::set<Address>> PRECOMPILES = {
    {EVMC_FRONTIER, {
        0x01,  // ecRecover
        0x02,  // SHA256
        0x03,  // RIPEMD160
        0x04,  // identity
    }},
    {EVMC_BYZANTIUM, {
        0x05,  // modexp
        0x06,  // ecAdd
        0x07,  // ecMul
        0x08,  // ecPairing
    }},
    {EVMC_ISTANBUL, {
        0x09,  // blake2f
    }},
    {EVMC_CANCUN, {
        0x0a,  // point_evaluation (KZG)
    }},
    {EVMC_PRAGUE, {
        0x0b,  // bls12_g1_add
        0x0c,  // bls12_g1_mul
        0x0d,  // bls12_g1_multiexp
        0x0e,  // bls12_g2_add
        0x0f,  // bls12_g2_mul
        0x10,  // bls12_g2_multiexp
        0x11,  // bls12_pairing
        0x12,  // bls12_map_fp_to_g1
        0x13,  // bls12_map_fp2_to_g2
    }},
};
```

---

## 附录 I: Verkle Trees 支持

### I.1 Verkle Trees 概述

Verkle Trees 是以太坊计划用于替代 Merkle Patricia Trie 的新状态树结构：

```
Merkle Patricia Trie vs Verkle Tree:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
特性              │ MPT          │ Verkle Tree
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
证明大小          │ ~3-4 KB      │ ~150-200 bytes
验证复杂度        │ O(log n)     │ O(1) 常数时间
密码学基础        │ Keccak256    │ Pedersen 承诺
分支因子          │ 16           │ 256
状态过期支持      │ 困难         │ 原生支持
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### I.2 EVM 影响

```
Verkle 相关 EIP:
┌─────────────────────────────────────────────────────────────┐
│ EIP-6800: Ethereum State using Verkle Trees                 │
│ - 状态树结构变化                                            │
│ - 新的地址空间布局                                          │
│ - Gas 成本重新计算                                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ EVM 操作码影响:                                              │
│                                                              │
│ SLOAD/SSTORE:                                               │
│   - 从 slot-based 变为 stem + suffix 模式                   │
│   - Gas 计算基于 witness 访问                               │
│                                                              │
│ BALANCE/EXTCODEHASH/EXTCODESIZE/EXTCODECOPY:                │
│   - 访问账户数据的 gas 变化                                 │
│   - 基于 verkle witness 的 gas 计费                         │
│                                                              │
│ CREATE/CREATE2:                                              │
│   - 代码分块存储 (31 字节/块)                               │
│   - 部署 gas 计算方式变化                                   │
└─────────────────────────────────────────────────────────────┘
```

### I.3 evmone Verkle 准备

```cpp
// evmone 的 Verkle 准备工作 (实验性)

// 新的状态访问接口
struct VerkleStateAccess {
    // 账户基本信息 (stem)
    struct AccountStem {
        uint8_t version;
        uint256 balance;
        uint64 nonce;
        bytes32 code_hash;
        uint64 code_size;
    };

    // 存储访问
    struct StorageAccess {
        bytes31 stem;      // 31 字节 stem
        uint8_t suffix;    // 1 字节 suffix (0-255)
        uint256 value;
    };

    // 代码访问 (分块)
    struct CodeChunk {
        bytes31 stem;
        uint8_t chunk_index;  // 0-127 for first 4KB
        bytes32 chunk_data;   // 31 bytes code + 1 byte metadata
    };
};

// Gas 计算变化
constexpr int64_t WITNESS_BRANCH_COST = 1900;   // 访问新 branch
constexpr int64_t WITNESS_CHUNK_COST = 200;     // 访问新 chunk

inline int64_t verkle_sload_gas(bool stem_warm, bool suffix_warm) {
    if (stem_warm && suffix_warm) {
        return 200;  // 完全热访问
    } else if (stem_warm) {
        return WITNESS_CHUNK_COST + 200;  // stem 热，suffix 冷
    } else {
        return WITNESS_BRANCH_COST + WITNESS_CHUNK_COST + 200;  // 全冷
    }
}
```

---

## 附录 J: 安全审计模式

### J.1 使用 evmone 进行合约安全分析

```cpp
// 安全分析 Host 实现
class SecurityAnalysisHost : public evmc::Host {
public:
    // 检测重入攻击
    struct ReentrancyTracker {
        std::stack<evmc::address> call_stack;
        std::set<evmc::address> in_call;

        bool check_reentrancy(const evmc::address& addr) {
            return in_call.count(addr) > 0;
        }

        void enter(const evmc::address& addr) {
            call_stack.push(addr);
            in_call.insert(addr);
        }

        void exit() {
            in_call.erase(call_stack.top());
            call_stack.pop();
        }
    };

    ReentrancyTracker reentrancy;

    // 检测存储冲突
    struct StorageAccessLog {
        std::map<std::pair<evmc::address, evmc::bytes32>,
                 std::vector<AccessType>> accesses;

        void log_access(const evmc::address& addr,
                       const evmc::bytes32& key,
                       AccessType type) {
            accesses[{addr, key}].push_back(type);
        }

        bool has_read_after_write(const evmc::address& addr,
                                  const evmc::bytes32& key) {
            auto& log = accesses[{addr, key}];
            bool written = false;
            for (auto t : log) {
                if (t == WRITE) written = true;
                if (t == READ && written) return true;
            }
            return false;
        }
    };

    StorageAccessLog storage_log;

    evmc::Result call(const evmc_message& msg) noexcept override {
        auto addr = evmc::address(msg.recipient);

        // 重入检测
        if (reentrancy.check_reentrancy(addr)) {
            report_vulnerability("REENTRANCY", addr, msg.sender);
        }

        reentrancy.enter(addr);
        auto result = execute_internal(msg);
        reentrancy.exit();

        return result;
    }

    evmc::bytes32 get_storage(const evmc::address& addr,
                               const evmc::bytes32& key) noexcept override {
        storage_log.log_access(addr, key, READ);
        return internal_get_storage(addr, key);
    }

    evmc_storage_status set_storage(const evmc::address& addr,
                                     const evmc::bytes32& key,
                                     const evmc::bytes32& value) noexcept override {
        storage_log.log_access(addr, key, WRITE);
        return internal_set_storage(addr, key, value);
    }
};
```

### J.2 常见漏洞检测模式

```cpp
// 整数溢出检测 (pre-Solidity 0.8)
class OverflowDetector {
public:
    void check_add(const intx::uint256& a, const intx::uint256& b) {
        intx::uint256 result = a + b;
        if (result < a) {
            report("INTEGER_OVERFLOW_ADD", a, b);
        }
    }

    void check_mul(const intx::uint256& a, const intx::uint256& b) {
        if (a != 0 && b != 0) {
            intx::uint256 result = a * b;
            if (result / a != b) {
                report("INTEGER_OVERFLOW_MUL", a, b);
            }
        }
    }

    void check_sub(const intx::uint256& a, const intx::uint256& b) {
        if (b > a) {
            report("INTEGER_UNDERFLOW_SUB", a, b);
        }
    }
};

// 未检查的外部调用检测
class UncheckedCallDetector {
    std::vector<CallInfo> pending_calls;

public:
    void on_call(const evmc_message& msg) {
        pending_calls.push_back({
            msg.recipient,
            msg.value,
            false  // return_checked
        });
    }

    void on_opcode(uint8_t opcode, const Stack& stack) {
        // 检测 CALL 后是否检查返回值
        if (!pending_calls.empty()) {
            auto& last_call = pending_calls.back();

            // ISZERO 检查返回值
            if (opcode == OP_ISZERO) {
                last_call.return_checked = true;
            }
            // JUMPI 基于返回值跳转
            else if (opcode == OP_JUMPI) {
                last_call.return_checked = true;
            }
            // 如果执行其他操作且未检查
            else if (!last_call.return_checked &&
                     opcode != OP_POP &&
                     opcode != OP_DUP1) {
                report("UNCHECKED_CALL_RETURN", last_call.recipient);
                pending_calls.pop_back();
            }
        }
    }
};

// 访问控制检测
class AccessControlAnalyzer {
    std::set<evmc::bytes32> admin_storage_slots;

public:
    void on_sstore(const evmc::address& contract,
                   const evmc::bytes32& slot,
                   const evmc::bytes32& old_value,
                   const evmc::bytes32& new_value,
                   const evmc::address& caller) {
        // 检测是否修改关键槽位
        if (admin_storage_slots.count(slot)) {
            // 检查是否有适当的访问控制
            if (!has_ownership_check(contract, caller)) {
                report("MISSING_ACCESS_CONTROL", contract, slot);
            }
        }
    }
};
```

### J.3 符号执行框架

```cpp
// 基于 evmone 的简化符号执行
class SymbolicExecutor {
    struct SymbolicValue {
        enum Type { CONCRETE, SYMBOLIC, MIXED };
        Type type;
        intx::uint256 concrete_value;
        std::string symbolic_expr;

        // 约束
        std::vector<std::string> constraints;
    };

    std::vector<SymbolicValue> symbolic_stack;
    std::map<evmc::bytes32, SymbolicValue> symbolic_storage;

public:
    void execute_symbolic(const uint8_t* code, size_t code_size) {
        size_t pc = 0;

        while (pc < code_size) {
            uint8_t opcode = code[pc];

            switch (opcode) {
            case OP_ADD: {
                auto b = symbolic_stack.back(); symbolic_stack.pop_back();
                auto a = symbolic_stack.back(); symbolic_stack.pop_back();

                if (a.type == CONCRETE && b.type == CONCRETE) {
                    symbolic_stack.push_back({
                        CONCRETE,
                        a.concrete_value + b.concrete_value,
                        ""
                    });
                } else {
                    symbolic_stack.push_back({
                        SYMBOLIC,
                        0,
                        "(" + a.symbolic_expr + " + " + b.symbolic_expr + ")"
                    });
                }
                break;
            }

            case OP_SLOAD: {
                auto key = symbolic_stack.back(); symbolic_stack.pop_back();

                if (key.type == CONCRETE) {
                    evmc::bytes32 slot;
                    intx::be::store(slot.bytes, key.concrete_value);

                    if (symbolic_storage.count(slot)) {
                        symbolic_stack.push_back(symbolic_storage[slot]);
                    } else {
                        // 创建新的符号值
                        symbolic_stack.push_back({
                            SYMBOLIC,
                            0,
                            "SLOAD(" + to_hex(slot) + ")"
                        });
                    }
                }
                break;
            }

            case OP_JUMPI: {
                auto dest = symbolic_stack.back(); symbolic_stack.pop_back();
                auto cond = symbolic_stack.back(); symbolic_stack.pop_back();

                if (cond.type == SYMBOLIC) {
                    // 路径分叉
                    fork_execution(cond.symbolic_expr, dest);
                }
                break;
            }
            }

            pc++;
        }
    }

    void fork_execution(const std::string& condition, const SymbolicValue& dest) {
        // 创建两个执行路径
        // Path 1: condition = true
        // Path 2: condition = false
        // 使用 Z3/SMT 求解器检查路径可行性
    }
};
```

---

## 附录 K: 真实 Bug 案例

### K.1 evmone 历史 Bug

```
evmone Bug 修复历史:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Bug #1: RETURNDATACOPY 越界读取 (2021)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: RETURNDATACOPY 未正确检查 offset + size 溢出
影响: 可能读取无效内存
修复:
```

```cpp
// 修复前
void op_returndatacopy(EvmState& state) {
    auto dest_offset = state.stack.pop();
    auto src_offset = state.stack.pop();
    auto size = state.stack.pop();

    // 缺少溢出检查！
    if (src_offset + size > state.return_data.size()) {
        state.status = EVMC_INVALID_MEMORY_ACCESS;
        return;
    }
}

// 修复后
void op_returndatacopy(EvmState& state) {
    auto dest_offset = state.stack.pop();
    auto src_offset = state.stack.pop();
    auto size = state.stack.pop();

    // 添加溢出检查
    if (src_offset > state.return_data.size() ||
        size > state.return_data.size() - src_offset) {
        state.status = EVMC_INVALID_MEMORY_ACCESS;
        return;
    }
}
```

```
Bug #2: CREATE2 地址计算不一致 (2020)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: 空 init_code 时 CREATE2 地址计算与 geth 不一致
原因: hash(salt || keccak256("")) vs hash(salt || "")
修复: 统一使用 keccak256(init_code) 即使 init_code 为空
```

```cpp
// 正确的 CREATE2 地址计算
evmc::address create2_address(
    const evmc::address& sender,
    const evmc::bytes32& salt,
    const uint8_t* init_code,
    size_t init_code_size
) {
    // 始终计算 init_code 的 hash，即使为空
    auto init_code_hash = ethash::keccak256(init_code, init_code_size);

    uint8_t buffer[1 + 20 + 32 + 32];
    buffer[0] = 0xff;
    std::memcpy(buffer + 1, sender.bytes, 20);
    std::memcpy(buffer + 21, salt.bytes, 32);
    std::memcpy(buffer + 53, init_code_hash.bytes, 32);

    auto hash = ethash::keccak256(buffer, sizeof(buffer));

    evmc::address addr;
    std::memcpy(addr.bytes, hash.bytes + 12, 20);
    return addr;
}
```

```
Bug #3: MCOPY 重叠区域处理 (2024, Cancun)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: MCOPY 在源和目标重叠时行为不正确
规范: 应该表现为 "先复制到临时缓冲区，再写入目标"
```

```cpp
// 修复后的 MCOPY 实现
void op_mcopy(EvmState& state) {
    auto dest = static_cast<size_t>(state.stack.pop());
    auto src = static_cast<size_t>(state.stack.pop());
    auto size = static_cast<size_t>(state.stack.pop());

    if (size == 0) return;

    // 扩展内存
    auto mem_cost = memory_expansion_cost(state, std::max(dest, src) + size);
    if (!state.consume_gas(mem_cost + 3 * ((size + 31) / 32))) {
        return;
    }

    // 使用 memmove 处理重叠（不是 memcpy！）
    std::memmove(state.memory.data() + dest,
                 state.memory.data() + src,
                 size);
}
```

### K.2 go-ethereum EVM Bug

```
geth EVM Bug 案例:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Bug #1: EIP-2929 访问列表 Gas 计算 (2021)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: 某些边界情况下访问列表 gas 计算不正确
场景: SLOAD 后紧跟 SSTORE 到同一槽位
```

```go
// 修复前 - gas 可能被多扣
func gasSLoadEIP2929(evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
    slot := common.Hash(stack.peek().Bytes32())
    if _, ok := evm.StateDB.GetTransientState(contract.Address(), slot); !ok {
        // 冷访问
        evm.StateDB.SetTransientState(contract.Address(), slot, common.Hash{1})
        return params.ColdSloadCostEIP2929, nil
    }
    return params.WarmStorageReadCostEIP2929, nil
}

// 修复后 - 正确使用 access list
func gasSLoadEIP2929(evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
    slot := common.Hash(stack.peek().Bytes32())
    if evm.StateDB.SlotInAccessList(contract.Address(), slot) {
        return params.WarmStorageReadCostEIP2929, nil
    }
    evm.StateDB.AddSlotToAccessList(contract.Address(), slot)
    return params.ColdSloadCostEIP2929, nil
}
```

```
Bug #2: SELFDESTRUCT 退款计算 (2022)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: EIP-3529 后退款计算仍使用旧逻辑
影响: 多次 SELFDESTRUCT 可获得过多退款
```

### K.3 差分测试发现的 Bug

```
通过 EVMFuzz 发现的不一致:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Case #1: EXTCODECOPY 对自毁合约的行为
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
输入: 合约 A 调用 SELFDESTRUCT，然后 EXTCODECOPY(A)
Geth:   返回空 (0 字节)
evmone: 返回原始代码
规范:   应返回空 (合约已标记为自毁)
结果:   evmone 修复

Case #2: DELEGATECALL 到预编译合约
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
输入: DELEGATECALL 到 ecRecover (0x01)
Geth:   执行预编译，使用 caller 的存储
evmone: 执行预编译，但 msg.sender 处理不同
规范:   预编译合约不应该被 DELEGATECALL
结果:   行为统一为失败

Case #3: 零 gas CALL 的返回值
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
输入: CALL with gas=0 to EOA
Geth:   成功 (返回 1)
evmone: 成功 (返回 1)
revm:   早期版本返回 0
规范:   应该成功
```

---

## 附录 L: Gas 优化技巧

### L.1 基于 EVM 内部实现的优化建议

```solidity
// 1. 存储槽打包优化
// 差: 每个变量占用独立槽位
contract Bad {
    uint256 a;  // slot 0
    uint256 b;  // slot 1
    uint256 c;  // slot 2
    // 3 个冷 SLOAD = 3 * 2100 = 6300 gas
}

// 好: 打包到单个槽位
contract Good {
    uint128 a;  // slot 0, 低 128 位
    uint128 b;  // slot 0, 高 128 位
    // 1 个冷 SLOAD = 2100 gas
    // 然后位操作提取 ~20 gas
}

// 2. 访问模式优化
contract AccessOptimization {
    mapping(address => uint256) public balances;

    // 差: 多次访问同一槽位
    function bad(address user) external view returns (uint256) {
        require(balances[user] > 0);      // SLOAD #1
        require(balances[user] < 1000);   // SLOAD #2 (虽然是 warm，仍有开销)
        return balances[user];            // SLOAD #3
    }

    // 好: 缓存到内存
    function good(address user) external view returns (uint256) {
        uint256 balance = balances[user]; // SLOAD #1 only
        require(balance > 0);             // MLOAD ~3 gas
        require(balance < 1000);          // MLOAD ~3 gas
        return balance;
    }
}

// 3. 内存 vs 存储
contract MemoryVsStorage {
    uint256[] public data;

    // 差: 循环中多次 SLOAD
    function sumBad() external view returns (uint256 total) {
        for (uint i = 0; i < data.length; i++) {
            total += data[i];  // 每次迭代 SLOAD
        }
    }

    // 好: 复制到内存后处理
    function sumGood() external view returns (uint256 total) {
        uint256[] memory localData = data;  // 一次性加载
        for (uint i = 0; i < localData.length; i++) {
            total += localData[i];  // MLOAD
        }
    }
}
```

### L.2 基于 EVM 操作码成本的优化

```
操作码 Gas 成本速查 (Cancun):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
操作               │ Gas   │ 优化建议
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SLOAD (cold)       │ 2100  │ 缓存到内存变量
SLOAD (warm)       │  100  │ 仍比 MLOAD 贵 33x
SSTORE (cold, 0→n) │ 22100 │ 批量写入，避免碎片化
SSTORE (warm)      │  100  │ 同一交易内重复写入便宜
MLOAD              │   3   │ 首选用于临时数据
MSTORE             │   3   │ 首选用于临时数据
CALL               │  100+ │ 使用 staticcall 如果不修改状态
CREATE             │ 32000 │ 使用 CREATE2 + 代理模式
KECCAK256          │ 30+6n │ 避免大数据 hash，考虑预计算
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

```solidity
// 4. 短路优化
contract ShortCircuit {
    mapping(address => bool) public whitelist;
    mapping(address => uint256) public balances;

    // 差: 总是执行两个 SLOAD
    function checkBad(address user) external view returns (bool) {
        return whitelist[user] && balances[user] > 100;
    }

    // 好: 把便宜的检查放前面
    function checkGood(address user, uint256 minBalance) external view returns (bool) {
        // 先检查 calldata (便宜) 再检查 storage (贵)
        if (minBalance == 0) return true;
        return balances[user] >= minBalance;
    }
}

// 5. 使用 Transient Storage (EIP-1153, Cancun)
contract TransientOptimization {
    // 传统: 使用 storage 做重入锁
    uint256 private _locked;  // 占用 storage 槽位

    modifier oldLock() {
        require(_locked == 0);
        _locked = 1;         // SSTORE: 22100 gas (cold, 0→1)
        _;
        _locked = 0;         // SSTORE: 2900 gas (warm, refund)
    }

    // Cancun: 使用 transient storage
    modifier newLock() {
        assembly {
            if tload(0) { revert(0, 0) }
            tstore(0, 1)     // TSTORE: 100 gas
        }
        _;
        assembly {
            tstore(0, 0)     // TSTORE: 100 gas
        }
    }
    // 节省: ~22000 gas per call
}

// 6. 使用 MCOPY (EIP-5656, Cancun)
contract McopyOptimization {
    // 传统: 循环复制
    function copyOld(bytes memory src) internal pure returns (bytes memory) {
        bytes memory dst = new bytes(src.length);
        for (uint i = 0; i < src.length; i++) {
            dst[i] = src[i];  // MLOAD + MSTORE per byte
        }
        return dst;
    }

    // Cancun: MCOPY
    function copyNew(bytes memory src) internal pure returns (bytes memory dst) {
        dst = new bytes(src.length);
        assembly {
            mcopy(add(dst, 32), add(src, 32), mload(src))
        }
        // Gas: 3 + 3 * (size / 32) vs size * ~6
    }
}
```

### L.3 编译器优化提示

```solidity
// 7. 利用编译器优化
contract CompilerHints {
    // 使用 immutable 代替普通变量 (部署时存入字节码)
    address public immutable owner;  // 读取是 PUSH，不是 SLOAD

    // 使用 constant 进行编译期计算
    uint256 public constant MAX_SUPPLY = 1000000 * 10**18;  // 编译期计算
    bytes32 public constant TYPEHASH = keccak256("Permit(...)");  // 编译期 hash

    constructor() {
        owner = msg.sender;
    }

    // 使用 unchecked 跳过溢出检查 (确保安全时)
    function sumUnchecked(uint256[] calldata arr) external pure returns (uint256 total) {
        unchecked {
            for (uint i = 0; i < arr.length; ++i) {  // ++i 比 i++ 略省 gas
                total += arr[i];
            }
        }
    }
}
```

---

## 附录 M: 跨链 EVM 变体

### M.1 zkEVM

```
zkEVM 类型对比:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
项目              │ 类型      │ 兼容性    │ 证明系统
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
zkSync Era        │ Type 4    │ 语言级    │ PLONK + FRI
Polygon zkEVM     │ Type 2/3  │ 字节码级  │ PIL + STARK
Scroll            │ Type 2    │ 字节码级  │ KZG + SNARK
Linea             │ Type 2    │ 字节码级  │ Lattice-based
Taiko             │ Type 1    │ 完全等价  │ 多证明系统
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Type 分类:
- Type 1: 完全以太坊等价 (可验证以太坊区块)
- Type 2: EVM 等价 (相同字节码，微小差异)
- Type 3: EVM 兼容 (大部分兼容，有修改)
- Type 4: 语言兼容 (Solidity 可编译，不同字节码)
```

```
zkEVM vs 标准 EVM 差异:
┌─────────────────────────────────────────────────────────────┐
│ 操作码差异:                                                  │
│                                                              │
│ SELFDESTRUCT: 多数 zkEVM 不支持或行为不同                   │
│ DIFFICULTY:   返回固定值或 0                                │
│ BLOCKHASH:    受限的历史深度                                │
│ 预编译:       部分 zkEVM 缺少某些预编译合约                 │
│                                                              │
│ Gas 计算:                                                    │
│ - zkEVM 可能有不同的 gas 定价                               │
│ - 某些操作在 ZK 证明中更昂贵                                │
│ - KECCAK256 在 zkEVM 中通常很贵                             │
└─────────────────────────────────────────────────────────────┘
```

### M.2 Arbitrum Stylus

```
Stylus 架构:
┌─────────────────────────────────────────────────────────────┐
│                     Arbitrum Stylus                          │
│                                                              │
│  ┌─────────────────┐     ┌─────────────────┐                │
│  │   EVM 合约      │     │   WASM 合约     │                │
│  │   (Solidity)    │     │ (Rust/C/C++)    │                │
│  └────────┬────────┘     └────────┬────────┘                │
│           │                       │                          │
│           ▼                       ▼                          │
│  ┌─────────────────────────────────────────┐                │
│  │           Stylus 运行时                  │                │
│  │  ┌─────────────┐  ┌─────────────────┐   │                │
│  │  │ geth EVM    │  │ wasmer (WASM)   │   │                │
│  │  │ 解释器      │  │ 解释器          │   │                │
│  │  └─────────────┘  └─────────────────┘   │                │
│  │         │                   │            │                │
│  │         └───────┬───────────┘            │                │
│  │                 ▼                        │                │
│  │  ┌───────────────────────────────────┐  │                │
│  │  │      共享状态访问接口              │  │                │
│  │  │   (EVM storage, balance, etc)     │  │                │
│  │  └───────────────────────────────────┘  │                │
│  └─────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

```rust
// Stylus Rust 合约示例
use stylus_sdk::{
    alloy_primitives::{Address, U256},
    prelude::*,
    storage::StorageU256,
};

#[storage]
#[entrypoint]
pub struct Counter {
    count: StorageU256,
}

#[public]
impl Counter {
    pub fn get(&self) -> U256 {
        self.count.get()
    }

    pub fn increment(&mut self) {
        let current = self.count.get();
        self.count.set(current + U256::from(1));
    }
}

// Gas 成本对比 (近似):
// EVM Solidity increment: ~26000 gas
// Stylus Rust increment:  ~5000 gas (5x 更便宜)
```

### M.3 Optimism (OP Stack)

```
OP Stack EVM 修改:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
组件                  │ 修改内容
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
交易类型              │ 新增 Deposit 交易 (Type 0x7E)
L1 数据费用           │ 额外收取 L1 calldata 费用
系统合约              │ L1Block, GasPriceOracle, L2ToL1MessagePasser
预编译                │ 无额外预编译 (不同于早期 OVM)
区块时间              │ 2 秒 (vs 以太坊 12 秒)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### M.4 其他 EVM 变体

```
其他 EVM 兼容链:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
链                │ EVM 实现      │ 主要差异
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BSC               │ geth fork     │ 更短区块时间，PoSA 共识
Polygon PoS       │ geth fork     │ Bor 共识，更短区块时间
Avalanche C-Chain │ coreth (fork) │ Snowman 共识，子网架构
Fantom            │ go-opera      │ Lachesis 共识，DAG 结构
Gnosis Chain      │ nethermind    │ POSDAO 共识
Celo              │ geth fork     │ 原生稳定币，手机优先
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

EVM 实现兼容性注意事项:
1. CHAINID: 每个链不同，需在合约中正确处理
2. BASEFEE: 非 EIP-1559 链可能返回 0 或不支持
3. PREVRANDAO: 非 PoS 链返回 difficulty
4. 预编译地址: 某些链有额外预编译或修改
5. Gas 限制: 每个链的区块 gas 限制不同
6. 硬分叉时间: EIP 激活时间与以太坊不同步
```

---

## 附录 N: 账户抽象 (Account Abstraction)

### N.1 ERC-4337 概述

```
ERC-4337 账户抽象架构:
┌─────────────────────────────────────────────────────────────┐
│                      用户操作流程                            │
│                                                              │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │  用户    │───▶│  Bundler     │───▶│  EntryPoint     │   │
│  │ (Wallet) │    │  (聚合器)    │    │  (入口合约)     │   │
│  └──────────┘    └──────────────┘    └────────┬────────┘   │
│       │                                        │             │
│       │ UserOperation                          ▼             │
│       │                              ┌─────────────────┐    │
│       │                              │  Smart Account  │    │
│       │                              │  (智能钱包)     │    │
│       └─────────────────────────────▶│                 │    │
│                                      └─────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### N.2 EVM 层面的 AA 支持

```go
// go-ethereum 中 EntryPoint 调用的处理
// 文件: core/vm/evm.go

func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
    // ERC-4337 EntryPoint 地址
    // 0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789

    // AA 调用特点:
    // 1. 从 EntryPoint 发起的调用
    // 2. 可能包含多个 UserOperation
    // 3. 需要特殊的 gas 计算

    if evm.Config.Tracer != nil {
        // 追踪 AA 调用以便调试
        evm.Config.Tracer.CaptureEnter(CALL, caller.Address(), addr, input, gas, value)
    }

    // ... 正常调用逻辑
}

// UserOperation 结构 (链下)
type UserOperation struct {
    Sender               common.Address
    Nonce                *big.Int
    InitCode             []byte      // 首次部署钱包时使用
    CallData             []byte      // 实际调用数据
    CallGasLimit         *big.Int
    VerificationGasLimit *big.Int
    PreVerificationGas   *big.Int
    MaxFeePerGas         *big.Int
    MaxPriorityFeePerGas *big.Int
    PaymasterAndData     []byte      // Paymaster 信息
    Signature            []byte
}
```

### N.3 EIP-7702: EOA 代码设置

```
EIP-7702 (Prague 升级):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
功能: 允许 EOA 临时设置代码，实现类似智能钱包的功能
交易类型: 0x04 (新交易类型)

交易结构:
┌─────────────────────────────────────────────────────────────┐
│ SetCodeTransaction                                          │
│ - chain_id                                                  │
│ - nonce                                                     │
│ - max_priority_fee_per_gas                                 │
│ - max_fee_per_gas                                          │
│ - gas_limit                                                │
│ - destination                                              │
│ - value                                                    │
│ - data                                                     │
│ - access_list                                              │
│ - authorization_list: [(chain_id, address, nonce, sig)]   │ ← 新字段
└─────────────────────────────────────────────────────────────┘
```

```cpp
// evmone 中 EIP-7702 的概念实现
struct Authorization {
    uint256 chain_id;
    evmc::address address;    // 要设置的代码地址
    uint64 nonce;
    uint8_t v;
    evmc::bytes32 r;
    evmc::bytes32 s;
};

class EIP7702Handler {
public:
    // 处理授权列表
    void process_authorizations(
        ExecutionState& state,
        const std::vector<Authorization>& auth_list
    ) {
        for (const auto& auth : auth_list) {
            // 1. 验证签名
            auto signer = ecrecover(auth);
            if (!signer) continue;

            // 2. 检查 nonce
            auto account_nonce = state.host.get_nonce(*signer);
            if (auth.nonce != account_nonce) continue;

            // 3. 设置代码委托
            // 特殊前缀 0xef0100 表示代码委托
            bytes delegation_code = {0xef, 0x01, 0x00};
            delegation_code.insert(
                delegation_code.end(),
                auth.address.bytes,
                auth.address.bytes + 20
            );

            state.host.set_code(*signer, delegation_code);

            // 4. 增加 nonce
            state.host.increment_nonce(*signer);
        }
    }

    // 执行时检查代码委托
    evmc::bytes32 get_effective_code(
        const evmc::address& addr,
        const ExecutionState& state
    ) {
        auto code = state.host.get_code(addr);

        // 检查是否是委托代码 (0xef0100 前缀)
        if (code.size() == 23 &&
            code[0] == 0xef && code[1] == 0x01 && code[2] == 0x00) {
            // 提取委托地址
            evmc::address delegate;
            std::memcpy(delegate.bytes, &code[3], 20);

            // 返回委托地址的代码
            return state.host.get_code(delegate);
        }

        return code;
    }
};
```

### N.4 Gas 计算差异

```
AA 相关 Gas 成本:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
操作                        │ 成本           │ 说明
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
UserOp 验证                 │ ~50,000+       │ 签名验证 + 状态检查
Paymaster 验证              │ ~30,000+       │ postOp 回调
钱包部署 (initCode)         │ 32,000+        │ CREATE2 成本
EIP-7702 授权处理           │ ~12,500/授权   │ Prague 新增
委托代码解析                │ ~100           │ 额外 EXTCODECOPY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 附录 O: 形式化验证

### O.1 KEVM - K 框架 EVM 语义

```
KEVM 架构:
┌─────────────────────────────────────────────────────────────┐
│                    K Framework                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  K 语言定义                              ││
│  │  syntax Instruction ::= "ADD" | "MUL" | "SUB" | ...     ││
│  │  rule <k> ADD => . ... </k>                             ││
│  │       <stack> I1 : I2 : S => I1 +Int I2 : S </stack>    ││
│  └─────────────────────────────────────────────────────────┘│
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              KEVM (EVM in K)                             ││
│  │  - 完整的 EVM 操作语义                                   ││
│  │  - 形式化的 gas 计算                                    ││
│  │  - 可验证的状态转换                                     ││
│  └─────────────────────────────────────────────────────────┘│
│                           │                                  │
│           ┌───────────────┼───────────────┐                 │
│           ▼               ▼               ▼                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   执行      │  │   验证      │  │   测试生成  │         │
│  │ (Concrete)  │  │ (Symbolic)  │  │  (Auto)     │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### O.2 K 语言 EVM 规范示例

```k
// KEVM 中 ADD 操作码的形式化定义
rule <k> ADD => . ... </k>
     <wordStack> W0 : W1 : WS => chop(W0 +Int W1) : WS </wordStack>
     <gas> G => G -Int Gverylow </gas>
  requires G >=Int Gverylow

// SLOAD 的形式化定义
rule <k> SLOAD => . ... </k>
     <wordStack> INDEX : WS => VALUE : WS </wordStack>
     <account>
       <acctID> ACCT </acctID>
       <storage> ... INDEX |-> VALUE ... </storage>
       ...
     </account>
     <gas> G => G -Int Gsload(ACCT, INDEX, SCHED) </gas>
  requires G >=Int Gsload(ACCT, INDEX, SCHED)

// CREATE2 地址计算的形式化验证
rule <k> CREATE2 => . ... </k>
     <wordStack> VALUE : OFFSET : SIZE : SALT : WS
              => #newAddr(ACCT, SALT, #range(LM, OFFSET, SIZE)) : WS
     </wordStack>
     <localMem> LM </localMem>
     <acctID> ACCT </acctID>
     ...

// 辅助函数
syntax Int ::= #newAddr(Int, Int, Bytes) [function]
rule #newAddr(ACCT, SALT, INITCODE)
  => #addr(keccak(#buf(1, 255) +Bytes #buf(20, ACCT)
                  +Bytes #buf(32, SALT)
                  +Bytes keccak(INITCODE)))
```

### O.3 与 evmone/geth 的验证对比

```cpp
// 使用 KEVM 验证 evmone 实现正确性

// 1. 生成测试用例
class KEVMTestGenerator {
public:
    // 从 K 规范自动生成边界测试
    std::vector<TestCase> generate_boundary_tests() {
        return {
            // 整数边界
            {.opcode = ADD, .inputs = {MAX_UINT256, 1}},
            {.opcode = SUB, .inputs = {0, 1}},
            {.opcode = MUL, .inputs = {MAX_UINT256, 2}},

            // Gas 边界
            {.opcode = SLOAD, .gas = 99},  // < cold cost
            {.opcode = CALL, .gas = 0},

            // 内存边界
            {.opcode = MLOAD, .offset = MAX_UINT64},
            {.opcode = MSTORE, .offset = MEMORY_LIMIT - 31},

            // 栈边界
            {.opcode = POP, .stack_size = 0},
            {.opcode = DUP16, .stack_size = 15},
        };
    }
};

// 2. 符号执行验证
class SymbolicVerifier {
    KEVMSpec kevm_spec;
    EvmoneExecutor evmone;

public:
    VerificationResult verify_opcode(Opcode op) {
        // 创建符号输入
        auto sym_stack = create_symbolic_stack(op.stack_inputs);
        auto sym_memory = create_symbolic_memory();
        auto sym_storage = create_symbolic_storage();

        // KEVM 符号执行
        auto kevm_result = kevm_spec.symbolic_execute(
            op, sym_stack, sym_memory, sym_storage
        );

        // evmone 执行
        auto evmone_result = evmone.execute(
            op, concretize(sym_stack), concretize(sym_memory)
        );

        // 验证等价性
        return verify_equivalence(kevm_result, evmone_result);
    }
};
```

### O.4 Act 语言规范

```act
// Act 语言 - 用于智能合约规范

// ERC20 transfer 规范
behaviour transfer of ERC20
interface transfer(address to, uint256 value)

// 前置条件
iff
    CALLER =/= to
    value <= balanceOf[CALLER]
    balanceOf[to] + value <= max_uint256

// 存储变化
storage
    balanceOf[CALLER] => balanceOf[CALLER] - value
    balanceOf[to] => balanceOf[to] + value

// 返回值
returns true

// Uniswap V2 swap 规范
behaviour swap of UniswapV2Pair
interface swap(uint amount0Out, uint amount1Out, address to, bytes data)

iff
    amount0Out > 0 or amount1Out > 0
    amount0Out < reserve0
    amount1Out < reserve1
    to =/= token0
    to =/= token1

storage
    reserve0 => reserve0 - amount0Out + amount0In
    reserve1 => reserve1 - amount1Out + amount1In

invariant
    // 恒定乘积公式 (考虑手续费)
    (reserve0 * 1000 - amount0In * 3) * (reserve1 * 1000 - amount1In * 3)
    >= reserve0_old * reserve1_old * 1000000
```

---

## 附录 P: 状态同步优化

### P.1 同步模式对比

```
以太坊客户端同步模式:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
模式          │ 数据量    │ 时间     │ 说明
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Full Sync     │ ~1.5 TB   │ 数周     │ 从创世块执行所有交易
Fast Sync     │ ~500 GB   │ 数天     │ 下载状态 + 最近区块
Snap Sync     │ ~500 GB   │ 数小时   │ 快照 + healing
Light Sync    │ ~1 GB     │ 分钟     │ 仅区块头
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### P.2 Snap Sync 实现

```go
// go-ethereum snap sync 核心流程
// 文件: eth/downloader/downloader.go

type SnapSync struct {
    // 状态快照
    snapshot *snapshot.Tree

    // 账户范围请求
    accountTasks map[common.Hash]*accountTask

    // 存储范围请求
    storageTasks map[common.Hash]*storageTask

    // Healing 队列
    healingQueue *prque.Prque
}

// 账户同步任务
type accountTask struct {
    start    common.Hash  // 起始账户 hash
    end      common.Hash  // 结束账户 hash
    accounts []*types.StateAccount
    proof    [][]byte     // Merkle 证明
}

func (s *SnapSync) processAccountRange(task *accountTask) error {
    // 1. 验证 Merkle 证明
    if err := verifyAccountProof(task); err != nil {
        return err
    }

    // 2. 存储账户数据
    for _, account := range task.accounts {
        s.snapshot.Update(account.Root, account)
    }

    // 3. 调度存储同步
    for _, account := range task.accounts {
        if account.Root != emptyRoot {
            s.scheduleStorageSync(account.Root)
        }
    }

    return nil
}

// Healing 过程 - 修复不完整的 trie
func (s *SnapSync) heal() error {
    for !s.healingQueue.Empty() {
        // 获取缺失的节点
        node := s.healingQueue.Pop()

        // 请求节点数据
        data, err := s.requestNode(node.Hash)
        if err != nil {
            return err
        }

        // 写入数据库
        s.db.Put(node.Hash.Bytes(), data)

        // 检查子节点
        for _, child := range parseChildren(data) {
            if !s.db.Has(child.Bytes()) {
                s.healingQueue.Push(child, priority)
            }
        }
    }

    return nil
}
```

### P.3 EVM 执行与状态同步的关系

```
状态同步期间的 EVM 执行:
┌─────────────────────────────────────────────────────────────┐
│                     Snap Sync 阶段                          │
│                                                              │
│  Phase 1: 下载快照                                          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 此阶段不执行 EVM                                        ││
│  │ 直接下载账户和存储数据                                  ││
│  └─────────────────────────────────────────────────────────┘│
│                           │                                  │
│                           ▼                                  │
│  Phase 2: Healing                                            │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 此阶段不执行 EVM                                        ││
│  │ 修复缺失的 trie 节点                                    ││
│  └─────────────────────────────────────────────────────────┘│
│                           │                                  │
│                           ▼                                  │
│  Phase 3: 追赶最新区块                                      │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 开始执行 EVM !!!                                        ││
│  │ - 执行快照点之后的所有区块                              ││
│  │ - 使用 evmone/geth EVM                                  ││
│  │ - 生成状态变更                                          ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘

同步期间 EVM 执行优化:
1. 使用 evmone 加速追赶阶段 (~3-6x faster)
2. 并行执行独立交易 (PEVM/Grevm)
3. 预取状态减少 I/O 等待
```

### P.4 evmone 在同步中的应用

```cpp
// 使用 evmone 加速区块同步
class FastBlockProcessor {
    evmc::VM* vm;
    StateDB* db;

public:
    // 批量执行区块
    void process_blocks(const std::vector<Block>& blocks) {
        for (const auto& block : blocks) {
            // 预取所有交易涉及的状态
            prefetch_state(block);

            // 使用 evmone 执行
            for (const auto& tx : block.transactions) {
                auto result = execute_transaction(tx);
                apply_state_changes(result);
            }

            // 验证状态根
            auto computed_root = db->compute_root();
            if (computed_root != block.state_root) {
                throw std::runtime_error("State root mismatch");
            }
        }
    }

    // 状态预取
    void prefetch_state(const Block& block) {
        std::vector<evmc::address> addresses;
        std::vector<std::pair<evmc::address, evmc::bytes32>> slots;

        for (const auto& tx : block.transactions) {
            addresses.push_back(tx.from);
            if (tx.to) {
                addresses.push_back(*tx.to);
            }
            // 解析访问列表
            for (const auto& [addr, keys] : tx.access_list) {
                addresses.push_back(addr);
                for (const auto& key : keys) {
                    slots.emplace_back(addr, key);
                }
            }
        }

        // 并行预取
        db->prefetch_accounts(addresses);
        db->prefetch_storage(slots);
    }
};
```

---

## 附录 Q: 性能剖析方法

### Q.1 Go pprof (go-ethereum)

```go
// go-ethereum 性能剖析
// 启用 pprof 端点
import _ "net/http/pprof"

func main() {
    // 启动 pprof HTTP 服务器
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // 正常启动 geth
    // ...
}

// 使用方法:
// 1. CPU 剖析
//    go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
//
// 2. 内存剖析
//    go tool pprof http://localhost:6060/debug/pprof/heap
//
// 3. Goroutine 剖析
//    go tool pprof http://localhost:6060/debug/pprof/goroutine
//
// 4. 阻塞剖析
//    go tool pprof http://localhost:6060/debug/pprof/block

// EVM 特定指标
type EVMMetrics struct {
    // 操作码执行计数
    OpcodeCount map[byte]uint64

    // Gas 消耗分布
    GasUsed map[byte]uint64

    // 执行时间
    ExecutionTime time.Duration

    // 状态访问
    StorageReads  uint64
    StorageWrites uint64
}

func (evm *EVM) collectMetrics(op OpCode, gas uint64, elapsed time.Duration) {
    if evm.Config.EnableMetrics {
        evm.metrics.OpcodeCount[byte(op)]++
        evm.metrics.GasUsed[byte(op)] += gas
        evm.metrics.ExecutionTime += elapsed
    }
}
```

### Q.2 C++ 剖析 (evmone)

```cpp
// evmone 性能剖析

// 1. 使用 perf
// $ perf record -g ./evmone-bench
// $ perf report

// 2. 使用 Valgrind/Callgrind
// $ valgrind --tool=callgrind ./evmone-bench
// $ kcachegrind callgrind.out.*

// 3. 内置计时器
class ExecutionProfiler {
    struct OpcodeStats {
        uint64_t count = 0;
        uint64_t total_cycles = 0;
        uint64_t min_cycles = UINT64_MAX;
        uint64_t max_cycles = 0;
    };

    std::array<OpcodeStats, 256> opcode_stats;

    // 使用 RDTSC 进行精确计时
    static inline uint64_t rdtsc() {
        uint32_t lo, hi;
        __asm__ volatile ("rdtsc" : "=a" (lo), "=d" (hi));
        return ((uint64_t)hi << 32) | lo;
    }

public:
    void record_opcode(uint8_t opcode, uint64_t cycles) {
        auto& stats = opcode_stats[opcode];
        stats.count++;
        stats.total_cycles += cycles;
        stats.min_cycles = std::min(stats.min_cycles, cycles);
        stats.max_cycles = std::max(stats.max_cycles, cycles);
    }

    void report() {
        std::cout << "Opcode Performance Report\n";
        std::cout << "========================\n";

        for (int i = 0; i < 256; i++) {
            const auto& stats = opcode_stats[i];
            if (stats.count > 0) {
                std::cout << std::hex << std::setw(2) << i
                          << ": count=" << std::dec << stats.count
                          << " avg=" << stats.total_cycles / stats.count
                          << " min=" << stats.min_cycles
                          << " max=" << stats.max_cycles
                          << "\n";
            }
        }
    }
};

// 4. 内存分析
class MemoryProfiler {
    size_t peak_memory = 0;
    size_t current_memory = 0;

public:
    void on_memory_resize(size_t old_size, size_t new_size) {
        current_memory = current_memory - old_size + new_size;
        peak_memory = std::max(peak_memory, current_memory);
    }

    void on_execution_end() {
        std::cout << "Peak memory usage: " << peak_memory << " bytes\n";
    }
};
```

### Q.3 火焰图生成

```bash
# go-ethereum 火焰图
# 1. 安装 go-torch 或使用 pprof 内置功能
go install github.com/uber/go-torch@latest

# 2. 收集剖析数据
curl -o cpu.pprof "http://localhost:6060/debug/pprof/profile?seconds=30"

# 3. 生成火焰图
go tool pprof -http=:8080 cpu.pprof
# 或
go-torch --file=flame.svg cpu.pprof

# evmone 火焰图
# 1. 使用 perf 收集
perf record -F 99 -g ./evmone-bench

# 2. 生成火焰图
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### Q.4 EVM 特定性能指标

```
EVM 性能关键指标:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
指标                    │ 测量方法           │ 优化目标
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
操作码吞吐量            │ ops/second         │ 最大化
状态读取延迟            │ μs/SLOAD           │ 最小化
状态写入延迟            │ μs/SSTORE          │ 最小化
内存分配频率            │ allocations/tx     │ 最小化
分支预测失败率          │ branch misses/%    │ < 5%
缓存命中率              │ cache hit/%        │ > 95%
gas/实际时间比          │ gas/μs             │ 一致性
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

典型瓶颈:
1. SLOAD/SSTORE: 数据库 I/O
2. KECCAK256: 密码学计算
3. CALL/CREATE: 状态管理开销
4. 大数运算: 256 位算术
```

---

## 附录 R: 历史状态查询

### R.1 Archive Node 架构

```
Archive Node vs Full Node:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
特性              │ Full Node    │ Archive Node
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
存储空间          │ ~1 TB        │ ~15+ TB
历史状态          │ 最近 128 块  │ 所有区块
eth_call 历史     │ 受限         │ 任意区块
debug_* API       │ 受限         │ 完整支持
同步时间          │ 数小时-天    │ 数周
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### R.2 历史状态存储

```go
// go-ethereum 历史状态存储
// 文件: core/state/database.go

// Archive 模式下的状态数据库
type ArchiveDatabase struct {
    // 底层 KV 存储
    diskdb ethdb.Database

    // Trie 缓存
    triedb *trie.Database

    // 不修剪旧状态
    pruning bool // = false for archive
}

// 获取历史状态
func (db *ArchiveDatabase) OpenTrie(root common.Hash) (Trie, error) {
    // Archive 模式: 任何历史状态根都可访问
    return trie.New(trie.TrieID(root), db.triedb)
}

// 历史状态查询
func (api *PublicBlockChainAPI) GetBalance(
    ctx context.Context,
    address common.Address,
    blockNrOrHash rpc.BlockNumberOrHash,
) (*hexutil.Big, error) {
    // 获取指定区块的状态
    state, _, err := api.b.StateAndHeaderByNumberOrHash(ctx, blockNrOrHash)
    if err != nil {
        return nil, err
    }

    // Archive node: 可查询任意历史区块
    // Full node: 只能查询最近 128 个区块
    return (*hexutil.Big)(state.GetBalance(address)), nil
}
```

### R.3 EVM 历史执行重放

```cpp
// evmone 历史交易重放
class HistoricalExecutor {
    ArchiveHost& archive;

public:
    // 在历史状态上重放交易
    evmc::Result replay_transaction(
        uint64_t block_number,
        const Transaction& tx
    ) {
        // 1. 获取历史区块头
        auto header = archive.get_block_header(block_number);

        // 2. 获取历史状态
        auto state_root = header.state_root;
        archive.set_state_root(state_root);

        // 3. 设置区块环境
        evmc_tx_context ctx{
            .block_number = header.number,
            .block_timestamp = header.timestamp,
            .block_gas_limit = header.gas_limit,
            .block_coinbase = header.coinbase,
            .block_difficulty = header.difficulty,
            .block_base_fee = header.base_fee,
        };

        // 4. 执行交易
        evmc_message msg{
            .kind = tx.to ? EVMC_CALL : EVMC_CREATE,
            .sender = tx.from,
            .recipient = tx.to.value_or(evmc::address{}),
            .value = tx.value,
            .input_data = tx.data.data(),
            .input_size = tx.data.size(),
            .gas = tx.gas_limit,
        };

        return vm->execute(archive, EVMC_CANCUN, msg, tx.data.data(), tx.data.size());
    }

    // 追踪历史交易
    TraceResult trace_transaction(
        uint64_t block_number,
        common::Hash tx_hash
    ) {
        // 获取区块和交易
        auto block = archive.get_block(block_number);
        auto tx_index = find_tx_index(block, tx_hash);

        // 重放之前的所有交易以获得正确状态
        for (size_t i = 0; i < tx_index; i++) {
            replay_transaction(block_number, block.transactions[i]);
        }

        // 使用追踪器执行目标交易
        TracingHost tracer(archive);
        auto result = vm->execute(
            tracer, EVMC_CANCUN,
            create_message(block.transactions[tx_index]),
            /* code */
        );

        return tracer.get_trace();
    }
};
```

### R.4 debug_traceTransaction 实现

```go
// go-ethereum debug API
// 文件: eth/tracers/api.go

func (api *API) TraceTransaction(ctx context.Context, hash common.Hash, config *TraceConfig) (interface{}, error) {
    // 1. 查找交易
    tx, blockHash, blockNumber, index, err := api.backend.GetTransaction(ctx, hash)
    if err != nil {
        return nil, err
    }

    // 2. 获取父区块状态 (需要 archive node)
    block, err := api.backend.BlockByHash(ctx, blockHash)
    if err != nil {
        return nil, err
    }

    parent, err := api.backend.BlockByNumber(ctx, rpc.BlockNumber(blockNumber-1))
    if err != nil {
        return nil, err
    }

    // 3. 创建状态数据库
    statedb, err := api.backend.StateAtBlock(ctx, parent, 0, nil, true, false)
    if err != nil {
        return nil, err
    }

    // 4. 重放之前的交易
    for i := uint64(0); i < index; i++ {
        tx := block.Transactions()[i]
        msg, _ := tx.AsMessage(signer, block.BaseFee())
        vmenv := vm.NewEVM(blockCtx, txCtx, statedb, chainConfig, vmConfig)
        statedb.Prepare(tx.Hash(), int(i))
        _, err := core.ApplyMessage(vmenv, msg, new(core.GasPool).AddGas(msg.Gas()))
        if err != nil {
            return nil, err
        }
        statedb.Finalise(true)
    }

    // 5. 使用追踪器执行目标交易
    tracer, err := tracers.New(config.Tracer, ctx)
    if err != nil {
        return nil, err
    }

    vmConfig := vm.Config{Tracer: tracer}
    vmenv := vm.NewEVM(blockCtx, txCtx, statedb, chainConfig, vmConfig)

    result, err := core.ApplyMessage(vmenv, msg, gasPool)
    if err != nil {
        return nil, err
    }

    return tracer.GetResult()
}
```

### R.5 状态存储优化

```
历史状态存储策略:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
策略              │ 空间      │ 查询速度  │ 适用场景
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
完整 Archive     │ 最大      │ 最快      │ 区块浏览器、分析
每 N 块快照      │ 中等      │ 中等      │ 一般历史查询
按需重放         │ 最小      │ 最慢      │ 偶尔历史查询
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

优化技术:
1. 状态差异存储 (只存储变更)
2. 压缩历史 trie 节点
3. 冷热数据分离
4. 索引加速查询
```

---

## 附录 S：EVM 调试工具

### S.1 go-ethereum 调试器实现

```go
// 文件: eth/tracers/js/tracer.go

// JavaScript 调试器 - 支持自定义追踪逻辑
type jsTracer struct {
    vm       *goja.Runtime
    env      *vm.EVM
    ctx      *tracerContext
    obj      *goja.Object

    // JS 回调函数
    result   goja.Callable
    fault    goja.Callable
    step     goja.Callable
    enter    goja.Callable
    exit     goja.Callable
}

// 单步执行回调
func (t *jsTracer) CaptureState(pc uint64, op vm.OpCode, gas, cost uint64,
    scope *vm.ScopeContext, rData []byte, depth int, err error) {

    // 构建步骤对象
    stepObj := t.vm.NewObject()
    stepObj.Set("pc", pc)
    stepObj.Set("op", op.String())
    stepObj.Set("gas", gas)
    stepObj.Set("gasCost", cost)
    stepObj.Set("depth", depth)

    // 栈信息
    stack := make([]string, len(scope.Stack.Data()))
    for i, val := range scope.Stack.Data() {
        stack[i] = val.Hex()
    }
    stepObj.Set("stack", stack)

    // 内存信息 (可选，消耗大)
    if t.ctx.enableMemory {
        stepObj.Set("memory", scope.Memory.Data())
    }

    // 调用 JS step 函数
    t.step(goja.Undefined(), stepObj, t.dbObj)
}
```

### S.2 evmone 调试追踪

```cpp
// evmone 追踪接口
class Tracer {
public:
    virtual ~Tracer() = default;

    // 指令执行前回调
    virtual void on_instruction_start(
        uint32_t pc,
        const intx::uint256* stack_top,
        int stack_height,
        int64_t gas,
        const ExecutionState& state) noexcept = 0;

    // 执行结束回调
    virtual void on_execution_end(const evmc_result& result) noexcept = 0;
};

// 调试追踪器实现
class DebugTracer : public Tracer {
    std::vector<TraceStep> steps_;
    bool capture_memory_;
    bool capture_stack_;

public:
    void on_instruction_start(
        uint32_t pc,
        const intx::uint256* stack_top,
        int stack_height,
        int64_t gas,
        const ExecutionState& state) noexcept override {

        TraceStep step;
        step.pc = pc;
        step.opcode = state.code[pc];
        step.gas = gas;
        step.depth = state.msg->depth;

        if (capture_stack_) {
            for (int i = 0; i < stack_height; ++i) {
                step.stack.push_back(stack_top[-i]);
            }
        }

        if (capture_memory_) {
            step.memory.assign(
                state.memory.begin(),
                state.memory.end()
            );
        }

        steps_.push_back(std::move(step));
    }

    const std::vector<TraceStep>& get_trace() const {
        return steps_;
    }
};

// 断点调试器
class BreakpointDebugger : public Tracer {
    std::set<uint32_t> breakpoints_;
    std::function<void(const ExecutionState&)> break_handler_;
    bool paused_ = false;

public:
    void add_breakpoint(uint32_t pc) {
        breakpoints_.insert(pc);
    }

    void remove_breakpoint(uint32_t pc) {
        breakpoints_.erase(pc);
    }

    void on_instruction_start(
        uint32_t pc,
        const intx::uint256* stack_top,
        int stack_height,
        int64_t gas,
        const ExecutionState& state) noexcept override {

        if (breakpoints_.count(pc) > 0) {
            paused_ = true;
            if (break_handler_) {
                break_handler_(state);
            }
            // 等待继续信号
            wait_for_continue();
        }
    }

    void continue_execution() {
        paused_ = false;
    }

    void step_over() {
        // 单步执行逻辑
    }
};
```

### S.3 调试器架构对比

```
┌─────────────────────────────────────────────────────────────────┐
│                     EVM 调试器架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   go-ethereum                      evmone                        │
│   ┌─────────────────────┐         ┌─────────────────────┐       │
│   │   Tracer Interface  │         │   Tracer Interface  │       │
│   │  ┌───────────────┐  │         │  ┌───────────────┐  │       │
│   │  │ CaptureStart  │  │         │  │ on_instruction│  │       │
│   │  │ CaptureState  │  │         │  │    _start     │  │       │
│   │  │ CaptureEnter  │  │         │  │ on_execution  │  │       │
│   │  │ CaptureExit   │  │         │  │    _end       │  │       │
│   │  │ CaptureEnd    │  │         │  └───────────────┘  │       │
│   │  └───────────────┘  │         └─────────────────────┘       │
│   └──────────┬──────────┘                    │                   │
│              │                               │                   │
│              ▼                               ▼                   │
│   ┌─────────────────────┐         ┌─────────────────────┐       │
│   │  Tracer 实现        │         │  Tracer 实现        │       │
│   │  ┌───────────────┐  │         │  ┌───────────────┐  │       │
│   │  │ structLogger  │  │         │  │ DebugTracer   │  │       │
│   │  │ callTracer    │  │         │  │ HistogramTr.  │  │       │
│   │  │ prestateTrace │  │         │  │ InstructionTr │  │       │
│   │  │ jsTracer      │  │         │  └───────────────┘  │       │
│   │  └───────────────┘  │         └─────────────────────┘       │
│   └─────────────────────┘                                        │
│                                                                  │
│   特点对比:                                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 特性          │ go-ethereum        │ evmone            │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │ JS 脚本支持   │ ✓ (goja)           │ ✗                 │   │
│   │ 内置追踪器    │ 多种               │ 基础              │   │
│   │ 性能开销      │ 较高               │ 较低              │   │
│   │ 断点支持      │ 有限               │ 可扩展            │   │
│   │ 远程调试      │ ✓ (RPC)            │ 需自定义          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 T：日志与事件系统

### T.1 LOG 操作码实现

```go
// go-ethereum LOG 操作码
// 文件: core/vm/instructions.go

func makeLog(size int) executionFunc {
    return func(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
        // 检查静态调用
        if interpreter.readOnly {
            return nil, ErrWriteProtection
        }

        stack := scope.Stack
        mStart, mSize := stack.pop(), stack.pop()

        // 获取 topics
        topics := make([]common.Hash, size)
        for i := 0; i < size; i++ {
            addr := stack.pop()
            topics[i] = common.Hash(addr.Bytes32())
        }

        // 读取数据
        data := scope.Memory.GetCopy(int64(mStart.Uint64()), int64(mSize.Uint64()))

        // 创建日志条目
        interpreter.evm.StateDB.AddLog(&types.Log{
            Address: scope.Contract.Address(),
            Topics:  topics,
            Data:    data,
            // BlockNumber 和 TxHash 在区块处理时填充
        })

        return nil, nil
    }
}

// Gas 计算
func gasLog(n int) gasFunc {
    return func(evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
        mSize := stack.Back(1)

        gas := uint64(n*params.LogTopicGas)           // 每个 topic 375 gas
        gas += params.LogGas                           // 基础 375 gas
        gas += params.LogDataGas * mSize.Uint64()     // 每字节 8 gas

        return gas, nil
    }
}
```

### T.2 evmone LOG 实现

```cpp
// evmone LOG 指令
template <size_t NumTopics>
evmc_status_code log(ExecutionState& state) noexcept {
    // 静态调用检查
    if (state.in_static_mode())
        return EVMC_STATIC_MODE_VIOLATION;

    const auto offset = state.stack.pop();
    const auto size = state.stack.pop();

    // 内存扩展 gas
    if (const auto cost = grow_memory(state, offset, size); cost < 0)
        return EVMC_OUT_OF_GAS;

    // LOG gas: 375 + 375*num_topics + 8*size
    const auto gas_cost = 375 + NumTopics * 375 + 8 * static_cast<int64_t>(size);
    if ((state.gas_left -= gas_cost) < 0)
        return EVMC_OUT_OF_GAS;

    // 收集 topics
    std::array<evmc::bytes32, NumTopics> topics;
    for (size_t i = 0; i < NumTopics; ++i) {
        topics[i] = intx::be::store<evmc::bytes32>(state.stack.pop());
    }

    // 获取数据
    const auto data = state.memory.data() + static_cast<size_t>(offset);

    // 发送到 host
    state.host.emit_log(
        state.msg->recipient,
        data,
        static_cast<size_t>(size),
        topics.data(),
        NumTopics
    );

    return EVMC_SUCCESS;
}
```

### T.3 Bloom Filter 实现

```go
// 文件: core/types/bloom9.go

const (
    BloomByteLength = 256
    BloomBitLength  = 8 * BloomByteLength // 2048 bits
)

type Bloom [BloomByteLength]byte

// 添加数据到 Bloom Filter
func (b *Bloom) Add(data []byte) {
    // Keccak256 哈希
    hash := crypto.Keccak256(data)

    // 取 3 个位置 (每个 11 bits = 0-2047)
    for i := 0; i < 6; i += 2 {
        // 取 16 bits，然后取低 11 bits
        bit := binary.BigEndian.Uint16(hash[i:i+2]) & 0x7FF

        // 设置对应位
        byteIndex := BloomByteLength - 1 - int(bit/8)
        bitIndex := bit % 8
        b[byteIndex] |= 1 << bitIndex
    }
}

// 从日志创建 Bloom Filter
func LogsBloom(logs []*Log) Bloom {
    var bloom Bloom
    for _, log := range logs {
        // 添加合约地址
        bloom.Add(log.Address.Bytes())

        // 添加所有 topics
        for _, topic := range log.Topics {
            bloom.Add(topic.Bytes())
        }
    }
    return bloom
}

// 检查是否可能包含
func (b Bloom) Test(data []byte) bool {
    hash := crypto.Keccak256(data)

    for i := 0; i < 6; i += 2 {
        bit := binary.BigEndian.Uint16(hash[i:i+2]) & 0x7FF
        byteIndex := BloomByteLength - 1 - int(bit/8)
        bitIndex := bit % 8

        if b[byteIndex]&(1<<bitIndex) == 0 {
            return false // 一定不包含
        }
    }
    return true // 可能包含
}
```

### T.4 事件索引与查询

```go
// 文件: core/rawdb/accessors_indexes.go

// 日志索引结构
type LogIndex struct {
    BlockNumber uint64
    TxIndex     uint32
    LogIndex    uint32
}

// 按 topic 索引日志
func WriteLogIndex(db ethdb.KeyValueWriter, topic common.Hash, index *LogIndex) {
    key := logIndexKey(topic, index.BlockNumber, index.TxIndex, index.LogIndex)
    db.Put(key, []byte{1})
}

// 日志过滤器
type FilterQuery struct {
    BlockHash *common.Hash     // 特定区块
    FromBlock *big.Int         // 起始区块
    ToBlock   *big.Int         // 结束区块
    Addresses []common.Address // 合约地址过滤
    Topics    [][]common.Hash  // Topics 过滤
}

// 过滤日志
func (f *Filter) Logs(ctx context.Context) ([]*types.Log, error) {
    var logs []*types.Log

    for blockNum := f.begin; blockNum <= f.end; blockNum++ {
        // 1. 先用 Bloom Filter 快速过滤
        header := f.backend.HeaderByNumber(ctx, rpc.BlockNumber(blockNum))
        if !f.bloomMatches(header.Bloom) {
            continue // 一定不匹配
        }

        // 2. 获取区块日志
        receipts := f.backend.GetReceipts(ctx, header.Hash())

        // 3. 精确匹配
        for _, receipt := range receipts {
            for _, log := range receipt.Logs {
                if f.matches(log) {
                    logs = append(logs, log)
                }
            }
        }
    }

    return logs, nil
}

// Bloom Filter 匹配
func (f *Filter) bloomMatches(bloom types.Bloom) bool {
    // 检查地址
    if len(f.addresses) > 0 {
        addressMatched := false
        for _, addr := range f.addresses {
            if bloom.Test(addr.Bytes()) {
                addressMatched = true
                break
            }
        }
        if !addressMatched {
            return false
        }
    }

    // 检查 topics
    for _, topicList := range f.topics {
        if len(topicList) == 0 {
            continue
        }

        topicMatched := false
        for _, topic := range topicList {
            if bloom.Test(topic.Bytes()) {
                topicMatched = true
                break
            }
        }
        if !topicMatched {
            return false
        }
    }

    return true
}
```

### T.5 日志系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    日志与事件系统架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   合约执行                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  emit Transfer(from, to, amount)                        │   │
│   │            │                                            │   │
│   │            ▼                                            │   │
│   │  ┌─────────────────┐                                   │   │
│   │  │ LOG3 操作码     │                                   │   │
│   │  │ topics[0]: keccak256("Transfer(address,address,uint256)")│
│   │  │ topics[1]: from                                      │   │
│   │  │ topics[2]: to                                        │   │
│   │  │ data: amount                                         │   │
│   │  └────────┬────────┘                                   │   │
│   └───────────┼─────────────────────────────────────────────┘   │
│               │                                                  │
│               ▼                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │               StateDB.AddLog()                          │   │
│   │  ┌─────────────────────────────────────────────────┐   │   │
│   │  │  Log {                                           │   │   │
│   │  │    Address: 0x...                                │   │   │
│   │  │    Topics: [topic0, topic1, topic2]              │   │   │
│   │  │    Data: 0x...                                   │   │   │
│   │  │    BlockNumber: pending                          │   │   │
│   │  │    TxHash: pending                               │   │   │
│   │  │    TxIndex: pending                              │   │   │
│   │  │    LogIndex: 0                                   │   │   │
│   │  │  }                                               │   │   │
│   │  └─────────────────────────────────────────────────┘   │   │
│   └────────────────────────┬────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              区块处理完成后                               │   │
│   │  ┌───────────────┐  ┌───────────────┐                   │   │
│   │  │ Receipt 生成  │  │ Bloom Filter  │                   │   │
│   │  │ - 填充元数据  │  │ - 聚合所有日志│                   │   │
│   │  │ - Logs 列表   │  │ - 存入区块头  │                   │   │
│   │  └───────────────┘  └───────────────┘                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   查询流程                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  eth_getLogs({address: 0x.., topics: [...]})            │   │
│   │            │                                            │   │
│   │            ▼                                            │   │
│   │  ┌─────────────────────────────────────────────────┐   │   │
│   │  │  1. 遍历区块范围                                 │   │   │
│   │  │  2. Bloom Filter 快速过滤 (O(1))                │   │   │
│   │  │  3. 获取候选区块的 Receipts                     │   │   │
│   │  │  4. 精确匹配日志条目                            │   │   │
│   │  │  5. 返回匹配结果                                │   │   │
│   │  └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Gas 成本 (LOG0-LOG4)                                           │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  基础: 375 gas                                          │   │
│   │  每个 topic: 375 gas                                    │   │
│   │  每字节数据: 8 gas                                      │   │
│   │                                                         │   │
│   │  LOG0: 375 + 8*size                                     │   │
│   │  LOG1: 750 + 8*size                                     │   │
│   │  LOG2: 1125 + 8*size                                    │   │
│   │  LOG3: 1500 + 8*size                                    │   │
│   │  LOG4: 1875 + 8*size                                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 U：内存池管理

### U.1 交易池架构

```go
// 文件: core/txpool/txpool.go

type TxPool struct {
    config      Config
    chainconfig *params.ChainConfig
    chain       BlockChain
    signer      types.Signer

    // 交易存储
    pending map[common.Address]*txList // 可执行交易
    queue   map[common.Address]*txList // 待处理交易

    // 价格排序
    priced *pricedList

    // 状态
    currentState  *state.StateDB
    pendingNonces *noncer

    // 限制
    currentMaxGas uint64
}

// 添加交易
func (pool *TxPool) Add(txs []*types.Transaction, local bool, sync bool) []error {
    // 1. 基础验证
    var errs []error
    for _, tx := range txs {
        if err := pool.validateTx(tx, local); err != nil {
            errs = append(errs, err)
            continue
        }

        // 2. 添加到队列
        pool.enqueueTx(tx.Hash(), tx, local, true)
    }

    // 3. 提升可执行交易
    pool.promoteExecutables(nil)

    return errs
}

// 交易验证
func (pool *TxPool) validateTx(tx *types.Transaction, local bool) error {
    // 大小检查
    if tx.Size() > txMaxSize {
        return ErrOversizedData
    }

    // Gas 限制
    if tx.Gas() > pool.currentMaxGas {
        return ErrGasLimit
    }

    // Gas 价格检查
    if !local && tx.GasTipCapIntCmp(pool.gasPrice) < 0 {
        return ErrUnderpriced
    }

    // 签名验证
    from, err := types.Sender(pool.signer, tx)
    if err != nil {
        return ErrInvalidSender
    }

    // Nonce 检查
    if pool.currentState.GetNonce(from) > tx.Nonce() {
        return ErrNonceTooLow
    }

    // 余额检查
    if pool.currentState.GetBalance(from).Cmp(tx.Cost()) < 0 {
        return ErrInsufficientFunds
    }

    // 内在 Gas 检查
    intrinsicGas, err := core.IntrinsicGas(tx.Data(), tx.AccessList(), tx.To() == nil, true, true, true)
    if err != nil || tx.Gas() < intrinsicGas {
        return ErrIntrinsicGas
    }

    return nil
}

// 提升可执行交易
func (pool *TxPool) promoteExecutables(accounts []common.Address) []*types.Transaction {
    var promoted []*types.Transaction

    for addr := range pool.queue {
        list := pool.queue[addr]
        nonce := pool.pendingNonces.get(addr)

        // 移动连续 nonce 的交易到 pending
        for _, tx := range list.Forward(nonce) {
            pool.pending[addr].Add(tx, pool.config.PriceBump)
            promoted = append(promoted, tx)
        }

        // 更新 nonce
        if list.Len() > 0 && list.txs.Get(nonce) != nil {
            pool.pendingNonces.set(addr, nonce+1)
        }
    }

    return promoted
}
```

### U.2 交易排序与替换

```go
// 按 Gas 价格排序的交易列表
type pricedList struct {
    all    *lookup
    urgent *priceHeap // 紧急交易 (高 gas price)
    floating *priceHeap // 浮动交易
}

// 价格堆实现
type priceHeap struct {
    baseFee *big.Int
    list    []*types.Transaction
}

func (h *priceHeap) Less(i, j int) bool {
    // EIP-1559: 比较有效 tip
    tipI := h.list[i].EffectiveGasTipIntCmp(h.list[j], h.baseFee)
    if tipI != 0 {
        return tipI > 0
    }
    // 相同 tip 时按 nonce 排序
    return h.list[i].Nonce() < h.list[j].Nonce()
}

// 交易替换逻辑
func (l *txList) Add(tx *types.Transaction, priceBump uint64) (bool, *types.Transaction) {
    old := l.txs.Get(tx.Nonce())

    // 新交易
    if old == nil {
        l.txs.Put(tx)
        return true, nil
    }

    // 替换检查: 新交易价格必须高于旧交易 * (100 + priceBump) / 100
    threshold := new(big.Int).Mul(old.GasPrice(), big.NewInt(100+int64(priceBump)))
    threshold.Div(threshold, big.NewInt(100))

    if tx.GasPriceCmp(old) < 0 || tx.GasPrice().Cmp(threshold) < 0 {
        return false, nil
    }

    // 替换
    l.txs.Put(tx)
    return true, old
}
```

### U.3 MEV 与交易排序

```
┌─────────────────────────────────────────────────────────────────┐
│                    交易池与 MEV 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   用户交易                        MEV Searcher                   │
│   ┌─────────┐                    ┌─────────────┐                │
│   │   Tx    │                    │ MEV Bundle  │                │
│   └────┬────┘                    │ - frontrun  │                │
│        │                         │ - backrun   │                │
│        │                         │ - sandwich  │                │
│        │                         └──────┬──────┘                │
│        │                                │                        │
│        ▼                                ▼                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Public Mempool                        │   │
│   │  ┌─────────────────────────────────────────────────┐    │   │
│   │  │  Pending Transactions (sorted by gas price)     │    │   │
│   │  │  [Tx1: 100 gwei] [Tx2: 90 gwei] [Tx3: 80 gwei] │    │   │
│   │  └─────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                          │                                       │
│                          │ 传统矿工/验证者                        │
│                          ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Block Builder                         │   │
│   │  方式1: 按 gas price 排序 (传统)                         │   │
│   │  方式2: PBS (Proposer-Builder Separation)               │   │
│   │                                                          │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  Block Template                                    │  │   │
│   │  │  1. MEV Bundle (if profitable)                     │  │   │
│   │  │  2. High gas price txs                             │  │   │
│   │  │  3. Remaining txs                                  │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   PBS 架构 (Post-Merge)                                          │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Proposer                     Builder                    │   │
│   │  (Validator)                  (Specialized)              │   │
│   │  ┌─────────┐                 ┌─────────────┐            │   │
│   │  │ Selects │◀── Bids ───────│  Builds     │            │   │
│   │  │ Header  │                 │  Blocks     │            │   │
│   │  └────┬────┘                 └──────┬──────┘            │   │
│   │       │                             │                    │   │
│   │       │ Commits                     │ MEV Extraction    │   │
│   │       ▼                             ▼                    │   │
│   │  ┌─────────┐                 ┌─────────────┐            │   │
│   │  │ Block   │◀── Reveals ────│  Bundle     │            │   │
│   │  │ Header  │                 │  Ordering   │            │   │
│   │  └─────────┘                 └─────────────┘            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 V：RPC 实现

### V.1 JSON-RPC 服务

```go
// 文件: rpc/server.go

type Server struct {
    services serviceRegistry
    codecs   mapset.Set
    run      int32
}

// 注册服务
func (s *Server) RegisterName(name string, receiver interface{}) error {
    return s.services.registerName(name, receiver)
}

// 处理请求
func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 读取请求体
    body, err := ioutil.ReadAll(r.Body)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // 解析 JSON-RPC
    var reqs []jsonrpcRequest
    if err := json.Unmarshal(body, &reqs); err != nil {
        // 尝试单个请求
        var req jsonrpcRequest
        if err := json.Unmarshal(body, &req); err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        reqs = []jsonrpcRequest{req}
    }

    // 并行处理批量请求
    responses := make([]jsonrpcResponse, len(reqs))
    var wg sync.WaitGroup

    for i, req := range reqs {
        wg.Add(1)
        go func(i int, req jsonrpcRequest) {
            defer wg.Done()
            responses[i] = s.handleRequest(req)
        }(i, req)
    }

    wg.Wait()

    // 返回响应
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(responses)
}

// 处理单个请求
func (s *Server) handleRequest(req jsonrpcRequest) jsonrpcResponse {
    // 查找方法
    service, method, err := s.services.lookup(req.Method)
    if err != nil {
        return jsonrpcResponse{
            ID:    req.ID,
            Error: &jsonrpcError{Code: -32601, Message: "Method not found"},
        }
    }

    // 调用方法
    result, err := method.call(service, req.Params)
    if err != nil {
        return jsonrpcResponse{
            ID:    req.ID,
            Error: &jsonrpcError{Code: -32000, Message: err.Error()},
        }
    }

    return jsonrpcResponse{
        ID:     req.ID,
        Result: result,
    }
}
```

### V.2 eth 命名空间实现

```go
// 文件: internal/ethapi/api.go

type EthereumAPI struct {
    b Backend
}

// eth_blockNumber
func (api *EthereumAPI) BlockNumber() hexutil.Uint64 {
    header, _ := api.b.HeaderByNumber(context.Background(), rpc.LatestBlockNumber)
    return hexutil.Uint64(header.Number.Uint64())
}

// eth_getBalance
func (api *EthereumAPI) GetBalance(ctx context.Context, address common.Address, blockNrOrHash rpc.BlockNumberOrHash) (*hexutil.Big, error) {
    state, _, err := api.b.StateAndHeaderByNumberOrHash(ctx, blockNrOrHash)
    if err != nil {
        return nil, err
    }
    return (*hexutil.Big)(state.GetBalance(address)), nil
}

// eth_call
func (api *EthereumAPI) Call(ctx context.Context, args TransactionArgs, blockNrOrHash rpc.BlockNumberOrHash, overrides *StateOverride) (hexutil.Bytes, error) {
    result, err := DoCall(ctx, api.b, args, blockNrOrHash, overrides, api.b.RPCEVMTimeout(), api.b.RPCGasCap())
    if err != nil {
        return nil, err
    }
    if len(result.Revert()) > 0 {
        return nil, newRevertError(result)
    }
    return result.Return(), result.Err
}

// eth_estimateGas
func (api *EthereumAPI) EstimateGas(ctx context.Context, args TransactionArgs, blockNrOrHash *rpc.BlockNumberOrHash) (hexutil.Uint64, error) {
    // 二分查找 gas 限制
    lo := params.TxGas - 1
    hi := api.b.RPCGasCap()

    // 如果指定了 gas，使用它作为上限
    if args.Gas != nil && uint64(*args.Gas) < hi {
        hi = uint64(*args.Gas)
    }

    // 执行并检查结果
    executable := func(gas uint64) (bool, *core.ExecutionResult, error) {
        args.Gas = (*hexutil.Uint64)(&gas)
        result, err := DoCall(ctx, api.b, args, *blockNrOrHash, nil, 0, gas)
        if err != nil {
            return false, nil, err
        }
        return result.Failed() == false, result, nil
    }

    // 二分查找
    for lo+1 < hi {
        mid := (lo + hi) / 2
        ok, _, err := executable(mid)
        if err != nil {
            return 0, err
        }
        if ok {
            hi = mid
        } else {
            lo = mid
        }
    }

    return hexutil.Uint64(hi), nil
}

// eth_sendRawTransaction
func (api *EthereumAPI) SendRawTransaction(ctx context.Context, input hexutil.Bytes) (common.Hash, error) {
    tx := new(types.Transaction)
    if err := tx.UnmarshalBinary(input); err != nil {
        return common.Hash{}, err
    }
    return SubmitTransaction(ctx, api.b, tx)
}
```

### V.3 WebSocket 订阅

```go
// 文件: rpc/subscription.go

type Subscription struct {
    ID        ID
    namespace string
    chanSize  int
    channel   chan interface{}
    err       chan error
}

// eth_subscribe 实现
func (api *EthereumAPI) Subscribe(ctx context.Context, subscriptionType string, args ...interface{}) (*rpc.Subscription, error) {
    notifier, supported := rpc.NotifierFromContext(ctx)
    if !supported {
        return nil, rpc.ErrNotificationsUnsupported
    }

    switch subscriptionType {
    case "newHeads":
        return api.subscribeNewHeads(ctx, notifier)
    case "logs":
        return api.subscribeLogs(ctx, notifier, args)
    case "newPendingTransactions":
        return api.subscribePendingTxs(ctx, notifier)
    default:
        return nil, fmt.Errorf("unknown subscription type: %s", subscriptionType)
    }
}

// 新区块头订阅
func (api *EthereumAPI) subscribeNewHeads(ctx context.Context, notifier *rpc.Notifier) (*rpc.Subscription, error) {
    subscription := notifier.CreateSubscription()

    go func() {
        headers := make(chan *types.Header)
        sub := api.b.SubscribeNewHead(headers)

        for {
            select {
            case header := <-headers:
                notifier.Notify(subscription.ID, header)
            case err := <-sub.Err():
                subscription.Err() <- err
                return
            case <-subscription.Err():
                sub.Unsubscribe()
                return
            }
        }
    }()

    return subscription, nil
}

// 日志订阅
func (api *EthereumAPI) subscribeLogs(ctx context.Context, notifier *rpc.Notifier, args []interface{}) (*rpc.Subscription, error) {
    // 解析过滤条件
    var filter FilterCriteria
    if len(args) > 0 {
        if err := json.Unmarshal(args[0].(json.RawMessage), &filter); err != nil {
            return nil, err
        }
    }

    subscription := notifier.CreateSubscription()

    go func() {
        logs := make(chan []*types.Log)
        sub := api.b.SubscribeLogs(ethereum.FilterQuery(filter), logs)

        for {
            select {
            case matched := <-logs:
                for _, log := range matched {
                    notifier.Notify(subscription.ID, log)
                }
            case err := <-sub.Err():
                subscription.Err() <- err
                return
            case <-subscription.Err():
                sub.Unsubscribe()
                return
            }
        }
    }()

    return subscription, nil
}
```

### V.4 Engine API (共识层接口)

```go
// 文件: eth/catalyst/api.go

type ConsensusAPI struct {
    eth *eth.Ethereum
}

// engine_newPayloadV3 - 接收新区块
func (api *ConsensusAPI) NewPayloadV3(params engine.ExecutableData, versionedHashes []common.Hash, beaconRoot *common.Hash) (engine.PayloadStatusV1, error) {
    // 验证 versioned hashes (blob 交易)
    if err := api.verifyBlobHashes(params.Transactions, versionedHashes); err != nil {
        return engine.PayloadStatusV1{Status: engine.INVALID}, err
    }

    // 构建区块
    block, err := engine.ExecutableDataToBlock(params, versionedHashes, beaconRoot)
    if err != nil {
        return engine.PayloadStatusV1{Status: engine.INVALID}, err
    }

    // 执行并验证
    if err := api.eth.BlockChain().InsertBlockWithoutSetHead(block); err != nil {
        return engine.PayloadStatusV1{
            Status:          engine.INVALID,
            ValidationError: err.Error(),
        }, nil
    }

    return engine.PayloadStatusV1{
        Status:          engine.VALID,
        LatestValidHash: &block.Hash(),
    }, nil
}

// engine_forkchoiceUpdatedV3 - 更新分叉选择
func (api *ConsensusAPI) ForkchoiceUpdatedV3(
    state engine.ForkchoiceStateV1,
    attrs *engine.PayloadAttributes,
) (engine.ForkChoiceResponse, error) {

    // 更新链头
    if err := api.eth.BlockChain().SetCanonical(state.HeadBlockHash); err != nil {
        return engine.ForkChoiceResponse{}, err
    }

    // 如果需要构建新区块
    if attrs != nil {
        payload, err := api.buildPayload(state.HeadBlockHash, attrs)
        if err != nil {
            return engine.ForkChoiceResponse{}, err
        }
        return engine.ForkChoiceResponse{
            PayloadStatus: engine.PayloadStatusV1{Status: engine.VALID},
            PayloadID:     &payload.ID,
        }, nil
    }

    return engine.ForkChoiceResponse{
        PayloadStatus: engine.PayloadStatusV1{Status: engine.VALID},
    }, nil
}

// engine_getPayloadV3 - 获取构建的区块
func (api *ConsensusAPI) GetPayloadV3(payloadID engine.PayloadID) (*engine.ExecutionPayloadEnvelope, error) {
    payload := api.localBlocks.get(payloadID)
    if payload == nil {
        return nil, engine.UnknownPayload
    }

    return &engine.ExecutionPayloadEnvelope{
        ExecutionPayload: payload.ExecutableData,
        BlockValue:       payload.fees,
        BlobsBundle:      payload.blobsBundle,
    }, nil
}
```

---

## 附录 W：硬分叉管理

### W.1 链配置

```go
// 文件: params/config.go

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
    ArrowGlacierBlock   *big.Int `json:"arrowGlacierBlock,omitempty"`
    GrayGlacierBlock    *big.Int `json:"grayGlacierBlock,omitempty"`

    // 时间戳激活 (Post-Merge)
    ShanghaiTime *uint64 `json:"shanghaiTime,omitempty"`
    CancunTime   *uint64 `json:"cancunTime,omitempty"`
    PragueTime   *uint64 `json:"pragueTime,omitempty"`
}

// 检查是否激活特定分叉
func (c *ChainConfig) IsLondon(num *big.Int) bool {
    return isBlockForked(c.LondonBlock, num)
}

func (c *ChainConfig) IsCancun(num *big.Int, time uint64) bool {
    return isTimestampForked(c.CancunTime, time)
}

func isBlockForked(s *big.Int, head *big.Int) bool {
    if s == nil || head == nil {
        return false
    }
    return s.Cmp(head) <= 0
}

func isTimestampForked(s *uint64, time uint64) bool {
    if s == nil {
        return false
    }
    return *s <= time
}

// 主网配置
var MainnetChainConfig = &ChainConfig{
    ChainID:             big.NewInt(1),
    HomesteadBlock:      big.NewInt(1_150_000),
    DAOForkBlock:        big.NewInt(1_920_000),
    EIP150Block:         big.NewInt(2_463_000),
    ByzantiumBlock:      big.NewInt(4_370_000),
    ConstantinopleBlock: big.NewInt(7_280_000),
    IstanbulBlock:       big.NewInt(9_069_000),
    BerlinBlock:         big.NewInt(12_244_000),
    LondonBlock:         big.NewInt(12_965_000),
    ShanghaiTime:        newUint64(1681338455),
    CancunTime:          newUint64(1710338135),
}
```

### W.2 分叉规则应用

```go
// 文件: core/vm/jump_table.go

func newFrontierInstructionSet() JumpTable {
    tbl := JumpTable{}
    // 基础指令集
    tbl[STOP] = &operation{execute: opStop, gasCost: constGasFunc(0)}
    tbl[ADD] = &operation{execute: opAdd, gasCost: constGasFunc(3)}
    // ... 其他 Frontier 指令
    return tbl
}

func newHomesteadInstructionSet() JumpTable {
    tbl := newFrontierInstructionSet()
    // Homestead 修改
    tbl[DELEGATECALL] = &operation{
        execute:    opDelegateCall,
        gasCost:    gasDelegateCall,
        dynamicGas: true,
    }
    return tbl
}

func newBerlinInstructionSet() JumpTable {
    tbl := newIstanbulInstructionSet()
    // EIP-2929: 访问列表
    tbl[SLOAD].gasCost = gasSLoadEIP2929
    tbl[EXTCODECOPY].gasCost = gasExtCodeCopyEIP2929
    return tbl
}

func newCancunInstructionSet() JumpTable {
    tbl := newShanghaiInstructionSet()
    // EIP-1153: 瞬态存储
    tbl[TLOAD] = &operation{execute: opTload, gasCost: constGasFunc(100)}
    tbl[TSTORE] = &operation{execute: opTstore, gasCost: constGasFunc(100)}
    // EIP-5656: MCOPY
    tbl[MCOPY] = &operation{execute: opMcopy, dynamicGas: true}
    // EIP-4844: BLOBHASH
    tbl[BLOBHASH] = &operation{execute: opBlobHash, gasCost: constGasFunc(3)}
    return tbl
}

// 根据配置选择指令集
func NewEVMInterpreter(evm *EVM) *EVMInterpreter {
    var table *JumpTable
    switch {
    case evm.chainRules.IsPrague:
        table = &pragueInstructionSet
    case evm.chainRules.IsCancun:
        table = &cancunInstructionSet
    case evm.chainRules.IsShanghai:
        table = &shanghaiInstructionSet
    case evm.chainRules.IsLondon:
        table = &londonInstructionSet
    // ... 其他分叉
    default:
        table = &frontierInstructionSet
    }
    return &EVMInterpreter{evm: evm, table: table}
}
```

### W.3 evmone 分叉处理

```cpp
// evmone 硬分叉配置
enum class evmc_revision {
    EVMC_FRONTIER = 0,
    EVMC_HOMESTEAD = 1,
    EVMC_TANGERINE_WHISTLE = 2,
    EVMC_SPURIOUS_DRAGON = 3,
    EVMC_BYZANTIUM = 4,
    EVMC_CONSTANTINOPLE = 5,
    EVMC_PETERSBURG = 6,
    EVMC_ISTANBUL = 7,
    EVMC_BERLIN = 8,
    EVMC_LONDON = 9,
    EVMC_PARIS = 10,
    EVMC_SHANGHAI = 11,
    EVMC_CANCUN = 12,
    EVMC_PRAGUE = 13,
};

// 根据分叉选择操作码表
const InstructionTable& get_instruction_table(evmc_revision rev) noexcept {
    static const auto tables = []() {
        std::array<InstructionTable, EVMC_MAX_REVISION + 1> tabs;
        for (int r = EVMC_FRONTIER; r <= EVMC_MAX_REVISION; ++r) {
            tabs[r] = create_instruction_table(static_cast<evmc_revision>(r));
        }
        return tabs;
    }();
    return tables[rev];
}

// 创建分叉特定的指令表
InstructionTable create_instruction_table(evmc_revision rev) noexcept {
    InstructionTable table;

    // 基础指令 (所有分叉)
    table[OP_STOP] = {op_stop, 0};
    table[OP_ADD] = {op_add, 3};
    // ...

    // Berlin 新增
    if (rev >= EVMC_BERLIN) {
        // EIP-2929 修改 gas
    }

    // London 新增
    if (rev >= EVMC_LONDON) {
        table[OP_BASEFEE] = {op_basefee, 2};
    }

    // Shanghai 新增
    if (rev >= EVMC_SHANGHAI) {
        table[OP_PUSH0] = {op_push0, 2};
    }

    // Cancun 新增
    if (rev >= EVMC_CANCUN) {
        table[OP_TLOAD] = {op_tload, 100};
        table[OP_TSTORE] = {op_tstore, 100};
        table[OP_MCOPY] = {op_mcopy, 3};
        table[OP_BLOBHASH] = {op_blobhash, 3};
        table[OP_BLOBBASEFEE] = {op_blobbasefee, 2};
    }

    return table;
}
```

---

## 附录 X：网络层

### X.1 devp2p 协议

```go
// 文件: p2p/server.go

type Server struct {
    Config
    listener     net.Listener
    ourHandshake *protoHandshake
    peers        map[discover.NodeID]*Peer
}

// 节点发现
type Table struct {
    mutex   sync.Mutex
    buckets [nBuckets]*bucket // Kademlia DHT
    nursery []*node
    db      *enode.DB
}

// 查找节点 (Kademlia lookup)
func (tab *Table) lookup(target enode.ID, refreshIfEmpty bool) []*node {
    var (
        result    nodesByDistance
        asked     = make(map[enode.ID]bool)
        pendingQueries = 0
        result_ch = make(chan []*node, alpha)
    )

    // 初始化: 取最近的节点
    result.entries = tab.closest(target, bucketSize, false).entries

    for {
        // 发送 alpha 个并发查询
        for i := 0; i < len(result.entries) && pendingQueries < alpha; i++ {
            n := result.entries[i]
            if !asked[n.ID()] {
                asked[n.ID()] = true
                pendingQueries++
                go func() {
                    result_ch <- tab.findnode(n, target)
                }()
            }
        }

        if pendingQueries == 0 {
            break
        }

        // 收集结果
        nodes := <-result_ch
        pendingQueries--
        for _, n := range nodes {
            result.push(n, bucketSize)
        }
    }

    return result.entries
}

// eth 协议握手
func (p *Peer) Handshake(network uint64, head common.Hash, genesis common.Hash) error {
    // 发送 Status 消息
    msg := &StatusPacket{
        ProtocolVersion: uint32(p.version),
        NetworkID:       network,
        TD:              p.td,
        Head:            head,
        Genesis:         genesis,
        ForkID:          forkid.NewID(p.chainConfig, genesis, p.headNumber),
    }

    errc := make(chan error, 2)
    go func() { errc <- p2p.Send(p.rw, StatusMsg, msg) }()
    go func() { errc <- p.readStatus(network, genesis) }()

    for i := 0; i < 2; i++ {
        if err := <-errc; err != nil {
            return err
        }
    }
    return nil
}
```

### X.2 区块同步协议

```go
// 文件: eth/downloader/downloader.go

// 同步模式
const (
    FullSync  SyncMode = iota // 完全验证所有区块
    SnapSync                   // 快照同步状态
    LightSync                  // 轻节点同步
)

// 下载器
type Downloader struct {
    mode    SyncMode
    peers   *peerSet
    stateDB ethdb.Database

    // 队列
    queue *queue

    // 同步进度
    syncStatsChainOrigin uint64
    syncStatsChainHeight uint64
}

// 同步流程
func (d *Downloader) synchronise(id string, hash common.Hash, td *big.Int, mode SyncMode) error {
    // 1. 获取枢轴点 (pivot)
    pivot := d.findAncestor(p, hash)

    // 2. 下载区块头
    headerCh := make(chan dataPack)
    go d.fetchHeaders(p, pivot, headerCh)

    // 3. 下载区块体
    bodyCh := make(chan dataPack)
    go d.fetchBodies(headerCh, bodyCh)

    // 4. 下载收据 (快照同步)
    if mode == SnapSync {
        receiptCh := make(chan dataPack)
        go d.fetchReceipts(headerCh, receiptCh)
    }

    // 5. 处理并导入
    return d.processFullSyncContent()
}

// 区块传播
func (pm *ProtocolManager) BroadcastBlock(block *types.Block, propagate bool) {
    hash := block.Hash()
    peers := pm.peers.PeersWithoutBlock(hash)

    if propagate {
        // 广播完整区块给 sqrt(peers)
        transferLen := int(math.Sqrt(float64(len(peers))))
        for _, peer := range peers[:transferLen] {
            peer.AsyncSendNewBlock(block, td)
        }
    }

    // 广播哈希给其他节点
    for _, peer := range peers[transferLen:] {
        peer.AsyncSendNewBlockHash(block)
    }
}
```

### X.3 网络架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    以太坊网络架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Discovery Layer                        │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│   │  │ discv4      │  │ discv5      │  │ DNS Discovery   │  │   │
│   │  │ (Kademlia)  │  │ (ENR based) │  │ (Bootstrap)     │  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│   └──────────────────────────┬──────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Transport Layer                        │   │
│   │  ┌─────────────────────────────────────────────────┐    │   │
│   │  │                   RLPx                           │    │   │
│   │  │  - ECIES 加密握手                                │    │   │
│   │  │  - 帧分包                                       │    │   │
│   │  │  - 消息多路复用                                  │    │   │
│   │  └─────────────────────────────────────────────────┘    │   │
│   └──────────────────────────┬──────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Protocol Layer                         │   │
│   │                                                          │   │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────────────┐   │   │
│   │  │ eth/68    │  │ snap/1    │  │ les/4 (Light)     │   │   │
│   │  │           │  │           │  │                    │   │   │
│   │  │ - Status  │  │ - Account │  │ - GetBlockHeaders │   │   │
│   │  │ - NewBlock│  │   Range   │  │ - GetProofs       │   │   │
│   │  │ - Txs     │  │ - Storage │  │                    │   │   │
│   │  │ - GetBlock│  │   Range   │  │                    │   │   │
│   │  │   Headers │  │ - ByteCodes│  │                    │   │   │
│   │  └───────────┘  └───────────┘  └───────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   消息类型 (eth/68)                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  0x00 Status           - 握手状态                        │   │
│   │  0x01 NewBlockHashes   - 新区块哈希通知                   │   │
│   │  0x02 Transactions     - 交易广播                        │   │
│   │  0x03 GetBlockHeaders  - 请求区块头                      │   │
│   │  0x04 BlockHeaders     - 区块头响应                      │   │
│   │  0x05 GetBlockBodies   - 请求区块体                      │   │
│   │  0x06 BlockBodies      - 区块体响应                      │   │
│   │  0x07 NewBlock         - 完整区块广播                    │   │
│   │  0x08 NewPooledTxHashes- 新交易哈希                      │   │
│   │  0x09 GetPooledTxs     - 请求交易详情                    │   │
│   │  0x0a PooledTxs        - 交易详情响应                    │   │
│   │  0x0b GetReceipts      - 请求收据                        │   │
│   │  0x0c Receipts         - 收据响应                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录 Y：数据库后端

### Y.1 LevelDB vs Pebble

```go
// 文件: ethdb/leveldb/leveldb.go

type Database struct {
    fn string      // 文件路径
    db *leveldb.DB // LevelDB 实例

    // 指标
    compTime  int64
    compRead  int64
    compWrite int64
}

func New(file string, cache int, handles int) (*Database, error) {
    // 配置 LevelDB
    opts := &opt.Options{
        OpenFilesCacheCapacity: handles,
        BlockCacheCapacity:     cache / 2 * opt.MiB,
        WriteBuffer:            cache / 4 * opt.MiB,
        Filter:                 filter.NewBloomFilter(10),
    }

    db, err := leveldb.OpenFile(file, opts)
    if err != nil {
        return nil, err
    }

    return &Database{fn: file, db: db}, nil
}

// 批量写入
func (db *Database) NewBatch() ethdb.Batch {
    return &batch{
        db: db.db,
        b:  new(leveldb.Batch),
    }
}

// Pebble 实现 (更高性能)
// 文件: ethdb/pebble/pebble.go

type Database struct {
    fn string
    db *pebble.DB
}

func New(file string, cache int) (*Database, error) {
    opts := &pebble.Options{
        Cache:                       pebble.NewCache(int64(cache * 1024 * 1024)),
        MaxOpenFiles:                500,
        MemTableSize:                64 * 1024 * 1024,
        MemTableStopWritesThreshold: 4,
        L0CompactionThreshold:       4,
        L0StopWritesThreshold:       12,
        LBaseMaxBytes:               64 * 1024 * 1024,
        Levels: []pebble.LevelOptions{
            {TargetFileSize: 2 * 1024 * 1024},
            {TargetFileSize: 4 * 1024 * 1024},
            {TargetFileSize: 8 * 1024 * 1024},
            {TargetFileSize: 16 * 1024 * 1024},
            {TargetFileSize: 32 * 1024 * 1024},
            {TargetFileSize: 64 * 1024 * 1024},
        },
    }

    db, err := pebble.Open(file, opts)
    return &Database{fn: file, db: db}, err
}
```

### Y.2 数据布局

```go
// 文件: core/rawdb/schema.go

// 键前缀定义
var (
    // 区块数据
    headerPrefix       = []byte("h") // headerPrefix + num + hash -> header
    bodyPrefix         = []byte("b") // bodyPrefix + num + hash -> body
    receiptsPrefix     = []byte("r") // receiptsPrefix + num + hash -> receipts
    headerNumberPrefix = []byte("H") // headerNumberPrefix + hash -> num

    // 状态数据
    CodePrefix     = []byte("c") // CodePrefix + code hash -> code
    accountPrefix  = []byte("a") // accountPrefix + address hash -> account
    storagePrefix  = []byte("s") // storagePrefix + address hash + slot hash -> value

    // Trie 节点
    trieNodePrefix = []byte("t") // trieNodePrefix + node hash -> node

    // 索引
    txLookupPrefix  = []byte("l") // txLookupPrefix + tx hash -> block num + index
    bloomBitsPrefix = []byte("B") // bloomBitsPrefix + bit + section + hash -> bloom bits
)

// 编码区块头键
func headerKey(number uint64, hash common.Hash) []byte {
    return append(append(headerPrefix, encodeBlockNumber(number)...), hash.Bytes()...)
}

// 读取区块头
func ReadHeader(db ethdb.Reader, hash common.Hash, number uint64) *types.Header {
    data, _ := db.Get(headerKey(number, hash))
    if len(data) == 0 {
        return nil
    }
    header := new(types.Header)
    if err := rlp.Decode(bytes.NewReader(data), header); err != nil {
        return nil
    }
    return header
}
```

### Y.3 数据库性能对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    数据库后端对比                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 特性          │ LevelDB      │ Pebble       │ MDBX       │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │ 语言          │ C++          │ Go           │ C          │   │
│   │ 写入性能      │ 中等         │ 高           │ 非常高     │   │
│   │ 读取性能      │ 高           │ 高           │ 非常高     │   │
│   │ 压缩          │ Snappy       │ Snappy/Zstd  │ LZ4        │   │
│   │ 并发写入      │ 单写入者     │ 单写入者     │ 多写入者   │   │
│   │ 内存映射      │ 部分         │ 部分         │ 完全       │   │
│   │ ACID          │ WAL          │ WAL          │ MVCC       │   │
│   │ 空间效率      │ 中等         │ 中等         │ 高         │   │
│   │ 维护者        │ Google       │ Cockroach    │ Erigon     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   存储空间对比 (主网完整节点)                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   LevelDB (Geth):     ~900 GB                           │   │
│   │   Pebble (Geth):      ~850 GB                           │   │
│   │   MDBX (Erigon):      ~550 GB                           │   │
│   │                                                          │   │
│   │   Archive Node:                                          │   │
│   │   LevelDB:            ~16 TB                            │   │
│   │   MDBX:               ~3 TB                             │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   LSM Tree vs B+ Tree                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   LevelDB/Pebble (LSM Tree)    MDBX (B+ Tree)           │   │
│   │                                                          │   │
│   │   ┌───────────────┐            ┌───────────────┐        │   │
│   │   │   MemTable    │            │   Root Page   │        │   │
│   │   │   (in memory) │            │               │        │   │
│   │   └───────┬───────┘            └───────┬───────┘        │   │
│   │           │ flush                      │                │   │
│   │           ▼                            ▼                │   │
│   │   ┌───────────────┐            ┌───────────────┐        │   │
│   │   │   L0 SSTable  │            │ Branch Pages  │        │   │
│   │   └───────┬───────┘            └───────┬───────┘        │   │
│   │           │ compact                    │                │   │
│   │           ▼                            ▼                │   │
│   │   ┌───────────────┐            ┌───────────────┐        │   │
│   │   │   L1 SSTable  │            │  Leaf Pages   │        │   │
│   │   └───────┬───────┘            │  (data)       │        │   │
│   │           │                    └───────────────┘        │   │
│   │           ▼                                             │   │
│   │   ┌───────────────┐                                     │   │
│   │   │   L2+ SSTable │                                     │   │
│   │   └───────────────┘                                     │   │
│   │                                                          │   │
│   │   优点:                        优点:                     │   │
│   │   - 顺序写入快                 - 读取快                  │   │
│   │   - 压缩高效                   - 无压缩开销              │   │
│   │   缺点:                        - 空间效率高              │   │
│   │   - 读取放大                   缺点:                     │   │
│   │   - 空间放大                   - 随机写入                │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Y.4 缓存策略

```go
// 状态缓存
type StateCache struct {
    // 账户缓存
    accounts *lru.Cache[common.Hash, []byte]
    // 存储缓存
    storage  *lru.Cache[common.Hash, []byte]
    // 代码缓存
    code     *lru.Cache[common.Hash, []byte]
    // Trie 节点缓存
    triedb   *trie.Database
}

// Trie 数据库缓存
type Database struct {
    diskdb ethdb.Database

    // 节点缓存
    cleans  *fastcache.Cache // 干净节点 (已持久化)
    dirties map[common.Hash]*cachedNode // 脏节点 (待写入)

    // 内存限制
    dirtiesSize common.StorageSize
    limit       common.StorageSize
}

// 提交缓存到磁盘
func (db *Database) Commit(root common.Hash, report bool) error {
    // 收集所有可达节点
    nodes := make(map[common.Hash][]byte)
    db.collectDirtyNodes(root, nodes)

    // 批量写入
    batch := db.diskdb.NewBatch()
    for hash, node := range nodes {
        batch.Put(hash[:], node)
    }

    // 清理脏缓存
    for hash := range nodes {
        delete(db.dirties, hash)
    }

    return batch.Write()
}
```

---

## 附录 Z：Witness 生成

### Z.1 无状态客户端概念

```
┌─────────────────────────────────────────────────────────────────┐
│                    无状态客户端架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   传统有状态客户端                 无状态客户端                   │
│   ┌─────────────────────┐         ┌─────────────────────┐       │
│   │  Full Node          │         │  Stateless Node     │       │
│   │  ┌───────────────┐  │         │  ┌───────────────┐  │       │
│   │  │ State DB      │  │         │  │ No State DB   │  │       │
│   │  │ ~900 GB       │  │         │  │ ~0 GB         │  │       │
│   │  │               │  │         │  │               │  │       │
│   │  │ 完整状态树    │  │         │  │ 仅验证 witness│  │       │
│   │  └───────────────┘  │         │  └───────────────┘  │       │
│   └─────────────────────┘         └─────────────────────┘       │
│            │                               │                     │
│            │ 执行区块                       │ 执行区块            │
│            ▼                               ▼                     │
│   ┌─────────────────────┐         ┌─────────────────────┐       │
│   │  从本地 DB 读取状态  │         │  从 witness 读取状态 │       │
│   │  验证状态根          │         │  验证 witness 证明   │       │
│   └─────────────────────┘         └─────────────────────┘       │
│                                                                  │
│   Witness 结构                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Block Witness {                                         │   │
│   │    // 区块执行需要的所有状态数据                          │   │
│   │    accounts: [                                           │   │
│   │      { address, nonce, balance, code_hash, storage_root }│   │
│   │    ],                                                    │   │
│   │    storage: [                                            │   │
│   │      { address, slot, value }                            │   │
│   │    ],                                                    │   │
│   │    codes: [                                              │   │
│   │      { code_hash, bytecode }                             │   │
│   │    ],                                                    │   │
│   │    // Merkle 证明                                        │   │
│   │    proofs: [                                             │   │
│   │      { path, nodes[] }                                   │   │
│   │    ]                                                     │   │
│   │  }                                                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Z.2 Witness 生成实现

```go
// 文件: core/state/witness.go

// Witness 收集器
type WitnessCollector struct {
    // 访问的账户
    accounts map[common.Address]*AccountWitness
    // 访问的存储槽
    storage map[common.Address]map[common.Hash]common.Hash
    // 访问的代码
    codes map[common.Hash][]byte
    // Merkle 证明节点
    proofNodes map[common.Hash][]byte
}

type AccountWitness struct {
    Address     common.Address
    Nonce       uint64
    Balance     *big.Int
    CodeHash    common.Hash
    StorageRoot common.Hash
    // 证明路径
    ProofPath   [][]byte
}

// 在执行时收集 witness
func (w *WitnessCollector) OnAccountAccess(addr common.Address, account *types.StateAccount) {
    if _, exists := w.accounts[addr]; !exists {
        w.accounts[addr] = &AccountWitness{
            Address:     addr,
            Nonce:       account.Nonce,
            Balance:     account.Balance,
            CodeHash:    account.CodeHash,
            StorageRoot: account.Root,
        }
    }
}

func (w *WitnessCollector) OnStorageAccess(addr common.Address, slot, value common.Hash) {
    if w.storage[addr] == nil {
        w.storage[addr] = make(map[common.Hash]common.Hash)
    }
    w.storage[addr][slot] = value
}

func (w *WitnessCollector) OnCodeAccess(codeHash common.Hash, code []byte) {
    if _, exists := w.codes[codeHash]; !exists {
        w.codes[codeHash] = code
    }
}

// 生成完整 witness
func (w *WitnessCollector) GenerateWitness(stateDB *StateDB) (*BlockWitness, error) {
    witness := &BlockWitness{
        Accounts: make([]AccountWitness, 0, len(w.accounts)),
        Storage:  make([]StorageWitness, 0),
        Codes:    make([]CodeWitness, 0, len(w.codes)),
    }

    // 收集账户证明
    for addr, acc := range w.accounts {
        proof, err := stateDB.GetProof(addr)
        if err != nil {
            return nil, err
        }
        acc.ProofPath = proof
        witness.Accounts = append(witness.Accounts, *acc)
    }

    // 收集存储证明
    for addr, slots := range w.storage {
        for slot, value := range slots {
            proof, err := stateDB.GetStorageProof(addr, slot)
            if err != nil {
                return nil, err
            }
            witness.Storage = append(witness.Storage, StorageWitness{
                Address:   addr,
                Slot:      slot,
                Value:     value,
                ProofPath: proof,
            })
        }
    }

    // 收集代码
    for hash, code := range w.codes {
        witness.Codes = append(witness.Codes, CodeWitness{
            Hash: hash,
            Code: code,
        })
    }

    return witness, nil
}
```

### Z.3 Verkle Witness (未来)

```go
// Verkle Trees 下的 witness 更加紧凑
type VerkleWitness struct {
    // 多项式承诺
    Commitments []VerkleCommitment
    // 开放证明 (单个证明可验证多个值)
    Proof       *ipa.MultiProof
    // 访问的键值对
    Keys        [][]byte
    Values      [][]byte
}

// Verkle witness 大小对比
// MPT witness:  ~800 KB/block (平均)
// Verkle witness: ~150 KB/block (平均)
// 压缩比: ~5x
```

---

## 附录 AA：Portal Network

### AA.1 Portal Network 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Portal Network 架构                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   设计目标: 去中心化轻客户端数据访问                              │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Portal Network                         │   │
│   │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────┐   │   │
│   │  │ History     │ │ Beacon      │ │ State           │   │   │
│   │  │ Network     │ │ Network     │ │ Network         │   │   │
│   │  │             │ │             │ │                 │   │   │
│   │  │ - 区块头    │ │ - 信标链数据│ │ - 状态数据      │   │   │
│   │  │ - 区块体    │ │ - 同步委员会│ │ - 状态证明      │   │   │
│   │  │ - 收据      │ │ - 轻客户端  │ │                 │   │   │
│   │  │             │ │   更新      │ │                 │   │   │
│   │  └──────┬──────┘ └──────┬──────┘ └────────┬────────┘   │   │
│   │         │               │                  │             │   │
│   │         └───────────────┼──────────────────┘             │   │
│   │                         │                                 │   │
│   │                         ▼                                 │   │
│   │  ┌─────────────────────────────────────────────────────┐│   │
│   │  │              Discovery v5 (DHT)                      ││   │
│   │  │  - 基于内容寻址                                      ││   │
│   │  │  - 数据按距离存储                                    ││   │
│   │  │  - 激励兼容                                          ││   │
│   │  └─────────────────────────────────────────────────────┘│   │
│   └─────────────────────────────────────────────────────────────┘
│                                                                  │
│   节点参与                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   Full Node ─────▶ 提供数据到 Portal Network            │   │
│   │                                                          │   │
│   │   Portal Node ◀───▶ 存储部分数据，响应查询              │   │
│   │                                                          │   │
│   │   Light Client ◀── 从 Portal Network 获取数据           │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### AA.2 内容寻址

```go
// Portal Network 内容 ID 计算
type ContentKey struct {
    NetworkID byte
    Key       []byte
}

// History Network 内容键
const (
    HistoryBlockHeader      byte = 0x00
    HistoryBlockBody        byte = 0x01
    HistoryBlockReceipts    byte = 0x02
    HistoryEpochAccumulator byte = 0x03
)

func HistoryBlockHeaderKey(blockHash common.Hash) ContentKey {
    return ContentKey{
        NetworkID: HistoryBlockHeader,
        Key:       blockHash.Bytes(),
    }
}

// 内容 ID = keccak256(content_key)
func (k ContentKey) ContentID() common.Hash {
    encoded := append([]byte{k.NetworkID}, k.Key...)
    return crypto.Keccak256Hash(encoded)
}

// 数据存储在 XOR 距离最近的节点
func XORDistance(a, b common.Hash) *big.Int {
    result := new(big.Int)
    for i := 0; i < 32; i++ {
        result.SetBit(result, i*8, uint(a[i]^b[i]))
    }
    return result
}

// 查找负责存储内容的节点
func (n *PortalNode) FindContentProviders(contentID common.Hash) []*Node {
    // 使用 discv5 递归查找
    return n.discv5.FindNodes(contentID)
}
```

---

## 附录 AB：SSZ 编码

### AB.1 SSZ 基础

```go
// Simple Serialize (SSZ) - 以太坊共识层序列化格式

// 与 RLP 对比
// RLP: 灵活，自描述，可变长度
// SSZ: 固定模式，高效 Merkle 化，适合零知识证明

// 基本类型序列化
type SSZEncoder struct{}

// uint64 编码 (小端序，固定 8 字节)
func (e *SSZEncoder) EncodeUint64(v uint64) []byte {
    buf := make([]byte, 8)
    binary.LittleEndian.PutUint64(buf, v)
    return buf
}

// 布尔值 (1 字节)
func (e *SSZEncoder) EncodeBool(v bool) []byte {
    if v {
        return []byte{0x01}
    }
    return []byte{0x00}
}

// 固定大小数组
func (e *SSZEncoder) EncodeFixedBytes(v []byte, size int) []byte {
    result := make([]byte, size)
    copy(result, v)
    return result
}

// 可变长度列表 (使用偏移量)
func (e *SSZEncoder) EncodeList(items [][]byte) []byte {
    // 计算固定部分大小 (偏移量)
    fixedSize := len(items) * 4

    // 构建偏移量和数据
    var result []byte
    var variableParts []byte
    currentOffset := uint32(fixedSize)

    for _, item := range items {
        // 写入偏移量
        offsetBytes := make([]byte, 4)
        binary.LittleEndian.PutUint32(offsetBytes, currentOffset)
        result = append(result, offsetBytes...)

        // 累积可变部分
        variableParts = append(variableParts, item...)
        currentOffset += uint32(len(item))
    }

    return append(result, variableParts...)
}
```

### AB.2 SSZ Merkle 化

```go
// SSZ 哈希树根计算
func HashTreeRoot(obj interface{}) common.Hash {
    chunks := Pack(obj)
    return Merkleize(chunks)
}

// 打包成 32 字节块
func Pack(obj interface{}) [][]byte {
    switch v := obj.(type) {
    case uint64:
        chunk := make([]byte, 32)
        binary.LittleEndian.PutUint64(chunk, v)
        return [][]byte{chunk}
    case []byte:
        return packBytes(v)
    case []uint64:
        return packUint64s(v)
    default:
        panic("unsupported type")
    }
}

func packBytes(data []byte) [][]byte {
    // 填充到 32 字节边界
    padded := make([]byte, ((len(data)+31)/32)*32)
    copy(padded, data)

    var chunks [][]byte
    for i := 0; i < len(padded); i += 32 {
        chunks = append(chunks, padded[i:i+32])
    }
    return chunks
}

// Merkle 化
func Merkleize(chunks [][]byte) common.Hash {
    // 填充到 2 的幂次
    n := nextPowerOfTwo(len(chunks))
    for len(chunks) < n {
        chunks = append(chunks, zeroHash[:])
    }

    // 构建 Merkle 树
    for len(chunks) > 1 {
        var nextLevel [][]byte
        for i := 0; i < len(chunks); i += 2 {
            combined := append(chunks[i], chunks[i+1]...)
            hash := sha256.Sum256(combined)
            nextLevel = append(nextLevel, hash[:])
        }
        chunks = nextLevel
    }

    return common.BytesToHash(chunks[0])
}

// 混合长度 (用于可变长度类型)
func MixInLength(root common.Hash, length uint64) common.Hash {
    lengthBytes := make([]byte, 32)
    binary.LittleEndian.PutUint64(lengthBytes, length)
    combined := append(root.Bytes(), lengthBytes...)
    hash := sha256.Sum256(combined)
    return common.BytesToHash(hash[:])
}
```

### AB.3 执行层 SSZ 提案

```go
// EIP-6493: 执行层 SSZ 化
// 将交易、区块等从 RLP 迁移到 SSZ

type SSZTransaction struct {
    ChainID       uint64
    Nonce         uint64
    MaxPriorityFee uint64
    MaxFee        uint64
    Gas           uint64
    To            *common.Address // Optional
    Value         *uint256.Int
    Data          []byte
    AccessList    []AccessTuple
    // EIP-4844
    MaxFeePerBlobGas uint64
    BlobVersionedHashes []common.Hash
}

// SSZ 执行负载 (用于共识层)
type SSZExecutionPayload struct {
    ParentHash    common.Hash
    FeeRecipient  common.Address
    StateRoot     common.Hash
    ReceiptsRoot  common.Hash
    LogsBloom     [256]byte
    PrevRandao    common.Hash
    BlockNumber   uint64
    GasLimit      uint64
    GasUsed       uint64
    Timestamp     uint64
    ExtraData     []byte   // max 32 bytes
    BaseFeePerGas *uint256.Int
    BlockHash     common.Hash
    Transactions  [][]byte // SSZ encoded transactions
    Withdrawals   []Withdrawal
    BlobGasUsed   uint64
    ExcessBlobGas uint64
}
```

---

## 附录 AC：MEV-Boost 详解

### AC.1 MEV-Boost 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEV-Boost 完整架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────┐                                           │
│   │    Searcher     │  发现 MEV 机会                            │
│   │  ┌───────────┐  │  - 套利                                   │
│   │  │ Bot/Script│  │  - 清算                                   │
│   │  └─────┬─────┘  │  - 三明治攻击                             │
│   └────────┼────────┘                                           │
│            │ 提交 Bundle                                         │
│            ▼                                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Block Builder                         │   │
│   │  ┌─────────────────────────────────────────────────┐    │   │
│   │  │  1. 收集 Bundles                                 │    │   │
│   │  │  2. 模拟执行，计算收益                           │    │   │
│   │  │  3. 排序优化，构建最优区块                       │    │   │
│   │  │  4. 计算出价金额                                 │    │   │
│   │  └─────────────────────────────────────────────────┘    │   │
│   └────────────────────────┬────────────────────────────────┘   │
│                            │ 提交 Bid + Block Header             │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                      Relay                               │   │
│   │  ┌─────────────────────────────────────────────────┐    │   │
│   │  │  - 验证 Builder 提交的区块                       │    │   │
│   │  │  - 托管区块内容 (防止 MEV 窃取)                  │    │   │
│   │  │  - 向 Proposer 展示出价                         │    │   │
│   │  │  - 收到签名后释放区块                           │    │   │
│   │  └─────────────────────────────────────────────────┘    │   │
│   │  常见 Relay: Flashbots, bloXroute, Ultrasound           │   │
│   └────────────────────────┬────────────────────────────────┘   │
│                            │ getHeader / getPayload              │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    MEV-Boost                             │   │
│   │  ┌─────────────────────────────────────────────────┐    │   │
│   │  │  - 本地运行的 sidecar                            │    │   │
│   │  │  - 连接多个 Relay                               │    │   │
│   │  │  - 选择最高出价                                 │    │   │
│   │  │  - 实现 Builder API                             │    │   │
│   │  └─────────────────────────────────────────────────┘    │   │
│   └────────────────────────┬────────────────────────────────┘   │
│                            │ Builder API                         │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                Consensus Client                          │   │
│   │  ┌─────────────────────────────────────────────────┐    │   │
│   │  │  Validator:                                      │    │   │
│   │  │  1. registerValidator - 注册接收 MEV              │    │   │
│   │  │  2. getHeader - 获取最佳出价                     │    │   │
│   │  │  3. 签名 blinded block                          │    │   │
│   │  │  4. getPayload - 获取完整区块                    │    │   │
│   │  │  5. 广播区块                                     │    │   │
│   │  └─────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### AC.2 Builder API

```go
// Builder API 接口定义
type BuilderAPI interface {
    // 验证者注册
    RegisterValidator(registrations []SignedValidatorRegistration) error

    // 获取区块头 (出价)
    GetHeader(slot uint64, parentHash common.Hash, pubkey BLSPubkey) (*SignedBuilderBid, error)

    // 获取完整区块
    GetPayload(signedBlindedBlock *SignedBlindedBeaconBlock) (*ExecutionPayload, error)

    // 提交区块 (Builder 使用)
    SubmitBlock(block *SignedBuilderBlock) error
}

// 验证者注册消息
type ValidatorRegistration struct {
    FeeRecipient common.Address
    GasLimit     uint64
    Timestamp    uint64
    Pubkey       BLSPubkey
}

// Builder 出价
type BuilderBid struct {
    Header *ExecutionPayloadHeader
    Value  *uint256.Int // Wei
    Pubkey BLSPubkey    // Builder's pubkey
}

// Blinded Block (不含交易内容)
type BlindedBeaconBlock struct {
    Slot          uint64
    ProposerIndex uint64
    ParentRoot    common.Hash
    StateRoot     common.Hash
    Body          *BlindedBeaconBlockBody
}

type BlindedBeaconBlockBody struct {
    // ... 其他字段
    ExecutionPayloadHeader *ExecutionPayloadHeader // 只有 header，没有 transactions
}
```

### AC.3 MEV-Boost 实现

```go
// MEV-Boost sidecar 实现
type MEVBoost struct {
    relays       []Relay
    registeredValidators map[BLSPubkey]*ValidatorRegistration
}

func (m *MEVBoost) GetHeader(slot uint64, parentHash common.Hash, pubkey BLSPubkey) (*SignedBuilderBid, error) {
    var bestBid *SignedBuilderBid
    var bestValue = big.NewInt(0)

    // 并行查询所有 relay
    results := make(chan *SignedBuilderBid, len(m.relays))
    for _, relay := range m.relays {
        go func(r Relay) {
            bid, err := r.GetHeader(slot, parentHash, pubkey)
            if err == nil {
                results <- bid
            } else {
                results <- nil
            }
        }(relay)
    }

    // 收集结果，选择最高出价
    for i := 0; i < len(m.relays); i++ {
        bid := <-results
        if bid != nil && bid.Message.Value.ToBig().Cmp(bestValue) > 0 {
            bestBid = bid
            bestValue = bid.Message.Value.ToBig()
        }
    }

    return bestBid, nil
}

func (m *MEVBoost) GetPayload(signedBlock *SignedBlindedBeaconBlock) (*ExecutionPayload, error) {
    // 从对应的 relay 获取完整 payload
    blockHash := signedBlock.Message.Body.ExecutionPayloadHeader.BlockHash

    for _, relay := range m.relays {
        payload, err := relay.GetPayload(signedBlock)
        if err == nil && payload.BlockHash == blockHash {
            return payload, nil
        }
    }

    return nil, errors.New("payload not found")
}
```

---

## 附录 AD：Hive 测试框架

### AD.1 Hive 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hive 测试框架架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   目的: 以太坊客户端互操作性测试                                     │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Hive Controller                      │   │
│   │  ┌─────────────────────────────────────────────────┐    │   │
│   │  │  - 管理 Docker 容器                              │    │   │
│   │  │  - 协调测试执行                                   │    │   │
│   │  │  - 收集结果                                      │    │   │
│   │  └─────────────────────────────────────────────────┘    │   │
│   └────────────────────────┬────────────────────────────────┘   │
│                            │                                    │
│            ┌───────────────┼───────────────┐                    │
│            │               │               │                    │
│            ▼               ▼               ▼                    │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│   │   Geth      │ │   Besu      │ │  Nethermind │               │
│   │  Container  │ │  Container  │ │  Container  │               │
│   └─────────────┘ └─────────────┘ └─────────────┘               │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│   │   Erigon    │ │    Reth     │ │   evmone    │               │
│   │  Container  │ │  Container  │ │  Container  │               │
│   └─────────────┘ └─────────────┘ └─────────────┘               │
│                                                                 │
│   测试类型                                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  1. devp2p    - P2P 协议测试                             │   │
│   │  2. eth       - eth 协议消息                             │   │
│   │  3. engine    - Engine API 测试                         │   │
│   │  4. sync      - 同步测试                                 │   │
│   │  5. rpc       - JSON-RPC 兼容性                          │   │
│   │  6. graphql   - GraphQL API                             │   │
│   │  7. evm       - EVM 执行一致性                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### AD.2 Hive 测试示例

```go
// Hive 测试套件示例
package main

import (
    "github.com/ethereum/hive/hivesim"
)

func main() {
    suite := hivesim.Suite{
        Name:        "eth-consensus",
        Description: "Ethereum execution layer consensus tests",
    }

    // 添加测试
    suite.Add(hivesim.TestSpec{
        Name:        "state-test",
        Description: "Run official state tests",
        Run:         runStateTests,
    })

    suite.Add(hivesim.TestSpec{
        Name:        "blockchain-test",
        Description: "Run official blockchain tests",
        Run:         runBlockchainTests,
    })

    hivesim.MustRunSuite(suite)
}

func runStateTests(t *hivesim.T) {
    // 获取测试客户端
    clients := t.Sim.ClientTypes()

    for _, clientType := range clients {
        t.Run(clientType.Name, func(t *hivesim.T) {
            // 启动客户端
            client := t.Sim.StartClient(clientType.Name, hivesim.Params{
                "HIVE_NETWORK_ID": "1337",
            })
            defer client.Shutdown()

            // 运行测试
            for _, test := range getStateTests() {
                result := executeStateTest(client, test)
                if !result.Pass {
                    t.Fatalf("State test failed: %s", result.Error)
                }
            }
        })
    }
}

func runBlockchainTests(t *hivesim.T) {
    clients := t.Sim.ClientTypes()

    for _, clientType := range clients {
        t.Run(clientType.Name, func(t *hivesim.T) {
            client := t.Sim.StartClient(clientType.Name, hivesim.Params{})
            defer client.Shutdown()

            // 导入区块并验证
            for _, block := range testBlocks {
                err := importBlock(client, block)
                if err != nil {
                    t.Fatalf("Block import failed: %v", err)
                }

                // 验证状态
                state, err := getState(client)
                if err != nil {
                    t.Fatalf("Failed to get state: %v", err)
                }

                if state.Root != block.ExpectedStateRoot {
                    t.Fatalf("State root mismatch: got %s, want %s",
                        state.Root, block.ExpectedStateRoot)
                }
            }
        })
    }
}
```

### AD.3 客户端 Docker 配置

```dockerfile
# Hive 客户端 Dockerfile 示例 (Geth)
FROM golang:1.21-alpine as builder

RUN apk add --no-cache gcc musl-dev linux-headers git

WORKDIR /go-ethereum
RUN git clone https://github.com/ethereum/go-ethereum.git .
RUN go build -o /geth ./cmd/geth

FROM alpine:latest
RUN apk add --no-cache ca-certificates
COPY --from=builder /geth /usr/local/bin/

# Hive 入口脚本
COPY hive-entrypoint.sh /

ENTRYPOINT ["/hive-entrypoint.sh"]
```

```bash
#!/bin/bash
# hive-entrypoint.sh

# 从环境变量读取配置
NETWORK_ID=${HIVE_NETWORK_ID:-1}
CHAIN_CONFIG=${HIVE_CHAIN_CONFIG:-""}

# 初始化创世区块
if [ -n "$HIVE_GENESIS" ]; then
    echo "$HIVE_GENESIS" > /genesis.json
    geth init /genesis.json
fi

# 启动 geth
exec geth \
    --networkid $NETWORK_ID \
    --http \
    --http.addr 0.0.0.0 \
    --http.api eth,net,web3,debug,engine \
    --authrpc.addr 0.0.0.0 \
    --authrpc.jwtsecret /jwtsecret \
    $@
```

### AD.4 测试结果分析

```go
// Hive 测试结果结构
type TestResult struct {
    Suite       string
    Test        string
    Client      string
    Pass        bool
    Details     string
    Duration    time.Duration
}

// 生成兼容性矩阵
func GenerateCompatibilityMatrix(results []TestResult) {
    matrix := make(map[string]map[string]bool)

    for _, r := range results {
        if matrix[r.Test] == nil {
            matrix[r.Test] = make(map[string]bool)
        }
        matrix[r.Test][r.Client] = r.Pass
    }

    // 输出矩阵
    fmt.Println("Test Compatibility Matrix:")
    fmt.Println("==========================")
    for test, clients := range matrix {
        fmt.Printf("%s:\n", test)
        for client, pass := range clients {
            status := "✓"
            if !pass {
                status = "✗"
            }
            fmt.Printf("  %s: %s\n", client, status)
        }
    }
}
```

---

## 参考资料

- [evmone GitHub](https://github.com/ethereum/evmone)
- [EVMC 标准](https://github.com/ethereum/evmc)
- [EVM 性能报告: Geth vs evmone](https://notes.ethereum.org/@ipsilon/evm-performance-report-geth-vs-evmone)
- [Go-ethereum EVM 优化报告](https://chfast.github.io/Go-ethereum-EVM-optimization-report/)
- [EOF 规范](https://eips.ethereum.org/EIPS/eip-3540)
- [EOF 官网](https://evmobjectformat.xyz/)
- [EIP-7692 EOF Meta](https://eips.ethereum.org/EIPS/eip-7692)
- [intx 库](https://github.com/chfast/intx)
- [uint256 库](https://github.com/holiman/uint256)
- [evm-bench 基准测试](https://github.com/ziyadedher/evm-bench)
- [evmone-compiler](https://github.com/megaeth-labs/evmone-compiler)
- [execution-spec-tests](https://github.com/ethereum/execution-spec-tests)
- [EVMFuzz 论文](https://onlinelibrary.wiley.com/doi/abs/10.1002/smr.2556)
- [Verkle Trees EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)
- [zkEVM 类型分类](https://vitalik.eth.limo/general/2022/08/04/zkevm.html)
- [Arbitrum Stylus](https://docs.arbitrum.io/stylus/stylus-gentle-introduction)
- [OP Stack Specs](https://specs.optimism.io/)
- [EIP-1153 Transient Storage](https://eips.ethereum.org/EIPS/eip-1153)
- [EIP-5656 MCOPY](https://eips.ethereum.org/EIPS/eip-5656)
- [ERC-4337 账户抽象](https://eips.ethereum.org/EIPS/eip-4337)
- [EIP-7702 EOA 代码设置](https://eips.ethereum.org/EIPS/eip-7702)
- [KEVM](https://github.com/runtimeverification/evm-semantics)
- [K Framework](https://kframework.org/)
- [Act 语言](https://github.com/ethereum/act)
- [Snap Sync 文档](https://geth.ethereum.org/docs/fundamentals/sync-modes)
- [devp2p 协议](https://github.com/ethereum/devp2p)
- [Engine API](https://github.com/ethereum/execution-apis/tree/main/src/engine)
- [LevelDB](https://github.com/google/leveldb)
- [Pebble](https://github.com/cockroachdb/pebble)
- [MDBX](https://github.com/erthink/libmdbx)
- [Flashbots MEV](https://docs.flashbots.net/)
- [PBS (Proposer-Builder Separation)](https://ethereum.org/en/roadmap/pbs/)
