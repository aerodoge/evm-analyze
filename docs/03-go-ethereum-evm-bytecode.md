# Go-Ethereum EVM 字节码篇

> 本文档深入讲解 EVM 字节码的编码、解码、反编译和分析技术。

## 目录

- [21. 字节码基础](#21-字节码基础)
- [22. 合约部署流程](#22-合约部署流程)
- [23. 函数选择器与 ABI 编码](#23-函数选择器与-abi-编码)
- [24. 字节码反编译](#24-字节码反编译)
- [25. 常见字节码模式](#25-常见字节码模式)
- [26. 字节码分析工具](#26-字节码分析工具)

---

## 21. 字节码基础

### 21.1 字节码结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    合约字节码结构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  部署时的交易 data:                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  ┌───────────────────┐ ┌──────────────────────────────┐ │  │
│  │  │   Creation Code   │ │   Constructor Arguments      │ │  │
│  │  │   (初始化代码)      │ │   (构造函数参数, ABI编码)      │ │  │
│  │  └─────────┬─────────┘ └──────────────────────────────┘ │  │
│  │            │                                             │  │
│  │            ▼                                             │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │  Creation Code 包含:                                 │ │  │
│  │  │  1. 初始化逻辑 (执行构造函数)                           │ │  │
│  │  │  2. Runtime Code (要部署的代码)                       │ │  │
│  │  │  3. CODECOPY + RETURN 将 runtime code 返回           │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  部署后存储在链上的:                                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │              Runtime Code (运行时代码)             │  │  │
│  │  │  • 函数分发逻辑                                    │  │  │
│  │  │  • 所有函数的实现                                  │  │  │
│  │  │  • 元数据哈希 (可选)                               │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 21.2 Creation Code 示例

```solidity
// 简单合约
contract Simple {
    uint256 public value;

    constructor(uint256 _value) {
        value = _value;
    }

    function setValue(uint256 _value) external {
        value = _value;
    }
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    Creation Code 解析                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Creation Code (伪代码):                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 1. 免费内存指针初始化                                  │  │
│  │  PUSH1 0x80                                              │  │
│  │  PUSH1 0x40                                              │  │
│  │  MSTORE          // memory[0x40] = 0x80                  │  │
│  │                                                          │  │
│  │  // 2. 检查 msg.value (非 payable 构造函数)               │  │
│  │  CALLVALUE                                               │  │
│  │  DUP1                                                    │  │
│  │  ISZERO                                                  │  │
│  │  PUSH2 label_ok                                          │  │
│  │  JUMPI                                                   │  │
│  │  PUSH1 0x00                                              │  │
│  │  DUP1                                                    │  │
│  │  REVERT          // 如果发送了 ETH，回滚                  │  │
│  │                                                          │  │
│  │  // 3. 加载构造函数参数                                    │  │
│  │  label_ok:                                               │  │
│  │  JUMPDEST                                                │  │
│  │  POP                                                     │  │
│  │  PUSH1 0x40                                              │  │
│  │  MLOAD           // 获取空闲内存指针                      │  │
│  │  PUSH1 0x20                                              │  │
│  │  DUP1                                                    │  │
│  │  CODESIZE        // 获取总代码大小                        │  │
│  │  SUB                                                     │  │
│  │  DUP3                                                    │  │
│  │  CODECOPY        // 复制构造函数参数到内存                 │  │
│  │  // ... 加载参数 ...                                     │  │
│  │                                                          │  │
│  │  // 4. 执行构造函数 (存储 value)                          │  │
│  │  PUSH1 0x00      // slot 0                               │  │
│  │  SSTORE          // storage[0] = _value                  │  │
│  │                                                          │  │
│  │  // 5. 返回 runtime code                                  │  │
│  │  PUSH1 runtime_size                                      │  │
│  │  DUP1                                                    │  │
│  │  PUSH1 runtime_offset                                    │  │
│  │  PUSH1 0x00                                              │  │
│  │  CODECOPY        // 复制 runtime code 到内存              │  │
│  │  PUSH1 0x00                                              │  │
│  │  RETURN          // 返回 runtime code                    │  │
│  │                                                          │  │
│  │  // 6. Runtime Code (嵌入在 creation code 中)             │  │
│  │  runtime_code:                                           │  │
│  │  ... (实际运行的代码) ...                                 │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 21.3 Runtime Code 结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Runtime Code 结构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  标准 Solidity Runtime Code 布局:                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  ┌─────────────────────────────────────────────────────┐│  │
│  │  │ 1. 函数分发器 (Dispatcher)                         ││  │
│  │  │    • 读取 calldata 前 4 字节 (函数选择器)          ││  │
│  │  │    • 与已知选择器比较                               ││  │
│  │  │    • 跳转到对应函数                                 ││  │
│  │  └─────────────────────────────────────────────────────┘│  │
│  │                     │                                    │  │
│  │     ┌───────────────┼───────────────┐                   │  │
│  │     ▼               ▼               ▼                   │  │
│  │  ┌───────┐     ┌───────┐      ┌───────┐                │  │
│  │  │ func1 │     │ func2 │      │ func3 │                │  │
│  │  │ 代码  │     │ 代码  │      │ 代码  │                │  │
│  │  └───────┘     └───────┘      └───────┘                │  │
│  │                                                          │  │
│  │  ┌─────────────────────────────────────────────────────┐│  │
│  │  │ 2. 函数实现                                        ││  │
│  │  │    • 参数解码                                       ││  │
│  │  │    • 业务逻辑                                       ││  │
│  │  │    • 返回值编码                                     ││  │
│  │  └─────────────────────────────────────────────────────┘│  │
│  │                                                          │  │
│  │  ┌─────────────────────────────────────────────────────┐│  │
│  │  │ 3. 辅助代码                                        ││  │
│  │  │    • 内部函数                                       ││  │
│  │  │    • 库代码                                         ││  │
│  │  │    • 错误处理                                       ││  │
│  │  └─────────────────────────────────────────────────────┘│  │
│  │                                                          │  │
│  │  ┌─────────────────────────────────────────────────────┐│  │
│  │  │ 4. 元数据 (可选, 末尾)                             ││  │
│  │  │    • Solidity 版本                                  ││  │
│  │  │    • IPFS/Swarm 哈希                               ││  │
│  │  │    • 格式: 0xa264... (CBOR 编码)                   ││  │
│  │  └─────────────────────────────────────────────────────┘│  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 22. 合约部署流程

### 22.1 部署交易处理

```go
// 文件: core/vm/evm.go

// Create 部署合约
func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *uint256.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
    // 1. 计算合约地址
    contractAddr = crypto.CreateAddress(caller.Address(), evm.StateDB.GetNonce(caller.Address()))

    return evm.create(caller, &codeAndHash{code: code}, gas, value, contractAddr, CREATE)
}

func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *uint256.Int, address common.Address, typ OpCode) ([]byte, common.Address, uint64, error) {
    // 2. 深度检查
    if evm.depth > int(params.CallCreateDepth) {
        return nil, common.Address{}, gas, ErrDepth
    }

    // 3. 余额检查
    if !value.IsZero() && !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value.ToBig()) {
		return nil, common.Address{}, gas, ErrInsufficientBalance
    }

    // 4. 增加 nonce
    nonce := evm.StateDB.GetNonce(caller.Address())
    evm.StateDB.SetNonce(caller.Address(), nonce+1)

    // 5. 检查地址冲突
    contractHash := evm.StateDB.GetCodeHash(address)
    if evm.StateDB.GetNonce(address) != 0 || (contractHash != (common.Hash{}) && contractHash != types.EmptyCodeHash) {
        return nil, common.Address{}, 0, ErrContractAddressCollision
    }

    // 6. 创建快照
    snapshot := evm.StateDB.Snapshot()
    
    // 7. 创建账户并设置 nonce = 1
    evm.StateDB.CreateAccount(address)
    evm.StateDB.SetNonce(address, 1)
    
    // 8. 转账
    evm.Context.Transfer(evm.StateDB, caller.Address(), address, value.ToBig())
    
    // 9. 创建合约对象并执行初始化代码
    contract := NewContract(caller, AccountRef(address), value, gas)
    contract.SetCodeOptionalHash(&address, codeAndHash)
    
    ret, err := evm.interpreter.Run(contract, nil, false)

    // 10. 检查代码大小限制 (24KB)
    if err == nil && len(ret) > params.MaxCodeSize {
        err = ErrMaxCodeSizeExceeded
    }
    
    // 11. 检查 EIP-3541: 禁止 0xEF 开头
    if err == nil && len(ret) > 0 && ret[0] == 0xEF {
        err = ErrInvalidCode
    }

    // 12. 支付代码存储 Gas (200 per byte)
    if err == nil {
        createDataGas := uint64(len(ret)) * params.CreateDataGas
        if contract.UseGas(createDataGas) {
            evm.StateDB.SetCode(address, ret)  // 存储 runtime code
        } else {
            err = ErrCodeStoreOutOfGas
        }
    }
    
    // 13. 错误回滚
    if err != nil {
        evm.StateDB.RevertToSnapshot(snapshot)
    }

    return ret, address, contract.Gas, err
}
```

### 22.2 地址计算

```go
// 文件: crypto/crypto.go

// CreateAddress 计算 CREATE 地址
// address = keccak256(rlp([sender, nonce]))[12:]
func CreateAddress(b common.Address, nonce uint64) common.Address {
    data, _ := rlp.EncodeToBytes([]interface{}{b, nonce})
    return common.BytesToAddress(Keccak256(data)[12:])
}

// CreateAddress2 计算 CREATE2 地址
// address = keccak256(0xff ++ sender ++ salt ++ keccak256(initCode))[12:]
func CreateAddress2(b common.Address, salt [32]byte, inithash []byte) common.Address {
    return common.BytesToAddress(Keccak256([]byte{0xff}, b.Bytes(), salt[:], inithash)[12:])
}
```

**地址计算示例：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    合约地址计算示例                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CREATE 地址计算:                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  sender = 0x6ac7ea33f8831ea9dcc53393aaa88b25a785dbf0     │  │
│  │  nonce = 0                                               │  │
│  │                                                          │  │
│  │  rlp([sender, nonce])                                    │  │
│  │  = rlp([0x6ac7ea33..., 0])                               │  │
│  │  = 0xd694 6ac7ea33f8831ea9dcc53393aaa88b25a785dbf0 80    │  │
│  │         ↑              ↑                            ↑    │  │
│  │       前缀          20字节地址                   nonce=0   │  │
│  │                                                          │  │
│  │  keccak256(rlp_data)                                     │  │
│  │  = 0xcd234a471b72ba2f1ccf0a70fcaba648a5eecd8defac28fe... │  │
│  │                                                          │  │
│  │  取后 20 字节                                             │  │
│  │  address = 0xcd234a471b72ba2f1ccf0a70fcaba648a5eecd8d    │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  CREATE2 地址计算:                                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  sender = 0x6ac7ea33f8831ea9dcc53393aaa88b25a785dbf0     │  │
│  │  salt = 0x0000...0001 (32字节)                           │  │
│  │  initCode = 0x6080604052... (合约创建代码)                 │  │
│  │                                                          │  │
│  │  initCodeHash = keccak256(initCode)                      │  │
│  │  = 0xabcd1234...                                         │  │
│  │                                                          │  │
│  │  data = 0xff ++ sender ++ salt ++ initCodeHash           │  │
│  │  = 0xff                                    (1 byte)      │  │
│  │    6ac7ea33f8831ea9dcc53393aaa88b25a785dbf0 (20 bytes)   │  │
│  │    0000...0001                             (32 bytes)    │  │
│  │    abcd1234...                             (32 bytes)    │  │
│  │                                                          │  │
│  │  keccak256(data)[12:]                                    │  │
│  │  address = 0x...                                         │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  CREATE2 的优势:                                                │
│  • 地址可预测 (部署前就知道)                                       │
│  • 相同参数 = 相同地址                                            │
│  • 支持反事实部署                                                 │
│  • 工厂模式常用                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 23. 函数选择器与 ABI 编码

### 23.1 函数选择器

```
┌─────────────────────────────────────────────────────────────────┐
│                      函数选择器                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  计算方法:                                                       │
│  selector = keccak256("functionName(type1,type2,...)")[0:4]     │
│                                                                 │
│  示例:                                                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  function transfer(address to, uint256 amount)           │  │
│  │                                                          │  │
│  │  签名字符串: "transfer(address,uint256)"                  │  │
│  │  注意: 不包含参数名，不包含空格，uint256 不简写为 uint     │  │
│  │                                                          │  │
│  │  keccak256("transfer(address,uint256)")                  │  │
│  │  = 0xa9059cbb2ab09eb219583f4a59a5d0623ade346d962bcd4e46b11da047c9049b│
│  │                                                          │  │
│  │  取前 4 字节: 0xa9059cbb                                 │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  常见选择器:                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  ERC20:                                                  │  │
│  │  0x70a08231  balanceOf(address)                         │  │
│  │  0xa9059cbb  transfer(address,uint256)                  │  │
│  │  0x23b872dd  transferFrom(address,address,uint256)      │  │
│  │  0x095ea7b3  approve(address,uint256)                   │  │
│  │  0xdd62ed3e  allowance(address,address)                 │  │
│  │  0x18160ddd  totalSupply()                              │  │
│  │                                                          │  │
│  │  ERC721:                                                 │  │
│  │  0x6352211e  ownerOf(uint256)                           │  │
│  │  0x42842e0e  safeTransferFrom(address,address,uint256)  │  │
│  │  0xb88d4fde  safeTransferFrom(address,address,uint256,bytes)│
│  │                                                          │  │
│  │  Uniswap V2:                                             │  │
│  │  0x022c0d9f  swap(uint256,uint256,address,bytes)        │  │
│  │  0x0902f1ac  getReserves()                              │  │
│  │  0xd21220a7  token1()                                   │  │
│  │  0x0dfe1681  token0()                                   │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 23.2 ABI 编码规则

```
┌─────────────────────────────────────────────────────────────────┐
│                      ABI 编码规则                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  静态类型 (固定大小):                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • uint<N> / int<N>: 补到 32 字节，右对齐                 │  │
│  │  • address: 补到 32 字节，右对齐                          │  │
│  │  • bool: 0 或 1，补到 32 字节                             │  │
│  │  • bytes<N>: 补到 32 字节，左对齐                         │  │
│  │  • 固定大小数组: 元素依次编码                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  动态类型 (可变大小):                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • bytes: 长度 + 数据 (补到 32 的倍数)                    │  │
│  │  • string: 同 bytes                                      │  │
│  │  • 动态数组: 长度 + 元素依次编码                          │  │
│  │  • 位置: 使用偏移量指向实际数据位置                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  编码示例: transfer(address,uint256)                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  调用: transfer(0x123..., 1000000)                       │  │
│  │                                                          │  │
│  │  calldata:                                               │  │
│  │  ┌─────────────────────────────────────────────────────┐│  │
│  │  │ 0xa9059cbb                                          ││  │
│  │  │ ← 函数选择器 (4 bytes)                               ││  │
│  │  │                                                     ││  │
│  │  │ 0000000000000000000000001234567890123456789012345678││  │
│  │  │ ← address to (32 bytes, 右对齐)                      ││  │
│  │  │                                                     ││  │
│  │  │ 00000000000000000000000000000000000000000000000000000f4240││
│  │  │ ← uint256 amount (32 bytes, 1000000 = 0xf4240)      ││  │
│  │  └─────────────────────────────────────────────────────┘│  │
│  │                                                          │  │
│  │  总长度: 4 + 32 + 32 = 68 bytes                         │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  编码示例: 动态参数                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  function test(uint256 a, string memory b, uint256 c)   │  │
│  │  调用: test(123, "hello", 456)                          │  │
│  │                                                          │  │
│  │  calldata 布局:                                          │  │
│  │  ┌─────────────────────────────────────────────────────┐│  │
│  │  │ [0x00-0x03]  函数选择器                             ││  │
│  │  │ [0x04-0x23]  a = 123 (静态)                        ││  │
│  │  │ [0x24-0x43]  b 的偏移量 = 0x60 (指向动态数据)       ││  │
│  │  │ [0x44-0x63]  c = 456 (静态)                        ││  │
│  │  │ [0x64-0x83]  b 的长度 = 5                          ││  │
│  │  │ [0x84-0xa3]  b 的数据 "hello" + padding            ││  │
│  │  └─────────────────────────────────────────────────────┘│  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 23.3 函数分发器字节码

```
┌─────────────────────────────────────────────────────────────────┐
│                    函数分发器字节码                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  典型的 Solidity 编译器生成的分发器:                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  // 读取 calldata 前 4 字节                              │  │
│  │  PUSH1 0x00                                              │  │
│  │  CALLDATALOAD    // 加载 calldata[0:32]                  │  │
│  │  PUSH1 0xe0      // 224 = 256 - 32                       │  │
│  │  SHR             // 右移 224 位，只保留前 4 字节          │  │
│  │                                                          │  │
│  │  // 与 transfer 选择器比较                                │  │
│  │  DUP1                                                    │  │
│  │  PUSH4 0xa9059cbb                                        │  │
│  │  EQ                                                      │  │
│  │  PUSH2 transfer_label                                    │  │
│  │  JUMPI           // 如果匹配，跳转到 transfer             │  │
│  │                                                          │  │
│  │  // 与 approve 选择器比较                                 │  │
│  │  DUP1                                                    │  │
│  │  PUSH4 0x095ea7b3                                        │  │
│  │  EQ                                                      │  │
│  │  PUSH2 approve_label                                     │  │
│  │  JUMPI           // 如果匹配，跳转到 approve              │  │
│  │                                                          │  │
│  │  // ... 更多函数 ...                                     │  │
│  │                                                          │  │
│  │  // 无匹配，检查 fallback                                 │  │
│  │  PUSH2 fallback_or_revert                                │  │
│  │  JUMP                                                    │  │
│  │                                                          │  │
│  │  // 或者直接 revert                                       │  │
│  │  PUSH1 0x00                                              │  │
│  │  DUP1                                                    │  │
│  │  REVERT                                                  │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  优化: 二分查找 (大合约)                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  如果合约有很多函数，编译器可能使用二分查找:              │  │
│  │                                                          │  │
│  │  // selector < 0x5000...?                                │  │
│  │  DUP1                                                    │  │
│  │  PUSH4 0x50000000                                        │  │
│  │  LT                                                      │  │
│  │  PUSH2 low_selectors                                     │  │
│  │  JUMPI                                                   │  │
│  │  // 处理 >= 0x5000... 的选择器                           │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 23.4 Go 代码: ABI 编解码

```go
package main

import (
	"fmt"
	"math/big"
	"strings"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

func main() {
	// 计算函数选择器
	transferSig := "transfer(address,uint256)"
	selector := crypto.Keccak256([]byte(transferSig))[:4]
	fmt.Printf("transfer 选择器: 0x%x\n", selector)

	// 使用 ABI 编码参数
	const abiJSON = `[{
        "name": "transfer",
        "type": "function",
        "inputs": [
            {"name": "to", "type": "address"},
            {"name": "amount", "type": "uint256"}
        ],
        "outputs": [{"name": "", "type": "bool"}]
    }]`

	parsedABI, err := abi.JSON(strings.NewReader(abiJSON))
	if err != nil {
		panic(err)
	}

	// 编码调用数据
	to := common.HexToAddress("0x1234567890123456789012345678901234567890")
	amount := big.NewInt(1000000000000000000) // 1e18

	callData, err := parsedABI.Pack("transfer", to, amount)
	if err != nil {
		panic(err)
	}

	fmt.Printf("编码后的 calldata: 0x%x\n", callData)
	fmt.Printf("长度: %d bytes\n", len(callData))

	// 解码调用数据
	method, err := parsedABI.MethodById(callData[:4])
	if err != nil {
		panic(err)
	}
	fmt.Printf("方法名: %s\n", method.Name)

	args, err := method.Inputs.Unpack(callData[4:])
	if err != nil {
		panic(err)
	}
	fmt.Printf("参数 to: %s\n", args[0].(common.Address).Hex())
	fmt.Printf("参数 amount: %s\n", args[1].(*big.Int).String())

	// 解码返回值
	returnData := common.Hex2Bytes("0000000000000000000000000000000000000000000000000000000000000001")
	var success bool
	err = parsedABI.UnpackIntoInterface(&success, "transfer", returnData)
	if err != nil {
		panic(err)
	}
	fmt.Printf("返回值: %v\n", success)
}
```

---

## 24. 字节码反编译

### 24.1 反编译原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    字节码反编译流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 线性扫描 (Linear Sweep)                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  从头开始顺序解析字节码                                    │  │
│  │  问题: PUSH 后的数据会被误解为操作码                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  2. 递归下降 (Recursive Descent)                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  从入口点开始，跟随控制流                                  │  │
│  │  遇到 JUMP/JUMPI 时递归分析目标                           │  │
│  │  问题: 动态跳转难以分析                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  3. 符号执行                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  模拟执行，跟踪符号值                                      │  │
│  │  可以解决一些动态跳转                                      │  │
│  │  计算复杂度高                                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  4. 模式匹配                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  识别已知的代码模式                                        │  │
│  │  如函数分发器、存储读写等                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 24.2 简单反汇编器

```go
package main

import (
	"encoding/hex"
	"fmt"

	"github.com/ethereum/go-ethereum/core/vm"
)

// Instruction 表示一条指令
type Instruction struct {
	Offset    uint64
	OpCode    vm.OpCode
	OpName    string
	Immediate []byte // PUSH 操作的立即数
}

// Disassemble 反汇编字节码
func Disassemble(code []byte) []Instruction {
	var instructions []Instruction

	for pc := uint64(0); pc < uint64(len(code)); {
		op := vm.OpCode(code[pc])
		inst := Instruction{
			Offset: pc,
			OpCode: op,
			OpName: op.String(),
		}

		// 处理 PUSH 操作码
		if op >= vm.PUSH1 && op <= vm.PUSH32 {
			size := int(op - vm.PUSH1 + 1)
			if int(pc)+1+size <= len(code) {
				inst.Immediate = code[pc+1 : pc+1+uint64(size)]
			}
			pc += uint64(size)
		}

		instructions = append(instructions, inst)
		pc++
	}

	return instructions
}

// PrintDisassembly 打印反汇编结果
func PrintDisassembly(instructions []Instruction) {
	for _, inst := range instructions {
		if len(inst.Immediate) > 0 {
			fmt.Printf("%04x: %-12s 0x%s\n", inst.Offset, inst.OpName, hex.EncodeToString(inst.Immediate))
		} else {
			fmt.Printf("%04x: %s\n", inst.Offset, inst.OpName)
		}
	}
}

// FindFunctionSelectors 查找函数选择器
func FindFunctionSelectors(code []byte) []string {
	var selectors []string

	for pc := 0; pc < len(code); {
		op := vm.OpCode(code[pc])

		// 查找 PUSH4 后跟 EQ 的模式
		if op == vm.PUSH4 && pc+5 < len(code) {
			selector := code[pc+1 : pc+5]

			// 检查接下来是否有 EQ 操作
			nextOp := vm.OpCode(code[pc+5])
			if nextOp == vm.EQ || nextOp == vm.DUP1 {
				selectors = append(selectors, fmt.Sprintf("0x%x", selector))
			}
			pc += 5
		} else if op >= vm.PUSH1 && op <= vm.PUSH32 {
			size := int(op - vm.PUSH1 + 1)
			pc += size + 1
		} else {
			pc++
		}
	}

	return selectors
}

// IdentifyContract 识别合约类型
func IdentifyContract(code []byte) string {
	selectors := FindFunctionSelectors(code)

	// 检查 ERC20
	erc20Selectors := map[string]bool{
		"0xa9059cbb": true, // transfer
		"0x70a08231": true, // balanceOf
		"0x095ea7b3": true, // approve
	}
	erc20Count := 0
	for _, sel := range selectors {
		if erc20Selectors[sel] {
			erc20Count++
		}
	}
	if erc20Count >= 2 {
		return "可能是 ERC20 代币"
	}

	// 检查 ERC721
	erc721Selectors := map[string]bool{
		"0x6352211e": true, // ownerOf
		"0x42842e0e": true, // safeTransferFrom
	}
	erc721Count := 0
	for _, sel := range selectors {
		if erc721Selectors[sel] {
			erc721Count++
		}
	}
	if erc721Count >= 1 {
		return "可能是 ERC721 NFT"
	}

	return "未知合约类型"
}

func main() {
	// 示例: 简单 ERC20 片段
	bytecodeHex := "6080604052600436106100295760003560e01c806370a08231146100" +
		"2e578063a9059cbb1461005b575b600080fd5b34801561003a5760" +
		"0080fd5b50610059600480360381019061005491906102a1565b61" +
		"008b565b005b34801561006757600080fd5b50610082600480360" +
		"3810190610071919061024e565b6100d3565b6040516100919190" +
		"6102f6565b60405180910390f35b"

	bytecode, _ := hex.DecodeString(bytecodeHex)

	fmt.Println("=== 反汇编结果 ===")
	instructions := Disassemble(bytecode)
	PrintDisassembly(instructions[:20]) // 只打印前 20 条

	fmt.Println("\n=== 函数选择器 ===")
	selectors := FindFunctionSelectors(bytecode)
	for _, sel := range selectors {
		fmt.Printf("  %s\n", sel)
	}

	fmt.Println("\n=== 合约识别 ===")
	fmt.Println(IdentifyContract(bytecode))
}
```

### 24.3 控制流图构建

```go
package main

import (
	"fmt"
)

// BasicBlock 基本块
type BasicBlock struct {
	StartPC      uint64
	EndPC        uint64
	Instructions []Instruction
	Successors   []*BasicBlock // 后继块
	Predecessors []*BasicBlock // 前驱块
	IsJumpDest   bool
}

// CFG 控制流图
type CFG struct {
	Blocks     []*BasicBlock
	BlockMap   map[uint64]*BasicBlock // PC -> Block
	EntryBlock *BasicBlock
}

// BuildCFG 构建控制流图
func BuildCFG(instructions []Instruction) *CFG {
	cfg := &CFG{
		BlockMap: make(map[uint64]*BasicBlock),
	}

	// 1. 识别基本块边界
	// 边界条件:
	// - 代码开始
	// - JUMPDEST
	// - JUMP/JUMPI 后的下一条指令
	// - STOP/RETURN/REVERT/INVALID

	leaders := make(map[uint64]bool)
	leaders[0] = true // 第一条指令是 leader

	for i, inst := range instructions {
		switch inst.OpCode {
		case 0x56: // JUMP
			// 下一条是 leader
			if i+1 < len(instructions) {
				leaders[instructions[i+1].Offset] = true
			}
		case 0x57: // JUMPI
			// 下一条是 leader (false 分支)
			if i+1 < len(instructions) {
				leaders[instructions[i+1].Offset] = true
			}
		case 0x5b: // JUMPDEST
			// 自身是 leader
			leaders[inst.Offset] = true
		case 0x00, 0xf3, 0xfd, 0xfe: // STOP, RETURN, REVERT, INVALID
			// 下一条是 leader
			if i+1 < len(instructions) {
				leaders[instructions[i+1].Offset] = true
			}
		}
	}

	// 2. 创建基本块
	var currentBlock *BasicBlock
	for _, inst := range instructions {
		if leaders[inst.Offset] {
			// 开始新块
			if currentBlock != nil {
				cfg.Blocks = append(cfg.Blocks, currentBlock)
			}
			currentBlock = &BasicBlock{
				StartPC:    inst.Offset,
				IsJumpDest: inst.OpCode == 0x5b,
			}
			cfg.BlockMap[inst.Offset] = currentBlock
		}

		if currentBlock != nil {
			currentBlock.Instructions = append(currentBlock.Instructions, inst)
			currentBlock.EndPC = inst.Offset
		}
	}
	if currentBlock != nil {
		cfg.Blocks = append(cfg.Blocks, currentBlock)
	}

	// 3. 建立边 (后继关系)
	for _, block := range cfg.Blocks {
		if len(block.Instructions) == 0 {
			continue
		}

		lastInst := block.Instructions[len(block.Instructions)-1]

		switch lastInst.OpCode {
		case 0x56: // JUMP
			// 需要静态分析跳转目标
			// 这里简化处理
		case 0x57: // JUMPI
			// false 分支: 下一个块
			// true 分支: 跳转目标
		case 0x00, 0xf3, 0xfd, 0xfe: // 终止指令
			// 没有后继
		default:
			// 顺序执行: 下一个块
			nextPC := lastInst.Offset + 1
			if lastInst.OpCode >= 0x60 && lastInst.OpCode <= 0x7f {
				nextPC += uint64(lastInst.OpCode - 0x5f)
			}
			if nextBlock, ok := cfg.BlockMap[nextPC]; ok {
				block.Successors = append(block.Successors, nextBlock)
				nextBlock.Predecessors = append(nextBlock.Predecessors, block)
			}
		}
	}

	if len(cfg.Blocks) > 0 {
		cfg.EntryBlock = cfg.Blocks[0]
	}

	return cfg
}

// PrintCFG 打印控制流图
func (cfg *CFG) Print() {
	fmt.Println("=== 控制流图 ===")
	for i, block := range cfg.Blocks {
		fmt.Printf("Block %d [%04x - %04x]", i, block.StartPC, block.EndPC)
		if block.IsJumpDest {
			fmt.Print(" (JUMPDEST)")
		}
		fmt.Println()

		for _, inst := range block.Instructions {
			fmt.Printf("  %04x: %s\n", inst.Offset, inst.OpName)
		}

		if len(block.Successors) > 0 {
			fmt.Print("  -> ")
			for j, succ := range block.Successors {
				if j > 0 {
					fmt.Print(", ")
				}
				fmt.Printf("[%04x]", succ.StartPC)
			}
			fmt.Println()
		}
		fmt.Println()
	}
}
```

---

## 25. 常见字节码模式

### 25.1 Solidity 编译器模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    常见 Solidity 字节码模式                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 内存初始化                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  6080604052   PUSH1 0x80, PUSH1 0x40, MSTORE            │  │
│  │  // memory[0x40] = 0x80 (空闲内存指针初始化)              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  2. 非 payable 检查                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  34          CALLVALUE                                   │  │
│  │  80          DUP1                                        │  │
│  │  15          ISZERO                                      │  │
│  │  61xxxx      PUSH2 xxxx                                  │  │
│  │  57          JUMPI                                       │  │
│  │  6000        PUSH1 0x00                                  │  │
│  │  80          DUP1                                        │  │
│  │  fd          REVERT                                      │  │
│  │  // if (msg.value != 0) revert();                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  3. 函数选择器提取                                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  6000        PUSH1 0x00                                  │  │
│  │  35          CALLDATALOAD                                │  │
│  │  60e0        PUSH1 0xe0                                  │  │
│  │  1c          SHR                                         │  │
│  │  // selector = calldata[0:4]                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  4. require 失败                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 带消息的 require                                      │  │
│  │  // Error(string) 选择器 = 0x08c379a0                    │  │
│  │  6308c379a0  PUSH4 0x08c379a0                            │  │
│  │  ...         (编码错误消息)                               │  │
│  │  fd          REVERT                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  5. SafeMath 检查 (0.8 之前)                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // a + b with overflow check                            │  │
│  │  01          ADD                                         │  │
│  │  80          DUP1                                        │  │
│  │  8x          DUPx (原来的 a)                             │  │
│  │  10          LT                                          │  │
│  │  15          ISZERO                                      │  │
│  │  ...         JUMPI (跳过 revert)                         │  │
│  │  fd          REVERT                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  6. Solidity 0.8+ 溢出检查                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // Panic(uint256) 选择器 = 0x4e487b71                   │  │
│  │  // Panic code 0x11 = overflow                           │  │
│  │  634e487b71  PUSH4 0x4e487b71                            │  │
│  │  ...         (编码 panic code)                           │  │
│  │  fd          REVERT                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  7. 存储变量读取                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  6000        PUSH1 0x00 (slot 0)                         │  │
│  │  54          SLOAD                                       │  │
│  │  // value = storage[0]                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  8. 存储变量写入                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ...         (value on stack)                            │  │
│  │  6000        PUSH1 0x00 (slot 0)                         │  │
│  │  55          SSTORE                                      │  │
│  │  // storage[0] = value                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  9. Mapping 访问                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // mapping(address => uint) balances; (slot 0)          │  │
│  │  // balances[addr]                                       │  │
│  │  ...         PUSH key (address)                          │  │
│  │  6000        PUSH1 0x00 (mapping slot)                   │  │
│  │  ...         MSTORE + MSTORE (key . slot)                │  │
│  │  6040        PUSH1 0x40 (size)                           │  │
│  │  6000        PUSH1 0x00 (offset)                         │  │
│  │  20          SHA3 (keccak256)                            │  │
│  │  54          SLOAD                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  10. 元数据哈希 (合约末尾)                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  a264697066735822...                                     │  │
│  │  // CBOR 编码的 IPFS 哈希                                │  │
│  │  // a2 = map with 2 elements                            │  │
│  │  // 64 = text(4) "ipfs"                                 │  │
│  │  // 58 22 = bytes(34) (IPFS hash)                       │  │
│  │  ...6473 6f6c6343 ...                                   │  │
│  │  // "solc" + version                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 25.2 代理合约模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    代理合约字节码模式                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  最小代理 (EIP-1167 Clone):                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  3d          RETURNDATASIZE  // 0 (初始)                 │  │
│  │  602d        PUSH1 0x2d      // 45 (runtime size)       │  │
│  │  8060        DUP1, PUSH1 0x0a                            │  │
│  │  3d          RETURNDATASIZE                              │  │
│  │  39          CODECOPY                                    │  │
│  │  81          DUP2                                        │  │
│  │  f3          RETURN                                      │  │
│  │  ------- 以上是 creation code -------                    │  │
│  │  363d3d373d3d3d363d73                                    │  │
│  │  CALLDATASIZE, RETURNDATASIZE, RETURNDATASIZE,          │  │
│  │  CALLDATACOPY, RETURNDATASIZE, RETURNDATASIZE,          │  │
│  │  RETURNDATASIZE, CALLDATASIZE, RETURNDATASIZE,          │  │
│  │  PUSH20                                                  │  │
│  │  xxxxxxxx...xxxxx (implementation address)              │  │
│  │  5af43d82803e903d91602b57fd5bf3                          │  │
│  │  GAS, DELEGATECALL, RETURNDATASIZE, DUP3, DUP1,         │  │
│  │  RETURNDATACOPY, SWAP1, RETURNDATASIZE, SWAP2,          │  │
│  │  PUSH1 0x2b, JUMPI, REVERT                              │  │
│  │  JUMPDEST, RETURN                                       │  │
│  │                                                          │  │
│  │  总长度: 45 bytes                                        │  │
│  │  功能: 将所有调用委托给 implementation                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  UUPS 代理:                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 从 EIP-1967 槽位读取实现地址                          │  │
│  │  // slot = keccak256("eip1967.proxy.implementation") - 1 │  │
│  │  //      = 0x360894a13ba1a3210667c828492db98dca3e2076... │  │
│  │                                                          │  │
│  │  7f360894...  PUSH32 slot                                │  │
│  │  54            SLOAD                                     │  │
│  │  // 然后 DELEGATECALL                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Transparent Proxy:                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  // 检查调用者是否为 admin                                │  │
│  │  33            CALLER                                    │  │
│  │  7fxxx...      PUSH32 admin_slot                         │  │
│  │  54            SLOAD                                     │  │
│  │  14            EQ                                        │  │
│  │  ...           JUMPI (to admin functions)                │  │
│  │  // 否则 DELEGATECALL 到实现                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 26. 字节码分析工具

### 26.1 常用工具

```
┌─────────────────────────────────────────────────────────────────┐
│                    字节码分析工具                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  在线工具:                                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • evm.codes           - 操作码参考 + Playground         │  │
│  │  • ethervm.io          - 在线反汇编                      │  │
│  │  • etherscan.io        - 合约反编译 (Decompile)         │  │
│  │  • contract-library.com - 合约反编译                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  命令行工具:                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • evmdis       - Go 实现的反汇编器                       │  │
│  │  • pyevmasm     - Python 汇编/反汇编器                    │  │
│  │  • panoramix    - 反编译器 (Etherscan 使用)              │  │
│  │  • heimdall-rs  - Rust 反编译器                          │  │
│  │  • whatsabi     - 函数选择器识别                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  安全审计工具:                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • Mythril      - 符号执行，漏洞检测                      │  │
│  │  • Slither      - 静态分析                               │  │
│  │  • Manticore    - 符号执行框架                           │  │
│  │  • Echidna      - 模糊测试                               │  │
│  │  • Foundry      - 测试框架，含调试器                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  IDE 插件:                                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • Solidity Visual Developer (VSCode)                   │  │
│  │  • Remix IDE 内置调试器                                   │  │
│  │  • Foundry forge debug                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 26.2 使用示例

```bash
# 1. 使用 cast (Foundry) 反汇编
cast disassemble 0x6080604052...

# 2. 获取合约字节码
cast code 0x... --rpc-url https://eth.llamarpc.com

# 3. 识别函数选择器
cast 4byte 0xa9059cbb
# 输出: transfer(address,uint256)

# 4. 计算函数选择器
cast sig "transfer(address,uint256)"
# 输出: 0xa9059cbb

# 5. ABI 编码调用数据
cast calldata "transfer(address,uint256)" 0x123... 1000

# 6. 解码调用数据
cast 4byte-decode 0xa9059cbb000000000000...
```

### 26.3 完整的分析示例

```go
package main

import (
	"encoding/hex"
	"fmt"
	"strings"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/crypto"
)

// BytecodeAnalyzer 字节码分析器
type BytecodeAnalyzer struct {
	code []byte
}

// NewAnalyzer 创建分析器
func NewAnalyzer(hexCode string) *BytecodeAnalyzer {
	code, _ := hex.DecodeString(strings.TrimPrefix(hexCode, "0x"))
	return &BytecodeAnalyzer{code: code}
}

// ExtractSelectors 提取函数选择器
func (a *BytecodeAnalyzer) ExtractSelectors() []string {
	var selectors []string

	for i := 0; i < len(a.code)-4; i++ {
		// 查找 PUSH4 (0x63)
		if a.code[i] == 0x63 {
			selector := fmt.Sprintf("0x%x", a.code[i+1:i+5])
			// 验证是否后跟 EQ 或 DUP
			if i+5 < len(a.code) {
				nextOp := a.code[i+5]
				if nextOp == 0x14 || nextOp == 0x80 { // EQ or DUP1
					selectors = append(selectors, selector)
				}
			}
		}
	}

	return unique(selectors)
}

// ExtractStrings 提取可能的字符串
func (a *BytecodeAnalyzer) ExtractStrings() []string {
	var strings []string

	// 查找连续的可打印 ASCII 字符
	var current []byte
	for _, b := range a.code {
		if b >= 0x20 && b <= 0x7e {
			current = append(current, b)
		} else {
			if len(current) >= 4 {
				strings = append(strings, string(current))
			}
			current = nil
		}
	}

	return strings
}

// HasMetadata 检查是否包含 Solidity 元数据
func (a *BytecodeAnalyzer) HasMetadata() bool {
	// Solidity 元数据以 0xa264 或 0xa165 开头
	if len(a.code) < 2 {
		return false
	}

	for i := len(a.code) - 50; i < len(a.code)-1; i++ {
		if i < 0 {
			continue
		}
		if (a.code[i] == 0xa2 && a.code[i+1] == 0x64) ||
			(a.code[i] == 0xa1 && a.code[i+1] == 0x65) {
			return true
		}
	}
	return false
}

// DetectProxyPattern 检测代理模式
func (a *BytecodeAnalyzer) DetectProxyPattern() string {
	codeHex := hex.EncodeToString(a.code)

	// EIP-1167 最小代理
	if strings.Contains(codeHex, "363d3d373d3d3d363d73") {
		return "EIP-1167 Minimal Proxy"
	}

	// UUPS/Transparent Proxy (包含 delegatecall 和 EIP-1967 槽位)
	if strings.Contains(codeHex, "360894a13ba1a3210667c828492db98dca3e2076") {
		return "EIP-1967 Proxy (UUPS/Transparent)"
	}

	// 检查是否有 DELEGATECALL (0xf4)
	for _, b := range a.code {
		if b == 0xf4 {
			return "Contains DELEGATECALL (possible proxy)"
		}
	}

	return "Not a proxy"
}

// KnownSelector 已知选择器数据库
var KnownSelectors = map[string]string{
	"0xa9059cbb": "transfer(address,uint256)",
	"0x70a08231": "balanceOf(address)",
	"0x095ea7b3": "approve(address,uint256)",
	"0x23b872dd": "transferFrom(address,address,uint256)",
	"0xdd62ed3e": "allowance(address,address)",
	"0x18160ddd": "totalSupply()",
	"0x313ce567": "decimals()",
	"0x06fdde03": "name()",
	"0x95d89b41": "symbol()",
	"0x6352211e": "ownerOf(uint256)",
	"0x42842e0e": "safeTransferFrom(address,address,uint256)",
	"0xb88d4fde": "safeTransferFrom(address,address,uint256,bytes)",
}

// LookupSelector 查找选择器含义
func LookupSelector(selector string) string {
	if sig, ok := KnownSelectors[selector]; ok {
		return sig
	}
	return "Unknown"
}

func unique(slice []string) []string {
	seen := make(map[string]bool)
	var result []string
	for _, s := range slice {
		if !seen[s] {
			seen[s] = true
			result = append(result, s)
		}
	}
	return result
}

func main() {
	// 示例: 分析一个简单合约
	bytecode := "608060405234801561001057600080fd5b50610150806100206000396000f3fe6080604052348015" +
		"61001057600080fd5b50600436106100365760003560e01c806370a082311461003b5780" +
		"63a9059cbb14610059575b600080fd5b610043610075565b6040516100509190610097" +
		"565b60405180910390f35b610073600480360381019061006e91906100e9565b610075" +
		"565b005b60005481565b8060008190555050565b6000819050919050565b610091816" +
		"10084565b82525050565b60006020820190506100ac6000830184610088565b929150" +
		"50565b600080fd5b6100bf81610084565b81146100ca57600080fd5b50565b600081" +
		"3590506100dd816100b6565b92915050565b6000602082840312156100f9576100f86" +
		"100b1565b5b6000610107848285016100ce565b91505092915050fea264697066735822"

	analyzer := NewAnalyzer(bytecode)

	fmt.Println("=== 字节码分析报告 ===")
	fmt.Printf("代码长度: %d bytes\n\n", len(analyzer.code))

	fmt.Println("函数选择器:")
	selectors := analyzer.ExtractSelectors()
	for _, sel := range selectors {
		sig := LookupSelector(sel)
		fmt.Printf("  %s → %s\n", sel, sig)
	}

	fmt.Println("\n代理模式检测:")
	fmt.Printf("  %s\n", analyzer.DetectProxyPattern())

	fmt.Println("\n元数据检测:")
	if analyzer.HasMetadata() {
		fmt.Println("  包含 Solidity 元数据")
	} else {
		fmt.Println("  未发现 Solidity 元数据")
	}

	fmt.Println("\n可能的字符串:")
	strings := analyzer.ExtractStrings()
	for _, s := range strings {
		if len(s) > 3 && len(s) < 50 {
			fmt.Printf("  \"%s\"\n", s)
		}
	}
}
```

---

## 下一篇预告

下一篇文档将讲解 **EVM 与 MEV**，包括：

- MEV 基础概念
- 三明治攻击
- 闪电贷套利
- Flashbots 与 MEV-Boost
- 抢跑保护策略
