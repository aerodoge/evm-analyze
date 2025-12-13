# Go-Ethereum EVM 深度解析：链上数据索引与分析

## 概述

链上数据索引是 DeFi 套利的重要基础设施。高效的数据索引能够帮助发现套利机会、分析历史表现、监控市场状态。本文深入讲解 The
Graph、Dune Analytics 等数据索引方案的集成与使用。

## 1. The Graph 协议集成

### 1.1 Subgraph 查询客户端

```go
package indexing

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"math/big"
	"net/http"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

// GraphQLClient The Graph 查询客户端
type GraphQLClient struct {
	endpoint   string
	httpClient *http.Client
	apiKey     string

	// 缓存
	cache *QueryCache

	// 速率限制
	rateLimiter *RateLimiter
}

// GraphQLRequest GraphQL 请求
type GraphQLRequest struct {
	Query     string                 `json:"query"`
	Variables map[string]interface{} `json:"variables,omitempty"`
}

// GraphQLResponse GraphQL 响应
type GraphQLResponse struct {
	Data   json.RawMessage `json:"data"`
	Errors []GraphQLError  `json:"errors,omitempty"`
}

// GraphQLError GraphQL 错误
type GraphQLError struct {
	Message   string `json:"message"`
	Locations []struct {
		Line   int `json:"line"`
		Column int `json:"column"`
	} `json:"locations"`
}

// NewGraphQLClient 创建 GraphQL 客户端
func NewGraphQLClient(endpoint, apiKey string) *GraphQLClient {
	return &GraphQLClient{
		endpoint: endpoint,
		httpClient: &http.Client{
			Timeout: 30 * time.Second,
		},
		apiKey:      apiKey,
		cache:       NewQueryCache(1000, 5*time.Minute),
		rateLimiter: NewRateLimiter(100, time.Second), // 100 QPS
	}
}

// Query 执行 GraphQL 查询
func (c *GraphQLClient) Query(ctx context.Context, query string, variables map[string]interface{}, result interface{}) error {
	// 检查缓存
	cacheKey := c.buildCacheKey(query, variables)
	if cached, ok := c.cache.Get(cacheKey); ok {
		return json.Unmarshal(cached, result)
	}

	// 速率限制
	if err := c.rateLimiter.Wait(ctx); err != nil {
		return err
	}

	// 构建请求
	reqBody := GraphQLRequest{
		Query:     query,
		Variables: variables,
	}

	body, err := json.Marshal(reqBody)
	if err != nil {
		return err
	}

	req, err := http.NewRequestWithContext(ctx, "POST", c.endpoint, bytes.NewReader(body))
	if err != nil {
		return err
	}

	req.Header.Set("Content-Type", "application/json")
	if c.apiKey != "" {
		req.Header.Set("Authorization", "Bearer "+c.apiKey)
	}

	// 发送请求
	resp, err := c.httpClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	// 读取响应
	respBody, err := io.ReadAll(resp.Body)
	if err != nil {
		return err
	}

	var graphResp GraphQLResponse
	if err := json.Unmarshal(respBody, &graphResp); err != nil {
		return err
	}

	if len(graphResp.Errors) > 0 {
		return fmt.Errorf("graphql error: %s", graphResp.Errors[0].Message)
	}

	// 缓存结果
	c.cache.Set(cacheKey, graphResp.Data)

	return json.Unmarshal(graphResp.Data, result)
}

func (c *GraphQLClient) buildCacheKey(query string, variables map[string]interface{}) string {
	data, _ := json.Marshal(variables)
	return fmt.Sprintf("%x", sha256.Sum256(append([]byte(query), data...)))
}
```

### 1.2 Uniswap V3 Subgraph 查询

```go
// UniswapV3Subgraph Uniswap V3 数据查询
type UniswapV3Subgraph struct {
client *GraphQLClient
}

// Pool 池子信息
type Pool struct {
ID                 string   `json:"id"`
Token0             Token    `json:"token0"`
Token1             Token    `json:"token1"`
FeeTier            string   `json:"feeTier"`
Liquidity          string   `json:"liquidity"`
SqrtPrice          string   `json:"sqrtPrice"`
Tick               string   `json:"tick"`
Token0Price        string   `json:"token0Price"`
Token1Price        string   `json:"token1Price"`
VolumeUSD          string   `json:"volumeUSD"`
TotalValueLockedUSD string  `json:"totalValueLockedUSD"`
}

// Token 代币信息
type Token struct {
ID       string `json:"id"`
Symbol   string `json:"symbol"`
Name     string `json:"name"`
Decimals string `json:"decimals"`
}

// Swap 交换记录
type Swap struct {
ID          string `json:"id"`
Timestamp   string `json:"timestamp"`
Pool        Pool   `json:"pool"`
Sender      string `json:"sender"`
Recipient   string `json:"recipient"`
Amount0     string `json:"amount0"`
Amount1     string `json:"amount1"`
AmountUSD   string `json:"amountUSD"`
SqrtPriceX96 string `json:"sqrtPriceX96"`
Tick        string `json:"tick"`
LogIndex    string `json:"logIndex"`
}

// NewUniswapV3Subgraph 创建 Uniswap V3 查询器
func NewUniswapV3Subgraph(endpoint, apiKey string) *UniswapV3Subgraph {
return &UniswapV3Subgraph{
client: NewGraphQLClient(endpoint, apiKey),
}
}

// GetTopPools 获取 TVL 最高的池子
func (s *UniswapV3Subgraph) GetTopPools(ctx context.Context, limit int) ([]Pool, error) {
query := `
        query GetTopPools($limit: Int!) {
            pools(
                first: $limit
                orderBy: totalValueLockedUSD
                orderDirection: desc
                where: { liquidity_gt: "0" }
            ) {
                id
                token0 {
                    id
                    symbol
                    name
                    decimals
                }
                token1 {
                    id
                    symbol
                    name
                    decimals
                }
                feeTier
                liquidity
                sqrtPrice
                tick
                token0Price
                token1Price
                volumeUSD
                totalValueLockedUSD
            }
        }
    `

variables := map[string]interface{}{
"limit": limit,
}

var result struct {
Pools []Pool `json:"pools"`
}

if err := s.client.Query(ctx, query, variables, &result); err != nil {
return nil, err
}

return result.Pools, nil
}

// GetPoolsByTokenPair 根据代币对获取池子
func (s *UniswapV3Subgraph) GetPoolsByTokenPair(ctx context.Context, token0, token1 string) ([]Pool, error) {
query := `
        query GetPoolsByTokenPair($token0: String!, $token1: String!) {
            pools(
                where: {
                    or: [
                        { token0: $token0, token1: $token1 },
                        { token0: $token1, token1: $token0 }
                    ]
                }
            ) {
                id
                token0 {
                    id
                    symbol
                    decimals
                }
                token1 {
                    id
                    symbol
                    decimals
                }
                feeTier
                liquidity
                sqrtPrice
                tick
                token0Price
                token1Price
                totalValueLockedUSD
            }
        }
    `

variables := map[string]interface{}{
"token0": strings.ToLower(token0),
"token1": strings.ToLower(token1),
}

var result struct {
Pools []Pool `json:"pools"`
}

if err := s.client.Query(ctx, query, variables, &result); err != nil {
return nil, err
}

return result.Pools, nil
}

// GetRecentSwaps 获取最近的交换
func (s *UniswapV3Subgraph) GetRecentSwaps(ctx context.Context, poolID string, limit int) ([]Swap, error) {
query := `
        query GetRecentSwaps($poolID: String!, $limit: Int!) {
            swaps(
                first: $limit
                orderBy: timestamp
                orderDirection: desc
                where: { pool: $poolID }
            ) {
                id
                timestamp
                pool {
                    id
                    token0 { symbol }
                    token1 { symbol }
                }
                sender
                recipient
                amount0
                amount1
                amountUSD
                sqrtPriceX96
                tick
                logIndex
            }
        }
    `

variables := map[string]interface{}{
"poolID": strings.ToLower(poolID),
"limit":  limit,
}

var result struct {
Swaps []Swap `json:"swaps"`
}

if err := s.client.Query(ctx, query, variables, &result); err != nil {
return nil, err
}

return result.Swaps, nil
}

// GetPoolDayData 获取池子每日数据
func (s *UniswapV3Subgraph) GetPoolDayData(ctx context.Context, poolID string, days int) ([]PoolDayData, error) {
query := `
        query GetPoolDayData($poolID: String!, $days: Int!) {
            poolDayDatas(
                first: $days
                orderBy: date
                orderDirection: desc
                where: { pool: $poolID }
            ) {
                date
                pool { id }
                liquidity
                sqrtPrice
                token0Price
                token1Price
                tick
                tvlUSD
                volumeToken0
                volumeToken1
                volumeUSD
                feesUSD
                txCount
                open
                high
                low
                close
            }
        }
    `

variables := map[string]interface{}{
"poolID": strings.ToLower(poolID),
"days":   days,
}

var result struct {
PoolDayDatas []PoolDayData `json:"poolDayDatas"`
}

if err := s.client.Query(ctx, query, variables, &result); err != nil {
return nil, err
}

return result.PoolDayDatas, nil
}

// PoolDayData 池子每日数据
type PoolDayData struct {
Date         int64  `json:"date"`
Pool         Pool   `json:"pool"`
Liquidity    string `json:"liquidity"`
SqrtPrice    string `json:"sqrtPrice"`
Token0Price  string `json:"token0Price"`
Token1Price  string `json:"token1Price"`
Tick         string `json:"tick"`
TvlUSD       string `json:"tvlUSD"`
VolumeToken0 string `json:"volumeToken0"`
VolumeToken1 string `json:"volumeToken1"`
VolumeUSD    string `json:"volumeUSD"`
FeesUSD      string `json:"feesUSD"`
TxCount      string `json:"txCount"`
Open         string `json:"open"`
High         string `json:"high"`
Low          string `json:"low"`
Close        string `json:"close"`
}
```

### 1.3 多 Subgraph 聚合查询

```go
// SubgraphAggregator 多 Subgraph 聚合器
type SubgraphAggregator struct {
uniswapV3  *UniswapV3Subgraph
uniswapV2  *UniswapV2Subgraph
sushiswap  *SushiswapSubgraph
curve      *CurveSubgraph
balancer   *BalancerSubgraph
}

// AggregatedPool 聚合池子信息
type AggregatedPool struct {
Protocol   string
Address    common.Address
Token0     common.Address
Token1     common.Address
Fee        *big.Int
TVL        *big.Float
Volume24h  *big.Float
Price      *big.Float
}

// NewSubgraphAggregator 创建聚合器
func NewSubgraphAggregator(config *SubgraphConfig) *SubgraphAggregator {
return &SubgraphAggregator{
uniswapV3: NewUniswapV3Subgraph(config.UniswapV3Endpoint, config.APIKey),
uniswapV2: NewUniswapV2Subgraph(config.UniswapV2Endpoint, config.APIKey),
sushiswap: NewSushiswapSubgraph(config.SushiswapEndpoint, config.APIKey),
curve:     NewCurveSubgraph(config.CurveEndpoint, config.APIKey),
balancer:  NewBalancerSubgraph(config.BalancerEndpoint, config.APIKey),
}
}

// GetAllPoolsForPair 获取所有协议的池子
func (a *SubgraphAggregator) GetAllPoolsForPair(ctx context.Context, token0, token1 string) ([]AggregatedPool, error) {
var allPools []AggregatedPool
var mu sync.Mutex
var wg sync.WaitGroup

// 并行查询各协议
queries := []struct {
name  string
query func () ([]AggregatedPool, error)
}{
{
name: "UniswapV3",
query: func () ([]AggregatedPool, error) {
pools, err := a.uniswapV3.GetPoolsByTokenPair(ctx, token0, token1)
if err != nil {
return nil, err
}
return a.convertUniV3Pools(pools), nil
},
},
{
name: "UniswapV2",
query: func () ([]AggregatedPool, error) {
pools, err := a.uniswapV2.GetPoolsByTokenPair(ctx, token0, token1)
if err != nil {
return nil, err
}
return a.convertUniV2Pools(pools), nil
},
},
{
name: "Sushiswap",
query: func () ([]AggregatedPool, error) {
pools, err := a.sushiswap.GetPoolsByTokenPair(ctx, token0, token1)
if err != nil {
return nil, err
}
return a.convertSushiPools(pools), nil
},
},
}

for _, q := range queries {
wg.Add(1)
go func (query func () ([]AggregatedPool, error)) {
defer wg.Done()
pools, err := query()
if err != nil {
return // 忽略错误，继续其他查询
}
mu.Lock()
allPools = append(allPools, pools...)
mu.Unlock()
}(q.query)
}

wg.Wait()

// 按 TVL 排序
sort.Slice(allPools, func (i, j int) bool {
return allPools[i].TVL.Cmp(allPools[j].TVL) > 0
})

return allPools, nil
}

// FindArbitrageOpportunities 查找套利机会
func (a *SubgraphAggregator) FindArbitrageOpportunities(ctx context.Context, minProfitBps int) ([]ArbitrageOpportunity, error) {
// 获取热门代币对
tokenPairs := []struct {
token0, token1 string
}{
{"0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", "0xA0b86a33E6441e8684636C6E5b1E9e5E3e8E8A8F"}, // WETH-USDC
{"0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", "0xdAC17F958D2ee523a2206206994597C13D831ec7"}, // WETH-USDT
{"0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599", "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"}, // WBTC-WETH
}

var opportunities []ArbitrageOpportunity

for _, pair := range tokenPairs {
pools, err := a.GetAllPoolsForPair(ctx, pair.token0, pair.token1)
if err != nil {
continue
}

// 比较不同协议的价格
for i := 0; i < len(pools); i++ {
for j := i + 1; j < len(pools); j++ {
priceDiff := a.calculatePriceDiff(pools[i].Price, pools[j].Price)
if priceDiff >= float64(minProfitBps)/10000 {
opportunities = append(opportunities, ArbitrageOpportunity{
BuyPool:      pools[i],
SellPool:     pools[j],
PriceDiffBps: int(priceDiff * 10000),
})
}
}
}
}

return opportunities, nil
}

func (a *SubgraphAggregator) calculatePriceDiff(price1, price2 *big.Float) float64 {
diff := new(big.Float).Sub(price1, price2)
diff.Abs(diff)
avg := new(big.Float).Add(price1, price2)
avg.Quo(avg, big.NewFloat(2))
result := new(big.Float).Quo(diff, avg)
f, _ := result.Float64()
return f
}
```

## 2. Dune Analytics 集成

### 2.1 Dune API 客户端

```go
// DuneClient Dune Analytics 客户端
type DuneClient struct {
apiKey     string
httpClient *http.Client
baseURL    string
}

// QueryExecution 查询执行信息
type QueryExecution struct {
ExecutionID     string    `json:"execution_id"`
State           string    `json:"state"`
SubmittedAt     time.Time `json:"submitted_at"`
ExpiresAt       time.Time `json:"expires_at"`
ExecutionStarted time.Time `json:"execution_started_at,omitempty"`
ExecutionEnded  time.Time `json:"execution_ended_at,omitempty"`
}

// QueryResult 查询结果
type QueryResult struct {
ExecutionID string `json:"execution_id"`
State       string `json:"state"`
Result      struct {
Rows     []map[string]interface{} `json:"rows"`
Metadata struct {
ColumnNames []string `json:"column_names"`
ColumnTypes []string `json:"column_types"`
RowCount    int      `json:"row_count"`
} `json:"metadata"`
} `json:"result"`
}

// NewDuneClient 创建 Dune 客户端
func NewDuneClient(apiKey string) *DuneClient {
return &DuneClient{
apiKey:  apiKey,
baseURL: "https://api.dune.com/api/v1",
httpClient: &http.Client{
Timeout: 60 * time.Second,
},
}
}

// ExecuteQuery 执行查询
func (c *DuneClient) ExecuteQuery(ctx context.Context, queryID int, parameters map[string]interface{}) (*QueryExecution, error) {
url := fmt.Sprintf("%s/query/%d/execute", c.baseURL, queryID)

body, _ := json.Marshal(map[string]interface{}{
"query_parameters": parameters,
})

req, err := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
if err != nil {
return nil, err
}

req.Header.Set("Content-Type", "application/json")
req.Header.Set("X-Dune-API-Key", c.apiKey)

resp, err := c.httpClient.Do(req)
if err != nil {
return nil, err
}
defer resp.Body.Close()

var result QueryExecution
if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
return nil, err
}

return &result, nil
}

// GetExecutionStatus 获取执行状态
func (c *DuneClient) GetExecutionStatus(ctx context.Context, executionID string) (*QueryExecution, error) {
url := fmt.Sprintf("%s/execution/%s/status", c.baseURL, executionID)

req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
if err != nil {
return nil, err
}

req.Header.Set("X-Dune-API-Key", c.apiKey)

resp, err := c.httpClient.Do(req)
if err != nil {
return nil, err
}
defer resp.Body.Close()

var result QueryExecution
if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
return nil, err
}

return &result, nil
}

// GetExecutionResults 获取执行结果
func (c *DuneClient) GetExecutionResults(ctx context.Context, executionID string) (*QueryResult, error) {
url := fmt.Sprintf("%s/execution/%s/results", c.baseURL, executionID)

req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
if err != nil {
return nil, err
}

req.Header.Set("X-Dune-API-Key", c.apiKey)

resp, err := c.httpClient.Do(req)
if err != nil {
return nil, err
}
defer resp.Body.Close()

var result QueryResult
if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
return nil, err
}

return &result, nil
}

// ExecuteAndWait 执行查询并等待结果
func (c *DuneClient) ExecuteAndWait(ctx context.Context, queryID int, parameters map[string]interface{}) (*QueryResult, error) {
// 执行查询
execution, err := c.ExecuteQuery(ctx, queryID, parameters)
if err != nil {
return nil, err
}

// 轮询状态
ticker := time.NewTicker(2 * time.Second)
defer ticker.Stop()

timeout := time.After(5 * time.Minute)

for {
select {
case <-ctx.Done():
return nil, ctx.Err()
case <-timeout:
return nil, fmt.Errorf("query timeout")
case <-ticker.C:
status, err := c.GetExecutionStatus(ctx, execution.ExecutionID)
if err != nil {
continue
}

switch status.State {
case "QUERY_STATE_COMPLETED":
return c.GetExecutionResults(ctx, execution.ExecutionID)
case "QUERY_STATE_FAILED":
return nil, fmt.Errorf("query failed")
case "QUERY_STATE_CANCELLED":
return nil, fmt.Errorf("query cancelled")
}
}
}
}
```

### 2.2 MEV 数据分析

```go
// MEVAnalyzer MEV 数据分析器
type MEVAnalyzer struct {
dune *DuneClient

// 预定义查询 ID
sandwichQueryID     int
arbitrageQueryID    int
liquidationQueryID  int
}

// SandwichAttack 三明治攻击数据
type SandwichAttack struct {
BlockNumber     int64     `json:"block_number"`
TxHash          string    `json:"tx_hash"`
Attacker        string    `json:"attacker"`
Victim          string    `json:"victim"`
Protocol        string    `json:"protocol"`
ProfitETH       float64   `json:"profit_eth"`
ProfitUSD       float64   `json:"profit_usd"`
GasCostETH      float64   `json:"gas_cost_eth"`
NetProfitETH    float64   `json:"net_profit_eth"`
Timestamp       time.Time `json:"timestamp"`
}

// ArbitrageTransaction 套利交易数据
type ArbitrageTransaction struct {
BlockNumber   int64     `json:"block_number"`
TxHash        string    `json:"tx_hash"`
Searcher      string    `json:"searcher"`
Protocols     []string  `json:"protocols"`
InputToken    string    `json:"input_token"`
OutputToken   string    `json:"output_token"`
InputAmount   float64   `json:"input_amount"`
OutputAmount  float64   `json:"output_amount"`
ProfitUSD     float64   `json:"profit_usd"`
GasCostUSD    float64   `json:"gas_cost_usd"`
NetProfitUSD  float64   `json:"net_profit_usd"`
Timestamp     time.Time `json:"timestamp"`
}

// NewMEVAnalyzer 创建 MEV 分析器
func NewMEVAnalyzer(apiKey string, config *MEVQueryConfig) *MEVAnalyzer {
return &MEVAnalyzer{
dune:               NewDuneClient(apiKey),
sandwichQueryID:    config.SandwichQueryID,
arbitrageQueryID:   config.ArbitrageQueryID,
liquidationQueryID: config.LiquidationQueryID,
}
}

// GetRecentSandwichAttacks 获取最近的三明治攻击
func (a *MEVAnalyzer) GetRecentSandwichAttacks(ctx context.Context, days int) ([]SandwichAttack, error) {
result, err := a.dune.ExecuteAndWait(ctx, a.sandwichQueryID, map[string]interface{}{
"days": days,
})
if err != nil {
return nil, err
}

var attacks []SandwichAttack
for _, row := range result.Result.Rows {
attack := SandwichAttack{
BlockNumber:  int64(row["block_number"].(float64)),
TxHash:       row["tx_hash"].(string),
Attacker:     row["attacker"].(string),
Victim:       row["victim"].(string),
Protocol:     row["protocol"].(string),
ProfitETH:    row["profit_eth"].(float64),
ProfitUSD:    row["profit_usd"].(float64),
GasCostETH:   row["gas_cost_eth"].(float64),
NetProfitETH: row["net_profit_eth"].(float64),
}
attacks = append(attacks, attack)
}

return attacks, nil
}

// GetTopArbitragers 获取顶级套利者
func (a *MEVAnalyzer) GetTopArbitragers(ctx context.Context, days int, limit int) ([]ArbitragerStats, error) {
// 自定义查询
queryID := 12345 // 需要创建对应的 Dune 查询

result, err := a.dune.ExecuteAndWait(ctx, queryID, map[string]interface{}{
"days":  days,
"limit": limit,
})
if err != nil {
return nil, err
}

var stats []ArbitragerStats
for _, row := range result.Result.Rows {
stat := ArbitragerStats{
Address:        row["address"].(string),
TotalTrades:    int(row["total_trades"].(float64)),
SuccessfulTrades: int(row["successful_trades"].(float64)),
TotalProfitUSD: row["total_profit_usd"].(float64),
AvgProfitUSD:   row["avg_profit_usd"].(float64),
WinRate:        row["win_rate"].(float64),
}
stats = append(stats, stat)
}

return stats, nil
}

// ArbitragerStats 套利者统计
type ArbitragerStats struct {
Address          string  `json:"address"`
TotalTrades      int     `json:"total_trades"`
SuccessfulTrades int     `json:"successful_trades"`
TotalProfitUSD   float64 `json:"total_profit_usd"`
AvgProfitUSD     float64 `json:"avg_profit_usd"`
WinRate          float64 `json:"win_rate"`
}

// AnalyzeProtocolVolume 分析协议交易量
func (a *MEVAnalyzer) AnalyzeProtocolVolume(ctx context.Context, protocol string, days int) (*ProtocolVolumeAnalysis, error) {
queryID := 12346 // 协议交易量分析查询

result, err := a.dune.ExecuteAndWait(ctx, queryID, map[string]interface{}{
"protocol": protocol,
"days":     days,
})
if err != nil {
return nil, err
}

if len(result.Result.Rows) == 0 {
return nil, fmt.Errorf("no data found")
}

row := result.Result.Rows[0]
return &ProtocolVolumeAnalysis{
Protocol:       protocol,
TotalVolumeUSD: row["total_volume_usd"].(float64),
TotalTrades:    int(row["total_trades"].(float64)),
UniqueTrades:   int(row["unique_traders"].(float64)),
AvgTradeSize:   row["avg_trade_size"].(float64),
DailyVolumes:   a.extractDailyVolumes(result),
}, nil
}

// ProtocolVolumeAnalysis 协议交易量分析
type ProtocolVolumeAnalysis struct {
Protocol       string    `json:"protocol"`
TotalVolumeUSD float64   `json:"total_volume_usd"`
TotalTrades    int       `json:"total_trades"`
UniqueTrades   int       `json:"unique_traders"`
AvgTradeSize   float64   `json:"avg_trade_size"`
DailyVolumes   []float64 `json:"daily_volumes"`
}

func (a *MEVAnalyzer) extractDailyVolumes(result *QueryResult) []float64 {
var volumes []float64
for _, row := range result.Result.Rows {
if v, ok := row["daily_volume"].(float64); ok {
volumes = append(volumes, v)
}
}
return volumes
}
```

## 3. 实时事件索引

### 3.1 事件订阅与索引

```go
// EventIndexer 事件索引器
type EventIndexer struct {
client     *ethclient.Client
wsClient   *ethclient.Client
db         *IndexerDB

// 事件处理器
handlers   map[common.Hash]EventHandler

// 索引状态
lastBlock  uint64

// 并发控制
workers    int
workChan   chan *types.Log
}

// EventHandler 事件处理器
type EventHandler func (log *types.Log) error

// IndexedEvent 索引的事件
type IndexedEvent struct {
ID          uint64         `db:"id"`
BlockNumber uint64         `db:"block_number"`
TxHash      common.Hash    `db:"tx_hash"`
LogIndex    uint           `db:"log_index"`
Address     common.Address `db:"address"`
Topics      []common.Hash  `db:"topics"`
Data        []byte         `db:"data"`
EventType   string         `db:"event_type"`
Decoded     json.RawMessage `db:"decoded"`
Timestamp   time.Time      `db:"timestamp"`
}

// NewEventIndexer 创建事件索引器
func NewEventIndexer(rpcURL, wsURL string, db *IndexerDB) (*EventIndexer, error) {
client, err := ethclient.Dial(rpcURL)
if err != nil {
return nil, err
}

wsClient, err := ethclient.Dial(wsURL)
if err != nil {
return nil, err
}

return &EventIndexer{
client:    client,
wsClient:  wsClient,
db:        db,
handlers:  make(map[common.Hash]EventHandler),
workers:   10,
workChan:  make(chan *types.Log, 1000),
}, nil
}

// RegisterHandler 注册事件处理器
func (i *EventIndexer) RegisterHandler(eventSig common.Hash, handler EventHandler) {
i.handlers[eventSig] = handler
}

// Start 开始索引
func (i *EventIndexer) Start(ctx context.Context, fromBlock uint64) error {
i.lastBlock = fromBlock

// 启动 worker
for w := 0; w < i.workers; w++ {
go i.worker(ctx)
}

// 先同步历史数据
if err := i.syncHistorical(ctx, fromBlock); err != nil {
return err
}

// 订阅新事件
return i.subscribeNewEvents(ctx)
}

// syncHistorical 同步历史数据
func (i *EventIndexer) syncHistorical(ctx context.Context, fromBlock uint64) error {
currentBlock, err := i.client.BlockNumber(ctx)
if err != nil {
return err
}

batchSize := uint64(1000)

for startBlock := fromBlock; startBlock < currentBlock; startBlock += batchSize {
endBlock := startBlock + batchSize - 1
if endBlock > currentBlock {
endBlock = currentBlock
}

// 获取事件签名列表
topics := make([]common.Hash, 0, len(i.handlers))
for sig := range i.handlers {
topics = append(topics, sig)
}

query := ethereum.FilterQuery{
FromBlock: big.NewInt(int64(startBlock)),
ToBlock:   big.NewInt(int64(endBlock)),
Topics:    [][]common.Hash{topics},
}

logs, err := i.client.FilterLogs(ctx, query)
if err != nil {
return err
}

for _, log := range logs {
logCopy := log
i.workChan <- &logCopy
}

i.lastBlock = endBlock
log.Printf("Indexed blocks %d to %d", startBlock, endBlock)
}

return nil
}

// subscribeNewEvents 订阅新事件
func (i *EventIndexer) subscribeNewEvents(ctx context.Context) error {
topics := make([]common.Hash, 0, len(i.handlers))
for sig := range i.handlers {
topics = append(topics, sig)
}

query := ethereum.FilterQuery{
Topics: [][]common.Hash{topics},
}

logsChan := make(chan types.Log)
sub, err := i.wsClient.SubscribeFilterLogs(ctx, query, logsChan)
if err != nil {
return err
}

for {
select {
case <-ctx.Done():
sub.Unsubscribe()
return ctx.Err()
case err := <-sub.Err():
return err
case log := <-logsChan:
i.workChan <- &log
}
}
}

// worker 工作协程
func (i *EventIndexer) worker(ctx context.Context) {
for {
select {
case <-ctx.Done():
return
case log := <-i.workChan:
i.processLog(ctx, log)
}
}
}

// processLog 处理日志
func (i *EventIndexer) processLog(ctx context.Context, log *types.Log) {
if len(log.Topics) == 0 {
return
}

handler, ok := i.handlers[log.Topics[0]]
if !ok {
return
}

// 调用处理器
if err := handler(log); err != nil {
log.Printf("Error processing log: %v", err)
}

// 存储到数据库
i.db.StoreEvent(&IndexedEvent{
BlockNumber: log.BlockNumber,
TxHash:      log.TxHash,
LogIndex:    log.Index,
Address:     log.Address,
Topics:      log.Topics,
Data:        log.Data,
Timestamp:   time.Now(),
})
}
```

### 3.2 DEX 交换事件索引

```go
// SwapIndexer DEX 交换索引器
type SwapIndexer struct {
*EventIndexer

// 解码器
uniswapV2ABI *abi.ABI
uniswapV3ABI *abi.ABI

// 池子缓存
poolCache *sync.Map
}

// IndexedSwap 索引的交换
type IndexedSwap struct {
ID          uint64         `db:"id"`
BlockNumber uint64         `db:"block_number"`
TxHash      common.Hash    `db:"tx_hash"`
LogIndex    uint           `db:"log_index"`
Protocol    string         `db:"protocol"`
PoolAddress common.Address `db:"pool_address"`
Sender      common.Address `db:"sender"`
Recipient   common.Address `db:"recipient"`
Token0      common.Address `db:"token0"`
Token1      common.Address `db:"token1"`
Amount0In   *big.Int       `db:"amount0_in"`
Amount1In   *big.Int       `db:"amount1_in"`
Amount0Out  *big.Int       `db:"amount0_out"`
Amount1Out  *big.Int       `db:"amount1_out"`
Price       *big.Float     `db:"price"`
Timestamp   time.Time      `db:"timestamp"`
}

// NewSwapIndexer 创建交换索引器
func NewSwapIndexer(rpcURL, wsURL string, db *IndexerDB) (*SwapIndexer, error) {
base, err := NewEventIndexer(rpcURL, wsURL, db)
if err != nil {
return nil, err
}

// 解析 ABI
uniswapV2ABI, _ := abi.JSON(strings.NewReader(UniswapV2PairABI))
uniswapV3ABI, _ := abi.JSON(strings.NewReader(UniswapV3PoolABI))

indexer := &SwapIndexer{
EventIndexer: base,
uniswapV2ABI: &uniswapV2ABI,
uniswapV3ABI: &uniswapV3ABI,
poolCache:    &sync.Map{},
}

// 注册事件处理器
// Uniswap V2 Swap 事件
indexer.RegisterHandler(
common.HexToHash("0xd78ad95fa46c994b6551d0da85fc275fe613ce37657fb8d5e3d130840159d822"),
indexer.handleUniswapV2Swap,
)

// Uniswap V3 Swap 事件
indexer.RegisterHandler(
common.HexToHash("0xc42079f94a6350d7e6235f29174924f928cc2ac818eb64fed8004e115fbcca67"),
indexer.handleUniswapV3Swap,
)

return indexer, nil
}

// handleUniswapV2Swap 处理 Uniswap V2 交换
func (i *SwapIndexer) handleUniswapV2Swap(log *types.Log) error {
event := struct {
Sender     common.Address
Amount0In  *big.Int
Amount1In  *big.Int
Amount0Out *big.Int
Amount1Out *big.Int
To         common.Address
}{}

if err := i.uniswapV2ABI.UnpackIntoInterface(&event, "Swap", log.Data); err != nil {
return err
}

// 获取池子信息
poolInfo, err := i.getPoolInfo(log.Address, "uniswapv2")
if err != nil {
return err
}

// 计算价格
price := i.calculatePrice(event.Amount0In, event.Amount1In, event.Amount0Out, event.Amount1Out)

swap := &IndexedSwap{
BlockNumber: log.BlockNumber,
TxHash:      log.TxHash,
LogIndex:    log.Index,
Protocol:    "UniswapV2",
PoolAddress: log.Address,
Sender:      event.Sender,
Recipient:   event.To,
Token0:      poolInfo.Token0,
Token1:      poolInfo.Token1,
Amount0In:   event.Amount0In,
Amount1In:   event.Amount1In,
Amount0Out:  event.Amount0Out,
Amount1Out:  event.Amount1Out,
Price:       price,
Timestamp:   time.Now(),
}

return i.db.StoreSwap(swap)
}

// handleUniswapV3Swap 处理 Uniswap V3 交换
func (i *SwapIndexer) handleUniswapV3Swap(log *types.Log) error {
event := struct {
Sender       common.Address
Recipient    common.Address
Amount0      *big.Int
Amount1      *big.Int
SqrtPriceX96 *big.Int
Liquidity    *big.Int
Tick         *big.Int
}{}

// Uniswap V3 的前两个参数在 topics 中
if len(log.Topics) < 3 {
return fmt.Errorf("invalid log topics")
}
event.Sender = common.BytesToAddress(log.Topics[1].Bytes())
event.Recipient = common.BytesToAddress(log.Topics[2].Bytes())

if err := i.uniswapV3ABI.UnpackIntoInterface(&event, "Swap", log.Data); err != nil {
return err
}

poolInfo, err := i.getPoolInfo(log.Address, "uniswapv3")
if err != nil {
return err
}

// 从 sqrtPriceX96 计算价格
price := i.sqrtPriceX96ToPrice(event.SqrtPriceX96, poolInfo.Token0Decimals, poolInfo.Token1Decimals)

// 确定输入输出
var amount0In, amount1In, amount0Out, amount1Out *big.Int
if event.Amount0.Sign() > 0 {
amount0In = event.Amount0
amount1Out = new(big.Int).Neg(event.Amount1)
} else {
amount1In = event.Amount1
amount0Out = new(big.Int).Neg(event.Amount0)
}

swap := &IndexedSwap{
BlockNumber: log.BlockNumber,
TxHash:      log.TxHash,
LogIndex:    log.Index,
Protocol:    "UniswapV3",
PoolAddress: log.Address,
Sender:      event.Sender,
Recipient:   event.Recipient,
Token0:      poolInfo.Token0,
Token1:      poolInfo.Token1,
Amount0In:   amount0In,
Amount1In:   amount1In,
Amount0Out:  amount0Out,
Amount1Out:  amount1Out,
Price:       price,
Timestamp:   time.Now(),
}

return i.db.StoreSwap(swap)
}

func (i *SwapIndexer) sqrtPriceX96ToPrice(sqrtPriceX96 *big.Int, decimals0, decimals1 int) *big.Float {
// price = (sqrtPriceX96 / 2^96)^2 * 10^(decimals0 - decimals1)
q96 := new(big.Int).Lsh(big.NewInt(1), 96)

sqrtPrice := new(big.Float).SetInt(sqrtPriceX96)
sqrtPrice.Quo(sqrtPrice, new(big.Float).SetInt(q96))

price := new(big.Float).Mul(sqrtPrice, sqrtPrice)

decimalAdjustment := new(big.Float).SetFloat64(math.Pow(10, float64(decimals0-decimals1)))
price.Mul(price, decimalAdjustment)

return price
}
```

## 4. 数据分析与可视化

### 4.1 套利机会分析器

```go
// ArbitrageAnalyzer 套利机会分析器
type ArbitrageAnalyzer struct {
db        *IndexerDB
subgraph  *SubgraphAggregator
dune      *DuneClient
}

// PriceDiscrepancy 价格差异
type PriceDiscrepancy struct {
TokenPair    string    `json:"token_pair"`
Protocol1    string    `json:"protocol1"`
Protocol2    string    `json:"protocol2"`
Price1       float64   `json:"price1"`
Price2       float64   `json:"price2"`
DiffPercent  float64   `json:"diff_percent"`
Liquidity1   float64   `json:"liquidity1"`
Liquidity2   float64   `json:"liquidity2"`
MaxTradeSize float64   `json:"max_trade_size"`
EstProfit    float64   `json:"est_profit"`
Timestamp    time.Time `json:"timestamp"`
}

// ArbitrageStats 套利统计
type ArbitrageStats struct {
TotalOpportunities int     `json:"total_opportunities"`
ExecutedTrades     int     `json:"executed_trades"`
SuccessfulTrades   int     `json:"successful_trades"`
TotalProfit        float64 `json:"total_profit"`
AvgProfit          float64 `json:"avg_profit"`
WinRate            float64 `json:"win_rate"`
BestTrade          float64 `json:"best_trade"`
WorstTrade         float64 `json:"worst_trade"`
}

// NewArbitrageAnalyzer 创建分析器
func NewArbitrageAnalyzer(db *IndexerDB, subgraph *SubgraphAggregator, dune *DuneClient) *ArbitrageAnalyzer {
return &ArbitrageAnalyzer{
db:       db,
subgraph: subgraph,
dune:     dune,
}
}

// FindPriceDiscrepancies 查找价格差异
func (a *ArbitrageAnalyzer) FindPriceDiscrepancies(ctx context.Context, minDiffPercent float64) ([]PriceDiscrepancy, error) {
// 获取最新交换数据
swaps, err := a.db.GetRecentSwaps(ctx, 100)
if err != nil {
return nil, err
}

// 按代币对分组
pricesByPair := make(map[string][]struct {
Protocol  string
Price     float64
Liquidity float64
})

for _, swap := range swaps {
pair := fmt.Sprintf("%s-%s", swap.Token0.Hex()[:8], swap.Token1.Hex()[:8])
pricesByPair[pair] = append(pricesByPair[pair], struct {
Protocol  string
Price     float64
Liquidity float64
}{
Protocol:  swap.Protocol,
Price:     swap.Price.Float64(),
Liquidity: 0, // 需要从其他来源获取
})
}

// 查找差异
var discrepancies []PriceDiscrepancy

for pair, prices := range pricesByPair {
for i := 0; i < len(prices); i++ {
for j := i + 1; j < len(prices); j++ {
diff := math.Abs(prices[i].Price-prices[j].Price) / prices[i].Price * 100

if diff >= minDiffPercent {
discrepancies = append(discrepancies, PriceDiscrepancy{
TokenPair:   pair,
Protocol1:   prices[i].Protocol,
Protocol2:   prices[j].Protocol,
Price1:      prices[i].Price,
Price2:      prices[j].Price,
DiffPercent: diff,
Timestamp:   time.Now(),
})
}
}
}
}

return discrepancies, nil
}

// GetHistoricalStats 获取历史统计
func (a *ArbitrageAnalyzer) GetHistoricalStats(ctx context.Context, days int) (*ArbitrageStats, error) {
// 从 Dune 获取历史数据
queryID := 12347 // 历史套利统计查询

result, err := a.dune.ExecuteAndWait(ctx, queryID, map[string]interface{}{
"days": days,
})
if err != nil {
return nil, err
}

if len(result.Result.Rows) == 0 {
return nil, fmt.Errorf("no data")
}

row := result.Result.Rows[0]
return &ArbitrageStats{
TotalOpportunities: int(row["total_opportunities"].(float64)),
ExecutedTrades:     int(row["executed_trades"].(float64)),
SuccessfulTrades:   int(row["successful_trades"].(float64)),
TotalProfit:        row["total_profit"].(float64),
AvgProfit:          row["avg_profit"].(float64),
WinRate:            row["win_rate"].(float64),
BestTrade:          row["best_trade"].(float64),
WorstTrade:         row["worst_trade"].(float64),
}, nil
}

// AnalyzeProfitableRoutes 分析盈利路径
func (a *ArbitrageAnalyzer) AnalyzeProfitableRoutes(ctx context.Context) ([]RouteAnalysis, error) {
// 获取历史套利交易
trades, err := a.db.GetArbitrageTrades(ctx, 1000)
if err != nil {
return nil, err
}

// 按路径分组统计
routeStats := make(map[string]*RouteAnalysis)

for _, trade := range trades {
routeKey := strings.Join(trade.Protocols, "->")

if _, ok := routeStats[routeKey]; !ok {
routeStats[routeKey] = &RouteAnalysis{
Route:     trade.Protocols,
RouteKey:  routeKey,
Trades:    0,
TotalProfit: 0,
}
}

stats := routeStats[routeKey]
stats.Trades++
stats.TotalProfit += trade.ProfitUSD

if trade.ProfitUSD > 0 {
stats.WinningTrades++
}
}

// 转换为切片并计算平均值
var results []RouteAnalysis
for _, stats := range routeStats {
stats.AvgProfit = stats.TotalProfit / float64(stats.Trades)
stats.WinRate = float64(stats.WinningTrades) / float64(stats.Trades)
results = append(results, *stats)
}

// 按总利润排序
sort.Slice(results, func (i, j int) bool {
return results[i].TotalProfit > results[j].TotalProfit
})

return results, nil
}

// RouteAnalysis 路径分析
type RouteAnalysis struct {
Route         []string `json:"route"`
RouteKey      string   `json:"route_key"`
Trades        int      `json:"trades"`
WinningTrades int      `json:"winning_trades"`
TotalProfit   float64  `json:"total_profit"`
AvgProfit     float64  `json:"avg_profit"`
WinRate       float64  `json:"win_rate"`
}
```

### 4.2 实时监控面板

```go
// DashboardServer 监控面板服务器
type DashboardServer struct {
analyzer *ArbitrageAnalyzer
indexer  *SwapIndexer

// WebSocket 连接
clients  map[*websocket.Conn]bool
mu       sync.RWMutex

// 实时数据
latestSwaps       []IndexedSwap
latestDiscrepancies []PriceDiscrepancy
}

// DashboardData 面板数据
type DashboardData struct {
Stats            ArbitrageStats      `json:"stats"`
RecentSwaps      []IndexedSwap       `json:"recent_swaps"`
Discrepancies    []PriceDiscrepancy  `json:"discrepancies"`
TopRoutes        []RouteAnalysis     `json:"top_routes"`
GasPrice         uint64              `json:"gas_price"`
ETHPrice         float64             `json:"eth_price"`
LastUpdated      time.Time           `json:"last_updated"`
}

// NewDashboardServer 创建面板服务器
func NewDashboardServer(analyzer *ArbitrageAnalyzer, indexer *SwapIndexer) *DashboardServer {
return &DashboardServer{
analyzer: analyzer,
indexer:  indexer,
clients:  make(map[*websocket.Conn]bool),
}
}

// Start 启动服务器
func (s *DashboardServer) Start(addr string) error {
http.HandleFunc("/api/dashboard", s.handleDashboard)
http.HandleFunc("/api/stats", s.handleStats)
http.HandleFunc("/api/swaps", s.handleSwaps)
http.HandleFunc("/ws", s.handleWebSocket)
http.Handle("/", http.FileServer(http.Dir("./static")))

// 启动数据更新协程
go s.updateLoop()

return http.ListenAndServe(addr, nil)
}

// handleDashboard 处理面板数据请求
func (s *DashboardServer) handleDashboard(w http.ResponseWriter, r *http.Request) {
ctx := r.Context()

// 获取统计数据
stats, err := s.analyzer.GetHistoricalStats(ctx, 7)
if err != nil {
http.Error(w, err.Error(), http.StatusInternalServerError)
return
}

// 获取最近交换
swaps, _ := s.indexer.db.GetRecentSwaps(ctx, 50)

// 获取价格差异
discrepancies, _ := s.analyzer.FindPriceDiscrepancies(ctx, 0.5)

// 获取顶级路径
routes, _ := s.analyzer.AnalyzeProfitableRoutes(ctx)
if len(routes) > 10 {
routes = routes[:10]
}

data := DashboardData{
Stats:         *stats,
RecentSwaps:   swaps,
Discrepancies: discrepancies,
TopRoutes:     routes,
LastUpdated:   time.Now(),
}

w.Header().Set("Content-Type", "application/json")
json.NewEncoder(w).Encode(data)
}

// handleWebSocket 处理 WebSocket 连接
func (s *DashboardServer) handleWebSocket(w http.ResponseWriter, r *http.Request) {
upgrader := websocket.Upgrader{
CheckOrigin: func (r *http.Request) bool { return true },
}

conn, err := upgrader.Upgrade(w, r, nil)
if err != nil {
return
}
defer conn.Close()

s.mu.Lock()
s.clients[conn] = true
s.mu.Unlock()

defer func () {
s.mu.Lock()
delete(s.clients, conn)
s.mu.Unlock()
}()

// 保持连接
for {
_, _, err := conn.ReadMessage()
if err != nil {
break
}
}
}

// broadcast 广播数据
func (s *DashboardServer) broadcast(data interface{}) {
msg, _ := json.Marshal(data)

s.mu.RLock()
defer s.mu.RUnlock()

for client := range s.clients {
client.WriteMessage(websocket.TextMessage, msg)
}
}

// updateLoop 数据更新循环
func (s *DashboardServer) updateLoop() {
ticker := time.NewTicker(5 * time.Second)
defer ticker.Stop()

for range ticker.C {
ctx := context.Background()

// 查找新的套利机会
discrepancies, err := s.analyzer.FindPriceDiscrepancies(ctx, 0.5)
if err == nil && len(discrepancies) > 0 {
s.broadcast(map[string]interface{}{
"type": "discrepancy",
"data": discrepancies,
})
}
}
}
```

## 5. 数据库存储

### 5.1 索引数据库

```go
// IndexerDB 索引数据库
type IndexerDB struct {
db *sql.DB
}

// NewIndexerDB 创建索引数据库
func NewIndexerDB(dsn string) (*IndexerDB, error) {
db, err := sql.Open("postgres", dsn)
if err != nil {
return nil, err
}

if err := db.Ping(); err != nil {
return nil, err
}

indexerDB := &IndexerDB{db: db}
if err := indexerDB.migrate(); err != nil {
return nil, err
}

return indexerDB, nil
}

// migrate 数据库迁移
func (d *IndexerDB) migrate() error {
schema := `
    CREATE TABLE IF NOT EXISTS events (
        id BIGSERIAL PRIMARY KEY,
        block_number BIGINT NOT NULL,
        tx_hash BYTEA NOT NULL,
        log_index INT NOT NULL,
        address BYTEA NOT NULL,
        topics BYTEA[],
        data BYTEA,
        event_type VARCHAR(50),
        decoded JSONB,
        timestamp TIMESTAMP DEFAULT NOW(),
        UNIQUE(tx_hash, log_index)
    );

    CREATE INDEX IF NOT EXISTS idx_events_block ON events(block_number);
    CREATE INDEX IF NOT EXISTS idx_events_address ON events(address);
    CREATE INDEX IF NOT EXISTS idx_events_type ON events(event_type);

    CREATE TABLE IF NOT EXISTS swaps (
        id BIGSERIAL PRIMARY KEY,
        block_number BIGINT NOT NULL,
        tx_hash BYTEA NOT NULL,
        log_index INT NOT NULL,
        protocol VARCHAR(50) NOT NULL,
        pool_address BYTEA NOT NULL,
        sender BYTEA NOT NULL,
        recipient BYTEA NOT NULL,
        token0 BYTEA NOT NULL,
        token1 BYTEA NOT NULL,
        amount0_in NUMERIC,
        amount1_in NUMERIC,
        amount0_out NUMERIC,
        amount1_out NUMERIC,
        price NUMERIC,
        timestamp TIMESTAMP DEFAULT NOW(),
        UNIQUE(tx_hash, log_index)
    );

    CREATE INDEX IF NOT EXISTS idx_swaps_block ON swaps(block_number);
    CREATE INDEX IF NOT EXISTS idx_swaps_pool ON swaps(pool_address);
    CREATE INDEX IF NOT EXISTS idx_swaps_protocol ON swaps(protocol);
    CREATE INDEX IF NOT EXISTS idx_swaps_timestamp ON swaps(timestamp);

    CREATE TABLE IF NOT EXISTS arbitrage_trades (
        id BIGSERIAL PRIMARY KEY,
        block_number BIGINT NOT NULL,
        tx_hash BYTEA NOT NULL,
        searcher BYTEA NOT NULL,
        protocols TEXT[],
        input_token BYTEA NOT NULL,
        output_token BYTEA NOT NULL,
        input_amount NUMERIC NOT NULL,
        output_amount NUMERIC NOT NULL,
        profit_usd NUMERIC,
        gas_cost_usd NUMERIC,
        net_profit_usd NUMERIC,
        timestamp TIMESTAMP DEFAULT NOW()
    );

    CREATE INDEX IF NOT EXISTS idx_arb_searcher ON arbitrage_trades(searcher);
    CREATE INDEX IF NOT EXISTS idx_arb_profit ON arbitrage_trades(net_profit_usd);
    `

_, err := d.db.Exec(schema)
return err
}

// StoreSwap 存储交换记录
func (d *IndexerDB) StoreSwap(swap *IndexedSwap) error {
_, err := d.db.Exec(`
        INSERT INTO swaps (
            block_number, tx_hash, log_index, protocol, pool_address,
            sender, recipient, token0, token1,
            amount0_in, amount1_in, amount0_out, amount1_out, price
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14)
        ON CONFLICT (tx_hash, log_index) DO NOTHING
    `,
swap.BlockNumber,
swap.TxHash.Bytes(),
swap.LogIndex,
swap.Protocol,
swap.PoolAddress.Bytes(),
swap.Sender.Bytes(),
swap.Recipient.Bytes(),
swap.Token0.Bytes(),
swap.Token1.Bytes(),
swap.Amount0In.String(),
swap.Amount1In.String(),
swap.Amount0Out.String(),
swap.Amount1Out.String(),
swap.Price.String(),
)
return err
}

// GetRecentSwaps 获取最近交换
func (d *IndexerDB) GetRecentSwaps(ctx context.Context, limit int) ([]IndexedSwap, error) {
rows, err := d.db.QueryContext(ctx, `
        SELECT
            id, block_number, tx_hash, log_index, protocol, pool_address,
            sender, recipient, token0, token1,
            amount0_in, amount1_in, amount0_out, amount1_out, price, timestamp
        FROM swaps
        ORDER BY block_number DESC, log_index DESC
        LIMIT $1
    `, limit)
if err != nil {
return nil, err
}
defer rows.Close()

var swaps []IndexedSwap
for rows.Next() {
var swap IndexedSwap
var txHash, poolAddr, sender, recipient, token0, token1 []byte
var amount0In, amount1In, amount0Out, amount1Out, price string

err := rows.Scan(
&swap.ID, &swap.BlockNumber, &txHash, &swap.LogIndex, &swap.Protocol, &poolAddr,
&sender, &recipient, &token0, &token1,
&amount0In, &amount1In, &amount0Out, &amount1Out, &price, &swap.Timestamp,
)
if err != nil {
continue
}

swap.TxHash = common.BytesToHash(txHash)
swap.PoolAddress = common.BytesToAddress(poolAddr)
swap.Sender = common.BytesToAddress(sender)
swap.Recipient = common.BytesToAddress(recipient)
swap.Token0 = common.BytesToAddress(token0)
swap.Token1 = common.BytesToAddress(token1)
swap.Amount0In, _ = new(big.Int).SetString(amount0In, 10)
swap.Amount1In, _ = new(big.Int).SetString(amount1In, 10)
swap.Amount0Out, _ = new(big.Int).SetString(amount0Out, 10)
swap.Amount1Out, _ = new(big.Int).SetString(amount1Out, 10)
swap.Price, _ = new(big.Float).SetString(price)

swaps = append(swaps, swap)
}

return swaps, nil
}

// GetArbitrageTrades 获取套利交易
func (d *IndexerDB) GetArbitrageTrades(ctx context.Context, limit int) ([]ArbitrageTransaction, error) {
rows, err := d.db.QueryContext(ctx, `
        SELECT
            block_number, tx_hash, searcher, protocols,
            input_token, output_token, input_amount, output_amount,
            profit_usd, gas_cost_usd, net_profit_usd, timestamp
        FROM arbitrage_trades
        ORDER BY block_number DESC
        LIMIT $1
    `, limit)
if err != nil {
return nil, err
}
defer rows.Close()

var trades []ArbitrageTransaction
for rows.Next() {
var trade ArbitrageTransaction
var txHash, searcher, inputToken, outputToken []byte
var protocols pq.StringArray

err := rows.Scan(
&trade.BlockNumber, &txHash, &searcher, &protocols,
&inputToken, &outputToken, &trade.InputAmount, &trade.OutputAmount,
&trade.ProfitUSD, &trade.GasCostUSD, &trade.NetProfitUSD, &trade.Timestamp,
)
if err != nil {
continue
}

trade.TxHash = common.BytesToHash(txHash).Hex()
trade.Searcher = common.BytesToAddress(searcher).Hex()
trade.Protocols = []string(protocols)
trade.InputToken = common.BytesToAddress(inputToken).Hex()
trade.OutputToken = common.BytesToAddress(outputToken).Hex()

trades = append(trades, trade)
}

return trades, nil
}
```

## 6. 完整使用示例

```go
func main() {
ctx := context.Background()

// 配置
config := &Config{
RPCURL:    "https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY",
WSURL:     "wss://eth-mainnet.g.alchemy.com/v2/YOUR_KEY",
GraphAPI:  "https://api.thegraph.com/subgraphs/name/uniswap/uniswap-v3",
DuneAPI:   "YOUR_DUNE_API_KEY",
DBConnStr: "postgres://user:pass@localhost/indexer",
}

// 初始化数据库
db, err := NewIndexerDB(config.DBConnStr)
if err != nil {
log.Fatal(err)
}

// 初始化 Subgraph 查询
subgraph := NewSubgraphAggregator(&SubgraphConfig{
UniswapV3Endpoint: config.GraphAPI,
APIKey:            "",
})

// 初始化 Dune 客户端
dune := NewDuneClient(config.DuneAPI)

// 初始化事件索引器
indexer, err := NewSwapIndexer(config.RPCURL, config.WSURL, db)
if err != nil {
log.Fatal(err)
}

// 初始化分析器
analyzer := NewArbitrageAnalyzer(db, subgraph, dune)

// 启动索引
go func () {
// 从最近 1000 个区块开始索引
currentBlock, _ := indexer.client.BlockNumber(ctx)
indexer.Start(ctx, currentBlock-1000)
}()

// 启动监控面板
dashboard := NewDashboardServer(analyzer, indexer)
log.Println("Dashboard starting on :8080")
log.Fatal(dashboard.Start(":8080"))
}
```

## 7. 总结

链上数据索引是套利机器人的重要组成部分：

1. **The Graph**: 提供标准化的 Subgraph 查询，适合获取历史数据和复杂聚合
2. **Dune Analytics**: 适合复杂的数据分析和统计查询
3. **实时索引**: 通过事件订阅实时追踪链上活动
4. **数据分析**: 通过历史数据分析发现套利模式和机会

关键实践：

- 使用缓存减少重复查询
- 并行查询多个数据源
- 建立完善的索引加速查询
- 实时监控与历史分析结合
