# Go-Ethereum EVM 深度解析：高级 Gas 优化

## 第一百一十节：EVM Gas 成本模型

### Gas 成本详解

```
EVM Gas 成本模型：
┌─────────────────────────────────────────────────────────────────────┐
│                       EVM Gas 成本结构                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  基础操作 Gas 成本                                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  算术运算:     ADD/SUB: 3    MUL/DIV: 5    EXP: 10+        │    │
│  │  比较运算:     LT/GT/EQ: 3   ISZERO: 3                     │    │
│  │  位运算:       AND/OR/XOR: 3  SHL/SHR: 3                   │    │
│  │  栈操作:       PUSH: 3       POP: 2       DUP: 3           │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  存储操作 Gas 成本 (EIP-2929 后)                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  SLOAD:        冷访问: 2100   热访问: 100                   │    │
│  │  SSTORE:       0→非0: 22100  非0→非0: 2900  非0→0: 退款    │    │
│  │  BALANCE:      冷访问: 2600   热访问: 100                   │    │
│  │  EXTCODESIZE:  冷访问: 2600   热访问: 100                   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  内存操作 Gas 成本                                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  MLOAD/MSTORE:   3 + 内存扩展成本                          │    │
│  │  内存扩展:       memory_cost = words * 3 + words² / 512    │    │
│  │  MSTORE8:        3                                         │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  调用操作 Gas 成本                                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  CALL:           冷: 2600 + 基础  热: 100 + 基础            │    │
│  │  STATICCALL:     同上                                       │    │
│  │  DELEGATECALL:   同上                                       │    │
│  │  CREATE:         32000                                      │    │
│  │  CREATE2:        32000 + 6 * 代码长度                       │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Gas 成本计算器

```go
package gasopt

import (
    "math/big"
)

// GasCosts EVM Gas 成本常量
var GasCosts = struct {
    // 基础操作
    Zero            uint64
    Base            uint64
    VeryLow         uint64
    Low             uint64
    Mid             uint64
    High            uint64

    // 存储操作 (EIP-2929)
    ColdSloadCost   uint64
    WarmStorageRead uint64
    SstoreSet       uint64
    SstoreReset     uint64
    SstoreClears    uint64

    // 内存操作
    Memory          uint64
    MemoryWord      uint64

    // 调用操作
    Call            uint64
    CallValue       uint64
    CallNewAccount  uint64
    ColdAccountAccess uint64
    WarmAccountAccess uint64

    // 日志操作
    Log             uint64
    LogData         uint64
    LogTopic        uint64

    // 创建操作
    Create          uint64
    Create2         uint64
    CodeDeposit     uint64

    // 特殊操作
    Keccak256       uint64
    Keccak256Word   uint64
    Copy            uint64
    Exp             uint64
    ExpByte         uint64
}{
    Zero:            0,
    Base:            2,
    VeryLow:         3,
    Low:             5,
    Mid:             8,
    High:            10,

    ColdSloadCost:   2100,
    WarmStorageRead: 100,
    SstoreSet:       22100,
    SstoreReset:     2900,
    SstoreClears:    4800,

    Memory:          3,
    MemoryWord:      3,

    Call:            0,
    CallValue:       9000,
    CallNewAccount:  25000,
    ColdAccountAccess: 2600,
    WarmAccountAccess: 100,

    Log:             375,
    LogData:         8,
    LogTopic:        375,

    Create:          32000,
    Create2:         32000,
    CodeDeposit:     200,

    Keccak256:       30,
    Keccak256Word:   6,
    Copy:            3,
    Exp:             10,
    ExpByte:         50,
}

// GasEstimator Gas 估算器
type GasEstimator struct {
    warmAccounts map[string]bool
    warmSlots    map[string]bool
}

// NewGasEstimator 创建估算器
func NewGasEstimator() *GasEstimator {
    return &GasEstimator{
        warmAccounts: make(map[string]bool),
        warmSlots:    make(map[string]bool),
    }
}

// EstimateSload 估算 SLOAD Gas
func (g *GasEstimator) EstimateSload(contract, slot string) uint64 {
    key := contract + ":" + slot
    if g.warmSlots[key] {
        return GasCosts.WarmStorageRead
    }
    g.warmSlots[key] = true
    return GasCosts.ColdSloadCost
}

// EstimateSstore 估算 SSTORE Gas
func (g *GasEstimator) EstimateSstore(contract, slot string, oldValue, newValue *big.Int) uint64 {
    key := contract + ":" + slot
    isWarm := g.warmSlots[key]
    g.warmSlots[key] = true

    var gas uint64 = 0

    // 冷访问成本
    if !isWarm {
        gas += GasCosts.ColdSloadCost
    }

    // 根据值变化计算成本
    oldIsZero := oldValue == nil || oldValue.Sign() == 0
    newIsZero := newValue == nil || newValue.Sign() == 0

    if oldIsZero && !newIsZero {
        // 0 -> 非0
        gas += GasCosts.SstoreSet
    } else if !oldIsZero && newIsZero {
        // 非0 -> 0 (有退款)
        gas += GasCosts.SstoreReset
    } else {
        // 非0 -> 非0
        gas += GasCosts.SstoreReset
    }

    return gas
}

// EstimateCall 估算 CALL Gas
func (g *GasEstimator) EstimateCall(to string, value *big.Int, isNew bool) uint64 {
    var gas uint64 = 0

    // 账户访问成本
    if g.warmAccounts[to] {
        gas += GasCosts.WarmAccountAccess
    } else {
        gas += GasCosts.ColdAccountAccess
        g.warmAccounts[to] = true
    }

    // 转账成本
    if value != nil && value.Sign() > 0 {
        gas += GasCosts.CallValue
    }

    // 新账户成本
    if isNew && value != nil && value.Sign() > 0 {
        gas += GasCosts.CallNewAccount
    }

    return gas
}

// EstimateMemory 估算内存扩展 Gas
func (g *GasEstimator) EstimateMemory(currentSize, newSize uint64) uint64 {
    if newSize <= currentSize {
        return 0
    }

    oldWords := (currentSize + 31) / 32
    newWords := (newSize + 31) / 32

    if newWords <= oldWords {
        return 0
    }

    oldCost := g.memoryGas(oldWords)
    newCost := g.memoryGas(newWords)

    return newCost - oldCost
}

// memoryGas 计算内存 Gas
func (g *GasEstimator) memoryGas(words uint64) uint64 {
    return words*GasCosts.MemoryWord + (words*words)/512
}

// EstimateKeccak256 估算 Keccak256 Gas
func (g *GasEstimator) EstimateKeccak256(dataSize uint64) uint64 {
    words := (dataSize + 31) / 32
    return GasCosts.Keccak256 + words*GasCosts.Keccak256Word
}

// EstimateLog 估算 LOG Gas
func (g *GasEstimator) EstimateLog(topics int, dataSize uint64) uint64 {
    return GasCosts.Log +
           uint64(topics)*GasCosts.LogTopic +
           dataSize*GasCosts.LogData
}

// EstimateCreate 估算 CREATE/CREATE2 Gas
func (g *GasEstimator) EstimateCreate(codeSize uint64, isCreate2 bool) uint64 {
    gas := GasCosts.Create
    if isCreate2 {
        words := (codeSize + 31) / 32
        gas += words * GasCosts.Keccak256Word
    }
    gas += codeSize * GasCosts.CodeDeposit
    return gas
}
```

---

## 第一百一十一节：Solidity Gas 优化技巧

### 合约优化模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

/// @title Gas 优化示例合约
/// @notice 展示各种 Gas 优化技巧
contract GasOptimizedContract {

    // ============ 存储优化 ============

    // 差: 使用多个 storage slot
    // uint256 public value1;  // slot 0
    // uint256 public value2;  // slot 1
    // uint256 public value3;  // slot 2

    // 好: 打包到同一个 slot
    struct PackedData {
        uint128 value1;  // 16 bytes
        uint64 value2;   // 8 bytes
        uint64 value3;   // 8 bytes
    }  // 总共 32 bytes = 1 slot

    PackedData public packedData;

    // 更进一步: 使用更小的类型
    struct TightlyPacked {
        uint32 timestamp;   // 4 bytes
        uint32 amount;      // 4 bytes
        uint16 fee;         // 2 bytes
        uint8 status;       // 1 byte
        bool active;        // 1 byte
        address user;       // 20 bytes
    }  // 总共 32 bytes = 1 slot

    // ============ 变量可见性优化 ============

    // 差: public 会自动生成 getter
    // uint256 public expensiveValue;

    // 好: internal/private 更便宜
    uint256 internal _value;

    // 如果需要读取，手动实现更优化的 getter
    function getValue() external view returns (uint256) {
        return _value;
    }

    // ============ 常量和不可变量 ============

    // 差: 存储变量
    // address public owner;

    // 好: immutable (部署时设置，存储在字节码中)
    address public immutable owner;

    // 最好: constant (编译时常量)
    uint256 public constant MAX_SUPPLY = 1000000;

    constructor() {
        owner = msg.sender;
    }

    // ============ 循环优化 ============

    uint256[] public values;

    // 差: 每次迭代都读取 storage
    function sumBad() external view returns (uint256 total) {
        for (uint256 i = 0; i < values.length; i++) {
            total += values[i];
        }
    }

    // 好: 缓存数组长度
    function sumGood() external view returns (uint256 total) {
        uint256 len = values.length;
        for (uint256 i = 0; i < len; i++) {
            total += values[i];
        }
    }

    // 更好: 使用 unchecked (如果确定不会溢出)
    function sumBest() external view returns (uint256 total) {
        uint256 len = values.length;
        for (uint256 i = 0; i < len; ) {
            total += values[i];
            unchecked { ++i; }
        }
    }

    // ============ 函数优化 ============

    // 差: 多次 storage 读取
    function processBad(uint256 amount) external {
        require(_value >= amount, "Insufficient");
        _value = _value - amount;
        // 更多使用 _value 的操作...
    }

    // 好: 缓存到 memory
    function processGood(uint256 amount) external {
        uint256 cached = _value;
        require(cached >= amount, "Insufficient");
        _value = cached - amount;
    }

    // ============ 错误处理优化 ============

    // 差: 长字符串错误消息
    // require(condition, "This is a very long error message that costs gas");

    // 好: 短字符串或自定义错误
    error InsufficientBalance(uint256 available, uint256 required);

    function withdrawOptimized(uint256 amount) external {
        uint256 balance = _value;
        if (balance < amount) {
            revert InsufficientBalance(balance, amount);
        }
        _value = balance - amount;
    }

    // ============ Calldata vs Memory ============

    // 差: memory 会复制数据
    function processBadArray(uint256[] memory data) external pure returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < data.length; i++) {
            sum += data[i];
        }
        return sum;
    }

    // 好: calldata 直接引用输入
    function processGoodArray(uint256[] calldata data) external pure returns (uint256) {
        uint256 sum;
        uint256 len = data.length;
        for (uint256 i = 0; i < len; ) {
            sum += data[i];
            unchecked { ++i; }
        }
        return sum;
    }

    // ============ 位操作优化 ============

    // 差: 使用除法和取模
    // uint256 result = value / 2;
    // uint256 remainder = value % 2;

    // 好: 使用位移
    function divideByPowerOf2(uint256 value, uint8 power) external pure returns (uint256) {
        return value >> power;
    }

    function multiplyByPowerOf2(uint256 value, uint8 power) external pure returns (uint256) {
        return value << power;
    }

    function isOdd(uint256 value) external pure returns (bool) {
        return value & 1 == 1;
    }

    // ============ 批量操作 ============

    mapping(address => uint256) public balances;

    // 差: 多次单独转账
    function transfersBad(address[] calldata recipients, uint256[] calldata amounts) external {
        for (uint256 i = 0; i < recipients.length; i++) {
            balances[msg.sender] -= amounts[i];
            balances[recipients[i]] += amounts[i];
        }
    }

    // 好: 批量处理，减少 storage 操作
    function transfersGood(address[] calldata recipients, uint256[] calldata amounts) external {
        uint256 totalAmount;
        uint256 len = recipients.length;

        // 先计算总量
        for (uint256 i = 0; i < len; ) {
            totalAmount += amounts[i];
            unchecked { ++i; }
        }

        // 一次性扣除发送者余额
        uint256 senderBalance = balances[msg.sender];
        require(senderBalance >= totalAmount, "Insufficient");
        balances[msg.sender] = senderBalance - totalAmount;

        // 分别增加接收者余额
        for (uint256 i = 0; i < len; ) {
            balances[recipients[i]] += amounts[i];
            unchecked { ++i; }
        }
    }
}
```

### 汇编优化

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

/// @title 汇编优化示例
contract AssemblyOptimized {

    // ============ 直接操作 Storage ============

    function getSlot(uint256 slot) external view returns (bytes32 value) {
        assembly {
            value := sload(slot)
        }
    }

    function setSlot(uint256 slot, bytes32 value) external {
        assembly {
            sstore(slot, value)
        }
    }

    // ============ 高效的地址检查 ============

    function isContract(address account) external view returns (bool result) {
        assembly {
            result := gt(extcodesize(account), 0)
        }
    }

    // ============ 高效的哈希计算 ============

    function efficientHash(bytes calldata data) external pure returns (bytes32 result) {
        assembly {
            // 直接从 calldata 计算 hash
            let ptr := mload(0x40)
            calldatacopy(ptr, data.offset, data.length)
            result := keccak256(ptr, data.length)
        }
    }

    // ============ 批量读取 ============

    function batchBalanceOf(
        address token,
        address[] calldata accounts
    ) external view returns (uint256[] memory results) {
        results = new uint256[](accounts.length);

        // balanceOf(address) selector
        bytes4 selector = 0x70a08231;

        assembly {
            let resultsPtr := add(results, 0x20)
            let accountsLen := accounts.length

            for { let i := 0 } lt(i, accountsLen) { i := add(i, 1) } {
                // 构造 calldata
                let ptr := mload(0x40)
                mstore(ptr, selector)
                mstore(add(ptr, 0x04), calldataload(add(accounts.offset, mul(i, 0x20))))

                // 调用
                let success := staticcall(gas(), token, ptr, 0x24, ptr, 0x20)

                if success {
                    mstore(add(resultsPtr, mul(i, 0x20)), mload(ptr))
                }
            }
        }
    }

    // ============ 高效的多调用 ============

    struct Call {
        address target;
        bytes callData;
    }

    function multicall(Call[] calldata calls) external returns (bytes[] memory results) {
        results = new bytes[](calls.length);

        for (uint256 i = 0; i < calls.length; ) {
            (bool success, bytes memory result) = calls[i].target.call(calls[i].callData);
            require(success, "Call failed");
            results[i] = result;
            unchecked { ++i; }
        }
    }

    // 更优化的 multicall (固定返回大小)
    function multicallFixed(
        address[] calldata targets,
        bytes[] calldata datas
    ) external returns (uint256[] memory results) {
        uint256 len = targets.length;
        results = new uint256[](len);

        assembly {
            let resultsPtr := add(results, 0x20)

            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                // 获取 target
                let target := calldataload(add(targets.offset, mul(i, 0x20)))

                // 获取 data offset 和 length
                let dataOffset := calldataload(add(datas.offset, mul(i, 0x20)))
                let dataPtr := add(datas.offset, dataOffset)
                let dataLen := calldataload(dataPtr)
                dataPtr := add(dataPtr, 0x20)

                // 复制 calldata
                let ptr := mload(0x40)
                calldatacopy(ptr, dataPtr, dataLen)

                // 调用
                let success := call(gas(), target, 0, ptr, dataLen, ptr, 0x20)

                if success {
                    mstore(add(resultsPtr, mul(i, 0x20)), mload(ptr))
                }
            }
        }
    }

    // ============ 零拷贝返回 ============

    function returnCalldata(bytes calldata data) external pure returns (bytes calldata) {
        return data;
    }

    // ============ 高效的条件检查 ============

    function efficientRequire(bool condition) external pure {
        assembly {
            if iszero(condition) {
                revert(0, 0)
            }
        }
    }

    // 带错误信息的高效 require
    function efficientRequireWithMessage(bool condition) external pure {
        assembly {
            if iszero(condition) {
                // 存储错误选择器: Error(string)
                mstore(0x00, 0x08c379a000000000000000000000000000000000000000000000000000000000)
                mstore(0x04, 0x20)
                mstore(0x24, 5)
                mstore(0x44, "Error")
                revert(0x00, 0x64)
            }
        }
    }
}
```

---

## 第一百一十二节：Go 层面的 Gas 优化

### 交易构建优化

```go
package gasopt

import (
    "context"
    "encoding/hex"
    "fmt"
    "math/big"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

// TransactionOptimizer 交易优化器
type TransactionOptimizer struct {
    client *ethclient.Client
}

// NewTransactionOptimizer 创建优化器
func NewTransactionOptimizer(client *ethclient.Client) *TransactionOptimizer {
    return &TransactionOptimizer{client: client}
}

// OptimizeCalldata 优化 calldata
func (o *TransactionOptimizer) OptimizeCalldata(data []byte) []byte {
    // 统计零字节和非零字节
    zeroBytes := 0
    nonZeroBytes := 0

    for _, b := range data {
        if b == 0 {
            zeroBytes++
        } else {
            nonZeroBytes++
        }
    }

    // calldata 成本: 零字节 4 gas, 非零字节 16 gas
    currentCost := zeroBytes*4 + nonZeroBytes*16

    fmt.Printf("Calldata 成本: %d gas (零字节: %d, 非零字节: %d)\n",
        currentCost, zeroBytes, nonZeroBytes)

    return data
}

// EstimateOptimalGas 估算最优 Gas
func (o *TransactionOptimizer) EstimateOptimalGas(
    ctx context.Context,
    from, to common.Address,
    data []byte,
    value *big.Int,
) (*GasEstimate, error) {
    // 估算 Gas
    msg := ethereum.CallMsg{
        From:  from,
        To:    &to,
        Data:  data,
        Value: value,
    }

    gasEstimate, err := o.client.EstimateGas(ctx, msg)
    if err != nil {
        return nil, err
    }

    // 获取当前 Gas 价格
    gasPrice, err := o.client.SuggestGasPrice(ctx)
    if err != nil {
        return nil, err
    }

    // 获取 baseFee 和 优先费
    header, err := o.client.HeaderByNumber(ctx, nil)
    if err != nil {
        return nil, err
    }

    baseFee := header.BaseFee
    suggestedTip, err := o.client.SuggestGasTipCap(ctx)
    if err != nil {
        return nil, err
    }

    // 计算最优参数
    return &GasEstimate{
        GasLimit:         gasEstimate,
        BaseFee:          baseFee,
        MaxPriorityFee:   suggestedTip,
        MaxFee:           new(big.Int).Add(new(big.Int).Mul(baseFee, big.NewInt(2)), suggestedTip),
        LegacyGasPrice:   gasPrice,
        EstimatedCostWei: new(big.Int).Mul(gasPrice, big.NewInt(int64(gasEstimate))),
    }, nil
}

// GasEstimate Gas 估算结果
type GasEstimate struct {
    GasLimit         uint64
    BaseFee          *big.Int
    MaxPriorityFee   *big.Int
    MaxFee           *big.Int
    LegacyGasPrice   *big.Int
    EstimatedCostWei *big.Int
}

// BuildOptimizedTx 构建优化交易
func (o *TransactionOptimizer) BuildOptimizedTx(
    ctx context.Context,
    from, to common.Address,
    data []byte,
    value *big.Int,
    nonce uint64,
) (*types.Transaction, error) {
    estimate, err := o.EstimateOptimalGas(ctx, from, to, data, value)
    if err != nil {
        return nil, err
    }

    chainID, err := o.client.ChainID(ctx)
    if err != nil {
        return nil, err
    }

    // 使用 EIP-1559 交易
    tx := types.NewTx(&types.DynamicFeeTx{
        ChainID:   chainID,
        Nonce:     nonce,
        GasTipCap: estimate.MaxPriorityFee,
        GasFeeCap: estimate.MaxFee,
        Gas:       estimate.GasLimit + estimate.GasLimit/10, // 10% 余量
        To:        &to,
        Value:     value,
        Data:      data,
    })

    return tx, nil
}

// CalldataBuilder 高效的 Calldata 构建器
type CalldataBuilder struct {
    data []byte
}

// NewCalldataBuilder 创建构建器
func NewCalldataBuilder(selector string) *CalldataBuilder {
    selectorBytes, _ := hex.DecodeString(selector)
    return &CalldataBuilder{
        data: selectorBytes,
    }
}

// AddAddress 添加地址参数
func (b *CalldataBuilder) AddAddress(addr common.Address) *CalldataBuilder {
    param := make([]byte, 32)
    copy(param[12:], addr.Bytes())
    b.data = append(b.data, param...)
    return b
}

// AddUint256 添加 uint256 参数
func (b *CalldataBuilder) AddUint256(value *big.Int) *CalldataBuilder {
    param := make([]byte, 32)
    if value != nil {
        value.FillBytes(param)
    }
    b.data = append(b.data, param...)
    return b
}

// AddBytes 添加 bytes 参数
func (b *CalldataBuilder) AddBytes(data []byte) *CalldataBuilder {
    // 计算偏移量
    currentLen := len(b.data) - 4 // 减去 selector
    offset := make([]byte, 32)
    big.NewInt(int64(currentLen + 32)).FillBytes(offset)
    b.data = append(b.data, offset...)

    // 长度
    length := make([]byte, 32)
    big.NewInt(int64(len(data))).FillBytes(length)
    b.data = append(b.data, length...)

    // 数据 (填充到 32 字节边界)
    paddedLen := ((len(data) + 31) / 32) * 32
    padded := make([]byte, paddedLen)
    copy(padded, data)
    b.data = append(b.data, padded...)

    return b
}

// Build 构建最终 calldata
func (b *CalldataBuilder) Build() []byte {
    return b.data
}

// PackedEncoder 紧凑编码器 (非标准 ABI)
type PackedEncoder struct {
    data []byte
}

// NewPackedEncoder 创建紧凑编码器
func NewPackedEncoder() *PackedEncoder {
    return &PackedEncoder{
        data: make([]byte, 0),
    }
}

// AddUint8 添加 uint8
func (e *PackedEncoder) AddUint8(v uint8) *PackedEncoder {
    e.data = append(e.data, v)
    return e
}

// AddUint16 添加 uint16
func (e *PackedEncoder) AddUint16(v uint16) *PackedEncoder {
    e.data = append(e.data, byte(v>>8), byte(v))
    return e
}

// AddUint24 添加 uint24
func (e *PackedEncoder) AddUint24(v uint32) *PackedEncoder {
    e.data = append(e.data, byte(v>>16), byte(v>>8), byte(v))
    return e
}

// AddAddress 添加地址 (20 bytes)
func (e *PackedEncoder) AddAddress(addr common.Address) *PackedEncoder {
    e.data = append(e.data, addr.Bytes()...)
    return e
}

// Build 构建
func (e *PackedEncoder) Build() []byte {
    return e.data
}

// 示例: 紧凑编码的 swap 路径
// 标准 ABI: 每个地址 32 bytes, 每个 fee 32 bytes
// 紧凑编码: 每个地址 20 bytes, 每个 fee 3 bytes
func BuildPackedSwapPath(tokens []common.Address, fees []uint32) []byte {
    encoder := NewPackedEncoder()

    for i, token := range tokens {
        encoder.AddAddress(token)
        if i < len(fees) {
            encoder.AddUint24(fees[i])
        }
    }

    return encoder.Build()
}
```

### Gas 监控和策略

```go
package gasopt

import (
    "context"
    "math/big"
    "sort"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/ethclient"
)

// GasMonitor Gas 监控器
type GasMonitor struct {
    client *ethclient.Client

    // 历史数据
    history     []GasDataPoint
    historyMu   sync.RWMutex
    maxHistory  int

    // 当前状态
    currentGas  *GasState
    stateMu     sync.RWMutex
}

// GasDataPoint Gas 数据点
type GasDataPoint struct {
    Timestamp     time.Time
    BlockNumber   uint64
    BaseFee       *big.Int
    GasUsedRatio  float64
    PriorityFees  []*big.Int
}

// GasState 当前 Gas 状态
type GasState struct {
    BaseFee         *big.Int
    SafeLow         *big.Int // 慢速
    Standard        *big.Int // 标准
    Fast            *big.Int // 快速
    Instant         *big.Int // 即时
    PendingCount    uint64
    NextBaseFee     *big.Int
}

// NewGasMonitor 创建监控器
func NewGasMonitor(client *ethclient.Client) *GasMonitor {
    return &GasMonitor{
        client:     client,
        history:    make([]GasDataPoint, 0),
        maxHistory: 1000,
    }
}

// Start 启动监控
func (m *GasMonitor) Start(ctx context.Context) {
    go m.monitorLoop(ctx)
}

// monitorLoop 监控循环
func (m *GasMonitor) monitorLoop(ctx context.Context) {
    ticker := time.NewTicker(12 * time.Second) // 每个区块
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            m.updateGasState(ctx)
        }
    }
}

// updateGasState 更新 Gas 状态
func (m *GasMonitor) updateGasState(ctx context.Context) {
    // 获取最新区块
    block, err := m.client.BlockByNumber(ctx, nil)
    if err != nil {
        return
    }

    // 获取待处理交易
    pendingCount, _ := m.client.PendingTransactionCount(ctx)

    // 收集区块内的优先费
    var priorityFees []*big.Int
    for _, tx := range block.Transactions() {
        if tx.Type() == 2 { // EIP-1559
            priorityFees = append(priorityFees, tx.GasTipCap())
        }
    }

    // 排序
    sort.Slice(priorityFees, func(i, j int) bool {
        return priorityFees[i].Cmp(priorityFees[j]) < 0
    })

    // 计算百分位
    baseFee := block.BaseFee()
    state := &GasState{
        BaseFee:      baseFee,
        PendingCount: pendingCount,
    }

    if len(priorityFees) > 0 {
        state.SafeLow = m.percentile(priorityFees, 10)
        state.Standard = m.percentile(priorityFees, 50)
        state.Fast = m.percentile(priorityFees, 75)
        state.Instant = m.percentile(priorityFees, 95)
    } else {
        defaultTip := big.NewInt(1e9) // 1 Gwei
        state.SafeLow = defaultTip
        state.Standard = new(big.Int).Mul(defaultTip, big.NewInt(2))
        state.Fast = new(big.Int).Mul(defaultTip, big.NewInt(3))
        state.Instant = new(big.Int).Mul(defaultTip, big.NewInt(5))
    }

    // 预测下一个 baseFee
    gasUsed := block.GasUsed()
    gasLimit := block.GasLimit()
    state.NextBaseFee = m.predictNextBaseFee(baseFee, gasUsed, gasLimit)

    // 更新状态
    m.stateMu.Lock()
    m.currentGas = state
    m.stateMu.Unlock()

    // 记录历史
    m.recordHistory(block.NumberU64(), baseFee, float64(gasUsed)/float64(gasLimit), priorityFees)
}

// percentile 计算百分位
func (m *GasMonitor) percentile(values []*big.Int, p int) *big.Int {
    if len(values) == 0 {
        return big.NewInt(0)
    }

    index := len(values) * p / 100
    if index >= len(values) {
        index = len(values) - 1
    }

    return new(big.Int).Set(values[index])
}

// predictNextBaseFee 预测下一个 baseFee
func (m *GasMonitor) predictNextBaseFee(currentBaseFee *big.Int, gasUsed, gasLimit uint64) *big.Int {
    // EIP-1559 公式
    // nextBaseFee = currentBaseFee * (1 + 0.125 * (gasUsed - target) / target)
    // target = gasLimit / 2

    target := gasLimit / 2

    var delta *big.Int
    if gasUsed > target {
        // 增加
        diff := new(big.Int).SetUint64(gasUsed - target)
        delta = new(big.Int).Mul(currentBaseFee, diff)
        delta.Div(delta, new(big.Int).SetUint64(target))
        delta.Div(delta, big.NewInt(8)) // / 8 = * 0.125
    } else if gasUsed < target {
        // 减少
        diff := new(big.Int).SetUint64(target - gasUsed)
        delta = new(big.Int).Mul(currentBaseFee, diff)
        delta.Div(delta, new(big.Int).SetUint64(target))
        delta.Div(delta, big.NewInt(8))
        delta.Neg(delta)
    } else {
        delta = big.NewInt(0)
    }

    nextBaseFee := new(big.Int).Add(currentBaseFee, delta)

    // 最小值
    if nextBaseFee.Sign() <= 0 {
        nextBaseFee = big.NewInt(1)
    }

    return nextBaseFee
}

// recordHistory 记录历史
func (m *GasMonitor) recordHistory(
    blockNumber uint64,
    baseFee *big.Int,
    gasUsedRatio float64,
    priorityFees []*big.Int,
) {
    m.historyMu.Lock()
    defer m.historyMu.Unlock()

    m.history = append(m.history, GasDataPoint{
        Timestamp:    time.Now(),
        BlockNumber:  blockNumber,
        BaseFee:      new(big.Int).Set(baseFee),
        GasUsedRatio: gasUsedRatio,
        PriorityFees: priorityFees,
    })

    // 保持最大长度
    if len(m.history) > m.maxHistory {
        m.history = m.history[1:]
    }
}

// GetCurrentState 获取当前状态
func (m *GasMonitor) GetCurrentState() *GasState {
    m.stateMu.RLock()
    defer m.stateMu.RUnlock()

    if m.currentGas == nil {
        return nil
    }

    return &GasState{
        BaseFee:      new(big.Int).Set(m.currentGas.BaseFee),
        SafeLow:      new(big.Int).Set(m.currentGas.SafeLow),
        Standard:     new(big.Int).Set(m.currentGas.Standard),
        Fast:         new(big.Int).Set(m.currentGas.Fast),
        Instant:      new(big.Int).Set(m.currentGas.Instant),
        PendingCount: m.currentGas.PendingCount,
        NextBaseFee:  new(big.Int).Set(m.currentGas.NextBaseFee),
    }
}

// GetOptimalGasPrice 获取最优 Gas 价格
func (m *GasMonitor) GetOptimalGasPrice(urgency GasUrgency) (*big.Int, *big.Int) {
    state := m.GetCurrentState()
    if state == nil {
        return big.NewInt(30e9), big.NewInt(2e9) // 默认值
    }

    var priorityFee *big.Int
    switch urgency {
    case UrgencySafeLow:
        priorityFee = state.SafeLow
    case UrgencyStandard:
        priorityFee = state.Standard
    case UrgencyFast:
        priorityFee = state.Fast
    case UrgencyInstant:
        priorityFee = state.Instant
    default:
        priorityFee = state.Standard
    }

    // maxFee = 2 * nextBaseFee + priorityFee
    maxFee := new(big.Int).Mul(state.NextBaseFee, big.NewInt(2))
    maxFee.Add(maxFee, priorityFee)

    return maxFee, priorityFee
}

// GasUrgency Gas 紧急程度
type GasUrgency int

const (
    UrgencySafeLow GasUrgency = iota
    UrgencyStandard
    UrgencyFast
    UrgencyInstant
)

// ShouldWaitForLowerGas 判断是否应该等待更低的 Gas
func (m *GasMonitor) ShouldWaitForLowerGas(maxAcceptableGwei float64) bool {
    state := m.GetCurrentState()
    if state == nil {
        return false
    }

    currentGwei := new(big.Float).SetInt(state.BaseFee)
    currentGwei.Quo(currentGwei, big.NewFloat(1e9))
    currentGweiFloat, _ := currentGwei.Float64()

    return currentGweiFloat > maxAcceptableGwei
}

// GetHistoricalAverage 获取历史平均值
func (m *GasMonitor) GetHistoricalAverage(hours int) *big.Int {
    m.historyMu.RLock()
    defer m.historyMu.RUnlock()

    cutoff := time.Now().Add(-time.Duration(hours) * time.Hour)

    var sum *big.Int = big.NewInt(0)
    var count int64 = 0

    for _, point := range m.history {
        if point.Timestamp.After(cutoff) {
            sum.Add(sum, point.BaseFee)
            count++
        }
    }

    if count == 0 {
        return big.NewInt(0)
    }

    return new(big.Int).Div(sum, big.NewInt(count))
}
```

---

## 本文总结

本文深入探讨了 EVM Gas 优化技术，主要涵盖：

### Gas 成本速查表

| 操作 | Gas 成本 | 优化建议 |
|------|---------|---------|
| SLOAD (冷) | 2100 | 缓存到 memory |
| SLOAD (热) | 100 | - |
| SSTORE (0→非0) | 22100 | 避免存储 |
| SSTORE (非0→非0) | 2900 | 批量更新 |
| CALL (冷) | 2600 | 预热账户 |
| 零字节 calldata | 4 | 使用更多零字节 |
| 非零字节 calldata | 16 | 紧凑编码 |

### 优化策略

```
Gas 优化层次：
┌─────────────────────────────────────────────────────────────────┐
│                       Gas 优化策略                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  合约层                                                          │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • 存储打包    • 常量/不可变量   • 短字符串错误         │    │
│  │ • unchecked   • calldata       • 汇编优化             │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
│  交易层                                                          │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • 紧凑 calldata • EIP-1559     • Gas 价格监控         │    │
│  │ • 批量操作      • 时机选择     • 预热账户/存储        │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 最佳实践

1. **存储优化**: 使用紧凑存储布局，最小化 SLOAD/SSTORE
2. **计算优化**: 使用 unchecked，位操作代替乘除
3. **Calldata 优化**: 使用紧凑编码，最大化零字节
4. **时机优化**: 监控 Gas 价格，选择低峰期执行

---

## 附录 A：EIP-4844 Blob 交易优化

### A.1 EIP-4844 概述

EIP-4844 (Proto-Danksharding) 在 Dencun 升级中引入，为 Layer2 提供更低成本的数据可用性。

```go
// EIP-4844 Blob 交易支持
package gas

import (
	"context"
	"crypto/sha256"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto/kzg4844"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/params"
)

// Blob 相关常量
const (
	BlobTxType             = 0x03
	FieldElementsPerBlob   = 4096
	BlobSize               = FieldElementsPerBlob * 32  // 128 KB
	MaxBlobsPerBlock       = 6
	TargetBlobsPerBlock    = 3
	BlobTxBlobGasPerBlob   = 1 << 17  // 131072
	MinBlobBaseFee         = 1        // 1 wei
	BlobBaseFeeUpdateFraction = 3338477 // controls blob base fee volatility
)

// BlobTransaction Blob 交易结构
type BlobTransaction struct {
	ChainID    *big.Int
	Nonce      uint64
	GasTipCap  *big.Int  // maxPriorityFeePerGas
	GasFeeCap  *big.Int  // maxFeePerGas
	Gas        uint64
	To         common.Address
	Value      *big.Int
	Data       []byte

	// Blob 特有字段
	BlobFeeCap *big.Int        // maxFeePerBlobGas
	BlobHashes []common.Hash   // blob versioned hashes
	Sidecar    *BlobTxSidecar  // blobs + commitments + proofs
}

// BlobTxSidecar Blob 附属数据
type BlobTxSidecar struct {
	Blobs       []kzg4844.Blob       // 实际 blob 数据
	Commitments []kzg4844.Commitment // KZG commitments
	Proofs      []kzg4844.Proof      // KZG proofs
}

// BlobClient Blob 交易客户端
type BlobClient struct {
	client *ethclient.Client
}

func NewBlobClient(client *ethclient.Client) *BlobClient {
	return &BlobClient{client: client}
}

// GetBlobBaseFee 获取当前 Blob 基础费用
func (b *BlobClient) GetBlobBaseFee(ctx context.Context) (*big.Int, error) {
	header, err := b.client.HeaderByNumber(ctx, nil)
	if err != nil {
		return nil, err
	}

	// Dencun 升级后，header 包含 excessBlobGas
	if header.ExcessBlobGas == nil {
		return big.NewInt(MinBlobBaseFee), nil
	}

	return CalcBlobBaseFee(*header.ExcessBlobGas), nil
}

// CalcBlobBaseFee 计算 Blob 基础费用
// 使用指数函数: blobBaseFee = MIN_BLOB_BASE_FEE * e^(excessBlobGas / BLOB_BASE_FEE_UPDATE_FRACTION)
func CalcBlobBaseFee(excessBlobGas uint64) *big.Int {
	// 简化实现，实际使用 fake exponential
	if excessBlobGas == 0 {
		return big.NewInt(MinBlobBaseFee)
	}

	// fake_exponential(MIN_BLOB_BASE_FEE, excess_blob_gas, BLOB_BASE_FEE_UPDATE_FRACTION)
	return fakeExponential(
		big.NewInt(MinBlobBaseFee),
		new(big.Int).SetUint64(excessBlobGas),
		big.NewInt(BlobBaseFeeUpdateFraction),
	)
}

// fakeExponential 伪指数函数
// 返回 factor * e^(numerator / denominator)
func fakeExponential(factor, numerator, denominator *big.Int) *big.Int {
	i := big.NewInt(1)
	output := big.NewInt(0)
	numeratorAccum := new(big.Int).Set(factor)
	numeratorAccum.Mul(numeratorAccum, denominator)

	for numeratorAccum.Sign() > 0 {
		output.Add(output, numeratorAccum)

		// numeratorAccum = numeratorAccum * numerator / (denominator * i)
		numeratorAccum.Mul(numeratorAccum, numerator)
		numeratorAccum.Div(numeratorAccum, denominator)
		numeratorAccum.Div(numeratorAccum, i)

		i.Add(i, big.NewInt(1))
	}

	return output.Div(output, denominator)
}

// PredictNextBlobBaseFee 预测下一个 Blob 基础费用
func (b *BlobClient) PredictNextBlobBaseFee(ctx context.Context) (*big.Int, error) {
	header, err := b.client.HeaderByNumber(ctx, nil)
	if err != nil {
		return nil, err
	}

	if header.ExcessBlobGas == nil || header.BlobGasUsed == nil {
		return big.NewInt(MinBlobBaseFee), nil
	}

	// 计算下一个区块的 excessBlobGas
	nextExcessBlobGas := CalcNextExcessBlobGas(
		*header.ExcessBlobGas,
		*header.BlobGasUsed,
	)

	return CalcBlobBaseFee(nextExcessBlobGas), nil
}

// CalcNextExcessBlobGas 计算下一个区块的 excessBlobGas
func CalcNextExcessBlobGas(parentExcessBlobGas, parentBlobGasUsed uint64) uint64 {
	targetBlobGasPerBlock := uint64(TargetBlobsPerBlock * BlobTxBlobGasPerBlob)

	excessBlobGas := parentExcessBlobGas + parentBlobGasUsed
	if excessBlobGas < targetBlobGasPerBlock {
		return 0
	}
	return excessBlobGas - targetBlobGasPerBlock
}

// CreateBlob 创建 Blob 数据
func CreateBlob(data []byte) (*kzg4844.Blob, *kzg4844.Commitment, *kzg4844.Proof, error) {
	var blob kzg4844.Blob

	// 数据必须小于 BlobSize
	if len(data) > BlobSize {
		return nil, nil, nil, fmt.Errorf("data too large for blob: %d > %d", len(data), BlobSize)
	}

	// 填充 blob
	copy(blob[:], data)

	// 生成 KZG commitment
	commitment, err := kzg4844.BlobToCommitment(blob)
	if err != nil {
		return nil, nil, nil, err
	}

	// 生成 KZG proof
	proof, err := kzg4844.ComputeBlobProof(blob, commitment)
	if err != nil {
		return nil, nil, nil, err
	}

	return &blob, &commitment, &proof, nil
}

// VersionedHash 计算 versioned hash
func VersionedHash(commitment kzg4844.Commitment) common.Hash {
	h := sha256.Sum256(commitment[:])
	h[0] = 0x01 // version byte
	return common.Hash(h)
}

// EstimateBlobGasCost 估算 Blob Gas 成本
func (b *BlobClient) EstimateBlobGasCost(ctx context.Context, numBlobs int) (*big.Int, error) {
	blobBaseFee, err := b.GetBlobBaseFee(ctx)
	if err != nil {
		return nil, err
	}

	// blob gas = numBlobs * BlobTxBlobGasPerBlob
	blobGas := uint64(numBlobs) * BlobTxBlobGasPerBlob

	// cost = blobGas * blobBaseFee
	cost := new(big.Int).SetUint64(blobGas)
	cost.Mul(cost, blobBaseFee)

	return cost, nil
}

// CompareCalldataVsBlob 比较 Calldata 和 Blob 成本
func (b *BlobClient) CompareCalldataVsBlob(ctx context.Context, dataSize int) (*CostComparison, error) {
	// 获取当前 Gas 价格
	header, err := b.client.HeaderByNumber(ctx, nil)
	if err != nil {
		return nil, err
	}

	baseFee := header.BaseFee
	blobBaseFee, _ := b.GetBlobBaseFee(ctx)

	// Calldata 成本
	// 假设 50% 零字节，50% 非零字节
	zeroBytes := dataSize / 2
	nonZeroBytes := dataSize - zeroBytes
	calldataGas := uint64(zeroBytes*4 + nonZeroBytes*16)
	calldataCost := new(big.Int).SetUint64(calldataGas)
	calldataCost.Mul(calldataCost, baseFee)

	// Blob 成本
	numBlobs := (dataSize + BlobSize - 1) / BlobSize // 向上取整
	if numBlobs > MaxBlobsPerBlock {
		numBlobs = MaxBlobsPerBlock
	}
	blobGas := uint64(numBlobs) * BlobTxBlobGasPerBlob
	blobCost := new(big.Int).SetUint64(blobGas)
	blobCost.Mul(blobCost, blobBaseFee)

	savings := new(big.Int).Sub(calldataCost, blobCost)
	savingsPercent := float64(0)
	if calldataCost.Sign() > 0 {
		savingsPercent = float64(savings.Int64()) / float64(calldataCost.Int64()) * 100
	}

	return &CostComparison{
		DataSize:        dataSize,
		CalldataGas:     calldataGas,
		CalldataCost:    calldataCost,
		NumBlobs:        numBlobs,
		BlobGas:         blobGas,
		BlobCost:        blobCost,
		Savings:         savings,
		SavingsPercent:  savingsPercent,
		RecommendBlob:   blobCost.Cmp(calldataCost) < 0,
	}, nil
}

// CostComparison 成本对比结果
type CostComparison struct {
	DataSize        int
	CalldataGas     uint64
	CalldataCost    *big.Int
	NumBlobs        int
	BlobGas         uint64
	BlobCost        *big.Int
	Savings         *big.Int
	SavingsPercent  float64
	RecommendBlob   bool
}
```

### A.2 Blob 交易构建

```go
// Blob 交易构建器
package gas

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/crypto/kzg4844"
	"github.com/ethereum/go-ethereum/ethclient"
)

// BlobTxBuilder Blob 交易构建器
type BlobTxBuilder struct {
	client     *ethclient.Client
	blobClient *BlobClient
	privateKey *ecdsa.PrivateKey
	chainID    *big.Int
}

func NewBlobTxBuilder(
	client *ethclient.Client,
	privateKey *ecdsa.PrivateKey,
	chainID *big.Int,
) *BlobTxBuilder {
	return &BlobTxBuilder{
		client:     client,
		blobClient: NewBlobClient(client),
		privateKey: privateKey,
		chainID:    chainID,
	}
}

// BuildBlobTx 构建 Blob 交易
func (b *BlobTxBuilder) BuildBlobTx(
	ctx context.Context,
	to common.Address,
	value *big.Int,
	data []byte,
	blobData [][]byte,  // 多个 blob 的数据
) (*types.Transaction, error) {
	// 获取 nonce
	from := crypto.PubkeyToAddress(b.privateKey.PublicKey)
	nonce, err := b.client.PendingNonceAt(ctx, from)
	if err != nil {
		return nil, err
	}

	// 获取 Gas 价格
	header, err := b.client.HeaderByNumber(ctx, nil)
	if err != nil {
		return nil, err
	}

	baseFee := header.BaseFee
	tip := big.NewInt(1e9) // 1 Gwei tip

	maxFeePerGas := new(big.Int).Mul(baseFee, big.NewInt(2))
	maxFeePerGas.Add(maxFeePerGas, tip)

	// 获取 Blob Gas 价格
	blobBaseFee, err := b.blobClient.GetBlobBaseFee(ctx)
	if err != nil {
		return nil, err
	}

	// maxFeePerBlobGas = 2 * currentBlobBaseFee
	maxFeePerBlobGas := new(big.Int).Mul(blobBaseFee, big.NewInt(2))

	// 构建 blobs
	var sidecar types.BlobTxSidecar
	var blobHashes []common.Hash

	for _, bd := range blobData {
		blob, commitment, proof, err := CreateBlob(bd)
		if err != nil {
			return nil, err
		}

		sidecar.Blobs = append(sidecar.Blobs, *blob)
		sidecar.Commitments = append(sidecar.Commitments, *commitment)
		sidecar.Proofs = append(sidecar.Proofs, *proof)
		blobHashes = append(blobHashes, VersionedHash(*commitment))
	}

	// 估算 Gas
	gasLimit, err := b.client.EstimateGas(ctx, ethereum.CallMsg{
		From:  from,
		To:    &to,
		Value: value,
		Data:  data,
	})
	if err != nil {
		gasLimit = 100000 // 默认值
	}

	// 构建交易
	tx := types.NewTx(&types.BlobTx{
		ChainID:    uint256.NewInt(b.chainID.Uint64()),
		Nonce:      nonce,
		GasTipCap:  uint256.NewInt(tip.Uint64()),
		GasFeeCap:  uint256.NewInt(maxFeePerGas.Uint64()),
		Gas:        gasLimit,
		To:         to,
		Value:      uint256.NewInt(value.Uint64()),
		Data:       data,
		BlobFeeCap: uint256.NewInt(maxFeePerBlobGas.Uint64()),
		BlobHashes: blobHashes,
		Sidecar:    &sidecar,
	})

	// 签名
	signedTx, err := types.SignTx(tx, types.NewCancunSigner(b.chainID), b.privateKey)
	if err != nil {
		return nil, err
	}

	return signedTx, nil
}

// SendBlobTx 发送 Blob 交易
func (b *BlobTxBuilder) SendBlobTx(ctx context.Context, tx *types.Transaction) (common.Hash, error) {
	err := b.client.SendTransaction(ctx, tx)
	if err != nil {
		return common.Hash{}, err
	}
	return tx.Hash(), nil
}

// L2 数据提交示例
const L2DataSubmissionExample = `
// Layer2 使用 Blob 提交数据示例
func submitL2Data(builder *BlobTxBuilder, l2Data []byte) error {
    ctx := context.Background()

    // 将 L2 数据分割成多个 blob
    var blobData [][]byte
    for i := 0; i < len(l2Data); i += BlobSize {
        end := i + BlobSize
        if end > len(l2Data) {
            end = len(l2Data)
        }
        blobData = append(blobData, l2Data[i:end])
    }

    // 构建并发送交易
    tx, err := builder.BuildBlobTx(
        ctx,
        sequencerInbox,  // L2 Sequencer Inbox 合约
        big.NewInt(0),
        nil,             // 无 calldata
        blobData,
    )
    if err != nil {
        return err
    }

    hash, err := builder.SendBlobTx(ctx, tx)
    if err != nil {
        return err
    }

    log.Printf("L2 data submitted with blob tx: %s", hash.Hex())
    return nil
}
`
```

---

## 附录 B：EIP-2930 Access List 优化

### B.1 Access List 概述

EIP-2930 引入访问列表，允许预声明交易将访问的地址和存储槽，减少冷访问成本。

```go
// EIP-2930 Access List 支持
package gas

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
)

// 冷/热访问 Gas 成本
const (
	ColdAccountAccessCost  = 2600  // 冷地址访问
	ColdSloadCost          = 2100  // 冷存储读取
	WarmAccessCost         = 100   // 热访问
	AccessListAddressCost  = 2400  // Access List 地址成本
	AccessListStorageCost  = 1900  // Access List 存储槽成本
)

// AccessListOptimizer Access List 优化器
type AccessListOptimizer struct {
	client *ethclient.Client
}

func NewAccessListOptimizer(client *ethclient.Client) *AccessListOptimizer {
	return &AccessListOptimizer{client: client}
}

// CreateAccessList 自动创建 Access List
func (a *AccessListOptimizer) CreateAccessList(
	ctx context.Context,
	from common.Address,
	to *common.Address,
	data []byte,
	value *big.Int,
) (types.AccessList, uint64, error) {
	msg := ethereum.CallMsg{
		From:  from,
		To:    to,
		Data:  data,
		Value: value,
	}

	// 使用 eth_createAccessList RPC
	result, err := a.client.CreateAccessList(ctx, msg)
	if err != nil {
		return nil, 0, err
	}

	return result.AccessList, result.GasUsed, nil
}

// OptimizeAccessList 优化 Access List
// 移除不划算的条目
func (a *AccessListOptimizer) OptimizeAccessList(
	ctx context.Context,
	accessList types.AccessList,
	from common.Address,
	to *common.Address,
	data []byte,
	value *big.Int,
) (types.AccessList, int64, error) {
	// 计算原始 Gas (无 Access List)
	originalGas, err := a.client.EstimateGas(ctx, ethereum.CallMsg{
		From:  from,
		To:    to,
		Data:  data,
		Value: value,
	})
	if err != nil {
		return nil, 0, err
	}

	// 计算带 Access List 的 Gas
	withAccessListGas, err := a.client.EstimateGas(ctx, ethereum.CallMsg{
		From:       from,
		To:         to,
		Data:       data,
		Value:      value,
		AccessList: accessList,
	})
	if err != nil {
		return nil, 0, err
	}

	savings := int64(originalGas) - int64(withAccessListGas)

	// 如果节省不明显，尝试精简 Access List
	if savings < 1000 {
		optimized := a.pruneAccessList(accessList)
		return optimized, savings, nil
	}

	return accessList, savings, nil
}

// pruneAccessList 精简 Access List
func (a *AccessListOptimizer) pruneAccessList(al types.AccessList) types.AccessList {
	var pruned types.AccessList

	for _, entry := range al {
		// 计算这个条目的成本
		entryCost := AccessListAddressCost
		entryCost += len(entry.StorageKeys) * AccessListStorageCost

		// 计算节省（假设每个都是冷访问）
		savings := ColdAccountAccessCost - WarmAccessCost // 地址节省
		savings += len(entry.StorageKeys) * (ColdSloadCost - WarmAccessCost)

		// 只保留划算的条目
		if savings > entryCost {
			pruned = append(pruned, entry)
		}
	}

	return pruned
}

// AnalyzeAccessListSavings 分析 Access List 节省
func (a *AccessListOptimizer) AnalyzeAccessListSavings(al types.AccessList) *AccessListAnalysis {
	analysis := &AccessListAnalysis{
		TotalAddresses:   len(al),
		TotalStorageKeys: 0,
	}

	for _, entry := range al {
		analysis.TotalStorageKeys += len(entry.StorageKeys)
	}

	// 计算 Access List 成本
	analysis.AccessListCost = analysis.TotalAddresses * AccessListAddressCost
	analysis.AccessListCost += analysis.TotalStorageKeys * AccessListStorageCost

	// 计算节省（从冷变热）
	// 地址节省: 2600 - 100 = 2500，但 AL 成本 2400，净节省 100
	analysis.AddressSavings = analysis.TotalAddresses * (ColdAccountAccessCost - WarmAccessCost - AccessListAddressCost)
	// 存储节省: 2100 - 100 = 2000，但 AL 成本 1900，净节省 100
	analysis.StorageSavings = analysis.TotalStorageKeys * (ColdSloadCost - WarmAccessCost - AccessListStorageCost)

	analysis.TotalSavings = analysis.AddressSavings + analysis.StorageSavings

	return analysis
}

// AccessListAnalysis Access List 分析结果
type AccessListAnalysis struct {
	TotalAddresses   int
	TotalStorageKeys int
	AccessListCost   int
	AddressSavings   int
	StorageSavings   int
	TotalSavings     int
}

// BuildAccessListTx 构建带 Access List 的交易
func BuildAccessListTx(
	chainID *big.Int,
	nonce uint64,
	to common.Address,
	value *big.Int,
	gasLimit uint64,
	gasPrice *big.Int,
	data []byte,
	accessList types.AccessList,
) *types.Transaction {
	return types.NewTx(&types.AccessListTx{
		ChainID:    chainID,
		Nonce:      nonce,
		To:         &to,
		Value:      value,
		Gas:        gasLimit,
		GasPrice:   gasPrice,
		Data:       data,
		AccessList: accessList,
	})
}

// 常见场景的 Access List 预设
var CommonAccessLists = map[string]types.AccessList{
	"uniswap_v2_swap": {
		{
			Address: common.HexToAddress("0x..."), // Router
			StorageKeys: []common.Hash{},
		},
		{
			Address: common.HexToAddress("0x..."), // Pair
			StorageKeys: []common.Hash{
				common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000008"), // reserve0, reserve1
			},
		},
	},
	"uniswap_v3_swap": {
		{
			Address: common.HexToAddress("0x..."), // Router
			StorageKeys: []common.Hash{},
		},
		{
			Address: common.HexToAddress("0x..."), // Pool
			StorageKeys: []common.Hash{
				common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000000"), // slot0
				common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000004"), // liquidity
			},
		},
	},
}
```

### B.2 Access List 实战应用

```go
// Access List 在套利中的应用
package gas

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
)

// ArbitrageAccessListBuilder 套利 Access List 构建器
type ArbitrageAccessListBuilder struct {
	client    *ethclient.Client
	optimizer *AccessListOptimizer
}

func NewArbitrageAccessListBuilder(client *ethclient.Client) *ArbitrageAccessListBuilder {
	return &ArbitrageAccessListBuilder{
		client:    client,
		optimizer: NewAccessListOptimizer(client),
	}
}

// BuildForDEXArbitrage 为 DEX 套利构建 Access List
func (b *ArbitrageAccessListBuilder) BuildForDEXArbitrage(
	ctx context.Context,
	path []common.Address,  // [tokenA, pool1, tokenB, pool2, tokenA]
) (types.AccessList, error) {
	var accessList types.AccessList

	for _, addr := range path {
		// 获取合约的常用存储槽
		slots, err := b.getCommonStorageSlots(ctx, addr)
		if err != nil {
			continue
		}

		accessList = append(accessList, types.AccessTuple{
			Address:     addr,
			StorageKeys: slots,
		})
	}

	return accessList, nil
}

// getCommonStorageSlots 获取常用存储槽
func (b *ArbitrageAccessListBuilder) getCommonStorageSlots(
	ctx context.Context,
	addr common.Address,
) ([]common.Hash, error) {
	// 这里可以根据合约类型返回预定义的存储槽
	// 实际实现需要识别合约类型

	// Uniswap V2 Pair 常用槽
	uniV2Slots := []common.Hash{
		common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000006"), // token0
		common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000007"), // token1
		common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000008"), // reserves
		common.HexToHash("0x000000000000000000000000000000000000000000000000000000000000000c"), // kLast
	}

	return uniV2Slots, nil
}

// PrewarmContracts 预热合约（通过 Access List）
func (b *ArbitrageAccessListBuilder) PrewarmContracts(
	ctx context.Context,
	contracts []common.Address,
) types.AccessList {
	var accessList types.AccessList

	for _, contract := range contracts {
		accessList = append(accessList, types.AccessTuple{
			Address:     contract,
			StorageKeys: []common.Hash{}, // 只预热地址
		})
	}

	return accessList
}

// 实际节省计算示例
const AccessListSavingsExample = `
// 示例：Uniswap V2 Swap

// 无 Access List:
// - Router 冷访问: 2600 gas
// - Pair 冷访问: 2600 gas
// - 第一次 SLOAD reserve: 2100 gas
// 总计冷访问: 7300 gas

// 有 Access List:
// - Router 在 AL: 2400 gas (预付) + 100 gas (访问) = 2500 gas
// - Pair 在 AL: 2400 gas + 100 gas = 2500 gas
// - reserve 在 AL: 1900 gas + 100 gas = 2000 gas
// 总计: 7000 gas

// 节省: 300 gas
// 但如果同一笔交易多次访问，节省更多

// 多次访问场景（如套利路径）:
// - 3 个 pool，每个访问 2 次
// 无 AL: 3 * 2600 + 3 * 2100 = 14100 gas
// 有 AL: 3 * 2400 + 3 * 1900 + 3 * 100 * 2 + 3 * 100 * 2 = 14100 gas
// 实际节省来自热访问: 每次热访问只需 100 gas
`
```

---

## 附录 C：EIP-1559 高级费用策略

### C.1 动态费用估算

```go
// EIP-1559 高级费用策略
package gas

import (
	"context"
	"math/big"
	"sort"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
)

// EIP1559FeeEstimator EIP-1559 费用估算器
type EIP1559FeeEstimator struct {
	client *ethclient.Client

	// 历史数据
	feeHistory    []FeeHistoryBlock
	historyMu     sync.RWMutex
	historyBlocks int
}

// FeeHistoryBlock 区块费用历史
type FeeHistoryBlock struct {
	BlockNumber  uint64
	BaseFee      *big.Int
	GasUsedRatio float64
	Rewards      []*big.Int // 不同百分位的奖励
}

// FeeEstimate 费用估算结果
type FeeEstimate struct {
	BaseFee           *big.Int
	MaxPriorityFee    *big.Int
	MaxFee            *big.Int
	EstimatedWaitTime time.Duration
	Confidence        float64  // 0-1
}

func NewEIP1559FeeEstimator(client *ethclient.Client) *EIP1559FeeEstimator {
	return &EIP1559FeeEstimator{
		client:        client,
		historyBlocks: 20,
	}
}

// GetFeeHistory 获取费用历史
func (e *EIP1559FeeEstimator) GetFeeHistory(ctx context.Context) error {
	// 使用 eth_feeHistory RPC
	history, err := e.client.FeeHistory(
		ctx,
		uint64(e.historyBlocks),
		nil, // 最新区块
		[]float64{10, 25, 50, 75, 90}, // 百分位
	)
	if err != nil {
		return err
	}

	e.historyMu.Lock()
	defer e.historyMu.Unlock()

	e.feeHistory = make([]FeeHistoryBlock, len(history.BaseFee)-1)
	for i := 0; i < len(history.BaseFee)-1; i++ {
		rewards := make([]*big.Int, len(history.Reward[i]))
		for j, r := range history.Reward[i] {
			rewards[j] = r
		}

		e.feeHistory[i] = FeeHistoryBlock{
			BlockNumber:  history.OldestBlock.Uint64() + uint64(i),
			BaseFee:      history.BaseFee[i],
			GasUsedRatio: history.GasUsedRatio[i],
			Rewards:      rewards,
		}
	}

	return nil
}

// EstimateFee 估算费用
func (e *EIP1559FeeEstimator) EstimateFee(ctx context.Context, urgency FeeUrgency) (*FeeEstimate, error) {
	// 更新历史
	if err := e.GetFeeHistory(ctx); err != nil {
		return nil, err
	}

	e.historyMu.RLock()
	defer e.historyMu.RUnlock()

	if len(e.feeHistory) == 0 {
		return e.getDefaultEstimate(ctx)
	}

	// 获取当前 baseFee
	header, err := e.client.HeaderByNumber(ctx, nil)
	if err != nil {
		return nil, err
	}
	currentBaseFee := header.BaseFee

	// 预测下一个 baseFee
	nextBaseFee := e.predictBaseFee(currentBaseFee, e.feeHistory[len(e.feeHistory)-1].GasUsedRatio)

	// 根据紧急程度选择优先费
	var priorityFee *big.Int
	var waitTime time.Duration
	var confidence float64

	switch urgency {
	case FeeUrgencySlow:
		priorityFee = e.getPercentilePriorityFee(10)
		waitTime = 5 * time.Minute
		confidence = 0.7
	case FeeUrgencyMedium:
		priorityFee = e.getPercentilePriorityFee(50)
		waitTime = 1 * time.Minute
		confidence = 0.85
	case FeeUrgencyFast:
		priorityFee = e.getPercentilePriorityFee(75)
		waitTime = 15 * time.Second
		confidence = 0.95
	case FeeUrgencyInstant:
		priorityFee = e.getPercentilePriorityFee(90)
		waitTime = 12 * time.Second
		confidence = 0.99
	}

	// 计算 maxFee
	// maxFee = 2 * nextBaseFee + priorityFee (适应 baseFee 波动)
	maxFee := new(big.Int).Mul(nextBaseFee, big.NewInt(2))
	maxFee.Add(maxFee, priorityFee)

	return &FeeEstimate{
		BaseFee:           nextBaseFee,
		MaxPriorityFee:    priorityFee,
		MaxFee:            maxFee,
		EstimatedWaitTime: waitTime,
		Confidence:        confidence,
	}, nil
}

// predictBaseFee 预测下一个 baseFee
func (e *EIP1559FeeEstimator) predictBaseFee(currentBaseFee *big.Int, gasUsedRatio float64) *big.Int {
	// EIP-1559: baseFee 变化 ±12.5%
	// 如果 gasUsedRatio > 0.5，baseFee 增加
	// 如果 gasUsedRatio < 0.5，baseFee 减少

	delta := gasUsedRatio - 0.5
	changePercent := delta * 0.25 // 最大 ±12.5%

	change := new(big.Float).SetInt(currentBaseFee)
	change.Mul(change, big.NewFloat(changePercent))

	changeInt, _ := change.Int(nil)
	nextBaseFee := new(big.Int).Add(currentBaseFee, changeInt)

	// 最小值
	if nextBaseFee.Cmp(big.NewInt(1)) < 0 {
		nextBaseFee = big.NewInt(1)
	}

	return nextBaseFee
}

// getPercentilePriorityFee 获取百分位优先费
func (e *EIP1559FeeEstimator) getPercentilePriorityFee(percentile int) *big.Int {
	var allFees []*big.Int

	for _, block := range e.feeHistory {
		for _, fee := range block.Rewards {
			if fee.Sign() > 0 {
				allFees = append(allFees, fee)
			}
		}
	}

	if len(allFees) == 0 {
		return big.NewInt(1e9) // 1 Gwei default
	}

	sort.Slice(allFees, func(i, j int) bool {
		return allFees[i].Cmp(allFees[j]) < 0
	})

	index := len(allFees) * percentile / 100
	if index >= len(allFees) {
		index = len(allFees) - 1
	}

	return new(big.Int).Set(allFees[index])
}

// FeeUrgency 费用紧急程度
type FeeUrgency int

const (
	FeeUrgencySlow FeeUrgency = iota
	FeeUrgencyMedium
	FeeUrgencyFast
	FeeUrgencyInstant
)

// getDefaultEstimate 默认估算
func (e *EIP1559FeeEstimator) getDefaultEstimate(ctx context.Context) (*FeeEstimate, error) {
	header, err := e.client.HeaderByNumber(ctx, nil)
	if err != nil {
		return nil, err
	}

	baseFee := header.BaseFee
	priorityFee := big.NewInt(2e9) // 2 Gwei

	maxFee := new(big.Int).Mul(baseFee, big.NewInt(2))
	maxFee.Add(maxFee, priorityFee)

	return &FeeEstimate{
		BaseFee:           baseFee,
		MaxPriorityFee:    priorityFee,
		MaxFee:            maxFee,
		EstimatedWaitTime: 30 * time.Second,
		Confidence:        0.8,
	}, nil
}
```

### C.2 MEV 感知费用策略

```go
// MEV 感知的费用策略
package gas

import (
	"context"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// MEVAwareFeeStrategy MEV 感知费用策略
type MEVAwareFeeStrategy struct {
	client         *ethclient.Client
	feeEstimator   *EIP1559FeeEstimator
	mevProtection  bool
}

func NewMEVAwareFeeStrategy(client *ethclient.Client) *MEVAwareFeeStrategy {
	return &MEVAwareFeeStrategy{
		client:       client,
		feeEstimator: NewEIP1559FeeEstimator(client),
	}
}

// GetFeeForArbitrage 获取套利交易的费用
func (m *MEVAwareFeeStrategy) GetFeeForArbitrage(
	ctx context.Context,
	expectedProfit *big.Int,
	competitionLevel CompetitionLevel,
) (*ArbitrageFeeEstimate, error) {
	// 基础费用估算
	baseEstimate, err := m.feeEstimator.EstimateFee(ctx, FeeUrgencyInstant)
	if err != nil {
		return nil, err
	}

	// 根据竞争程度调整
	var priorityMultiplier float64
	var profitSharePercent float64

	switch competitionLevel {
	case CompetitionLow:
		priorityMultiplier = 1.0
		profitSharePercent = 10 // 愿意分享 10% 利润给矿工
	case CompetitionMedium:
		priorityMultiplier = 1.5
		profitSharePercent = 30
	case CompetitionHigh:
		priorityMultiplier = 2.5
		profitSharePercent = 50
	case CompetitionExtreme:
		priorityMultiplier = 5.0
		profitSharePercent = 70
	}

	// 计算最大可接受的优先费
	// maxPriorityFee = min(baseEstimate * multiplier, profit * sharePercent / gasLimit)
	adjustedPriority := new(big.Float).SetInt(baseEstimate.MaxPriorityFee)
	adjustedPriority.Mul(adjustedPriority, big.NewFloat(priorityMultiplier))
	adjustedPriorityInt, _ := adjustedPriority.Int(nil)

	// 基于利润的上限
	profitShare := new(big.Float).SetInt(expectedProfit)
	profitShare.Mul(profitShare, big.NewFloat(profitSharePercent/100))
	profitShareInt, _ := profitShare.Int(nil)

	// 取较小值
	finalPriority := adjustedPriorityInt
	if profitShareInt.Cmp(adjustedPriorityInt) < 0 {
		finalPriority = profitShareInt
	}

	// 计算 maxFee
	maxFee := new(big.Int).Mul(baseEstimate.BaseFee, big.NewInt(2))
	maxFee.Add(maxFee, finalPriority)

	return &ArbitrageFeeEstimate{
		BaseFee:              baseEstimate.BaseFee,
		RecommendedPriority:  finalPriority,
		MaxFee:               maxFee,
		ExpectedProfit:       expectedProfit,
		NetProfit:            new(big.Int).Sub(expectedProfit, finalPriority),
		CompetitionLevel:     competitionLevel,
	}, nil
}

// ArbitrageFeeEstimate 套利费用估算
type ArbitrageFeeEstimate struct {
	BaseFee              *big.Int
	RecommendedPriority  *big.Int
	MaxFee               *big.Int
	ExpectedProfit       *big.Int
	NetProfit            *big.Int
	CompetitionLevel     CompetitionLevel
}

// CompetitionLevel 竞争程度
type CompetitionLevel int

const (
	CompetitionLow CompetitionLevel = iota
	CompetitionMedium
	CompetitionHigh
	CompetitionExtreme
)

// DetectCompetitionLevel 检测竞争程度
func (m *MEVAwareFeeStrategy) DetectCompetitionLevel(
	ctx context.Context,
	opportunity common.Hash, // 机会的唯一标识
) (CompetitionLevel, error) {
	// 检查 mempool 中是否有类似交易
	// 实际实现需要 mempool 访问

	// 简化实现：基于历史数据估算
	return CompetitionMedium, nil
}

// DynamicFeeAdjuster 动态费用调整器
type DynamicFeeAdjuster struct {
	strategy       *MEVAwareFeeStrategy
	initialFee     *big.Int
	maxFee         *big.Int
	stepIncrease   *big.Int
	attempts       int
	maxAttempts    int
}

// NewDynamicFeeAdjuster 创建动态费用调整器
func NewDynamicFeeAdjuster(
	strategy *MEVAwareFeeStrategy,
	initialFee, maxFee *big.Int,
	maxAttempts int,
) *DynamicFeeAdjuster {
	step := new(big.Int).Sub(maxFee, initialFee)
	step.Div(step, big.NewInt(int64(maxAttempts)))

	return &DynamicFeeAdjuster{
		strategy:     strategy,
		initialFee:   initialFee,
		maxFee:       maxFee,
		stepIncrease: step,
		maxAttempts:  maxAttempts,
	}
}

// GetNextFee 获取下一个费用（用于重试）
func (d *DynamicFeeAdjuster) GetNextFee() (*big.Int, bool) {
	if d.attempts >= d.maxAttempts {
		return nil, false
	}

	fee := new(big.Int).Set(d.initialFee)
	increment := new(big.Int).Mul(d.stepIncrease, big.NewInt(int64(d.attempts)))
	fee.Add(fee, increment)

	if fee.Cmp(d.maxFee) > 0 {
		fee = d.maxFee
	}

	d.attempts++
	return fee, true
}

// 费用策略总结
const FeeStrategySummary = `
EIP-1559 费用策略要点：

1. BaseFee 预测
   - 监控 gasUsedRatio
   - 预测 1-2 个区块后的 baseFee
   - 为波动留出余量 (2x baseFee)

2. PriorityFee 选择
   - 低优先级 (10%): 适合非紧急交易
   - 中优先级 (50%): 标准交易
   - 高优先级 (75%): 需要快速确认
   - 即时 (90%): 套利/清算

3. MEV 感知调整
   - 检测竞争程度
   - 根据利润调整优先费
   - 动态重试机制

4. 最佳实践
   - 始终设置合理的 maxFee 上限
   - 监控交易状态，必要时替换
   - 考虑使用 Flashbots 避免竞争
`
```

### C.3 Gas 价格预警系统

```go
// Gas 价格预警系统
package gas

import (
	"context"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/ethclient"
)

// GasAlertSystem Gas 预警系统
type GasAlertSystem struct {
	client      *ethclient.Client
	estimator   *EIP1559FeeEstimator

	// 阈值配置
	lowThreshold  *big.Int  // 低于此值触发买入提醒
	highThreshold *big.Int  // 高于此值触发等待提醒

	// 订阅者
	subscribers []AlertSubscriber
	subMu       sync.RWMutex

	// 状态
	lastAlert   time.Time
	alertCooldown time.Duration
}

// AlertSubscriber 预警订阅者
type AlertSubscriber interface {
	OnGasAlert(alert *GasAlert)
}

// GasAlert 预警信息
type GasAlert struct {
	Type        AlertType
	CurrentGas  *big.Int
	Threshold   *big.Int
	Suggestion  string
	Timestamp   time.Time
}

// AlertType 预警类型
type AlertType int

const (
	AlertTypeLow AlertType = iota    // Gas 很低，适合发送交易
	AlertTypeHigh                    // Gas 很高，建议等待
	AlertTypeSpike                   // Gas 突然飙升
	AlertTypeDrop                    // Gas 突然下降
)

func NewGasAlertSystem(client *ethclient.Client) *GasAlertSystem {
	return &GasAlertSystem{
		client:        client,
		estimator:     NewEIP1559FeeEstimator(client),
		lowThreshold:  big.NewInt(10e9),  // 10 Gwei
		highThreshold: big.NewInt(50e9),  // 50 Gwei
		alertCooldown: 5 * time.Minute,
	}
}

// SetThresholds 设置阈值
func (g *GasAlertSystem) SetThresholds(low, high *big.Int) {
	g.lowThreshold = low
	g.highThreshold = high
}

// Subscribe 订阅预警
func (g *GasAlertSystem) Subscribe(sub AlertSubscriber) {
	g.subMu.Lock()
	defer g.subMu.Unlock()
	g.subscribers = append(g.subscribers, sub)
}

// Start 启动监控
func (g *GasAlertSystem) Start(ctx context.Context) {
	go g.monitorLoop(ctx)
}

// monitorLoop 监控循环
func (g *GasAlertSystem) monitorLoop(ctx context.Context) {
	ticker := time.NewTicker(12 * time.Second)
	defer ticker.Stop()

	var lastBaseFee *big.Int

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			g.checkAndAlert(ctx, lastBaseFee)
		}
	}
}

// checkAndAlert 检查并发送预警
func (g *GasAlertSystem) checkAndAlert(ctx context.Context, lastBaseFee *big.Int) {
	header, err := g.client.HeaderByNumber(ctx, nil)
	if err != nil {
		return
	}

	currentBaseFee := header.BaseFee

	// 检查是否需要预警
	var alert *GasAlert

	// 低 Gas 预警
	if currentBaseFee.Cmp(g.lowThreshold) < 0 {
		alert = &GasAlert{
			Type:       AlertTypeLow,
			CurrentGas: currentBaseFee,
			Threshold:  g.lowThreshold,
			Suggestion: "Gas 价格很低，适合发送非紧急交易",
			Timestamp:  time.Now(),
		}
	}

	// 高 Gas 预警
	if currentBaseFee.Cmp(g.highThreshold) > 0 {
		alert = &GasAlert{
			Type:       AlertTypeHigh,
			CurrentGas: currentBaseFee,
			Threshold:  g.highThreshold,
			Suggestion: "Gas 价格较高，建议等待或使用低优先级",
			Timestamp:  time.Now(),
		}
	}

	// 突变检测
	if lastBaseFee != nil {
		change := new(big.Int).Sub(currentBaseFee, lastBaseFee)
		changePercent := new(big.Float).SetInt(change)
		changePercent.Quo(changePercent, new(big.Float).SetInt(lastBaseFee))
		changePercentFloat, _ := changePercent.Float64()

		if changePercentFloat > 0.25 { // 上涨超过 25%
			alert = &GasAlert{
				Type:       AlertTypeSpike,
				CurrentGas: currentBaseFee,
				Threshold:  lastBaseFee,
				Suggestion: "Gas 价格突然上涨，可能有大额交易或攻击",
				Timestamp:  time.Now(),
			}
		} else if changePercentFloat < -0.20 { // 下跌超过 20%
			alert = &GasAlert{
				Type:       AlertTypeDrop,
				CurrentGas: currentBaseFee,
				Threshold:  lastBaseFee,
				Suggestion: "Gas 价格突然下跌，是发送交易的好时机",
				Timestamp:  time.Now(),
			}
		}
	}

	// 发送预警
	if alert != nil && time.Since(g.lastAlert) > g.alertCooldown {
		g.notify(alert)
		g.lastAlert = time.Now()
	}
}

// notify 通知订阅者
func (g *GasAlertSystem) notify(alert *GasAlert) {
	g.subMu.RLock()
	defer g.subMu.RUnlock()

	for _, sub := range g.subscribers {
		go sub.OnGasAlert(alert)
	}
}

// WebhookAlertSubscriber Webhook 订阅者
type WebhookAlertSubscriber struct {
	webhookURL string
}

func (w *WebhookAlertSubscriber) OnGasAlert(alert *GasAlert) {
	// 发送 HTTP POST 到 webhook
	// 实现略
}

// LogAlertSubscriber 日志订阅者
type LogAlertSubscriber struct{}

func (l *LogAlertSubscriber) OnGasAlert(alert *GasAlert) {
	log.Printf("[GAS ALERT] Type: %d, Current: %s, Threshold: %s, Suggestion: %s",
		alert.Type, alert.CurrentGas, alert.Threshold, alert.Suggestion)
}
```
