# Go-Ethereum EVM 深度解析：Layer2 套利

## 第一百零三节：Layer2 基础架构

### L2 生态概览

```
Layer2 生态系统：
┌─────────────────────────────────────────────────────────────────────┐
│                         Layer 2 生态                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │    Optimistic   │  │    ZK Rollup    │  │    Validium     │     │
│  │    Rollup       │  │                 │  │                 │     │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤     │
│  │ • Arbitrum One  │  │ • zkSync Era    │  │ • StarkEx       │     │
│  │ • Optimism      │  │ • Scroll        │  │ • Immutable X   │     │
│  │ • Base          │  │ • Linea         │  │                 │     │
│  │ • Mantle        │  │ • Polygon zkEVM │  │                 │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│           │                   │                    │                │
│           └───────────────────┼────────────────────┘                │
│                               │                                     │
│                               ▼                                     │
│                    ┌─────────────────┐                             │
│                    │   Ethereum L1   │                             │
│                    │   (Data Layer)  │                             │
│                    └─────────────────┘                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 多链客户端管理

```go
package layer2

import (
    "context"
    "fmt"
    "math/big"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

// ChainID 链标识
type ChainID uint64

const (
    ChainEthereum   ChainID = 1
    ChainArbitrum   ChainID = 42161
    ChainOptimism   ChainID = 10
    ChainBase       ChainID = 8453
    ChainZkSync     ChainID = 324
    ChainPolygon    ChainID = 137
    ChainScroll     ChainID = 534352
    ChainLinea      ChainID = 59144
)

// ChainConfig 链配置
type ChainConfig struct {
    ChainID         ChainID
    Name            string
    RpcURL          string
    WsURL           string
    ExplorerURL     string
    NativeToken     string
    BlockTime       time.Duration
    ConfirmBlocks   uint64
    GasMultiplier   float64

    // L2 特定配置
    L1ChainID       ChainID        // 对应的 L1 链
    BridgeAddress   common.Address // 官方桥地址
    SequencerURL    string         // 排序器地址
}

// MultiChainClient 多链客户端
type MultiChainClient struct {
    clients     map[ChainID]*ChainClient
    configs     map[ChainID]*ChainConfig
    mu          sync.RWMutex
}

// ChainClient 单链客户端
type ChainClient struct {
    config      *ChainConfig
    httpClient  *ethclient.Client
    wsClient    *ethclient.Client

    // 状态
    blockNumber uint64
    gasPrice    *big.Int
    baseFee     *big.Int

    // 监控
    latency     time.Duration
    isHealthy   bool
    lastUpdate  time.Time

    mu          sync.RWMutex
}

// NewMultiChainClient 创建多链客户端
func NewMultiChainClient(configs []*ChainConfig) (*MultiChainClient, error) {
    mcc := &MultiChainClient{
        clients: make(map[ChainID]*ChainClient),
        configs: make(map[ChainID]*ChainConfig),
    }

    for _, config := range configs {
        client, err := mcc.connectChain(config)
        if err != nil {
            return nil, fmt.Errorf("连接链 %s 失败: %w", config.Name, err)
        }

        mcc.clients[config.ChainID] = client
        mcc.configs[config.ChainID] = config
    }

    return mcc, nil
}

// connectChain 连接单条链
func (mcc *MultiChainClient) connectChain(config *ChainConfig) (*ChainClient, error) {
    httpClient, err := ethclient.Dial(config.RpcURL)
    if err != nil {
        return nil, fmt.Errorf("HTTP 连接失败: %w", err)
    }

    var wsClient *ethclient.Client
    if config.WsURL != "" {
        wsClient, err = ethclient.Dial(config.WsURL)
        if err != nil {
            // WebSocket 可选，不影响主功能
            fmt.Printf("警告: %s WebSocket 连接失败: %v\n", config.Name, err)
        }
    }

    client := &ChainClient{
        config:     config,
        httpClient: httpClient,
        wsClient:   wsClient,
        isHealthy:  true,
        lastUpdate: time.Now(),
    }

    // 初始化状态
    if err := client.updateState(context.Background()); err != nil {
        return nil, err
    }

    return client, nil
}

// updateState 更新链状态
func (cc *ChainClient) updateState(ctx context.Context) error {
    start := time.Now()

    // 获取区块号
    blockNumber, err := cc.httpClient.BlockNumber(ctx)
    if err != nil {
        cc.isHealthy = false
        return err
    }

    // 获取 gas 价格
    gasPrice, err := cc.httpClient.SuggestGasPrice(ctx)
    if err != nil {
        cc.isHealthy = false
        return err
    }

    // 获取最新区块的 baseFee
    block, err := cc.httpClient.BlockByNumber(ctx, nil)
    if err == nil && block.BaseFee() != nil {
        cc.baseFee = block.BaseFee()
    }

    cc.mu.Lock()
    cc.blockNumber = blockNumber
    cc.gasPrice = gasPrice
    cc.latency = time.Since(start)
    cc.isHealthy = true
    cc.lastUpdate = time.Now()
    cc.mu.Unlock()

    return nil
}

// GetClient 获取特定链的客户端
func (mcc *MultiChainClient) GetClient(chainID ChainID) (*ChainClient, error) {
    mcc.mu.RLock()
    defer mcc.mu.RUnlock()

    client, exists := mcc.clients[chainID]
    if !exists {
        return nil, fmt.Errorf("链 %d 未配置", chainID)
    }

    return client, nil
}

// GetAllClients 获取所有客户端
func (mcc *MultiChainClient) GetAllClients() map[ChainID]*ChainClient {
    mcc.mu.RLock()
    defer mcc.mu.RUnlock()

    result := make(map[ChainID]*ChainClient)
    for k, v := range mcc.clients {
        result[k] = v
    }
    return result
}

// StartHealthMonitor 启动健康监控
func (mcc *MultiChainClient) StartHealthMonitor(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            mcc.checkHealth(ctx)
        }
    }
}

// checkHealth 检查所有链健康状态
func (mcc *MultiChainClient) checkHealth(ctx context.Context) {
    mcc.mu.RLock()
    clients := make([]*ChainClient, 0, len(mcc.clients))
    for _, c := range mcc.clients {
        clients = append(clients, c)
    }
    mcc.mu.RUnlock()

    var wg sync.WaitGroup
    for _, client := range clients {
        wg.Add(1)
        go func(c *ChainClient) {
            defer wg.Done()
            c.updateState(ctx)
        }(client)
    }
    wg.Wait()
}

// GetHealthStatus 获取健康状态
func (mcc *MultiChainClient) GetHealthStatus() map[ChainID]HealthStatus {
    mcc.mu.RLock()
    defer mcc.mu.RUnlock()

    status := make(map[ChainID]HealthStatus)
    for chainID, client := range mcc.clients {
        client.mu.RLock()
        status[chainID] = HealthStatus{
            ChainID:     chainID,
            Name:        client.config.Name,
            IsHealthy:   client.isHealthy,
            BlockNumber: client.blockNumber,
            Latency:     client.latency,
            LastUpdate:  client.lastUpdate,
        }
        client.mu.RUnlock()
    }

    return status
}

// HealthStatus 健康状态
type HealthStatus struct {
    ChainID     ChainID
    Name        string
    IsHealthy   bool
    BlockNumber uint64
    Latency     time.Duration
    LastUpdate  time.Time
}

// Call 调用合约
func (cc *ChainClient) Call(ctx context.Context, msg ethereum.CallMsg) ([]byte, error) {
    return cc.httpClient.CallContract(ctx, msg, nil)
}

// SendTransaction 发送交易
func (cc *ChainClient) SendTransaction(ctx context.Context, tx *types.Transaction) error {
    return cc.httpClient.SendTransaction(ctx, tx)
}

// GetBlockNumber 获取区块号
func (cc *ChainClient) GetBlockNumber() uint64 {
    cc.mu.RLock()
    defer cc.mu.RUnlock()
    return cc.blockNumber
}

// GetGasPrice 获取 gas 价格
func (cc *ChainClient) GetGasPrice() *big.Int {
    cc.mu.RLock()
    defer cc.mu.RUnlock()
    return new(big.Int).Set(cc.gasPrice)
}

// GetBaseFee 获取 baseFee
func (cc *ChainClient) GetBaseFee() *big.Int {
    cc.mu.RLock()
    defer cc.mu.RUnlock()
    if cc.baseFee == nil {
        return nil
    }
    return new(big.Int).Set(cc.baseFee)
}
```

### L2 特定功能

```go
package layer2

import (
    "context"
    "fmt"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/accounts/abi"
)

// L2Client L2 特定客户端
type L2Client struct {
    *ChainClient
    l1Client    *ChainClient
}

// NewL2Client 创建 L2 客户端
func NewL2Client(l2Client, l1Client *ChainClient) *L2Client {
    return &L2Client{
        ChainClient: l2Client,
        l1Client:    l1Client,
    }
}

// ArbitrumClient Arbitrum 特定功能
type ArbitrumClient struct {
    *L2Client
    sequencerInbox common.Address
    outbox         common.Address
}

// NewArbitrumClient 创建 Arbitrum 客户端
func NewArbitrumClient(l2Client, l1Client *ChainClient) *ArbitrumClient {
    return &ArbitrumClient{
        L2Client:       NewL2Client(l2Client, l1Client),
        sequencerInbox: common.HexToAddress("0x1c479675ad559DC151F6Ec7ed3FbF8ceE79582B6"),
        outbox:         common.HexToAddress("0x0B9857ae2D4A3DBe74ffE1d7DF045bb7F96E4840"),
    }
}

// GetL1GasPrice 获取 Arbitrum L1 Gas 价格
func (ac *ArbitrumClient) GetL1GasPrice(ctx context.Context) (*big.Int, error) {
    // Arbitrum ArbGasInfo 预编译合约
    arbGasInfo := common.HexToAddress("0x000000000000000000000000000000000000006C")

    // getL1BaseFeeEstimate() selector
    data := common.Hex2Bytes("f5d6ded7")

    result, err := ac.Call(ctx, ethereum.CallMsg{
        To:   &arbGasInfo,
        Data: data,
    })
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

// EstimateL1DataFee 估算 L1 数据费用
func (ac *ArbitrumClient) EstimateL1DataFee(ctx context.Context, txData []byte) (*big.Int, error) {
    // ArbGasInfo.getL1GasEstimate(data)
    arbGasInfo := common.HexToAddress("0x000000000000000000000000000000000000006C")

    // 构造调用数据
    abiDef, _ := abi.JSON(strings.NewReader(`[{
        "inputs": [{"type": "bytes"}],
        "name": "gasEstimateL1Component",
        "outputs": [
            {"type": "uint64"},
            {"type": "uint256"},
            {"type": "uint256"}
        ],
        "stateMutability": "view",
        "type": "function"
    }]`))

    data, err := abiDef.Pack("gasEstimateL1Component", txData)
    if err != nil {
        return nil, err
    }

    result, err := ac.Call(ctx, ethereum.CallMsg{
        To:   &arbGasInfo,
        Data: data,
    })
    if err != nil {
        return nil, err
    }

    // 解析结果
    outputs, err := abiDef.Unpack("gasEstimateL1Component", result)
    if err != nil {
        return nil, err
    }

    // 返回 L1 数据费用估算
    if len(outputs) >= 2 {
        return outputs[1].(*big.Int), nil
    }

    return big.NewInt(0), nil
}

// OptimismClient Optimism/Base 特定功能
type OptimismClient struct {
    *L2Client
    gasPriceOracle common.Address
}

// NewOptimismClient 创建 Optimism 客户端
func NewOptimismClient(l2Client, l1Client *ChainClient) *OptimismClient {
    return &OptimismClient{
        L2Client:       NewL2Client(l2Client, l1Client),
        gasPriceOracle: common.HexToAddress("0x420000000000000000000000000000000000000F"),
    }
}

// GetL1Fee 获取 Optimism L1 费用
func (oc *OptimismClient) GetL1Fee(ctx context.Context, txData []byte) (*big.Int, error) {
    // GasPriceOracle.getL1Fee(bytes)
    abiDef, _ := abi.JSON(strings.NewReader(`[{
        "inputs": [{"type": "bytes"}],
        "name": "getL1Fee",
        "outputs": [{"type": "uint256"}],
        "stateMutability": "view",
        "type": "function"
    }]`))

    data, err := abiDef.Pack("getL1Fee", txData)
    if err != nil {
        return nil, err
    }

    result, err := oc.Call(ctx, ethereum.CallMsg{
        To:   &oc.gasPriceOracle,
        Data: data,
    })
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

// GetL1GasUsed 获取 L1 gas 使用量
func (oc *OptimismClient) GetL1GasUsed(ctx context.Context, txData []byte) (*big.Int, error) {
    abiDef, _ := abi.JSON(strings.NewReader(`[{
        "inputs": [{"type": "bytes"}],
        "name": "getL1GasUsed",
        "outputs": [{"type": "uint256"}],
        "stateMutability": "view",
        "type": "function"
    }]`))

    data, err := abiDef.Pack("getL1GasUsed", txData)
    if err != nil {
        return nil, err
    }

    result, err := oc.Call(ctx, ethereum.CallMsg{
        To:   &oc.gasPriceOracle,
        Data: data,
    })
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

// GetScalar 获取费用标量
func (oc *OptimismClient) GetScalar(ctx context.Context) (*big.Int, error) {
    // scalar() selector
    data := common.Hex2Bytes("f45e65d8")

    result, err := oc.Call(ctx, ethereum.CallMsg{
        To:   &oc.gasPriceOracle,
        Data: data,
    })
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}
```

---

## 第一百零四节：跨链价格监控

### 跨链价格聚合器

```go
package layer2

import (
    "context"
    "fmt"
    "math/big"
    "sort"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/common"
)

// CrossChainPriceMonitor 跨链价格监控器
type CrossChainPriceMonitor struct {
    multiClient *MultiChainClient

    // 价格缓存
    prices      map[ChainID]map[common.Address]*TokenPrice
    priceMu     sync.RWMutex

    // DEX 配置
    dexConfigs  map[ChainID][]*DEXConfig

    // 订阅者
    subscribers []chan *PriceUpdate
    subMu       sync.RWMutex
}

// TokenPrice 代币价格
type TokenPrice struct {
    Token       common.Address
    ChainID     ChainID
    PriceUSD    *big.Float
    PriceETH    *big.Float
    Liquidity   *big.Int
    LastUpdate  time.Time
    Source      string
}

// DEXConfig DEX 配置
type DEXConfig struct {
    Name        string
    ChainID     ChainID
    Factory     common.Address
    Router      common.Address
    Type        DEXType // UniswapV2, UniswapV3, Curve 等
    Fee         uint64  // 基点
}

// DEXType DEX 类型
type DEXType int

const (
    DEXTypeUniswapV2 DEXType = iota
    DEXTypeUniswapV3
    DEXTypeCurve
    DEXTypeBalancer
    DEXTypeSushiSwap
)

// PriceUpdate 价格更新
type PriceUpdate struct {
    Token       common.Address
    ChainID     ChainID
    OldPrice    *big.Float
    NewPrice    *big.Float
    Change      float64
    Timestamp   time.Time
}

// NewCrossChainPriceMonitor 创建监控器
func NewCrossChainPriceMonitor(multiClient *MultiChainClient) *CrossChainPriceMonitor {
    return &CrossChainPriceMonitor{
        multiClient: multiClient,
        prices:      make(map[ChainID]map[common.Address]*TokenPrice),
        dexConfigs:  initDEXConfigs(),
        subscribers: make([]chan *PriceUpdate, 0),
    }
}

// initDEXConfigs 初始化 DEX 配置
func initDEXConfigs() map[ChainID][]*DEXConfig {
    configs := make(map[ChainID][]*DEXConfig)

    // Ethereum
    configs[ChainEthereum] = []*DEXConfig{
        {
            Name:    "Uniswap V2",
            ChainID: ChainEthereum,
            Factory: common.HexToAddress("0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f"),
            Router:  common.HexToAddress("0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D"),
            Type:    DEXTypeUniswapV2,
            Fee:     30,
        },
        {
            Name:    "Uniswap V3",
            ChainID: ChainEthereum,
            Factory: common.HexToAddress("0x1F98431c8aD98523631AE4a59f267346ea31F984"),
            Router:  common.HexToAddress("0xE592427A0AEce92De3Edee1F18E0157C05861564"),
            Type:    DEXTypeUniswapV3,
            Fee:     0, // 动态费率
        },
        {
            Name:    "SushiSwap",
            ChainID: ChainEthereum,
            Factory: common.HexToAddress("0xC0AEe478e3658e2610c5F7A4A2E1777cE9e4f2Ac"),
            Router:  common.HexToAddress("0xd9e1cE17f2641f24aE83637ab66a2cca9C378B9F"),
            Type:    DEXTypeSushiSwap,
            Fee:     30,
        },
    }

    // Arbitrum
    configs[ChainArbitrum] = []*DEXConfig{
        {
            Name:    "Uniswap V3",
            ChainID: ChainArbitrum,
            Factory: common.HexToAddress("0x1F98431c8aD98523631AE4a59f267346ea31F984"),
            Router:  common.HexToAddress("0xE592427A0AEce92De3Edee1F18E0157C05861564"),
            Type:    DEXTypeUniswapV3,
            Fee:     0,
        },
        {
            Name:    "SushiSwap",
            ChainID: ChainArbitrum,
            Factory: common.HexToAddress("0xc35DADB65012eC5796536bD9864eD8773aBc74C4"),
            Router:  common.HexToAddress("0x1b02dA8Cb0d097eB8D57A175b88c7D8b47997506"),
            Type:    DEXTypeSushiSwap,
            Fee:     30,
        },
        {
            Name:    "Camelot",
            ChainID: ChainArbitrum,
            Factory: common.HexToAddress("0x6EcCab422D763aC031210895C81787E87B43A652"),
            Router:  common.HexToAddress("0xc873fEcbd354f5A56E00E710B90EF4201db2448d"),
            Type:    DEXTypeUniswapV2,
            Fee:     30,
        },
    }

    // Optimism
    configs[ChainOptimism] = []*DEXConfig{
        {
            Name:    "Uniswap V3",
            ChainID: ChainOptimism,
            Factory: common.HexToAddress("0x1F98431c8aD98523631AE4a59f267346ea31F984"),
            Router:  common.HexToAddress("0xE592427A0AEce92De3Edee1F18E0157C05861564"),
            Type:    DEXTypeUniswapV3,
            Fee:     0,
        },
        {
            Name:    "Velodrome",
            ChainID: ChainOptimism,
            Factory: common.HexToAddress("0x25CbdDb98b35ab1FF77413456B31EC81A6B6B746"),
            Router:  common.HexToAddress("0x9c12939390052919aF3155f41Bf4160Fd3666A6f"),
            Type:    DEXTypeUniswapV2,
            Fee:     0, // 动态费率
        },
    }

    // Base
    configs[ChainBase] = []*DEXConfig{
        {
            Name:    "Uniswap V3",
            ChainID: ChainBase,
            Factory: common.HexToAddress("0x33128a8fC17869897dcE68Ed026d694621f6FDfD"),
            Router:  common.HexToAddress("0x2626664c2603336E57B271c5C0b26F421741e481"),
            Type:    DEXTypeUniswapV3,
            Fee:     0,
        },
        {
            Name:    "Aerodrome",
            ChainID: ChainBase,
            Factory: common.HexToAddress("0x420DD381b31aEf6683db6B902084cB0FFECe40Da"),
            Router:  common.HexToAddress("0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43"),
            Type:    DEXTypeUniswapV2,
            Fee:     0,
        },
    }

    return configs
}

// StartMonitoring 开始监控
func (pm *CrossChainPriceMonitor) StartMonitoring(ctx context.Context, tokens []common.Address) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            pm.updatePrices(ctx, tokens)
        }
    }
}

// updatePrices 更新所有链上的价格
func (pm *CrossChainPriceMonitor) updatePrices(ctx context.Context, tokens []common.Address) {
    clients := pm.multiClient.GetAllClients()

    var wg sync.WaitGroup
    for chainID, client := range clients {
        wg.Add(1)
        go func(cid ChainID, c *ChainClient) {
            defer wg.Done()
            pm.updateChainPrices(ctx, cid, c, tokens)
        }(chainID, client)
    }
    wg.Wait()
}

// updateChainPrices 更新单链价格
func (pm *CrossChainPriceMonitor) updateChainPrices(
    ctx context.Context,
    chainID ChainID,
    client *ChainClient,
    tokens []common.Address,
) {
    dexConfigs := pm.dexConfigs[chainID]
    if len(dexConfigs) == 0 {
        return
    }

    for _, token := range tokens {
        price, err := pm.getTokenPrice(ctx, chainID, client, token, dexConfigs)
        if err != nil {
            continue
        }

        pm.updatePriceCache(chainID, token, price)
    }
}

// getTokenPrice 获取代币价格
func (pm *CrossChainPriceMonitor) getTokenPrice(
    ctx context.Context,
    chainID ChainID,
    client *ChainClient,
    token common.Address,
    dexConfigs []*DEXConfig,
) (*TokenPrice, error) {
    // 从多个 DEX 获取价格并聚合
    var prices []*big.Float
    var totalLiquidity *big.Int = big.NewInt(0)

    for _, dex := range dexConfigs {
        price, liquidity, err := pm.getPriceFromDEX(ctx, client, token, dex)
        if err != nil {
            continue
        }

        if price != nil && price.Sign() > 0 {
            prices = append(prices, price)
            totalLiquidity.Add(totalLiquidity, liquidity)
        }
    }

    if len(prices) == 0 {
        return nil, fmt.Errorf("无法获取 %s 在链 %d 上的价格", token.Hex(), chainID)
    }

    // 使用中位数价格
    medianPrice := calculateMedian(prices)

    return &TokenPrice{
        Token:      token,
        ChainID:    chainID,
        PriceUSD:   medianPrice,
        Liquidity:  totalLiquidity,
        LastUpdate: time.Now(),
        Source:     "aggregated",
    }, nil
}

// getPriceFromDEX 从特定 DEX 获取价格
func (pm *CrossChainPriceMonitor) getPriceFromDEX(
    ctx context.Context,
    client *ChainClient,
    token common.Address,
    dex *DEXConfig,
) (*big.Float, *big.Int, error) {
    // 获取 WETH 地址 (根据链不同)
    weth := getWETHAddress(dex.ChainID)

    // 获取池子地址
    poolAddress, err := pm.getPoolAddress(ctx, client, dex, token, weth)
    if err != nil {
        return nil, nil, err
    }

    // 获取储备
    reserve0, reserve1, err := pm.getReserves(ctx, client, poolAddress, dex.Type)
    if err != nil {
        return nil, nil, err
    }

    // 计算价格
    price := new(big.Float).Quo(
        new(big.Float).SetInt(reserve1),
        new(big.Float).SetInt(reserve0),
    )

    // 流动性
    liquidity := new(big.Int).Add(reserve0, reserve1)

    return price, liquidity, nil
}

// getWETHAddress 获取 WETH 地址
func getWETHAddress(chainID ChainID) common.Address {
    wethAddresses := map[ChainID]common.Address{
        ChainEthereum: common.HexToAddress("0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"),
        ChainArbitrum: common.HexToAddress("0x82aF49447D8a07e3bd95BD0d56f35241523fBab1"),
        ChainOptimism: common.HexToAddress("0x4200000000000000000000000000000000000006"),
        ChainBase:     common.HexToAddress("0x4200000000000000000000000000000000000006"),
        ChainPolygon:  common.HexToAddress("0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619"),
    }

    return wethAddresses[chainID]
}

// getPoolAddress 获取池子地址
func (pm *CrossChainPriceMonitor) getPoolAddress(
    ctx context.Context,
    client *ChainClient,
    dex *DEXConfig,
    token0, token1 common.Address,
) (common.Address, error) {
    // 根据 DEX 类型调用不同的工厂方法
    switch dex.Type {
    case DEXTypeUniswapV2, DEXTypeSushiSwap:
        return pm.getV2PairAddress(ctx, client, dex.Factory, token0, token1)
    case DEXTypeUniswapV3:
        return pm.getV3PoolAddress(ctx, client, dex.Factory, token0, token1, 3000)
    default:
        return common.Address{}, fmt.Errorf("不支持的 DEX 类型")
    }
}

// getV2PairAddress 获取 V2 池子地址
func (pm *CrossChainPriceMonitor) getV2PairAddress(
    ctx context.Context,
    client *ChainClient,
    factory, token0, token1 common.Address,
) (common.Address, error) {
    // getPair(address,address) selector: 0xe6a43905
    data := make([]byte, 68)
    copy(data[0:4], common.Hex2Bytes("e6a43905"))
    copy(data[16:36], token0.Bytes())
    copy(data[48:68], token1.Bytes())

    result, err := client.Call(ctx, ethereum.CallMsg{
        To:   &factory,
        Data: data,
    })
    if err != nil {
        return common.Address{}, err
    }

    return common.BytesToAddress(result), nil
}

// getV3PoolAddress 获取 V3 池子地址
func (pm *CrossChainPriceMonitor) getV3PoolAddress(
    ctx context.Context,
    client *ChainClient,
    factory, token0, token1 common.Address,
    fee uint64,
) (common.Address, error) {
    // getPool(address,address,uint24) selector: 0x1698ee82
    data := make([]byte, 100)
    copy(data[0:4], common.Hex2Bytes("1698ee82"))
    copy(data[16:36], token0.Bytes())
    copy(data[48:68], token1.Bytes())
    new(big.Int).SetUint64(fee).FillBytes(data[68:100])

    result, err := client.Call(ctx, ethereum.CallMsg{
        To:   &factory,
        Data: data,
    })
    if err != nil {
        return common.Address{}, err
    }

    return common.BytesToAddress(result), nil
}

// getReserves 获取储备
func (pm *CrossChainPriceMonitor) getReserves(
    ctx context.Context,
    client *ChainClient,
    pool common.Address,
    dexType DEXType,
) (*big.Int, *big.Int, error) {
    switch dexType {
    case DEXTypeUniswapV2, DEXTypeSushiSwap:
        // getReserves() selector: 0x0902f1ac
        data := common.Hex2Bytes("0902f1ac")

        result, err := client.Call(ctx, ethereum.CallMsg{
            To:   &pool,
            Data: data,
        })
        if err != nil {
            return nil, nil, err
        }

        if len(result) < 64 {
            return nil, nil, fmt.Errorf("无效的储备数据")
        }

        reserve0 := new(big.Int).SetBytes(result[0:32])
        reserve1 := new(big.Int).SetBytes(result[32:64])

        return reserve0, reserve1, nil

    case DEXTypeUniswapV3:
        // slot0() selector: 0x3850c7bd
        data := common.Hex2Bytes("3850c7bd")

        result, err := client.Call(ctx, ethereum.CallMsg{
            To:   &pool,
            Data: data,
        })
        if err != nil {
            return nil, nil, err
        }

        // 从 sqrtPriceX96 计算储备比例
        if len(result) < 32 {
            return nil, nil, fmt.Errorf("无效的 slot0 数据")
        }

        sqrtPriceX96 := new(big.Int).SetBytes(result[0:32])
        // 简化：返回虚拟储备
        reserve0 := big.NewInt(1e18)
        price := new(big.Int).Mul(sqrtPriceX96, sqrtPriceX96)
        price.Div(price, new(big.Int).Exp(big.NewInt(2), big.NewInt(192), nil))
        reserve1 := new(big.Int).Mul(reserve0, price)

        return reserve0, reserve1, nil

    default:
        return nil, nil, fmt.Errorf("不支持的 DEX 类型")
    }
}

// updatePriceCache 更新价格缓存
func (pm *CrossChainPriceMonitor) updatePriceCache(
    chainID ChainID,
    token common.Address,
    newPrice *TokenPrice,
) {
    pm.priceMu.Lock()
    defer pm.priceMu.Unlock()

    if _, exists := pm.prices[chainID]; !exists {
        pm.prices[chainID] = make(map[common.Address]*TokenPrice)
    }

    oldPrice := pm.prices[chainID][token]
    pm.prices[chainID][token] = newPrice

    // 通知订阅者
    if oldPrice != nil {
        change := calculatePriceChange(oldPrice.PriceUSD, newPrice.PriceUSD)
        if change > 0.001 || change < -0.001 { // 0.1% 变化阈值
            pm.notifySubscribers(&PriceUpdate{
                Token:     token,
                ChainID:   chainID,
                OldPrice:  oldPrice.PriceUSD,
                NewPrice:  newPrice.PriceUSD,
                Change:    change,
                Timestamp: time.Now(),
            })
        }
    }
}

// GetPrice 获取价格
func (pm *CrossChainPriceMonitor) GetPrice(chainID ChainID, token common.Address) (*TokenPrice, error) {
    pm.priceMu.RLock()
    defer pm.priceMu.RUnlock()

    if chainPrices, exists := pm.prices[chainID]; exists {
        if price, exists := chainPrices[token]; exists {
            return price, nil
        }
    }

    return nil, fmt.Errorf("价格未找到")
}

// GetCrossChainPrices 获取跨链价格
func (pm *CrossChainPriceMonitor) GetCrossChainPrices(token common.Address) map[ChainID]*TokenPrice {
    pm.priceMu.RLock()
    defer pm.priceMu.RUnlock()

    result := make(map[ChainID]*TokenPrice)
    for chainID, chainPrices := range pm.prices {
        if price, exists := chainPrices[token]; exists {
            result[chainID] = price
        }
    }

    return result
}

// Subscribe 订阅价格更新
func (pm *CrossChainPriceMonitor) Subscribe() <-chan *PriceUpdate {
    ch := make(chan *PriceUpdate, 100)

    pm.subMu.Lock()
    pm.subscribers = append(pm.subscribers, ch)
    pm.subMu.Unlock()

    return ch
}

// notifySubscribers 通知订阅者
func (pm *CrossChainPriceMonitor) notifySubscribers(update *PriceUpdate) {
    pm.subMu.RLock()
    defer pm.subMu.RUnlock()

    for _, ch := range pm.subscribers {
        select {
        case ch <- update:
        default:
            // 通道已满，跳过
        }
    }
}

// calculateMedian 计算中位数
func calculateMedian(prices []*big.Float) *big.Float {
    if len(prices) == 0 {
        return new(big.Float)
    }

    // 排序
    sort.Slice(prices, func(i, j int) bool {
        return prices[i].Cmp(prices[j]) < 0
    })

    mid := len(prices) / 2
    if len(prices)%2 == 0 {
        sum := new(big.Float).Add(prices[mid-1], prices[mid])
        return sum.Quo(sum, big.NewFloat(2))
    }

    return prices[mid]
}

// calculatePriceChange 计算价格变化
func calculatePriceChange(oldPrice, newPrice *big.Float) float64 {
    if oldPrice == nil || oldPrice.Sign() == 0 {
        return 0
    }

    diff := new(big.Float).Sub(newPrice, oldPrice)
    change := new(big.Float).Quo(diff, oldPrice)
    result, _ := change.Float64()

    return result
}
```

---

## 第一百零五节：跨链套利检测

### 跨链套利机会识别

```go
package layer2

import (
    "context"
    "fmt"
    "math/big"
    "sort"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/common"
)

// CrossChainArbitrageDetector 跨链套利检测器
type CrossChainArbitrageDetector struct {
    priceMonitor    *CrossChainPriceMonitor
    multiClient     *MultiChainClient
    bridgeManager   *BridgeManager

    // 配置
    minProfitBps    uint64  // 最小利润基点
    maxLatencyMs    int64   // 最大延迟毫秒
    enabledChains   []ChainID

    // 结果通道
    opportunities   chan *ArbitrageOpportunity
}

// ArbitrageOpportunity 套利机会
type ArbitrageOpportunity struct {
    Token           common.Address
    SourceChain     ChainID
    TargetChain     ChainID
    SourcePrice     *big.Float
    TargetPrice     *big.Float
    PriceDiffBps    uint64          // 价格差基点
    EstimatedProfit *big.Int        // 预估利润 (wei)
    BridgeCost      *big.Int        // 桥接成本
    GasCost         *big.Int        // Gas 成本
    NetProfit       *big.Int        // 净利润
    OptimalAmount   *big.Int        // 最优交易量
    BridgeRoute     *BridgeRoute    // 桥接路线
    Timestamp       time.Time
    Confidence      float64         // 置信度
}

// BridgeManager 桥接管理器
type BridgeManager struct {
    bridges map[string]*BridgeConfig
}

// BridgeConfig 桥接配置
type BridgeConfig struct {
    Name            string
    SourceChain     ChainID
    TargetChain     ChainID
    BridgeAddress   common.Address
    Fee             uint64          // 基点
    MinAmount       *big.Int
    MaxAmount       *big.Int
    EstimatedTime   time.Duration
}

// BridgeRoute 桥接路线
type BridgeRoute struct {
    Bridge          *BridgeConfig
    Fee             *big.Int
    EstimatedTime   time.Duration
}

// NewCrossChainArbitrageDetector 创建检测器
func NewCrossChainArbitrageDetector(
    priceMonitor *CrossChainPriceMonitor,
    multiClient *MultiChainClient,
) *CrossChainArbitrageDetector {
    return &CrossChainArbitrageDetector{
        priceMonitor:  priceMonitor,
        multiClient:   multiClient,
        bridgeManager: NewBridgeManager(),
        minProfitBps:  50,  // 0.5%
        maxLatencyMs:  5000,
        enabledChains: []ChainID{
            ChainEthereum,
            ChainArbitrum,
            ChainOptimism,
            ChainBase,
        },
        opportunities: make(chan *ArbitrageOpportunity, 100),
    }
}

// NewBridgeManager 创建桥接管理器
func NewBridgeManager() *BridgeManager {
    bm := &BridgeManager{
        bridges: make(map[string]*BridgeConfig),
    }

    // 注册桥接配置
    bm.registerBridges()

    return bm
}

// registerBridges 注册桥接
func (bm *BridgeManager) registerBridges() {
    // Arbitrum 官方桥
    bm.bridges["eth-arb-official"] = &BridgeConfig{
        Name:          "Arbitrum Official Bridge",
        SourceChain:   ChainEthereum,
        TargetChain:   ChainArbitrum,
        BridgeAddress: common.HexToAddress("0x8315177aB297bA92A06054cE80a67Ed4DBd7ed3a"),
        Fee:           0,
        MinAmount:     big.NewInt(1e15),
        MaxAmount:     new(big.Int).Mul(big.NewInt(1e18), big.NewInt(10000)),
        EstimatedTime: 10 * time.Minute,
    }

    // Optimism 官方桥
    bm.bridges["eth-op-official"] = &BridgeConfig{
        Name:          "Optimism Official Bridge",
        SourceChain:   ChainEthereum,
        TargetChain:   ChainOptimism,
        BridgeAddress: common.HexToAddress("0x99C9fc46f92E8a1c0deC1b1747d010903E884bE1"),
        Fee:           0,
        MinAmount:     big.NewInt(1e15),
        MaxAmount:     new(big.Int).Mul(big.NewInt(1e18), big.NewInt(10000)),
        EstimatedTime: 20 * time.Minute,
    }

    // Base 官方桥
    bm.bridges["eth-base-official"] = &BridgeConfig{
        Name:          "Base Official Bridge",
        SourceChain:   ChainEthereum,
        TargetChain:   ChainBase,
        BridgeAddress: common.HexToAddress("0x49048044D57e1C92A77f79988d21Fa8fAF74E97e"),
        Fee:           0,
        MinAmount:     big.NewInt(1e15),
        MaxAmount:     new(big.Int).Mul(big.NewInt(1e18), big.NewInt(10000)),
        EstimatedTime: 20 * time.Minute,
    }

    // Across Protocol (快速桥)
    bm.bridges["across-eth-arb"] = &BridgeConfig{
        Name:          "Across Protocol",
        SourceChain:   ChainEthereum,
        TargetChain:   ChainArbitrum,
        BridgeAddress: common.HexToAddress("0x5c7BCd6E7De5423a257D81B442095A1a6ced35C5"),
        Fee:           10, // 0.1%
        MinAmount:     big.NewInt(1e16),
        MaxAmount:     new(big.Int).Mul(big.NewInt(1e18), big.NewInt(1000)),
        EstimatedTime: 2 * time.Minute,
    }

    // Stargate (LayerZero)
    bm.bridges["stargate-eth-arb"] = &BridgeConfig{
        Name:          "Stargate Finance",
        SourceChain:   ChainEthereum,
        TargetChain:   ChainArbitrum,
        BridgeAddress: common.HexToAddress("0x8731d54E9D02c286767d56ac03e8037C07e01e98"),
        Fee:           6, // 0.06%
        MinAmount:     big.NewInt(1e17),
        MaxAmount:     new(big.Int).Mul(big.NewInt(1e18), big.NewInt(5000)),
        EstimatedTime: 3 * time.Minute,
    }
}

// StartDetection 开始检测
func (d *CrossChainArbitrageDetector) StartDetection(ctx context.Context, tokens []common.Address) {
    // 订阅价格更新
    priceUpdates := d.priceMonitor.Subscribe()

    for {
        select {
        case <-ctx.Done():
            return
        case update := <-priceUpdates:
            // 检查该代币在其他链上的价格
            d.checkArbitrageOpportunity(ctx, update.Token)
        }
    }
}

// checkArbitrageOpportunity 检查套利机会
func (d *CrossChainArbitrageDetector) checkArbitrageOpportunity(ctx context.Context, token common.Address) {
    // 获取所有链上的价格
    prices := d.priceMonitor.GetCrossChainPrices(token)
    if len(prices) < 2 {
        return
    }

    // 比较所有链对
    for sourceChain, sourcePrice := range prices {
        for targetChain, targetPrice := range prices {
            if sourceChain == targetChain {
                continue
            }

            // 计算价格差
            diffBps := d.calculatePriceDiffBps(sourcePrice.PriceUSD, targetPrice.PriceUSD)

            // 检查是否超过阈值
            if diffBps < d.minProfitBps {
                continue
            }

            // 获取桥接路线
            bridgeRoute := d.bridgeManager.GetBestRoute(sourceChain, targetChain)
            if bridgeRoute == nil {
                continue
            }

            // 计算预估利润
            opportunity := d.calculateOpportunity(
                ctx,
                token,
                sourceChain,
                targetChain,
                sourcePrice,
                targetPrice,
                diffBps,
                bridgeRoute,
            )

            if opportunity != nil && opportunity.NetProfit.Sign() > 0 {
                select {
                case d.opportunities <- opportunity:
                default:
                    // 通道已满
                }
            }
        }
    }
}

// calculatePriceDiffBps 计算价格差基点
func (d *CrossChainArbitrageDetector) calculatePriceDiffBps(price1, price2 *big.Float) uint64 {
    if price1 == nil || price2 == nil || price1.Sign() == 0 {
        return 0
    }

    diff := new(big.Float).Sub(price2, price1)
    ratio := new(big.Float).Quo(diff, price1)
    ratioFloat, _ := ratio.Float64()

    // 转换为基点
    bps := uint64(ratioFloat * 10000)
    if bps < 0 {
        return 0
    }

    return bps
}

// GetBestRoute 获取最佳桥接路线
func (bm *BridgeManager) GetBestRoute(source, target ChainID) *BridgeRoute {
    var bestRoute *BridgeRoute
    var lowestFee uint64 = ^uint64(0)

    for _, bridge := range bm.bridges {
        if bridge.SourceChain == source && bridge.TargetChain == target {
            if bridge.Fee < lowestFee {
                lowestFee = bridge.Fee
                bestRoute = &BridgeRoute{
                    Bridge:        bridge,
                    Fee:           big.NewInt(int64(bridge.Fee)),
                    EstimatedTime: bridge.EstimatedTime,
                }
            }
        }
    }

    return bestRoute
}

// calculateOpportunity 计算套利机会
func (d *CrossChainArbitrageDetector) calculateOpportunity(
    ctx context.Context,
    token common.Address,
    sourceChain, targetChain ChainID,
    sourcePrice, targetPrice *TokenPrice,
    diffBps uint64,
    bridgeRoute *BridgeRoute,
) *ArbitrageOpportunity {
    // 估算最优交易量 (使用较小池子的流动性)
    optimalAmount := d.calculateOptimalAmount(sourcePrice, targetPrice)

    // 计算毛利润
    grossProfit := d.calculateGrossProfit(optimalAmount, diffBps)

    // 计算桥接成本
    bridgeCost := d.calculateBridgeCost(optimalAmount, bridgeRoute)

    // 估算 Gas 成本
    gasCost := d.estimateGasCost(ctx, sourceChain, targetChain)

    // 计算净利润
    netProfit := new(big.Int).Sub(grossProfit, bridgeCost)
    netProfit.Sub(netProfit, gasCost)

    // 计算置信度
    confidence := d.calculateConfidence(sourcePrice, targetPrice, bridgeRoute)

    return &ArbitrageOpportunity{
        Token:           token,
        SourceChain:     sourceChain,
        TargetChain:     targetChain,
        SourcePrice:     sourcePrice.PriceUSD,
        TargetPrice:     targetPrice.PriceUSD,
        PriceDiffBps:    diffBps,
        EstimatedProfit: grossProfit,
        BridgeCost:      bridgeCost,
        GasCost:         gasCost,
        NetProfit:       netProfit,
        OptimalAmount:   optimalAmount,
        BridgeRoute:     bridgeRoute,
        Timestamp:       time.Now(),
        Confidence:      confidence,
    }
}

// calculateOptimalAmount 计算最优交易量
func (d *CrossChainArbitrageDetector) calculateOptimalAmount(
    sourcePrice, targetPrice *TokenPrice,
) *big.Int {
    // 使用较小流动性的 1%
    minLiquidity := sourcePrice.Liquidity
    if targetPrice.Liquidity.Cmp(minLiquidity) < 0 {
        minLiquidity = targetPrice.Liquidity
    }

    // 1% 的流动性
    optimal := new(big.Int).Div(minLiquidity, big.NewInt(100))

    // 设置上限
    maxAmount := new(big.Int).Mul(big.NewInt(1e18), big.NewInt(100)) // 100 ETH
    if optimal.Cmp(maxAmount) > 0 {
        return maxAmount
    }

    return optimal
}

// calculateGrossProfit 计算毛利润
func (d *CrossChainArbitrageDetector) calculateGrossProfit(amount *big.Int, diffBps uint64) *big.Int {
    profit := new(big.Int).Mul(amount, big.NewInt(int64(diffBps)))
    return profit.Div(profit, big.NewInt(10000))
}

// calculateBridgeCost 计算桥接成本
func (d *CrossChainArbitrageDetector) calculateBridgeCost(amount *big.Int, route *BridgeRoute) *big.Int {
    if route == nil {
        return big.NewInt(0)
    }

    cost := new(big.Int).Mul(amount, big.NewInt(int64(route.Bridge.Fee)))
    return cost.Div(cost, big.NewInt(10000))
}

// estimateGasCost 估算 Gas 成本
func (d *CrossChainArbitrageDetector) estimateGasCost(
    ctx context.Context,
    sourceChain, targetChain ChainID,
) *big.Int {
    totalGas := big.NewInt(0)

    // 源链 Gas
    sourceClient, _ := d.multiClient.GetClient(sourceChain)
    if sourceClient != nil {
        gasPrice := sourceClient.GetGasPrice()
        // Swap + Bridge ≈ 300k gas
        gas := new(big.Int).Mul(gasPrice, big.NewInt(300000))
        totalGas.Add(totalGas, gas)
    }

    // 目标链 Gas
    targetClient, _ := d.multiClient.GetClient(targetChain)
    if targetClient != nil {
        gasPrice := targetClient.GetGasPrice()
        // Swap ≈ 150k gas
        gas := new(big.Int).Mul(gasPrice, big.NewInt(150000))
        totalGas.Add(totalGas, gas)
    }

    return totalGas
}

// calculateConfidence 计算置信度
func (d *CrossChainArbitrageDetector) calculateConfidence(
    sourcePrice, targetPrice *TokenPrice,
    route *BridgeRoute,
) float64 {
    confidence := 1.0

    // 价格新鲜度
    sourceAge := time.Since(sourcePrice.LastUpdate).Seconds()
    targetAge := time.Since(targetPrice.LastUpdate).Seconds()

    if sourceAge > 10 {
        confidence *= 0.8
    }
    if targetAge > 10 {
        confidence *= 0.8
    }

    // 流动性因素
    minLiquidity := new(big.Int).Mul(big.NewInt(1e18), big.NewInt(10)) // 10 ETH
    if sourcePrice.Liquidity.Cmp(minLiquidity) < 0 {
        confidence *= 0.7
    }
    if targetPrice.Liquidity.Cmp(minLiquidity) < 0 {
        confidence *= 0.7
    }

    // 桥接时间因素
    if route != nil && route.EstimatedTime > 10*time.Minute {
        confidence *= 0.6
    }

    return confidence
}

// GetOpportunities 获取机会通道
func (d *CrossChainArbitrageDetector) GetOpportunities() <-chan *ArbitrageOpportunity {
    return d.opportunities
}
```

---

## 第一百零六节：跨链套利执行

### 跨链套利执行器

```go
package layer2

import (
    "context"
    "crypto/ecdsa"
    "fmt"
    "math/big"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/crypto"
)

// CrossChainArbitrageExecutor 跨链套利执行器
type CrossChainArbitrageExecutor struct {
    multiClient   *MultiChainClient
    bridgeManager *BridgeManager
    privateKey    *ecdsa.PrivateKey
    address       common.Address

    // 执行状态
    pendingTxs    map[common.Hash]*PendingExecution
    pendingMu     sync.RWMutex

    // 配置
    maxConcurrent int
    semaphore     chan struct{}
}

// PendingExecution 待执行状态
type PendingExecution struct {
    Opportunity   *ArbitrageOpportunity
    Stage         ExecutionStage
    SourceTxHash  common.Hash
    BridgeTxHash  common.Hash
    TargetTxHash  common.Hash
    StartTime     time.Time
    LastUpdate    time.Time
    Error         error
}

// ExecutionStage 执行阶段
type ExecutionStage int

const (
    StageInitiated ExecutionStage = iota
    StageSourceSwapPending
    StageSourceSwapConfirmed
    StageBridgePending
    StageBridgeConfirmed
    StageTargetSwapPending
    StageTargetSwapConfirmed
    StageCompleted
    StageFailed
)

// ExecutionResult 执行结果
type ExecutionResult struct {
    Opportunity   *ArbitrageOpportunity
    Success       bool
    ActualProfit  *big.Int
    TotalGasUsed  uint64
    TotalGasCost  *big.Int
    SourceTxHash  common.Hash
    TargetTxHash  common.Hash
    Duration      time.Duration
    Error         error
}

// NewCrossChainArbitrageExecutor 创建执行器
func NewCrossChainArbitrageExecutor(
    multiClient *MultiChainClient,
    bridgeManager *BridgeManager,
    privateKey *ecdsa.PrivateKey,
) *CrossChainArbitrageExecutor {
    return &CrossChainArbitrageExecutor{
        multiClient:   multiClient,
        bridgeManager: bridgeManager,
        privateKey:    privateKey,
        address:       crypto.PubkeyToAddress(privateKey.PublicKey),
        pendingTxs:    make(map[common.Hash]*PendingExecution),
        maxConcurrent: 5,
        semaphore:     make(chan struct{}, 5),
    }
}

// Execute 执行套利
func (e *CrossChainArbitrageExecutor) Execute(
    ctx context.Context,
    opportunity *ArbitrageOpportunity,
) (*ExecutionResult, error) {
    // 获取信号量
    select {
    case e.semaphore <- struct{}{}:
        defer func() { <-e.semaphore }()
    case <-ctx.Done():
        return nil, ctx.Err()
    }

    startTime := time.Now()
    result := &ExecutionResult{
        Opportunity: opportunity,
    }

    // 创建执行状态
    execution := &PendingExecution{
        Opportunity: opportunity,
        Stage:       StageInitiated,
        StartTime:   startTime,
        LastUpdate:  startTime,
    }

    // 步骤1: 源链交换
    sourceTxHash, err := e.executeSourceSwap(ctx, opportunity)
    if err != nil {
        execution.Stage = StageFailed
        execution.Error = err
        result.Error = fmt.Errorf("源链交换失败: %w", err)
        return result, err
    }
    execution.SourceTxHash = sourceTxHash
    execution.Stage = StageSourceSwapPending
    result.SourceTxHash = sourceTxHash

    // 等待源链确认
    sourceReceipt, err := e.waitForConfirmation(ctx, opportunity.SourceChain, sourceTxHash)
    if err != nil {
        execution.Stage = StageFailed
        execution.Error = err
        result.Error = fmt.Errorf("源链确认失败: %w", err)
        return result, err
    }
    execution.Stage = StageSourceSwapConfirmed
    result.TotalGasUsed += sourceReceipt.GasUsed

    // 步骤2: 桥接
    bridgeTxHash, err := e.executeBridge(ctx, opportunity)
    if err != nil {
        execution.Stage = StageFailed
        execution.Error = err
        result.Error = fmt.Errorf("桥接失败: %w", err)
        return result, err
    }
    execution.BridgeTxHash = bridgeTxHash
    execution.Stage = StageBridgePending

    // 等待桥接完成
    err = e.waitForBridgeCompletion(ctx, opportunity, bridgeTxHash)
    if err != nil {
        execution.Stage = StageFailed
        execution.Error = err
        result.Error = fmt.Errorf("桥接确认失败: %w", err)
        return result, err
    }
    execution.Stage = StageBridgeConfirmed

    // 步骤3: 目标链交换
    targetTxHash, err := e.executeTargetSwap(ctx, opportunity)
    if err != nil {
        execution.Stage = StageFailed
        execution.Error = err
        result.Error = fmt.Errorf("目标链交换失败: %w", err)
        return result, err
    }
    execution.TargetTxHash = targetTxHash
    execution.Stage = StageTargetSwapPending
    result.TargetTxHash = targetTxHash

    // 等待目标链确认
    targetReceipt, err := e.waitForConfirmation(ctx, opportunity.TargetChain, targetTxHash)
    if err != nil {
        execution.Stage = StageFailed
        execution.Error = err
        result.Error = fmt.Errorf("目标链确认失败: %w", err)
        return result, err
    }
    execution.Stage = StageCompleted
    result.TotalGasUsed += targetReceipt.GasUsed

    // 计算实际利润
    result.Success = true
    result.ActualProfit = e.calculateActualProfit(sourceReceipt, targetReceipt)
    result.Duration = time.Since(startTime)

    return result, nil
}

// executeSourceSwap 执行源链交换
func (e *CrossChainArbitrageExecutor) executeSourceSwap(
    ctx context.Context,
    opportunity *ArbitrageOpportunity,
) (common.Hash, error) {
    sourceClient, err := e.multiClient.GetClient(opportunity.SourceChain)
    if err != nil {
        return common.Hash{}, err
    }

    // 构建交换交易
    swapData := e.buildSwapData(opportunity.Token, opportunity.OptimalAmount, true)

    // 获取 nonce
    nonce, err := sourceClient.httpClient.PendingNonceAt(ctx, e.address)
    if err != nil {
        return common.Hash{}, err
    }

    // 获取 gas 价格
    gasPrice := sourceClient.GetGasPrice()

    // 构建交易
    tx := types.NewTransaction(
        nonce,
        getRouterAddress(opportunity.SourceChain),
        big.NewInt(0),
        uint64(300000),
        gasPrice,
        swapData,
    )

    // 签名
    chainID := big.NewInt(int64(opportunity.SourceChain))
    signedTx, err := types.SignTx(tx, types.NewEIP155Signer(chainID), e.privateKey)
    if err != nil {
        return common.Hash{}, err
    }

    // 发送
    err = sourceClient.SendTransaction(ctx, signedTx)
    if err != nil {
        return common.Hash{}, err
    }

    return signedTx.Hash(), nil
}

// executeBridge 执行桥接
func (e *CrossChainArbitrageExecutor) executeBridge(
    ctx context.Context,
    opportunity *ArbitrageOpportunity,
) (common.Hash, error) {
    sourceClient, err := e.multiClient.GetClient(opportunity.SourceChain)
    if err != nil {
        return common.Hash{}, err
    }

    bridge := opportunity.BridgeRoute.Bridge

    // 构建桥接调用数据
    bridgeData := e.buildBridgeData(
        opportunity.Token,
        opportunity.OptimalAmount,
        opportunity.TargetChain,
    )

    // 获取 nonce
    nonce, err := sourceClient.httpClient.PendingNonceAt(ctx, e.address)
    if err != nil {
        return common.Hash{}, err
    }

    gasPrice := sourceClient.GetGasPrice()

    tx := types.NewTransaction(
        nonce,
        bridge.BridgeAddress,
        big.NewInt(0),
        uint64(500000),
        gasPrice,
        bridgeData,
    )

    chainID := big.NewInt(int64(opportunity.SourceChain))
    signedTx, err := types.SignTx(tx, types.NewEIP155Signer(chainID), e.privateKey)
    if err != nil {
        return common.Hash{}, err
    }

    err = sourceClient.SendTransaction(ctx, signedTx)
    if err != nil {
        return common.Hash{}, err
    }

    return signedTx.Hash(), nil
}

// executeTargetSwap 执行目标链交换
func (e *CrossChainArbitrageExecutor) executeTargetSwap(
    ctx context.Context,
    opportunity *ArbitrageOpportunity,
) (common.Hash, error) {
    targetClient, err := e.multiClient.GetClient(opportunity.TargetChain)
    if err != nil {
        return common.Hash{}, err
    }

    // 构建交换数据 (卖出)
    swapData := e.buildSwapData(opportunity.Token, opportunity.OptimalAmount, false)

    nonce, err := targetClient.httpClient.PendingNonceAt(ctx, e.address)
    if err != nil {
        return common.Hash{}, err
    }

    gasPrice := targetClient.GetGasPrice()

    tx := types.NewTransaction(
        nonce,
        getRouterAddress(opportunity.TargetChain),
        big.NewInt(0),
        uint64(300000),
        gasPrice,
        swapData,
    )

    chainID := big.NewInt(int64(opportunity.TargetChain))
    signedTx, err := types.SignTx(tx, types.NewEIP155Signer(chainID), e.privateKey)
    if err != nil {
        return common.Hash{}, err
    }

    err = targetClient.SendTransaction(ctx, signedTx)
    if err != nil {
        return common.Hash{}, err
    }

    return signedTx.Hash(), nil
}

// waitForConfirmation 等待确认
func (e *CrossChainArbitrageExecutor) waitForConfirmation(
    ctx context.Context,
    chainID ChainID,
    txHash common.Hash,
) (*types.Receipt, error) {
    client, err := e.multiClient.GetClient(chainID)
    if err != nil {
        return nil, err
    }

    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    timeout := time.After(5 * time.Minute)

    for {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-timeout:
            return nil, fmt.Errorf("等待确认超时")
        case <-ticker.C:
            receipt, err := client.httpClient.TransactionReceipt(ctx, txHash)
            if err != nil {
                continue
            }

            if receipt.Status == types.ReceiptStatusSuccessful {
                return receipt, nil
            }

            return nil, fmt.Errorf("交易失败")
        }
    }
}

// waitForBridgeCompletion 等待桥接完成
func (e *CrossChainArbitrageExecutor) waitForBridgeCompletion(
    ctx context.Context,
    opportunity *ArbitrageOpportunity,
    bridgeTxHash common.Hash,
) error {
    targetClient, err := e.multiClient.GetClient(opportunity.TargetChain)
    if err != nil {
        return err
    }

    // 获取桥接前的余额
    token := opportunity.Token
    weth := getWETHAddress(opportunity.TargetChain)
    if token == common.HexToAddress("0x0000000000000000000000000000000000000000") {
        token = weth
    }

    balanceBefore, err := e.getTokenBalance(ctx, targetClient, token, e.address)
    if err != nil {
        return err
    }

    // 等待余额变化
    estimatedTime := opportunity.BridgeRoute.EstimatedTime
    timeout := time.After(estimatedTime * 3) // 3倍估算时间
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-timeout:
            return fmt.Errorf("桥接超时")
        case <-ticker.C:
            balanceAfter, err := e.getTokenBalance(ctx, targetClient, token, e.address)
            if err != nil {
                continue
            }

            // 检查余额是否增加
            diff := new(big.Int).Sub(balanceAfter, balanceBefore)
            expectedAmount := new(big.Int).Mul(opportunity.OptimalAmount, big.NewInt(99))
            expectedAmount.Div(expectedAmount, big.NewInt(100)) // 允许 1% 滑点

            if diff.Cmp(expectedAmount) >= 0 {
                return nil
            }
        }
    }
}

// getTokenBalance 获取代币余额
func (e *CrossChainArbitrageExecutor) getTokenBalance(
    ctx context.Context,
    client *ChainClient,
    token, account common.Address,
) (*big.Int, error) {
    // balanceOf(address) selector: 0x70a08231
    data := make([]byte, 36)
    copy(data[0:4], common.Hex2Bytes("70a08231"))
    copy(data[16:36], account.Bytes())

    result, err := client.Call(ctx, ethereum.CallMsg{
        To:   &token,
        Data: data,
    })
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

// buildSwapData 构建交换数据
func (e *CrossChainArbitrageExecutor) buildSwapData(
    token common.Address,
    amount *big.Int,
    isBuy bool,
) []byte {
    // 简化实现：使用 V2 Router
    // swapExactTokensForTokens / swapExactETHForTokens
    // 实际实现需要根据具体情况构建
    return nil
}

// buildBridgeData 构建桥接数据
func (e *CrossChainArbitrageExecutor) buildBridgeData(
    token common.Address,
    amount *big.Int,
    targetChain ChainID,
) []byte {
    // 根据不同桥接协议构建数据
    // 实际实现需要根据具体桥接协议
    return nil
}

// calculateActualProfit 计算实际利润
func (e *CrossChainArbitrageExecutor) calculateActualProfit(
    sourceReceipt, targetReceipt *types.Receipt,
) *big.Int {
    // 从事件日志中提取实际交换数量
    // 计算实际利润
    return big.NewInt(0)
}

// getRouterAddress 获取路由地址
func getRouterAddress(chainID ChainID) common.Address {
    routers := map[ChainID]common.Address{
        ChainEthereum: common.HexToAddress("0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D"),
        ChainArbitrum: common.HexToAddress("0x1b02dA8Cb0d097eB8D57A175b88c7D8b47997506"),
        ChainOptimism: common.HexToAddress("0xE592427A0AEce92De3Edee1F18E0157C05861564"),
        ChainBase:     common.HexToAddress("0x2626664c2603336E57B271c5C0b26F421741e481"),
    }
    return routers[chainID]
}
```

---

## 附录 A: ZK Rollup 深入解析

### A.1 ZK Rollup 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZK Rollup 架构                                │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    L2 执行层                               │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐          │  │
│  │  │ 用户交易   │─▶│ 排序器     │─▶│ 状态更新   │          │  │
│  │  │            │  │ Sequencer  │  │            │          │  │
│  │  └────────────┘  └────────────┘  └────────────┘          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    证明生成层                              │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐          │  │
│  │  │ 交易批次   │─▶│ ZK Prover  │─▶│ 有效性证明 │          │  │
│  │  │ Batch      │  │            │  │ Validity   │          │  │
│  │  └────────────┘  └────────────┘  └────────────┘          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    L1 验证层                               │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐          │  │
│  │  │ State Root │  │ Verifier   │  │ 最终性     │          │  │
│  │  │ 状态根     │  │ 合约验证   │  │ Finality   │          │  │
│  │  └────────────┘  └────────────┘  └────────────┘          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  主要 ZK Rollup:                                                │
│  - zkSync Era: zkEVM, 原生账户抽象                              │
│  - StarkNet: Cairo 语言, STARK 证明                            │
│  - Polygon zkEVM: 完全 EVM 等价                                 │
│  - Scroll: zkEVM, 字节码级兼容                                  │
│  - Linea: ConsenSys 开发, Type-2 zkEVM                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### A.2 ZK Rollup 客户端实现

```go
package zkrollup

import (
    "context"
    "encoding/json"
    "fmt"
    "math/big"
    "net/http"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

// ZKRollupType ZK Rollup 类型
type ZKRollupType int

const (
    ZKSyncEra ZKRollupType = iota
    StarkNet
    PolygonZkEVM
    Scroll
    Linea
)

// ZKRollupClient ZK Rollup 通用客户端
type ZKRollupClient struct {
    client     *ethclient.Client
    rollupType ZKRollupType
    chainID    *big.Int

    // L1 连接
    l1Client   *ethclient.Client
    l1Contracts *L1Contracts
}

// L1Contracts L1 合约地址
type L1Contracts struct {
    RollupAddress  common.Address
    BridgeAddress  common.Address
    VerifierAddress common.Address
}

// BatchInfo 批次信息
type BatchInfo struct {
    BatchNumber   uint64
    Timestamp     time.Time
    StateRoot     common.Hash
    TxCount       uint64
    L1TxHash      common.Hash
    Status        BatchStatus
}

// BatchStatus 批次状态
type BatchStatus int

const (
    BatchPending BatchStatus = iota
    BatchCommitted
    BatchProven
    BatchFinalized
)

// NewZKRollupClient 创建 ZK Rollup 客户端
func NewZKRollupClient(rpcURL string, rollupType ZKRollupType, l1Client *ethclient.Client) (*ZKRollupClient, error) {
    client, err := ethclient.Dial(rpcURL)
    if err != nil {
        return nil, err
    }

    chainID, err := client.ChainID(context.Background())
    if err != nil {
        return nil, err
    }

    return &ZKRollupClient{
        client:     client,
        rollupType: rollupType,
        chainID:    chainID,
        l1Client:   l1Client,
        l1Contracts: getL1Contracts(rollupType),
    }, nil
}

func getL1Contracts(rollupType ZKRollupType) *L1Contracts {
    switch rollupType {
    case ZKSyncEra:
        return &L1Contracts{
            RollupAddress:   common.HexToAddress("0x32400084C286CF3E17e7B677ea9583e60a000324"),
            BridgeAddress:   common.HexToAddress("0x57891966931Eb4Bb6FB81430E6cE0A03AAbDe063"),
        }
    case PolygonZkEVM:
        return &L1Contracts{
            RollupAddress:   common.HexToAddress("0x5132A183E9F3CB7C848b0AAC5Ae0c4f0491B7aB2"),
            BridgeAddress:   common.HexToAddress("0x2a3DD3EB832aF982ec71669E178424b10Dca2EDe"),
        }
    case Scroll:
        return &L1Contracts{
            RollupAddress:   common.HexToAddress("0xa13BAF47339d63B743e7Da8741db5456DAc1E556"),
            BridgeAddress:   common.HexToAddress("0xD8A791fE2bE73eb6E6cF1eb0cb3F36adC9B3F8f9"),
        }
    default:
        return &L1Contracts{}
    }
}

// GetLatestBatchInfo 获取最新批次信息
func (c *ZKRollupClient) GetLatestBatchInfo(ctx context.Context) (*BatchInfo, error) {
    switch c.rollupType {
    case ZKSyncEra:
        return c.getZKSyncBatchInfo(ctx)
    case PolygonZkEVM:
        return c.getPolygonZkEVMBatchInfo(ctx)
    case Scroll:
        return c.getScrollBatchInfo(ctx)
    default:
        return nil, fmt.Errorf("unsupported rollup type")
    }
}

func (c *ZKRollupClient) getZKSyncBatchInfo(ctx context.Context) (*BatchInfo, error) {
    // zkSync Era 使用 zks_ RPC 方法
    var result struct {
        L1BatchNumber    string `json:"l1BatchNumber"`
        Timestamp        int64  `json:"timestamp"`
        RootHash         string `json:"rootHash"`
        CommitTxHash     string `json:"commitTxHash"`
        ProveTxHash      string `json:"proveTxHash"`
        ExecuteTxHash    string `json:"executeTxHash"`
    }

    err := c.client.Client().CallContext(ctx, &result, "zks_L1BatchNumber")
    if err != nil {
        return nil, err
    }

    batchNum, _ := new(big.Int).SetString(result.L1BatchNumber, 0)

    // 获取批次详情
    var batchDetails struct {
        Number       uint64      `json:"number"`
        Timestamp    int64       `json:"timestamp"`
        RootHash     common.Hash `json:"rootHash"`
        Status       string      `json:"status"`
    }

    err = c.client.Client().CallContext(ctx, &batchDetails, "zks_getL1BatchDetails", batchNum.Uint64())
    if err != nil {
        return nil, err
    }

    status := BatchPending
    switch batchDetails.Status {
    case "committed":
        status = BatchCommitted
    case "proven":
        status = BatchProven
    case "executed":
        status = BatchFinalized
    }

    return &BatchInfo{
        BatchNumber: batchDetails.Number,
        Timestamp:   time.Unix(batchDetails.Timestamp, 0),
        StateRoot:   batchDetails.RootHash,
        Status:      status,
    }, nil
}

// GetFinalityStatus 获取交易最终性状态
func (c *ZKRollupClient) GetFinalityStatus(ctx context.Context, txHash common.Hash) (*FinalityStatus, error) {
    // 获取交易所在批次
    receipt, err := c.client.TransactionReceipt(ctx, txHash)
    if err != nil {
        return nil, err
    }

    // 获取该批次的状态
    batchInfo, err := c.getBatchByBlockNumber(ctx, receipt.BlockNumber.Uint64())
    if err != nil {
        return nil, err
    }

    return &FinalityStatus{
        TxHash:        txHash,
        BatchNumber:   batchInfo.BatchNumber,
        L2BlockNumber: receipt.BlockNumber.Uint64(),
        BatchStatus:   batchInfo.Status,
        L1TxHash:      batchInfo.L1TxHash,
        IsFinal:       batchInfo.Status == BatchFinalized,
    }, nil
}

// FinalityStatus 最终性状态
type FinalityStatus struct {
    TxHash        common.Hash
    BatchNumber   uint64
    L2BlockNumber uint64
    BatchStatus   BatchStatus
    L1TxHash      common.Hash
    IsFinal       bool
}

// WaitForFinality 等待交易最终性
func (c *ZKRollupClient) WaitForFinality(ctx context.Context, txHash common.Hash, timeout time.Duration) error {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    timeoutCh := time.After(timeout)

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-timeoutCh:
            return fmt.Errorf("timeout waiting for finality")
        case <-ticker.C:
            status, err := c.GetFinalityStatus(ctx, txHash)
            if err != nil {
                continue
            }
            if status.IsFinal {
                return nil
            }
        }
    }
}

// EstimateProofTime 估算证明生成时间
func (c *ZKRollupClient) EstimateProofTime(ctx context.Context) (time.Duration, error) {
    // 获取最近几个批次的证明时间
    latestBatch, err := c.GetLatestBatchInfo(ctx)
    if err != nil {
        return 0, err
    }

    // 计算平均证明时间
    var totalTime time.Duration
    count := 0

    for i := uint64(0); i < 10 && latestBatch.BatchNumber > i; i++ {
        batchNum := latestBatch.BatchNumber - i
        batch, err := c.getBatchByNumber(ctx, batchNum)
        if err != nil || batch.Status != BatchFinalized {
            continue
        }

        // 证明时间 = 最终化时间 - 提交时间
        // 这里简化处理
        totalTime += 30 * time.Minute
        count++
    }

    if count == 0 {
        return 30 * time.Minute, nil // 默认估计
    }

    return totalTime / time.Duration(count), nil
}
```

### A.3 zkSync Era 特定功能

```go
// ZKSyncEraClient zkSync Era 专用客户端
type ZKSyncEraClient struct {
    *ZKRollupClient
}

// NewZKSyncEraClient 创建 zkSync Era 客户端
func NewZKSyncEraClient(rpcURL string, l1Client *ethclient.Client) (*ZKSyncEraClient, error) {
    base, err := NewZKRollupClient(rpcURL, ZKSyncEra, l1Client)
    if err != nil {
        return nil, err
    }
    return &ZKSyncEraClient{base}, nil
}

// GetMainContractAddress 获取主合约地址
func (c *ZKSyncEraClient) GetMainContractAddress(ctx context.Context) (common.Address, error) {
    var result common.Address
    err := c.client.Client().CallContext(ctx, &result, "zks_getMainContract")
    return result, err
}

// GetBridgeContracts 获取桥合约地址
func (c *ZKSyncEraClient) GetBridgeContracts(ctx context.Context) (*BridgeContracts, error) {
    var result BridgeContracts
    err := c.client.Client().CallContext(ctx, &result, "zks_getBridgeContracts")
    return &result, err
}

// BridgeContracts 桥合约
type BridgeContracts struct {
    L1Erc20DefaultBridge common.Address `json:"l1Erc20DefaultBridge"`
    L2Erc20DefaultBridge common.Address `json:"l2Erc20DefaultBridge"`
    L1WethBridge         common.Address `json:"l1WethBridge"`
    L2WethBridge         common.Address `json:"l2WethBridge"`
}

// GetL1GasPrice 获取 L1 gas 价格
func (c *ZKSyncEraClient) GetL1GasPrice(ctx context.Context) (*big.Int, error) {
    var result string
    err := c.client.Client().CallContext(ctx, &result, "zks_getL1GasPrice")
    if err != nil {
        return nil, err
    }
    gasPrice, _ := new(big.Int).SetString(result, 0)
    return gasPrice, nil
}

// EstimateFee 估算交易费用
func (c *ZKSyncEraClient) EstimateFee(ctx context.Context, tx *types.Transaction) (*ZKSyncFee, error) {
    msg := map[string]interface{}{
        "from":  tx.From(),
        "to":    tx.To(),
        "data":  tx.Data(),
        "value": tx.Value(),
    }

    var result ZKSyncFee
    err := c.client.Client().CallContext(ctx, &result, "zks_estimateFee", msg)
    return &result, err
}

// ZKSyncFee 费用估算
type ZKSyncFee struct {
    GasLimit          *big.Int `json:"gas_limit"`
    GasPerPubdataLimit *big.Int `json:"gas_per_pubdata_limit"`
    MaxFeePerGas      *big.Int `json:"max_fee_per_gas"`
    MaxPriorityFeePerGas *big.Int `json:"max_priority_fee_per_gas"`
}

// GetAccountAbstractionNonce 获取 AA nonce
func (c *ZKSyncEraClient) GetAccountAbstractionNonce(ctx context.Context, address common.Address) (uint64, error) {
    var result string
    err := c.client.Client().CallContext(ctx, &result, "eth_getTransactionCount", address, "pending")
    if err != nil {
        return 0, err
    }
    nonce, _ := new(big.Int).SetString(result, 0)
    return nonce.Uint64(), nil
}

// SendRawTransaction 发送原始交易 (支持 EIP-712 签名)
func (c *ZKSyncEraClient) SendRawTransactionWithPaymaster(
    ctx context.Context,
    tx *types.Transaction,
    paymaster common.Address,
    paymasterInput []byte,
) (common.Hash, error) {
    // zkSync Era 支持 Paymaster
    // 构建 EIP-712 交易

    return common.Hash{}, nil
}
```

### A.4 Polygon zkEVM 客户端

```go
// PolygonZkEVMClient Polygon zkEVM 专用客户端
type PolygonZkEVMClient struct {
    *ZKRollupClient
}

// NewPolygonZkEVMClient 创建 Polygon zkEVM 客户端
func NewPolygonZkEVMClient(rpcURL string, l1Client *ethclient.Client) (*PolygonZkEVMClient, error) {
    base, err := NewZKRollupClient(rpcURL, PolygonZkEVM, l1Client)
    if err != nil {
        return nil, err
    }
    return &PolygonZkEVMClient{base}, nil
}

// GetBatchByNumber 获取批次信息
func (c *PolygonZkEVMClient) GetBatchByNumber(ctx context.Context, batchNum uint64) (*PolygonBatch, error) {
    var result PolygonBatch
    err := c.client.Client().CallContext(ctx, &result, "zkevm_getBatchByNumber", batchNum)
    return &result, err
}

// PolygonBatch Polygon zkEVM 批次
type PolygonBatch struct {
    Number          uint64        `json:"number"`
    Coinbase        common.Address `json:"coinbase"`
    StateRoot       common.Hash   `json:"stateRoot"`
    GlobalExitRoot  common.Hash   `json:"globalExitRoot"`
    LocalExitRoot   common.Hash   `json:"localExitRoot"`
    AccInputHash    common.Hash   `json:"accInputHash"`
    Timestamp       uint64        `json:"timestamp"`
    SendSequencesTxHash common.Hash `json:"sendSequencesTxHash"`
    VerifyBatchTxHash   common.Hash `json:"verifyBatchTxHash"`
    Closed          bool          `json:"closed"`
}

// IsVirtualized 检查批次是否已虚拟化
func (c *PolygonZkEVMClient) IsVirtualized(ctx context.Context, batchNum uint64) (bool, error) {
    var result bool
    err := c.client.Client().CallContext(ctx, &result, "zkevm_isBlockVirtualized", batchNum)
    return result, err
}

// IsConsolidated 检查批次是否已整合 (证明已验证)
func (c *PolygonZkEVMClient) IsConsolidated(ctx context.Context, batchNum uint64) (bool, error) {
    var result bool
    err := c.client.Client().CallContext(ctx, &result, "zkevm_isBlockConsolidated", batchNum)
    return result, err
}

// GetVerifiedBatchNumber 获取已验证的批次号
func (c *PolygonZkEVMClient) GetVerifiedBatchNumber(ctx context.Context) (uint64, error) {
    var result string
    err := c.client.Client().CallContext(ctx, &result, "zkevm_verifiedBatchNumber")
    if err != nil {
        return 0, err
    }
    num, _ := new(big.Int).SetString(result, 0)
    return num.Uint64(), nil
}

// GetExitRootByGlobalExitRoot 获取退出根
func (c *PolygonZkEVMClient) GetExitRootByGlobalExitRoot(ctx context.Context, globalExitRoot common.Hash) (*ExitRoot, error) {
    var result ExitRoot
    err := c.client.Client().CallContext(ctx, &result, "zkevm_getExitRootByGlobalExitRoot", globalExitRoot)
    return &result, err
}

// ExitRoot 退出根
type ExitRoot struct {
    MainnetExitRoot common.Hash `json:"mainnetExitRoot"`
    RollupExitRoot  common.Hash `json:"rollupExitRoot"`
    GlobalExitRoot  common.Hash `json:"globalExitRoot"`
}
```

### A.5 ZK Rollup 套利策略

```go
// ZKRollupArbitrage ZK Rollup 套利
type ZKRollupArbitrage struct {
    clients map[ZKRollupType]*ZKRollupClient
    l1Client *ethclient.Client
}

// ZKArbitrageOpportunity 套利机会
type ZKArbitrageOpportunity struct {
    SourceChain    ZKRollupType
    TargetChain    ZKRollupType
    Token          common.Address
    SourcePrice    *big.Float
    TargetPrice    *big.Float
    PriceDiffBps   int
    EstimatedProfit *big.Int
    ProofTime      time.Duration  // 证明时间影响套利时效
    GasCost        *big.Int
    Risk           ArbitrageRisk
}

// ArbitrageRisk 风险等级
type ArbitrageRisk int

const (
    RiskLow ArbitrageRisk = iota
    RiskMedium
    RiskHigh
)

// NewZKRollupArbitrage 创建 ZK 套利器
func NewZKRollupArbitrage(l1Client *ethclient.Client, configs map[ZKRollupType]string) (*ZKRollupArbitrage, error) {
    clients := make(map[ZKRollupType]*ZKRollupClient)

    for rollupType, rpcURL := range configs {
        client, err := NewZKRollupClient(rpcURL, rollupType, l1Client)
        if err != nil {
            return nil, err
        }
        clients[rollupType] = client
    }

    return &ZKRollupArbitrage{
        clients:  clients,
        l1Client: l1Client,
    }, nil
}

// FindOpportunities 查找套利机会
func (a *ZKRollupArbitrage) FindOpportunities(ctx context.Context, token common.Address) ([]ZKArbitrageOpportunity, error) {
    var opportunities []ZKArbitrageOpportunity

    // 获取各链上的价格
    prices := make(map[ZKRollupType]*big.Float)

    for rollupType, client := range a.clients {
        price, err := a.getTokenPrice(ctx, client, token)
        if err != nil {
            continue
        }
        prices[rollupType] = price
    }

    // 比较价格差异
    for source, sourcePrice := range prices {
        for target, targetPrice := range prices {
            if source == target {
                continue
            }

            diff := new(big.Float).Sub(targetPrice, sourcePrice)
            ratio := new(big.Float).Quo(diff, sourcePrice)
            ratioF, _ := ratio.Float64()
            diffBps := int(ratioF * 10000)

            if diffBps > 50 { // 至少 0.5% 价差
                // 获取证明时间
                proofTime, _ := a.clients[source].EstimateProofTime(ctx)

                // 评估风险
                risk := a.evaluateRisk(diffBps, proofTime)

                opportunities = append(opportunities, ZKArbitrageOpportunity{
                    SourceChain:  source,
                    TargetChain:  target,
                    Token:        token,
                    SourcePrice:  sourcePrice,
                    TargetPrice:  targetPrice,
                    PriceDiffBps: diffBps,
                    ProofTime:    proofTime,
                    Risk:         risk,
                })
            }
        }
    }

    return opportunities, nil
}

// evaluateRisk 评估风险
func (a *ZKRollupArbitrage) evaluateRisk(diffBps int, proofTime time.Duration) ArbitrageRisk {
    // 证明时间越长，价格波动风险越高
    if proofTime > 2*time.Hour {
        return RiskHigh
    }

    // 价差越小，被抢先风险越高
    if diffBps < 100 {
        return RiskHigh
    }

    if proofTime > 1*time.Hour || diffBps < 200 {
        return RiskMedium
    }

    return RiskLow
}

// ExecuteArbitrage 执行套利
func (a *ZKRollupArbitrage) ExecuteArbitrage(
    ctx context.Context,
    opp ZKArbitrageOpportunity,
    amount *big.Int,
    auth *bind.TransactOpts,
) (*ArbitrageResult, error) {
    result := &ArbitrageResult{
        Opportunity: opp,
        StartTime:   time.Now(),
    }

    // 1. 在源链买入
    sourceTx, err := a.buyOnSource(ctx, opp.SourceChain, opp.Token, amount, auth)
    if err != nil {
        result.Error = err
        return result, err
    }
    result.SourceTxHash = sourceTx.Hash()

    // 2. 等待证明生成 (或使用快速桥)
    if opp.ProofTime > 30*time.Minute {
        // 使用快速桥 (如 Orbiter, LayerZero)
        bridgeTx, err := a.bridgeWithFastPath(ctx, opp, amount, auth)
        if err != nil {
            result.Error = err
            return result, err
        }
        result.BridgeTxHash = bridgeTx.Hash()
    } else {
        // 等待原生桥
        err = a.clients[opp.SourceChain].WaitForFinality(ctx, sourceTx.Hash(), opp.ProofTime*2)
        if err != nil {
            result.Error = err
            return result, err
        }

        bridgeTx, err := a.bridgeWithNative(ctx, opp, amount, auth)
        if err != nil {
            result.Error = err
            return result, err
        }
        result.BridgeTxHash = bridgeTx.Hash()
    }

    // 3. 在目标链卖出
    targetTx, err := a.sellOnTarget(ctx, opp.TargetChain, opp.Token, amount, auth)
    if err != nil {
        result.Error = err
        return result, err
    }
    result.TargetTxHash = targetTx.Hash()

    // 4. 计算实际利润
    result.EndTime = time.Now()
    result.ActualProfit = a.calculateActualProfit(result)

    return result, nil
}

// ArbitrageResult 套利结果
type ArbitrageResult struct {
    Opportunity   ZKArbitrageOpportunity
    SourceTxHash  common.Hash
    BridgeTxHash  common.Hash
    TargetTxHash  common.Hash
    StartTime     time.Time
    EndTime       time.Time
    ActualProfit  *big.Int
    GasSpent      *big.Int
    Error         error
}

// CompareProofSystems 比较证明系统
func CompareProofSystems() {
    /*
    ZK 证明系统比较:

    | 系统     | 证明时间  | 证明大小 | 验证成本 | 代表项目      |
    |----------|----------|---------|---------|--------------|
    | SNARK    | 分钟级   | ~200B   | 低      | zkSync      |
    | STARK    | 秒级     | ~100KB  | 中      | StarkNet    |
    | PLONK    | 分钟级   | ~500B   | 低      | Polygon     |
    | Halo2    | 分钟级   | ~200B   | 低      | Scroll      |

    选择考量:
    - SNARK: 需要可信设置，但证明小
    - STARK: 无需可信设置，量子安全，但证明大
    - PLONK: 通用可信设置，可更新
    - Halo2: 无需可信设置，证明小
    */
}
```

### A.6 ZK Rollup 对比

```
┌────────────────────────────────────────────────────────────────────────┐
│                    ZK Rollup 主要项目对比                                │
├──────────┬───────────┬───────────┬───────────┬──────────┬─────────────┤
│ 特性     │ zkSync Era │ StarkNet  │ Polygon   │ Scroll   │ Linea       │
│          │            │           │ zkEVM     │          │             │
├──────────┼───────────┼───────────┼───────────┼──────────┼─────────────┤
│ 证明系统 │ SNARK     │ STARK     │ PLONK     │ Halo2    │ SNARK       │
│ EVM 兼容 │ 字节码    │ Cairo VM  │ 完全等价  │ 字节码   │ 字节码      │
│ 账户抽象 │ 原生支持  │ 原生支持  │ ERC-4337  │ ERC-4337 │ ERC-4337    │
│ 证明时间 │ ~10分钟   │ ~分钟级   │ ~30分钟   │ ~20分钟  │ ~15分钟     │
│ TPS      │ ~2000     │ ~500      │ ~2000     │ ~1000    │ ~1500       │
│ 最终性   │ ~小时级   │ ~小时级   │ ~小时级   │ ~小时级  │ ~小时级     │
├──────────┼───────────┼───────────┼───────────┼──────────┼─────────────┤
│ 套利优势 │ AA 支持   │ 低费用    │ 兼容性好  │ 安全性高 │ 生态完善    │
│ 套利劣势 │ 验证慢    │ 语言学习  │ 证明慢    │ 较新     │ 中心化      │
└──────────┴───────────┴───────────┴───────────┴──────────┴─────────────┘
```

---

## 本文总结

本文深入探讨了 Layer2 跨链套利，主要涵盖：

### L2 生态架构

```
跨链套利架构：
┌─────────────────────────────────────────────────────────────────┐
│                      跨链套利系统                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │ 价格监控层   │────▶│ 套利检测层   │────▶│ 执行层       │    │
│  │              │     │              │     │              │    │
│  │ • 多链价格   │     │ • 价差计算   │     │ • 源链交换   │    │
│  │ • DEX 聚合   │     │ • 成本估算   │     │ • 跨链桥接   │    │
│  │ • 实时更新   │     │ • 机会排序   │     │ • 目标链交换 │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│                                                                 │
│  ──────────────────────────────────────────────────────────    │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      基础设施层                           │  │
│  │  • 多链客户端   • 健康监控   • 桥接管理   • 状态追踪     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 核心组件

| 组件 | 功能 | 关键特性 |
|------|------|---------|
| MultiChainClient | 多链连接管理 | 健康监控、状态同步 |
| L2Client | L2 特定功能 | L1 费用估算、排序器交互 |
| CrossChainPriceMonitor | 跨链价格监控 | DEX 聚合、中位数计算 |
| ArbitrageDetector | 套利检测 | 价差计算、置信度评估 |
| ArbitrageExecutor | 套利执行 | 原子执行、状态追踪 |
| BridgeManager | 桥接管理 | 路线优化、费用比较 |

### 关键挑战

1. **延迟风险**: 桥接时间导致价格变化
2. **Gas 成本**: L1 数据费用显著影响利润
3. **流动性分散**: 不同链流动性差异大
4. **桥接风险**: 桥接协议安全性和可靠性
```
