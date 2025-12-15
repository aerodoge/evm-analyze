# Solidity 内联汇编完全指南

## 目录

1. [概述](#1-概述)
2. [Yul 语言基础](#2-yul-语言基础)
3. [内存模型](#3-内存模型)
4. [存储模型](#4-存储模型)
5. [Calldata 操作](#5-calldata-操作)
6. [EVM 操作码参考](#6-evm-操作码参考)
7. [常见模式与最佳实践](#7-常见模式与最佳实践)
8. [Gas 优化技巧](#8-gas-优化技巧)
9. [安全注意事项](#9-安全注意事项)
10. [实战案例](#10-实战案例)

---

## 1. 概述

### 1.1 什么是内联汇编

Solidity内联汇编允许在合约中直接编写接近EVM操作码的低级代码。它使用一种叫做**Yul**的中间语言。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AssemblyIntro {
    function add(uint256 a, uint256 b) external pure returns (uint256 result) {
        // 使用 assembly 块进入汇编模式
        assembly {
            result := add(a, b)
        }
    }
}
```

### 1.2 为什么使用汇编

| 场景         | 说明                                    |
|------------|---------------------------------------|
| **Gas 优化** | 绕过Solidity的安全检查，减少不必要的操作              |
| **访问底层功能** | 直接使用EVM操作码（如`create2`, `extcodecopy`） |
| **内存精确控制** | 手动管理内存布局，避免Solidity的自动管理开销            |
| **实现特殊逻辑** | 如代理合约的`delegatecall`转发                |
| **突破语言限制** | 实现Solidity无法表达的操作                     |

### 1.3 汇编的风险

```
⚠️ 警告：汇编代码绕过 Solidity 的安全检查

- 无溢出保护（Solidity 0.8+ 的自动检查不适用）
- 无类型检查（所有值都是256位）
- 无边界检查（数组越界不会报错）
- 内存管理需要手动处理
- 代码更难审计和维护
```

---

## 2. Yul 语言基础

### 2.1 基本语法

```solidity
contract YulBasics {
    function basics() external pure returns (uint256) {
        assembly {
            // ========== 字面量 ==========
            let a := 100              // 十进制
            let b := 0xff             // 十六进制
            let c := true             // 布尔值 (1)
            let d := false            // 布尔值 (0)

            // ========== 变量声明 ==========
            let x := 0                // 声明并初始化
            let y                     // 声明，默认值为 0
            x := add(x, 1)           // 赋值

            // ========== 注释 ==========
            // 单行注释
            /* 多行注释 */

            // ========== 返回值 ==========
            mstore(0x00, a)
            return (0x00, 32)
        }
    }
}
```

### 2.2 算术运算

```solidity
contract ArithmeticOps {
    function arithmetic(uint256 a, uint256 b) external pure returns (
        uint256 sum,
        uint256 diff,
        uint256 prod,
        uint256 quot,
        uint256 remain
    ) {
        assembly {
            // 加法
            sum := add(a, b)

            // 减法（如果 a < b，会下溢环绕）
            diff := sub(a, b)

            // 乘法
            prod := mul(a, b)

            // 除法（除以 0 返回 0，不会 revert）
            quot := div(a, b)

            // 取模
            remain := mod(a, b)

            // 有符号运算
            // sdiv(a, b)  - 有符号除法
            // smod(a, b)  - 有符号取模

            // 幂运算
            // exp(base, exponent)

            // 加法取模（避免溢出）
            // addmod(a, b, N) = (a + b) % N
            // mulmod(a, b, N) = (a * b) % N
        }
    }
}
```

### 2.3 比较运算

```solidity
contract ComparisonOps {
    function compare(uint256 a, uint256 b) external pure returns (
        bool isLess,
        bool isGreater,
        bool isEqual,
        bool isZero
    ) {
        assembly {
            // 小于（返回1或0）
            isLess := lt(a, b)

            // 大于
            isGreater := gt(a, b)

            // 等于
            isEqual := eq(a, b)

            // 是否为零
            isZero := iszero(a)

            // 有符号比较
            // slt(a, b) - 有符号小于
            // sgt(a, b) - 有符号大于
        }
    }
}
```

### 2.4 位运算

```solidity
contract BitwiseOps {
    function bitwise(uint256 a, uint256 b) external pure returns (
        uint256 andResult,
        uint256 orResult,
        uint256 xorResult,
        uint256 notResult,
        uint256 shlResult,
        uint256 shrResult
    ) {
        assembly {
            // 按位与
            andResult := and(a, b)

            // 按位或
            orResult := or(a, b)

            // 按位异或
            xorResult := xor(a, b)

            // 按位取反
            notResult := not(a)

            // 左移：shl(shift, value) = value << shift
            shlResult := shl(8, a)      // a << 8

            // 右移：shr(shift, value) = value >> shift
            shrResult := shr(8, a)      // a >> 8

            // 有符号右移（算术右移，保留符号位）
            // sar(shift, value)

            // 提取单个字节
            // byte(n, x) - 从左数第 n 个字节（0-31）
            let firstByte := byte(0, a)   // 最高字节
            let lastByte := byte(31, a)   // 最低字节
        }
    }
}
```

### 2.5 控制流

```solidity
contract ControlFlow {
    // ========== if 语句 ==========
    function ifStatement(uint256 x) external pure returns (uint256 result) {
        assembly {
            // if 条件为非零则执行
            // 注意：没有 else！
            if gt(x, 10) {
                result := 1
            }

            // 模拟 if-else：使用两个 if
            if gt(x, 10) {
                result := 1
            }
            if iszero(gt(x, 10)) {
                result := 0
            }
        }
    }

    // ========== switch 语句 ==========
    function switchStatement(uint256 x) external pure returns (uint256 result) {
        assembly {
            switch x
            case 0 {
                result := 100
            }
            case 1 {
                result := 200
            }
            case 2 {
                result := 300
            }
            default {
                result := 999
            }
            // 注意：没有 fall-through，不需要 break
        }
    }

    // ========== for 循环 ==========
    function forLoop(uint256 n) external pure returns (uint256 sum) {
        assembly {
            // for { 初始化 } 条件 { 后处理 } { 循环体 }
            for {let i := 0} lt(i, n) {i := add(i, 1)} {
                sum := add(sum, i)
            }
        }
    }

    // ========== while 循环（用 for 模拟）==========
    function whileLoop(uint256 n) external pure returns (uint256 result) {
        assembly {
            let i := n
            // while (i > 0) 的等价形式
            for {} gt(i, 0) {} {
                result := add(result, i)
                i := sub(i, 1)
            }
        }
    }

    // ========== break 和 continue ==========
    function breakContinue(uint256 n) external pure returns (uint256 sum) {
        assembly {
            for {let i := 0} lt(i, n) {i := add(i, 1)} {
                // continue：跳过偶数
                if iszero(mod(i, 2)) {
                    continue
                }

                sum := add(sum, i)

                // break：超过 100 就停止
                if gt(sum, 100) {
                    break
                }
            }
        }
    }
}
```

### 2.6 函数定义

```solidity
contract YulFunctions {
    function demonstrateFunctions(uint256 a, uint256 b) external pure returns (uint256) {
        assembly {
            // ========== 定义函数 ==========
            // function 函数名(参数) -> 返回值 { 函数体 }

            // 无返回值函数
            function log(value) {
                // 可以调用其他 Yul 内置函数
                mstore(0, value)
            }

            // 单返回值函数
            function double(x) -> result {
                result := mul(x, 2)
            }

            // 多返回值函数
            function divmod(x, y) -> q, r {
                q := div(x, y)
                r := mod(x, y)
            }

            // 递归函数
            function factorial(n) -> result {
                switch n
                case 0 {result := 1}
                case 1 {result := 1}
                default {
                    result := mul(n, factorial(sub(n, 1)))
                }
            }

            // ========== 调用函数 ==========
            let doubled := double(a)
            let q, r := divmod(a, b)
            let fact := factorial(5)

            // 返回结果
            mstore(0, add(doubled, fact))
            return (0, 32)
        }
    }
}
```

---

## 3. 内存模型

### 3.1 内存布局

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Solidity 内存布局                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   0x00 - 0x3f  (64 bytes)   Scratch Space（临时空间）                 │
│                              用于哈希等操作的临时存储                   │
│                                                                     │
│   0x40 - 0x5f  (32 bytes)   Free Memory Pointer（空闲内存指针）        │
│                              指向下一个可用的内存位置                   │
│                                                                     │
│   0x60 - 0x7f  (32 bytes)   Zero Slot（零槽）                        │
│                              用于动态数组的初始值                      │
│                                                                     │
│   0x80 - ...                 Free Memory（可用内存）                  │
│                              Solidity 分配内存从这里开始               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 内存操作

```solidity
contract MemoryOperations {
    function memoryBasics() external pure returns (bytes32 result) {
        assembly {
            // ========== 读取内存 ==========
            // mload(offset) - 从 offset 读取 32 字节
            let freePtr := mload(0x40)  // 读取空闲内存指针

            // ========== 写入内存 ==========
            // mstore(offset, value) - 在 offset 写入 32 字节
            mstore(freePtr, 0x1234)

            // mstore8(offset, value) - 在 offset 写入 1 字节
            mstore8(add(freePtr, 32), 0xff)

            // ========== 更新空闲指针 ==========
            // 使用内存后必须更新！
            mstore(0x40, add(freePtr, 64))

            // ========== 内存大小 ==========
            // msize() - 返回已访问的最高内存地址
            let memSize := msize()

            result := mload(freePtr)
        }
    }

    // ========== 使用 Scratch Space ==========
    function useScratchSpace(bytes32 a, bytes32 b) external pure returns (bytes32) {
        assembly {
            // Scratch space 可以自由使用，但是临时的
            // 任何 Solidity 操作都可能覆盖它
            mstore(0x00, a)
            mstore(0x20, b)
            let hash := keccak256(0x00, 0x40)
            mstore(0x00, hash)
            return (0x00, 32)
        }
    }
}
```

### 3.3 动态内存分配

```solidity
contract DynamicMemory {
    // 安全的内存分配函数
    function allocate(uint256 size) internal pure returns (uint256 ptr) {
        assembly {
            // 获取当前空闲指针
            ptr := mload(0x40)

            // 更新空闲指针（32 字节对齐）
            let newPtr := add(ptr, and(add(size, 31), not(31)))
            mstore(0x40, newPtr)
        }
    }

    function createArray(uint256 length) external pure returns (bytes memory) {
        assembly {
            // 分配内存：32 字节长度 + length * 32 字节数据
            let ptr := mload(0x40)
            let size := add(32, mul(length, 32))

            // 存储长度
            mstore(ptr, length)

            // 初始化数据（可选）
            for {let i := 0} lt(i, length) {i := add(i, 1)} {
                mstore(add(add(ptr, 32), mul(i, 32)), i)
            }

            // 更新空闲指针
            mstore(0x40, add(ptr, size))

            // 返回内存指针
            return (ptr, size)
        }
    }
}
```

### 3.4 内存复制

```solidity
contract MemoryCopy {
    // 手动内存复制
    function copyMemory(
        uint256 src,
        uint256 dest,
        uint256 length
    ) internal pure {
        assembly {
            // 按 32 字节块复制
            for {let i := 0} lt(i, length) {i := add(i, 32)} {
                mstore(add(dest, i), mload(add(src, i)))
            }
        }
    }

    // Cancun 升级后的 MCOPY（更高效）
    function copyMemoryMcopy(
        uint256 src,
        uint256 dest,
        uint256 length
    ) internal pure {
        assembly {
            // mcopy(dest, src, length) - EIP-5656
            // 需要 Cancun 升级后才能使用
            // mcopy(dest, src, length)
        }
    }

    // 复制 bytes 数据
    function cloneBytes(bytes memory data) external pure returns (bytes memory result) {
        assembly {
            let length := mload(data)
            let totalSize := add(32, length)

            // 分配新内存
            result := mload(0x40)
            mstore(0x40, add(result, and(add(totalSize, 31), not(31))))

            // 复制长度
            mstore(result, length)

            // 复制数据
            let src := add(data, 32)
            let dest := add(result, 32)

            for {let i := 0} lt(i, length) {i := add(i, 32)} {
                mstore(add(dest, i), mload(add(src, i)))
            }
        }
    }
}
```

---

## 4. 存储模型

### 4.1 存储布局

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Solidity 存储布局                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   每个存储槽 = 32 字节 (256 bits)                                     │
│   共有 2^256 个槽位可用                                               │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ Slot 0:  第一个状态变量                                       │   │
│   │ Slot 1:  第二个状态变量                                       │   │
│   │ ...                                                         │   │
│   │ Slot N:  mapping/动态数组的基础槽位                            │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   紧凑打包：小于 32 字节的变量会被打包到同一个槽位                         │
│                                                                     │
│   contract Example {                                                │
│       uint128 a;  // slot 0 低 128 位                                │
│       uint128 b;  // slot 0 高 128 位                                │
│       uint256 c;  // slot 1 (完整 256 位)                            │
│   }                                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 基本存储操作

```solidity
contract StorageOperations {
    uint256 public slot0;     // slot 0
    uint256 public slot1;     // slot 1
    address public slot2;     // slot 2

    function storageBasics(uint256 value) external {
        assembly {
            // ========== 读取存储 ==========
            // sload(slot) - 读取指定槽位
            let current := sload(0)

            // ========== 写入存储 ==========
            // sstore(slot, value) - 写入指定槽位
            sstore(0, value)

            // 读取后写入
            let old := sload(1)
            sstore(1, add(old, 1))
        }
    }

    // 获取变量的存储槽位
    function getSlot() external pure returns (uint256 s0, uint256 s1, uint256 s2) {
        assembly {
            // 使用 .slot 获取变量的存储槽位
            s0 := slot0.slot
            s1 := slot1.slot
            s2 := slot2.slot
        }
    }
}
```

### 4.3 紧凑存储的读写

```solidity
contract PackedStorage {
    // 打包存储：slot 0
    // [  8 bits  ][  8 bits  ][ 160 bits ][ 80 bits ]
    // [  status  ][  flags   ][  owner   ][ amount  ]

    uint8 public status;     // slot 0, offset 0
    uint8 public flags;      // slot 0, offset 8
    address public owner;    // slot 0, offset 16
    uint80 public amount;    // slot 0, offset 176

    function readPackedValues() external view returns (
        uint8 _status,
        uint8 _flags,
        address _owner,
        uint80 _amount
    ) {
        assembly {
            let packed := sload(0)

            // 提取各字段（从右到左）
            _amount := and(packed, 0xffffffffffffffffffff)  // 低 80 位

            _owner := and(shr(80, packed), 0xffffffffffffffffffffffffffffffffffffffff)  // 160 位

            _flags := and(shr(240, packed), 0xff)  // 8 位

            _status := shr(248, packed)  // 最高 8 位
        }
    }

    function writePackedValues(
        uint8 _status,
        uint8 _flags,
        address _owner,
        uint80 _amount
    ) external {
        assembly {
            // 构建打包值
            let packed := or(
                or(
                    shl(248, _status),
                    shl(240, _flags)
                ),
                or(
                    shl(80, _owner),
                    _amount
                )
            )
            sstore(0, packed)
        }
    }

    // 只更新单个字段
    function updateStatus(uint8 newStatus) external {
        assembly {
            let packed := sload(0)

            // 清除旧值
            packed := and(packed, not(shl(248, 0xff)))

            // 设置新值
            packed := or(packed, shl(248, newStatus))

            sstore(0, packed)
        }
    }
}
```

### 4.4 Mapping 存储计算

```solidity
contract MappingStorage {
    mapping(address => uint256) public balances;           // slot 0
    mapping(address => mapping(address => uint256)) public allowances;  // slot 1

    function getBalanceSlot(address account) external pure returns (bytes32 slot) {
        assembly {
            // mapping 的值存储在 keccak256(key . baseSlot)
            // 这里 "." 表示拼接

            // 将 key 和 baseSlot 写入内存
            mstore(0x00, account)
            mstore(0x20, 0)  // balances 的 baseSlot 是 0

            // 计算存储槽
            slot := keccak256(0x00, 0x40)
        }
    }

    function readBalance(address account) external view returns (uint256 balance) {
        assembly {
            mstore(0x00, account)
            mstore(0x20, 0)
            let slot := keccak256(0x00, 0x40)
            balance := sload(slot)
        }
    }

    function writeBalance(address account, uint256 value) external {
        assembly {
            mstore(0x00, account)
            mstore(0x20, 0)
            let slot := keccak256(0x00, 0x40)
            sstore(slot, value)
        }
    }

    // 嵌套 mapping
    function getAllowanceSlot(address owner, address spender) external pure returns (bytes32 slot) {
        assembly {
            // 第一层：keccak256(owner . baseSlot)
            mstore(0x00, owner)
            mstore(0x20, 1)  // allowances 的 baseSlot 是 1
            let firstLevel := keccak256(0x00, 0x40)

            // 第二层：keccak256(spender . firstLevel)
            mstore(0x00, spender)
            mstore(0x20, firstLevel)
            slot := keccak256(0x00, 0x40)
        }
    }
}
```

### 4.5 动态数组存储

```solidity
contract DynamicArrayStorage {
    uint256[] public arr;  // slot 0

    function getArrayLength() external view returns (uint256 length) {
        assembly {
            // 数组长度存储在 baseSlot
            length := sload(0)
        }
    }

    function getElementSlot(uint256 index) external pure returns (bytes32 slot) {
        assembly {
            // 元素存储在 keccak256(baseSlot) + index
            mstore(0x00, 0)  // baseSlot
            let dataStart := keccak256(0x00, 0x20)
            slot := add(dataStart, index)
        }
    }

    function readElement(uint256 index) external view returns (uint256 value) {
        assembly {
            // 检查边界
            let length := sload(0)
            if iszero(lt(index, length)) {
                revert(0, 0)
            }

            // 计算元素槽位
            mstore(0x00, 0)
            let dataStart := keccak256(0x00, 0x20)
            value := sload(add(dataStart, index))
        }
    }

    function pushElement(uint256 value) external {
        assembly {
            // 获取当前长度
            let length := sload(0)

            // 计算新元素的槽位
            mstore(0x00, 0)
            let dataStart := keccak256(0x00, 0x20)

            // 写入新元素
            sstore(add(dataStart, length), value)

            // 更新长度
            sstore(0, add(length, 1))
        }
    }
}
```

---

## 5. Calldata 操作

### 5.1 Calldata 结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Calldata 结构                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────────┬──────────────────────────────────────────────┐   │
│   │  0-3 bytes   │           4+ bytes                           │   │
│   │  Function    │           ABI 编码的参数                       │   │
│   │  Selector    │                                              │   │
│   └──────────────┴──────────────────────────────────────────────┘   │
│                                                                     │
│   示例: transfer(address to, uint256 amount)                         │
│                                                                     │
│   Byte 0-3:    0xa9059cbb  (函数选择器)                              │
│   Byte 4-35:   to 地址 (32 字节，左填充)                               │
│   Byte 36-67:  amount 金额 (32 字节)                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 读取 Calldata

```solidity
contract CalldataOperations {
    function readCalldata() external pure returns (
        bytes4 selector,
        uint256 firstArg,
        uint256 dataSize
    ) {
        assembly {
            // 获取 calldata 大小
            dataSize := calldatasize()

            // 读取 calldata
            // calldataload(offset) - 从 offset 读取 32 字节
            let word := calldataload(0)

            // 提取函数选择器（前 4 字节）
            selector := shr(224, word)

            // 读取第一个参数（跳过 4 字节选择器）
            firstArg := calldataload(4)
        }
    }

    // 读取动态类型参数
    function readDynamicData(bytes calldata data) external pure returns (
        uint256 offset,
        uint256 length,
        bytes32 firstWord
    ) {
        assembly {
            // 对于 bytes calldata data：
            // data.offset - 数据在 calldata 中的偏移
            // data.length - 数据长度

            offset := data.offset
            length := data.length

            // 读取数据的第一个 32 字节
            if gt(length, 0) {
                firstWord := calldataload(data.offset)
            }
        }
    }

    // 复制 calldata 到内存
    function copyCalldataToMemory() external pure returns (bytes memory) {
        assembly {
            let size := calldatasize()
            let ptr := mload(0x40)

            // 存储长度
            mstore(ptr, size)

            // calldatacopy(destOffset, offset, size)
            calldatacopy(add(ptr, 32), 0, size)

            // 更新空闲指针
            mstore(0x40, add(add(ptr, 32), and(add(size, 31), not(31))))

            return (ptr, add(32, size))
        }
    }
}
```

### 5.3 Calldata 与 Memory 的选择

```solidity
contract CalldataVsMemory {
    // ✅ 使用 calldata - 更节省 gas
    function processCalldata(bytes calldata data) external pure returns (bytes32) {
        assembly {
            // 直接从 calldata 读取，无需复制
            mstore(0, calldataload(data.offset))
            return (0, 32)
        }
    }

    // ❌ 使用 memory - 需要复制，消耗更多 gas
    function processMemory(bytes memory data) external pure returns (bytes32) {
        assembly {
            // data 已经在内存中
            mstore(0, mload(add(data, 32)))
            return (0, 32)
        }
    }
}
```

---

## 6. EVM 操作码参考

### 6.1 环境信息

```solidity
contract EnvironmentOps {
    function getEnvironmentInfo() external view returns (
        address _caller,
        address _origin,
        address _this,
        uint256 _value,
        uint256 _gasPrice,
        uint256 _gasLeft,
        uint256 _blockNumber,
        uint256 _timestamp,
        address _coinbase,
        uint256 _difficulty,
        uint256 _gasLimit,
        uint256 _chainId,
        uint256 _selfBalance
    ) {
        assembly {
            _caller := caller()           // msg.sender
            _origin := origin()           // tx.origin
            _this := address()            // address(this)
            _value := callvalue()         // msg.value
            _gasPrice := gasprice()       // tx.gasprice
            _gasLeft := gas()             // 剩余 gas
            _blockNumber := number()      // block.number
            _timestamp := timestamp()     // block.timestamp
            _coinbase := coinbase()       // block.coinbase
            _difficulty := prevrandao()   // block.prevrandao (原 difficulty)
            _gasLimit := gaslimit()       // block.gaslimit
            _chainId := chainid()         // block.chainid
            _selfBalance := selfbalance() // address(this).balance
        }
    }

    // 获取外部账户/合约信息
    function getExternalInfo(address target) external view returns (
        uint256 balance,
        uint256 codeSize,
        bytes32 codeHash
    ) {
        assembly {
            balance := balance(target)        // 账户余额
            codeSize := extcodesize(target)   // 代码大小
            codeHash := extcodehash(target)   // 代码哈希
        }
    }

    // 复制外部合约代码
    function copyExternalCode(address target) external view returns (bytes memory code) {
        assembly {
            let size := extcodesize(target)
            code := mload(0x40)
            mstore(code, size)

            // extcodecopy(addr, destOffset, offset, size)
            extcodecopy(target, add(code, 32), 0, size)

            mstore(0x40, add(add(code, 32), and(add(size, 31), not(31))))
        }
    }
}
```

### 6.2 外部调用

```solidity
contract ExternalCalls {
    // ========== call ==========
    function makeCall(
        address target,
        bytes memory data
    ) external payable returns (bool success, bytes memory returnData) {
        assembly {
            // call(gas, addr, value, argsOffset, argsSize, retOffset, retSize)
            success := call(
                gas(),                // 转发所有 gas
                target,               // 目标地址
                callvalue(),          // 转发所有 ETH
                add(data, 32),        // 输入数据位置（跳过长度）
                mload(data),          // 输入数据长度
                0,                    // 输出位置（先不指定）
                0                     // 输出长度（先不指定）
            )

            // 获取返回数据
            let retSize := returndatasize()
            returnData := mload(0x40)
            mstore(returnData, retSize)
            returndatacopy(add(returnData, 32), 0, retSize)
            mstore(0x40, add(add(returnData, 32), and(add(retSize, 31), not(31))))
        }
    }

    // ========== staticcall (只读调用) ==========
    function makeStaticCall(
        address target,
        bytes memory data
    ) external view returns (bool success, bytes memory returnData) {
        assembly {
            // staticcall(gas, addr, argsOffset, argsSize, retOffset, retSize)
            success := staticcall(
                gas(),
                target,
                add(data, 32),
                mload(data),
                0,
                0
            )

            let retSize := returndatasize()
            returnData := mload(0x40)
            mstore(returnData, retSize)
            returndatacopy(add(returnData, 32), 0, retSize)
            mstore(0x40, add(add(returnData, 32), and(add(retSize, 31), not(31))))
        }
    }

    // ========== delegatecall ==========
    function makeDelegateCall(
        address target,
        bytes memory data
    ) external returns (bool success, bytes memory returnData) {
        assembly {
            // delegatecall(gas, addr, argsOffset, argsSize, retOffset, retSize)
            // 注意：在目标合约的上下文中执行，但使用当前合约的存储
            success := delegatecall(
                gas(),
                target,
                add(data, 32),
                mload(data),
                0,
                0
            )

            let retSize := returndatasize()
            returnData := mload(0x40)
            mstore(returnData, retSize)
            returndatacopy(add(returnData, 32), 0, retSize)
            mstore(0x40, add(add(returnData, 32), and(add(retSize, 31), not(31))))
        }
    }
}
```

### 6.3 合约创建与销毁

```solidity
contract ContractLifecycle {
    // ========== create ==========
    function deployCreate(bytes memory bytecode) external payable returns (address deployed) {
        assembly {
            // create(value, offset, size)
            deployed := create(
                callvalue(),          // 发送的 ETH
                add(bytecode, 32),    // 字节码位置
                mload(bytecode)       // 字节码长度
            )

            // 检查是否成功
            if iszero(extcodesize(deployed)) {
                revert(0, 0)
            }
        }
    }

    // ========== create2 ==========
    function deployCreate2(
        bytes memory bytecode,
        bytes32 salt
    ) external payable returns (address deployed) {
        assembly {
            // create2(value, offset, size, salt)
            deployed := create2(
                callvalue(),
                add(bytecode, 32),
                mload(bytecode),
                salt
            )

            if iszero(extcodesize(deployed)) {
                revert(0, 0)
            }
        }
    }

    // ========== selfdestruct ==========
    function destroy(address payable recipient) external {
        assembly {
            // selfdestruct(recipient)
            // 警告：Cancun 后行为改变，不再删除代码
            selfdestruct(recipient)
        }
    }
}
```

### 6.4 日志操作

```solidity
contract LogOperations {
    function emitLogs() external {
        assembly {
            let ptr := mload(0x40)

            // ========== log0 (无 topic) ==========
            mstore(ptr, 0x1234)
            log0(ptr, 32)

            // ========== log1 (1 个 topic) ==========
            let topic1 := 0xaabbccdd00000000000000000000000000000000000000000000000000000000
            mstore(ptr, 0x5678)
            log1(ptr, 32, topic1)

            // ========== log2 (2 个 topics) ==========
            let topic2 := 0x1122334400000000000000000000000000000000000000000000000000000000
            mstore(ptr, 0x9abc)
            log2(ptr, 32, topic1, topic2)

            // ========== log3 (3 个 topics) ==========
            let topic3 := 0x5566778800000000000000000000000000000000000000000000000000000000
            log3(ptr, 32, topic1, topic2, topic3)

            // ========== log4 (4 个 topics) ==========
            let topic4 := 0x99aabbcc00000000000000000000000000000000000000000000000000000000
            log4(ptr, 32, topic1, topic2, topic3, topic4)
        }
    }

    // 模拟 Transfer 事件
    event Transfer(address indexed from, address indexed to, uint256 value);

    function emitTransfer(address from, address to, uint256 value) external {
        assembly {
            // Transfer(address,address,uint256) 的签名哈希
            let signature := 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef

            // indexed 参数作为 topic
            let topic1 := from
            let topic2 := to

            // 非 indexed 参数作为 data
            mstore(0, value)
            log3(0, 32, signature, topic1, topic2)
        }
    }
}
```

### 6.5 哈希与加密

```solidity
contract HashOperations {
    function hashFunctions(bytes memory data) external view returns (
        bytes32 keccakHash,
        bytes32 sha256Hash
    ) {
        assembly {
            // keccak256
            keccakHash := keccak256(add(data, 32), mload(data))

            // SHA256 (预编译合约调用)
            // 地址 0x02
            let ptr := mload(0x40)
            let success := staticcall(
                gas(),
                0x02,                 // SHA256 预编译地址
                add(data, 32),        // 输入
                mload(data),          // 输入长度
                ptr,                  // 输出位置
                32                    // 输出长度
            )
            sha256Hash := mload(ptr)
        }
    }

    // 高效的双值哈希
    function hash2(bytes32 a, bytes32 b) external pure returns (bytes32 result) {
        assembly {
            mstore(0x00, a)
            mstore(0x20, b)
            result := keccak256(0x00, 0x40)
        }
    }

    // ecrecover
    function recoverSigner(
        bytes32 hash,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external view returns (address signer) {
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, hash)
            mstore(add(ptr, 32), v)
            mstore(add(ptr, 64), r)
            mstore(add(ptr, 96), s)

            // 调用 ecrecover 预编译 (地址 0x01)
            let success := staticcall(
                gas(),
                0x01,
                ptr,
                128,
                ptr,
                32
            )

            if success {
                signer := mload(ptr)
            }
        }
    }
}
```

---

## 7. 常见模式与最佳实践

### 7.1 代理合约转发

```solidity
contract MinimalProxy {
    address immutable implementation;

    constructor(address _implementation) {
        implementation = _implementation;
    }

    fallback() external payable {
        assembly {
            // 复制 calldata
            calldatacopy(0, 0, calldatasize())

            // delegatecall 到实现合约
            let result := delegatecall(
                gas(),
                sload(implementation.slot),
                0,
                calldatasize(),
                0,
                0
            )

            // 复制返回数据
            returndatacopy(0, 0, returndatasize())

            // 根据结果返回或回滚
            switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return (0, returndatasize())
            }
        }
    }
}
```

### 7.2 安全的低级调用

```solidity
contract SafeCall {
    error CallFailed(bytes data);

    function safeCall(
        address target,
        bytes memory data
    ) external payable returns (bytes memory result) {
        bool success;
        assembly {
            success := call(
                gas(),
                target,
                callvalue(),
                add(data, 32),
                mload(data),
                0,
                0
            )

            let retSize := returndatasize()
            result := mload(0x40)
            mstore(result, retSize)
            returndatacopy(add(result, 32), 0, retSize)
            mstore(0x40, add(add(result, 32), and(add(retSize, 31), not(31))))
        }

        if (!success) {
            // 使用 Solidity 处理错误
            assembly {
                revert(add(result, 32), mload(result))
            }
        }
    }

    // 带 gas 限制的调用
    function callWithGasLimit(
        address target,
        bytes memory data,
        uint256 gasLimit
    ) external returns (bool success) {
        assembly {
            success := call(
                gasLimit,             // 限制 gas
                target,
                0,
                add(data, 32),
                mload(data),
                0,
                0
            )
        }
    }
}
```

### 7.3 高效的数据编码

```solidity
contract EfficientEncoding {
    // 手动编码 transfer(address,uint256) 调用
    function encodeTransfer(
        address to,
        uint256 amount
    ) external pure returns (bytes memory) {
        assembly {
            let ptr := mload(0x40)

            // 函数选择器
            mstore(ptr, 0xa9059cbb00000000000000000000000000000000000000000000000000000000)

            // 参数
            mstore(add(ptr, 4), to)
            mstore(add(ptr, 36), amount)

            // 更新空闲指针并返回
            mstore(0x40, add(ptr, 68))

            // 返回编码后的数据
            return (ptr, 68)
        }
    }

    // 高效的多参数编码
    function encodeMultiple(
        uint256 a,
        address b,
        bool c
    ) external pure returns (bytes memory result) {
        assembly {
            result := mload(0x40)
            mstore(result, 96)  // 长度

            mstore(add(result, 32), a)
            mstore(add(result, 64), b)
            mstore(add(result, 96), c)

            mstore(0x40, add(result, 128))
        }
    }
}
```

### 7.4 批量操作

```solidity
contract BatchOperations {
    // 批量读取余额
    function batchBalanceOf(
        address token,
        address[] calldata accounts
    ) external view returns (uint256[] memory balances) {
        assembly {
            let length := accounts.length

            // 分配返回数组
            balances := mload(0x40)
            mstore(balances, length)
            let dataPtr := add(balances, 32)

            // 准备 balanceOf 调用
            let callPtr := add(dataPtr, mul(length, 32))
            mstore(callPtr, 0x70a0823100000000000000000000000000000000000000000000000000000000)

            // 遍历查询
            for {let i := 0} lt(i, length) {i := add(i, 1)} {
                // 设置查询地址
                let account := calldataload(add(accounts.offset, mul(i, 32)))
                mstore(add(callPtr, 4), account)

                // 调用
                let success := staticcall(gas(), token, callPtr, 36, dataPtr, 32)

                if iszero(success) {
                    mstore(dataPtr, 0)
                }

                dataPtr := add(dataPtr, 32)
            }

            // 更新空闲指针
            mstore(0x40, dataPtr)
        }
    }
}
```

---

## 8. Gas 优化技巧

### 8.1 存储优化

```solidity
contract StorageOptimization {
    // ❌ 低效：多次 sload/sstore
    function inefficient(uint256 value) external {
        uint256 slot0 = 0; // 假设这是存储变量
        assembly {
            let current := sload(0)
            sstore(0, add(current, 1))
            current := sload(0)  // 重复读取
            sstore(0, add(current, value))
        }
    }

    // ✅ 高效：缓存存储值
    function efficient(uint256 value) external {
        assembly {
            let current := sload(0)
            current := add(current, 1)
            current := add(current, value)
            sstore(0, current)  // 只写入一次
        }
    }

    // 利用冷/热存储访问
    // 首次访问（冷）：2100 gas
    // 后续访问（热）：100 gas
    function accessPattern() external {
        assembly {
            // 第一次访问 slot 0（冷）：2100 gas
            let val := sload(0)

            // 同一交易中再次访问（热）：100 gas
            val := sload(0)
        }
    }
}
```

### 8.2 内存优化

```solidity
contract MemoryOptimization {
    // ✅ 使用 scratch space 进行临时计算
    function useScrachSpace(bytes32 a, bytes32 b) external pure returns (bytes32) {
        assembly {
            mstore(0x00, a)
            mstore(0x20, b)
            mstore(0x00, keccak256(0x00, 0x40))
            return (0x00, 0x20)
        }
    }

    // ✅ 避免不必要的内存扩展
    function avoidMemoryExpansion() external pure returns (uint256) {
        assembly {
            // 内存扩展是二次成本
            // 尽量重用低地址内存
            mstore(0x00, 123)
            return (0x00, 32)
        }
    }

    // ❌ 低效：过度使用内存
    function inefficientMemory(uint256 n) external pure returns (uint256 sum) {
        assembly {
            let ptr := mload(0x40)
            for {let i := 0} lt(i, n) {i := add(i, 1)} {
                mstore(add(ptr, mul(i, 32)), i)  // 线性扩展内存
            }
            // 计算...
        }
    }

    // ✅ 高效：原地计算
    function efficientCompute(uint256 n) external pure returns (uint256 sum) {
        assembly {
            for {let i := 0} lt(i, n) {i := add(i, 1)} {
                sum := add(sum, i)  // 不需要存储中间值
            }
        }
    }
}
```

### 8.3 短路计算

```solidity
contract ShortCircuit {
    // ✅ 使用 if 代替 require 组合条件
    function efficientCheck(uint256 a, uint256 b) external pure returns (bool) {
        assembly {
            // 短路：第一个条件失败就停止
            if iszero(gt(a, 0)) {
                revert(0, 0)
            }
            if iszero(lt(b, 100)) {
                revert(0, 0)
            }
            mstore(0, 1)
            return (0, 32)
        }
    }

    // ✅ 常量比较放前面
    function orderMatters(uint256 value, address addr) external view returns (bool) {
        assembly {
            // 先检查便宜的常量比较
            if iszero(gt(value, 0)) {
                mstore(0, 0)
                return (0, 32)
            }

            // 再检查昂贵的外部调用
            if iszero(extcodesize(addr)) {
                mstore(0, 0)
                return (0, 32)
            }

            mstore(0, 1)
            return (0, 32)
        }
    }
}
```

### 8.4 位操作技巧

```solidity
contract BitTricks {
    // 检查是否为 2 的幂
    function isPowerOfTwo(uint256 x) external pure returns (bool result) {
        assembly {
            // x & (x - 1) == 0 且 x != 0
            result := and(gt(x, 0), iszero(and(x, sub(x, 1))))
        }
    }

    // 快速除以 2 的幂
    function divByPowerOfTwo(uint256 x, uint256 shift) external pure returns (uint256) {
        assembly {
            mstore(0, shr(shift, x))
            return (0, 32)
        }
    }

    // 快速乘以 2 的幂
    function mulByPowerOfTwo(uint256 x, uint256 shift) external pure returns (uint256) {
        assembly {
            mstore(0, shl(shift, x))
            return (0, 32)
        }
    }

    // 获取最低有效位
    function lowestBit(uint256 x) external pure returns (uint256) {
        assembly {
            // x & (-x) 或 x & (x ^ (x - 1))
            mstore(0, and(x, sub(0, x)))
            return (0, 32)
        }
    }

    // 计算 1 的个数（popcount）
    function popcount(uint256 x) external pure returns (uint256 count) {
        assembly {
            for {} gt(x, 0) {} {
                count := add(count, and(x, 1))
                x := shr(1, x)
            }
        }
    }

    // 清除最低有效位
    function clearLowestBit(uint256 x) external pure returns (uint256) {
        assembly {
            mstore(0, and(x, sub(x, 1)))
            return (0, 32)
        }
    }
}
```

---

## 9. 安全注意事项

### 9.1 常见陷阱

```solidity
contract SecurityPitfalls {
    // ========== 陷阱 1: 除以零不会 revert ==========
    function divisionByZero(uint256 a, uint256 b) external pure returns (uint256) {
        assembly {
            // 危险：b = 0 时返回 0，不会 revert
            mstore(0, div(a, b))
            return (0, 32)
        }
    }

    // ✅ 安全版本
    function safeDivision(uint256 a, uint256 b) external pure returns (uint256) {
        assembly {
            if iszero(b) {
                revert(0, 0)
            }
            mstore(0, div(a, b))
            return (0, 32)
        }
    }

    // ========== 陷阱 2: 溢出没有检查 ==========
    function overflow(uint256 a, uint256 b) external pure returns (uint256) {
        assembly {
            // 危险：可能溢出
            mstore(0, add(a, b))
            return (0, 32)
        }
    }

    // ✅ 安全版本
    function safeAdd(uint256 a, uint256 b) external pure returns (uint256 result) {
        assembly {
            result := add(a, b)
            if lt(result, a) {
                revert(0, 0)
            }
        }
    }

    // ========== 陷阱 3: call 返回值未检查 ==========
    function uncheckedCall(address target) external {
        assembly {
            // 危险：忽略返回值
            pop(call(gas(), target, 0, 0, 0, 0, 0))
        }
    }

    // ✅ 安全版本
    function checkedCall(address target) external {
        assembly {
            let success := call(gas(), target, 0, 0, 0, 0, 0)
            if iszero(success) {
                returndatacopy(0, 0, returndatasize())
                revert(0, returndatasize())
            }
        }
    }

    // ========== 陷阱 4: 内存指针未更新 ==========
    function memoryLeak() external pure returns (bytes memory) {
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, 100)
            // 危险：没有更新 0x40！
            // 后续 Solidity 代码会覆盖这块内存
            return (ptr, 32)
        }
    }

    // ✅ 安全版本
    function safeMemory() external pure returns (bytes memory) {
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, 32)          // 长度
            mstore(add(ptr, 32), 100) // 数据
            mstore(0x40, add(ptr, 64)) // 更新指针！
            return (ptr, 64)
        }
    }
}
```

### 9.2 外部调用安全

```solidity
contract ExternalCallSafety {
    // 安全的 ETH 转账
    function safeTransferETH(address to, uint256 amount) internal {
        assembly {
            // 使用 call 而非 transfer/send
            // 转发足够的 gas
            let success := call(gas(), to, amount, 0, 0, 0, 0)

            if iszero(success) {
                // 检查是否是因为余额不足
                if gt(amount, selfbalance()) {
                    // 自定义错误：InsufficientBalance()
                    mstore(0, 0xf4d678b8)
                    revert(0, 4)
                }
                // 其他原因失败
                revert(0, 0)
            }
        }
    }

    // 安全的 ERC20 转账（处理非标准返回值）
    function safeTransferERC20(
        address token,
        address to,
        uint256 amount
    ) internal {
        assembly {
            let ptr := mload(0x40)

            // transfer(address,uint256)
            mstore(ptr, 0xa9059cbb00000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 4), to)
            mstore(add(ptr, 36), amount)

            let success := call(gas(), token, 0, ptr, 68, ptr, 32)

            // 检查调用是否成功
            if iszero(success) {
                revert(0, 0)
            }

            // 检查返回值（如果有）
            // 某些代币不返回值，某些返回 true
            let retSize := returndatasize()
            if gt(retSize, 0) {
                returndatacopy(ptr, 0, 32)
                if iszero(mload(ptr)) {
                    revert(0, 0)
                }
            }
        }
    }
}
```

### 9.3 重入保护

```solidity
contract ReentrancyGuard {
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;
    uint256 private _status;

    constructor() {
        _status = _NOT_ENTERED;
    }

    modifier nonReentrant() {
        assembly {
            // 检查是否已进入
            if eq(sload(_status.slot), _ENTERED) {
                // ReentrancyGuardReentrantCall()
                mstore(0, 0x3ee5aeb5)
                revert(0, 4)
            }

            // 设置为已进入
            sstore(_status.slot, _ENTERED)
        }

        _;

        assembly {
            // 重置状态
            sstore(_status.slot, _NOT_ENTERED)
        }
    }
}
```

### 9.4 安全检查清单

```solidity
/**
 * ╔═══════════════════════════════════════════════════════════════════╗
 * ║                    汇编代码安全检查清单                              ║
 * ╠═══════════════════════════════════════════════════════════════════╣
 * ║                                                                   ║
 * ║  □ 数值安全                                                        ║
 * ║    ├── 检查除法除数不为零                                           ║
 * ║    ├── 检查加法/乘法溢出                                            ║
 * ║    ├── 检查减法下溢                                                ║
 * ║    └── 注意有符号/无符号运算差异                                     ║
 * ║                                                                   ║
 * ║  □ 内存安全                                                        ║
 * ║    ├── 使用 mload(0x40) 获取空闲指针                                ║
 * ║    ├── 使用后更新空闲指针                                           ║
 * ║    ├── 注意 scratch space 会被覆盖                                 ║
 * ║    └── 避免读取未初始化的内存                                        ║
 * ║                                                                   ║
 * ║  □ 存储安全                                                        ║
 * ║    ├── 正确计算 mapping/array 槽位                                 ║
 * ║    ├── 不要硬编码槽位号                                             ║
 * ║    └── 使用 .slot 获取变量槽位                                      ║
 * ║                                                                   ║
 * ║  □ 外部调用                                                        ║
 * ║    ├── 始终检查 call/delegatecall 返回值                           ║
 * ║    ├── 正确处理 returndatasize                                     ║
 * ║    ├── 考虑 gas 限制（防 DoS）                                     ║
 * ║    └── 注意重入风险                                                ║
 * ║                                                                    ║
 * ║  □ 类型安全                                                        ║
 * ║    ├── Yul 中所有值都是 256 位                                     ║
 * ║    ├── 手动进行类型边界检查                                        ║
 * ║    ├── 地址需要清除高位：and(x, 0xff...ff)                         ║
 * ║    └── 布尔值应为 0 或 1                                           ║
 * ║                                                                    ║
 * ╚═══════════════════════════════════════════════════════════════════╝
 */
```

---

## 10. 实战案例

### 10.1 高效的 ERC20 实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EfficientERC20 {
    // 存储布局
    // slot 0: totalSupply
    // slot 1: balanceOf mapping base
    // slot 2: allowance mapping base
    // slot 3-4: name (动态)
    // slot 5-6: symbol (动态)
    // slot 7: decimals

    uint256 private _totalSupply;
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(string memory _name, string memory _symbol, uint256 initialSupply) {
        name = _name;
        symbol = _symbol;
        _mint(msg.sender, initialSupply);
    }

    function totalSupply() external view returns (uint256) {
        return _totalSupply;
    }

    // 高效的 balanceOf
    function balanceOf(address account) external view returns (uint256 balance) {
        assembly {
            // 计算 slot: keccak256(account . 1)
            mstore(0x00, account)
            mstore(0x20, 1)
            balance := sload(keccak256(0x00, 0x40))
        }
    }

    // 高效的 transfer
    function transfer(address to, uint256 amount) external returns (bool) {
        assembly {
            // 获取 caller
            let from := caller()

            // 计算 from 的 balance slot
            mstore(0x00, from)
            mstore(0x20, 1)
            let fromSlot := keccak256(0x00, 0x40)
            let fromBalance := sload(fromSlot)

            // 检查余额
            if lt(fromBalance, amount) {
                // InsufficientBalance(address,uint256,uint256)
                mstore(0x00, 0xf4d678b8)
                mstore(0x04, from)
                mstore(0x24, amount)
                mstore(0x44, fromBalance)
                revert(0x00, 0x64)
            }

            // 更新 from 余额
            sstore(fromSlot, sub(fromBalance, amount))

            // 计算 to 的 balance slot
            mstore(0x00, to)
            mstore(0x20, 1)
            let toSlot := keccak256(0x00, 0x40)

            // 更新 to 余额
            sstore(toSlot, add(sload(toSlot), amount))

            // 发出 Transfer 事件
            mstore(0x00, amount)
            log3(
                0x00,
                0x20,
            // Transfer(address,address,uint256)
                0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef,
                from,
                to
            )

            // 返回 true
            mstore(0x00, 1)
            return (0x00, 0x20)
        }
    }

    // 高效的 approve
    function approve(address spender, uint256 amount) external returns (bool) {
        assembly {
            let owner := caller()

            // 计算 allowance slot
            // keccak256(spender . keccak256(owner . 2))
            mstore(0x00, owner)
            mstore(0x20, 2)
            let innerHash := keccak256(0x00, 0x40)

            mstore(0x00, spender)
            mstore(0x20, innerHash)
            let slot := keccak256(0x00, 0x40)

            // 设置 allowance
            sstore(slot, amount)

            // 发出 Approval 事件
            mstore(0x00, amount)
            log3(
                0x00,
                0x20,
            // Approval(address,address,uint256)
                0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925,
                owner,
                spender
            )

            mstore(0x00, 1)
            return (0x00, 0x20)
        }
    }

    function allowance(address owner, address spender) external view returns (uint256 result) {
        assembly {
            mstore(0x00, owner)
            mstore(0x20, 2)
            let innerHash := keccak256(0x00, 0x40)

            mstore(0x00, spender)
            mstore(0x20, innerHash)
            result := sload(keccak256(0x00, 0x40))
        }
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        assembly {
            let spender := caller()

            // 检查 allowance
            mstore(0x00, from)
            mstore(0x20, 2)
            let innerHash := keccak256(0x00, 0x40)

            mstore(0x00, spender)
            mstore(0x20, innerHash)
            let allowanceSlot := keccak256(0x00, 0x40)
            let currentAllowance := sload(allowanceSlot)

            // 如果不是无限授权，检查并扣减
            if lt(currentAllowance, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff) {
                if lt(currentAllowance, amount) {
                    revert(0, 0)
                }
                sstore(allowanceSlot, sub(currentAllowance, amount))
            }

            // from 余额
            mstore(0x00, from)
            mstore(0x20, 1)
            let fromSlot := keccak256(0x00, 0x40)
            let fromBalance := sload(fromSlot)

            if lt(fromBalance, amount) {
                revert(0, 0)
            }

            sstore(fromSlot, sub(fromBalance, amount))

            // to 余额
            mstore(0x00, to)
            mstore(0x20, 1)
            let toSlot := keccak256(0x00, 0x40)
            sstore(toSlot, add(sload(toSlot), amount))

            // Transfer 事件
            mstore(0x00, amount)
            log3(
                0x00,
                0x20,
                0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef,
                from,
                to
            )

            mstore(0x00, 1)
            return (0x00, 0x20)
        }
    }

    function _mint(address to, uint256 amount) internal {
        assembly {
            // 更新总供应量
            let supply := sload(0)
            sstore(0, add(supply, amount))

            // 更新余额
            mstore(0x00, to)
            mstore(0x20, 1)
            let slot := keccak256(0x00, 0x40)
            sstore(slot, add(sload(slot), amount))

            // Transfer 事件 (from = address(0))
            mstore(0x00, amount)
            log3(
                0x00,
                0x20,
                0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef,
                0,
                to
            )
        }
    }
}
```

### 10.2 最小代理工厂

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MinimalProxyFactory {
    event ProxyDeployed(address proxy, address implementation);

    /**
     * @notice 部署 EIP-1167 最小代理
     * @dev 代理字节码：
     *      363d3d373d3d3d363d73<impl>5af43d82803e903d91602b57fd5bf3
     */
    function deployProxy(address implementation) external returns (address proxy) {
        assembly {
            // 获取空闲内存
            let ptr := mload(0x40)

            // 存储创建代码
            // 前 10 字节：部署代码
            mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)

            // 实现地址（20 字节）
            mstore(add(ptr, 0x14), shl(0x60, implementation))

            // 后 15 字节：运行时代码后缀
            mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf3000000000000000000000000000000000000)

            // 部署
            proxy := create(0, ptr, 0x37)

            if iszero(proxy) {
                revert(0, 0)
            }
        }

        emit ProxyDeployed(proxy, implementation);
    }

    /**
     * @notice 使用 CREATE2 部署确定性代理
     */
    function deployProxyDeterministic(
        address implementation,
        bytes32 salt
    ) external returns (address proxy) {
        assembly {
            let ptr := mload(0x40)

            mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
            mstore(add(ptr, 0x14), shl(0x60, implementation))
            mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf3000000000000000000000000000000000000)

            proxy := create2(0, ptr, 0x37, salt)

            if iszero(proxy) {
                revert(0, 0)
            }
        }

        emit ProxyDeployed(proxy, implementation);
    }

    /**
     * @notice 预测 CREATE2 地址
     */
    function predictProxyAddress(
        address implementation,
        bytes32 salt
    ) external view returns (address predicted) {
        assembly {
            let ptr := mload(0x40)

            // 构建 init code
            mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
            mstore(add(ptr, 0x14), shl(0x60, implementation))
            mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf3000000000000000000000000000000000000)

            // 计算 init code hash
            let initCodeHash := keccak256(ptr, 0x37)

            // CREATE2 地址计算
            // keccak256(0xff ++ factory ++ salt ++ initCodeHash)
            mstore(ptr, 0xff00000000000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 1), shl(0x60, address()))
            mstore(add(ptr, 21), salt)
            mstore(add(ptr, 53), initCodeHash)

            predicted := and(
                keccak256(ptr, 85),
                0xffffffffffffffffffffffffffffffffffffffff
            )
        }
    }
}
```

### 10.3 高效的签名验证

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EfficientSignature {
    // EIP-712 域分隔符
    bytes32 public immutable DOMAIN_SEPARATOR;

    // Permit typehash
    bytes32 public constant PERMIT_TYPEHASH =
    keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");

    mapping(address => uint256) public nonces;

    constructor(string memory name, string memory version) {
        DOMAIN_SEPARATOR = _buildDomainSeparator(name, version);
    }

    function _buildDomainSeparator(
        string memory name,
        string memory version
    ) internal view returns (bytes32) {
        bytes32 result;
        assembly {
            let ptr := mload(0x40)

            // EIP712Domain typehash
            mstore(ptr, 0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f)

            // keccak256(name)
            let nameLen := mload(name)
            mstore(add(ptr, 32), keccak256(add(name, 32), nameLen))

            // keccak256(version)
            let versionLen := mload(version)
            mstore(add(ptr, 64), keccak256(add(version, 32), versionLen))

            // chainId
            mstore(add(ptr, 96), chainid())

            // verifyingContract
            mstore(add(ptr, 128), address())

            result := keccak256(ptr, 160)
        }
        return result;
    }

    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        assembly {
            // 检查 deadline
            if gt(timestamp(), deadline) {
                // ExpiredSignature()
                mstore(0, 0x0819bdcd)
                revert(0, 4)
            }

            // 获取 nonce
            mstore(0, owner)
            mstore(32, 0) // nonces slot
            let nonceSlot := keccak256(0, 64)
            let nonce := sload(nonceSlot)

            // 构建 struct hash
            let ptr := mload(0x40)
            mstore(ptr, 0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9) // PERMIT_TYPEHASH
            mstore(add(ptr, 32), owner)
            mstore(add(ptr, 64), spender)
            mstore(add(ptr, 96), value)
            mstore(add(ptr, 128), nonce)
            mstore(add(ptr, 160), deadline)
            let structHash := keccak256(ptr, 192)

            // 构建 digest
            mstore(ptr, 0x1901000000000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 2), sload(DOMAIN_SEPARATOR.slot))
            mstore(add(ptr, 34), structHash)
            let digest := keccak256(ptr, 66)

            // ecrecover
            mstore(ptr, digest)
            mstore(add(ptr, 32), v)
            mstore(add(ptr, 64), r)
            mstore(add(ptr, 96), s)

            let success := staticcall(gas(), 1, ptr, 128, ptr, 32)
            let recovered := mload(ptr)

            // 验证签名
            if or(iszero(success), iszero(eq(recovered, owner))) {
                // InvalidSignature()
                mstore(0, 0x8baa579f)
                revert(0, 4)
            }

            // 增加 nonce
            sstore(nonceSlot, add(nonce, 1))

            // 设置 allowance（简化版，实际需要正确的 slot 计算）
            // ... allowance 逻辑
        }
    }
}
```

---

## 参考资料

- [Solidity 官方文档 - 内联汇编](https://docs.soliditylang.org/en/latest/assembly.html)
- [Yul 语言规范](https://docs.soliditylang.org/en/latest/yul.html)
- [EVM 操作码参考](https://www.evm.codes/)
- [EIP-1167 最小代理](https://eips.ethereum.org/EIPS/eip-1167)
- [EIP-712 类型化数据签名](https://eips.ethereum.org/EIPS/eip-712)
- [Solmate - Gas 优化库](https://github.com/transmissions11/solmate)
- [Solady - 高度优化库](https://github.com/Vectorized/solady)
