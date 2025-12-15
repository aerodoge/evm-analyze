# 消息中间件在交易系统中的深度应用指南

## 目录

1. [消息中间件概述](#1-消息中间件概述)
2. [消息中间件在交易系统中的核心作用](#2-消息中间件在交易系统中的核心作用)
3. [主流消息中间件内部原理](#3-主流消息中间件内部原理)
4. [常见交易系统架构](#4-常见交易系统架构)
5. [中间件在交易系统中的应用场景](#5-中间件在交易系统中的应用场景)
6. [高性能交易系统中间件实践](#6-高性能交易系统中间件实践)
7. [最佳实践与注意事项](#7-最佳实践与注意事项)

---

## 1. 消息中间件概述

### 1.1 什么是消息中间件

消息中间件（Message-Oriented Middleware, MOM）是一种支持在分布式系统中发送和接收消息的基础设施软件。它在消息的发送者和接收者之间起到解耦、缓冲和异步通信的作用。

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Producer  │────▶│  Message Broker  │────▶│   Consumer  │
│  (生产者)    │     │   (消息代理)      │     │   (消费者)   │
└─────────────┘     └──────────────────┘     └─────────────┘
                           │
                    ┌──────┴──────┐
                    │             │
               ┌────▼────┐  ┌────▼────┐
               │  Queue  │  │  Topic  │
               │  (队列)  │  │  (主题) │
               └─────────┘  └─────────┘
```

### 1.2 消息中间件的核心特性

| 特性      | 描述            | 在交易系统中的意义     |
|---------|---------------|---------------|
| **解耦**  | 生产者和消费者无需直接通信 | 订单系统与风控系统独立演进 |
| **异步**  | 消息发送后无需等待处理完成 | 提高订单提交响应速度    |
| **削峰**  | 缓冲突发流量        | 应对开盘/收盘交易高峰   |
| **可靠性** | 消息持久化与确认机制    | 确保交易指令不丢失     |
| **顺序性** | 保证消息按序处理      | 保证同一账户订单顺序执行  |

### 1.3 消息传递模式

#### 1.3.1 点对点模式 (Point-to-Point)

```
                    ┌─────────────────────────────────┐
                    │           Queue                 │
┌──────────┐        │  ┌───┬───┬───┬───┬───┬───┐    │        ┌──────────┐
│ Producer │───────▶│  │ M │ M │ M │ M │ M │ M │    │───────▶│ Consumer │
└──────────┘        │  │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │    │        └──────────┘
                    │  └───┴───┴───┴───┴───┴───┘    │
                    └─────────────────────────────────┘

特点：一条消息只能被一个消费者消费
应用：订单处理、任务分发
```

#### 1.3.2 发布/订阅模式 (Publish/Subscribe)

```
                                    ┌──────────────┐
                              ┌────▶│ Subscriber 1 │  (行情展示)
                              │     └──────────────┘
┌───────────┐    ┌────────┐   │     ┌──────────────┐
│ Publisher │───▶│ Topic  │───┼────▶│ Subscriber 2 │  (风控监控)
└───────────┘    └────────┘   │     └──────────────┘
  (行情源)        (行情主题)    │     ┌──────────────┐
                              └────▶│ Subscriber 3 │  (策略引擎)
                                    └──────────────┘

特点：一条消息可以被多个消费者消费
应用：行情分发、事件广播
```

---

## 2. 消息中间件在交易系统中的核心作用

### 2.1 交易系统为什么需要消息中间件

交易系统具有以下特点，使得消息中间件成为必需组件：

```
交易系统特点                          消息中间件解决方案
─────────────────────────────────────────────────────────────────
高并发        ──────────────────────▶  消息缓冲与削峰填谷
低延迟要求    ──────────────────────▶  异步处理与批量优化
高可靠性     ──────────────────────▶  消息持久化与重试机制
系统解耦     ──────────────────────▶  生产消费分离
实时性       ──────────────────────▶  推模式消息分发
可扩展性     ──────────────────────▶  水平扩展与分区
```

### 2.2 核心作用详解

#### 2.2.1 订单流转与处理

```
┌─────────────────────────────────────────────────────────────────────┐
│                        订单处理流程                                  │
└─────────────────────────────────────────────────────────────────────┘

  用户下单          订单队列           订单处理           成交通知
     │                │                  │                  │
     ▼                ▼                  ▼                  ▼
┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   API   │───▶│ Order Queue │───▶│Order Engine │───▶│Notification │
│ Gateway │    │  订单队列    │    │  撮合引擎    │    │   Queue     │
└─────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                     │                   │                  │
                     │              ┌────┴────┐             │
                     │              ▼         ▼             ▼
                     │         ┌───────┐ ┌───────┐    ┌──────────┐
                     │         │ 成交  │ │ 拒绝  │    │ 推送服务  │
                     │         └───────┘ └───────┘    └──────────┘
                     │
              ┌──────┴──────┐
              ▼             ▼
        ┌──────────┐  ┌──────────┐
        │ 风控检查  │  │ 日志记录  │
        └──────────┘  └──────────┘
```

**Go 代码示例 - 订单消息结构**：

```go
package order

import (
	"encoding/json"
	"time"
)

// OrderMessage 订单消息结构
type OrderMessage struct {
	OrderID     string            `json:"order_id"`
	UserID      string            `json:"user_id"`
	Symbol      string            `json:"symbol"`        // 交易对 BTC/USDT
	Side        OrderSide         `json:"side"`          // BUY/SELL
	Type        OrderType         `json:"type"`          // LIMIT/MARKET
	Price       string            `json:"price"`         // 价格（使用字符串避免精度问题）
	Quantity    string            `json:"quantity"`      // 数量
	TimeInForce TimeInForce       `json:"time_in_force"` // GTC/IOC/FOK
	Timestamp   int64             `json:"timestamp"`     // 下单时间戳（纳秒）
	Priority    int               `json:"priority"`      // 优先级
	Metadata    map[string]string `json:"metadata"`      // 扩展字段
}

type OrderSide string

const (
	OrderSideBuy  OrderSide = "BUY"
	OrderSideSell OrderSide = "SELL"
)

type OrderType string

const (
	OrderTypeLimit  OrderType = "LIMIT"
	OrderTypeMarket OrderType = "MARKET"
)

type TimeInForce string

const (
	TimeInForceGTC TimeInForce = "GTC" // Good Till Cancel
	TimeInForceIOC TimeInForce = "IOC" // Immediate Or Cancel
	TimeInForceFOK TimeInForce = "FOK" // Fill Or Kill
)

// Serialize 序列化订单消息
func (o *OrderMessage) Serialize() ([]byte, error) {
	return json.Marshal(o)
}

// GetPartitionKey 获取分区键（保证同一用户订单顺序）
func (o *OrderMessage) GetPartitionKey() string {
	return o.UserID + ":" + o.Symbol
}
```

#### 2.2.2 行情数据分发

```
┌─────────────────────────────────────────────────────────────────────┐
│                        行情分发架构                                  │
└─────────────────────────────────────────────────────────────────────┘

     交易所/链上数据                    消息中间件                    消费端
┌──────────────────┐           ┌────────────────────┐         ┌─────────────┐
│   Binance API    │──┐        │                    │    ┌───▶│  Web终端     │
├──────────────────┤  │        │   Topic: ticker    │────┤    └─────────────┘
│    OKX API       │──┼───────▶│   ┌─────────────┐  │    │    ┌─────────────┐
├──────────────────┤  │        │   │ Partition 0 │  │    ├───▶│  移动App     │
│  Blockchain RPC  │──┤        │   │ Partition 1 │  │    │    └─────────────┘
├──────────────────┤  │        │   │ Partition 2 │  │    │    ┌─────────────┐
│   DEX Events     │──┘        │   └─────────────┘  │    └───▶│  策略引擎    │
└──────────────────┘           │                    │         └─────────────┘
                               │   Topic: depth     │         ┌─────────────┐
    行情聚合服务                 │   ┌─────────────┐  │    ┌───▶│  做市系统    │
┌──────────────────┐           │   │ Partition 0 │  │────┤    └─────────────┘
│  行情清洗/聚合     │──────────▶│   │ Partition 1 │  │    │    ┌─────────────┐
│  价格计算/校验     │           │   └─────────────┘  │    └───▶│  风控系统    │
└──────────────────┘           └────────────────────┘         └─────────────┘
```

**Go 代码示例 - 行情消息处理**：

```go
package market

import (
	"context"
	"sync"
	"time"
)

// TickerMessage 行情Tick消息
type TickerMessage struct {
	Symbol      string `json:"symbol"`
	Exchange    string `json:"exchange"`
	BidPrice    string `json:"bid_price"`    // 买一价
	BidQty      string `json:"bid_qty"`      // 买一量
	AskPrice    string `json:"ask_price"`    // 卖一价
	AskQty      string `json:"ask_qty"`      // 卖一量
	LastPrice   string `json:"last_price"`   // 最新成交价
	Volume24h   string `json:"volume_24h"`   // 24小时成交量
	Timestamp   int64  `json:"timestamp"`    // 时间戳（微秒）
	SequenceNum uint64 `json:"sequence_num"` // 序列号
}

// DepthMessage 深度数据消息
type DepthMessage struct {
	Symbol     string     `json:"symbol"`
	Exchange   string     `json:"exchange"`
	Bids       [][]string `json:"bids"` // [[price, qty], ...]
	Asks       [][]string `json:"asks"`
	Timestamp  int64      `json:"timestamp"`
	UpdateID   uint64     `json:"update_id"`
	IsSnapshot bool       `json:"is_snapshot"` // 是否是全量快照
}

// MarketDataPublisher 行情数据发布器
type MarketDataPublisher struct {
	producer    MessageProducer
	bufferSize  int
	buffer      chan *TickerMessage
	batchSize   int
	flushPeriod time.Duration
	wg          sync.WaitGroup
}

// MessageProducer 消息生产者接口
type MessageProducer interface {
	Send(ctx context.Context, topic string, key string, value []byte) error
	SendBatch(ctx context.Context, topic string, messages []Message) error
	Close() error
}

type Message struct {
	Key   string
	Value []byte
}

// NewMarketDataPublisher 创建行情发布器
func NewMarketDataPublisher(producer MessageProducer, opts ...Option) *MarketDataPublisher {
	p := &MarketDataPublisher{
		producer:    producer,
		bufferSize:  10000,
		batchSize:   100,
		flushPeriod: 10 * time.Millisecond,
	}
	for _, opt := range opts {
		opt(p)
	}
	p.buffer = make(chan *TickerMessage, p.bufferSize)
	return p
}

type Option func(*MarketDataPublisher)

func WithBufferSize(size int) Option {
	return func(p *MarketDataPublisher) {
		p.bufferSize = size
	}
}

func WithBatchSize(size int) Option {
	return func(p *MarketDataPublisher) {
		p.batchSize = size
	}
}

// Start 启动批量发送协程
func (p *MarketDataPublisher) Start(ctx context.Context) {
	p.wg.Add(1)
	go func() {
		defer p.wg.Done()

		batch := make([]*TickerMessage, 0, p.batchSize)
		ticker := time.NewTicker(p.flushPeriod)
		defer ticker.Stop()

		for {
			select {
			case <-ctx.Done():
				// 发送剩余消息
				if len(batch) > 0 {
					p.flushBatch(context.Background(), batch)
				}
				return

			case msg := <-p.buffer:
				batch = append(batch, msg)
				if len(batch) >= p.batchSize {
					p.flushBatch(ctx, batch)
					batch = batch[:0]
				}

			case <-ticker.C:
				if len(batch) > 0 {
					p.flushBatch(ctx, batch)
					batch = batch[:0]
				}
			}
		}
	}()
}

// Publish 发布行情消息（非阻塞）
func (p *MarketDataPublisher) Publish(msg *TickerMessage) bool {
	select {
	case p.buffer <- msg:
		return true
	default:
		// 缓冲区满，丢弃消息（行情数据可容忍丢失）
		return false
	}
}

func (p *MarketDataPublisher) flushBatch(ctx context.Context, batch []*TickerMessage) {
	messages := make([]Message, len(batch))
	for i, msg := range batch {
		data, _ := msg.Serialize()
		messages[i] = Message{
			Key:   msg.Symbol,
			Value: data,
		}
	}
	_ = p.producer.SendBatch(ctx, "market.ticker", messages)
}

func (t *TickerMessage) Serialize() ([]byte, error) {
	// 使用高性能序列化库如 sonic 或 msgpack
	return json.Marshal(t)
}
```

#### 2.2.3 风控事件处理

```
┌─────────────────────────────────────────────────────────────────────┐
│                        风控消息流                                     │
└─────────────────────────────────────────────────────────────────────┘

  交易事件                   风控队列                     风控处理
     │                         │                          │
     ▼                         ▼                          ▼
┌─────────────┐         ┌─────────────┐           ┌─────────────────┐
│  订单创建    │────────▶│             │           │  实时风控检查     │
├─────────────┤         │   Risk      │           ├─────────────────┤
│  订单成交    │────────▶│   Event     │──────────▶│ • 仓位检查        │
├─────────────┤         │   Queue     │           │ • 资金检查       │
│  价格变动    │────────▶│             │           │ • 频率检查       │
├─────────────┤         │   Priority  │           │ • 黑名单检查     │
│  仓位变化    │────────▶│   Queue     │           └─────────────────┘
└─────────────┘         └─────────────┘                   │
                               │                          │
                        ┌──────┴──────┐            ┌──────┴──────┐
                        ▼             ▼            ▼             ▼
                   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
                   │ 普通事件 │  │ 紧急事件 │   │  通过   │   │  拦截   │
                   └─────────┘  └─────────┘  └─────────┘  └─────────┘
```

**Go 代码示例 - 风控事件消息**：

```go
package risk

import (
	"context"
	"time"
)

// RiskEventType 风控事件类型
type RiskEventType string

const (
	RiskEventOrderCreated    RiskEventType = "ORDER_CREATED"
	RiskEventOrderFilled     RiskEventType = "ORDER_FILLED"
	RiskEventPositionChanged RiskEventType = "POSITION_CHANGED"
	RiskEventPriceAlert      RiskEventType = "PRICE_ALERT"
	RiskEventLiquidation     RiskEventType = "LIQUIDATION"
)

// RiskLevel 风险等级
type RiskLevel int

const (
	RiskLevelLow      RiskLevel = 1
	RiskLevelMedium   RiskLevel = 2
	RiskLevelHigh     RiskLevel = 3
	RiskLevelCritical RiskLevel = 4
)

// RiskEvent 风控事件
type RiskEvent struct {
	EventID     string                 `json:"event_id"`
	EventType   RiskEventType          `json:"event_type"`
	UserID      string                 `json:"user_id"`
	AccountID   string                 `json:"account_id"`
	Symbol      string                 `json:"symbol"`
	RiskLevel   RiskLevel              `json:"risk_level"`
	Payload     map[string]interface{} `json:"payload"`
	Timestamp   int64                  `json:"timestamp"`
	RequireSync bool                   `json:"require_sync"` // 是否需要同步处理
}

// RiskEventProcessor 风控事件处理器
type RiskEventProcessor struct {
	consumer    MessageConsumer
	handlers    map[RiskEventType]RiskHandler
	workerCount int
}

type MessageConsumer interface {
	Subscribe(ctx context.Context, topics []string) error
	Poll(ctx context.Context, timeout time.Duration) (*ConsumerMessage, error)
	Commit(ctx context.Context, msg *ConsumerMessage) error
}

type ConsumerMessage struct {
	Topic     string
	Partition int
	Offset    int64
	Key       []byte
	Value     []byte
	Timestamp time.Time
}

type RiskHandler func(ctx context.Context, event *RiskEvent) (*RiskResult, error)

type RiskResult struct {
	Passed    bool         `json:"passed"`
	RiskScore float64      `json:"risk_score"`
	Actions   []RiskAction `json:"actions"`
	Reason    string       `json:"reason"`
}

type RiskAction string

const (
	RiskActionNone      RiskAction = "NONE"
	RiskActionWarn      RiskAction = "WARN"
	RiskActionReject    RiskAction = "REJECT"
	RiskActionFreeze    RiskAction = "FREEZE"
	RiskActionLiquidate RiskAction = "LIQUIDATE"
)

// NewRiskEventProcessor 创建风控事件处理器
func NewRiskEventProcessor(consumer MessageConsumer, workerCount int) *RiskEventProcessor {
	return &RiskEventProcessor{
		consumer:    consumer,
		handlers:    make(map[RiskEventType]RiskHandler),
		workerCount: workerCount,
	}
}

// RegisterHandler 注册事件处理器
func (p *RiskEventProcessor) RegisterHandler(eventType RiskEventType, handler RiskHandler) {
	p.handlers[eventType] = handler
}

// Start 启动处理器
func (p *RiskEventProcessor) Start(ctx context.Context) error {
	if err := p.consumer.Subscribe(ctx, []string{
		"risk.events.normal",
		"risk.events.priority",
	}); err != nil {
		return err
	}

	// 启动多个 worker
	for i := 0; i < p.workerCount; i++ {
		go p.worker(ctx, i)
	}

	return nil
}

func (p *RiskEventProcessor) worker(ctx context.Context, workerID int) {
	for {
		select {
		case <-ctx.Done():
			return
		default:
		}

		msg, err := p.consumer.Poll(ctx, 100*time.Millisecond)
		if err != nil || msg == nil {
			continue
		}

		var event RiskEvent
		if err := json.Unmarshal(msg.Value, &event); err != nil {
			// 记录错误，跳过此消息
			continue
		}

		handler, ok := p.handlers[event.EventType]
		if !ok {
			// 未知事件类型
			_ = p.consumer.Commit(ctx, msg)
			continue
		}

		// 根据风险等级设置超时
		timeout := p.getTimeout(event.RiskLevel)
		processCtx, cancel := context.WithTimeout(ctx, timeout)

		result, err := handler(processCtx, &event)
		cancel()

		if err != nil {
			// 处理失败，根据策略决定是否重试
			continue
		}

		// 处理风控结果
		p.handleResult(&event, result)

		// 提交偏移量
		_ = p.consumer.Commit(ctx, msg)
	}
}

func (p *RiskEventProcessor) getTimeout(level RiskLevel) time.Duration {
	switch level {
	case RiskLevelCritical:
		return 10 * time.Millisecond
	case RiskLevelHigh:
		return 50 * time.Millisecond
	case RiskLevelMedium:
		return 100 * time.Millisecond
	default:
		return 500 * time.Millisecond
	}
}

func (p *RiskEventProcessor) handleResult(event *RiskEvent, result *RiskResult) {
	// 发送风控结果到相应的处理模块
	// ...
}
```

#### 2.2.4 交易日志与审计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        交易审计日志流                                │
└─────────────────────────────────────────────────────────────────────┘

    业务系统                    日志队列                    存储/分析
       │                          │                          │
       ▼                          ▼                          ▼
┌─────────────┐            ┌─────────────┐            ┌─────────────┐
│  订单系统   │──────┐     │             │     ┌─────▶│ Elasticsearch│
├─────────────┤      │     │             │     │      └─────────────┘
│  撮合系统   │──────┼────▶│  Audit Log  │─────┤      ┌─────────────┐
├─────────────┤      │     │   Topic     │     ├─────▶│  ClickHouse │
│  清结算系统  │──────┤     │             │     │      └─────────────┘
├─────────────┤      │     │ (高吞吐量)   │     │      ┌─────────────┐
│  风控系统   │──────┘     │             │     └─────▶│  HDFS/S3    │
└─────────────┘            └─────────────┘            └─────────────┘
                                                           │
                                                    ┌──────┴──────┐
                                                    ▼             ▼
                                              ┌─────────┐   ┌─────────┐
                                              │实时监控  │   │离线分析  │
                                              └─────────┘   └─────────┘
```

### 2.3 消息中间件选型对比

| 特性       | Kafka     | RabbitMQ | RocketMQ | Pulsar | ZeroMQ |
|----------|-----------|----------|----------|--------|--------|
| **吞吐量**  | 极高 (百万/秒) | 中等 (万/秒) | 高 (十万/秒) | 极高     | 极高     |
| **延迟**   | 毫秒级       | 微秒-毫秒    | 毫秒级      | 毫秒级    | 微秒级    |
| **持久化**  | 磁盘        | 内存+磁盘    | 磁盘       | 分层存储   | 无      |
| **消息顺序** | 分区内有序     | 队列内有序    | 队列内有序    | 分区内有序  | 无保证    |
| **事务支持** | 支持        | 支持       | 支持       | 支持     | 不支持    |
| **协议**   | 自有协议      | AMQP     | 自有协议     | 自有协议   | 自有协议   |
| **适用场景** | 日志/行情     | 订单/通知    | 交易/金融    | 多租户    | 内部通信   |

**交易系统选型建议**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        中间件选型决策树                                │
└─────────────────────────────────────────────────────────────────────┘

                        需要什么类型的消息传递？
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
         高吞吐量          低延迟需求        复杂路由
              │                │                │
              ▼                ▼                ▼
    ┌─────────────────┐ ┌─────────────┐ ┌─────────────┐
    │ • 行情数据分发    │ │ • 订单处理   │ │ • 通知系统    │
    │ • 日志收集       │ │ • 内部通信    │ │ • 事件路由   │
    │ • 监控数据       │ │ • 实时计算    │ │ • 工作流     │
    └────────┬────────┘ └──────┬──────┘ └──────┬──────┘
             │                 │               │
             ▼                 ▼               ▼
    ┌─────────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Kafka/Pulsar   │ │   ZeroMQ/   │ │  RabbitMQ   │
    │                 │ │  Aeron/LMAX │ │             │
    └─────────────────┘ └─────────────┘ └─────────────┘
```

---

## 3. 主流消息中间件内部原理

### 3.1 Apache Kafka 深度剖析

Kafka 是目前交易系统中使用最广泛的消息中间件之一，特别适合高吞吐量的行情数据分发和日志收集场景。

#### 3.1.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Kafka 集群架构                                     │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────┐
                              │   ZooKeeper /   │
                              │   KRaft 集群     │
                              │  (元数据管理)    │
                              └────────┬────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                           │
           ▼                           ▼                           ▼
    ┌──────────────┐             ┌─────────────┐             ┌─────────────┐
    │  Broker 1    │             │  Broker 2   │             │  Broker 3   │
    │  (Controller)│◀──────────▶ │             │◀──────────▶ │             │
    ├──────────────┤             ├─────────────┤             ├─────────────┤
    │ Topic A-P0   │             │ Topic A-P1  │             │ Topic A-P2  │
    │ (Leader)     │             │ (Leader)    │             │ (Leader)    │
    │ Topic A-P1   │             │ Topic A-P2  │             │ Topic A-P0  │
    │ (Follower)   │             │ (Follower)  │             │ (Follower)  │
    └──────────────┘             └─────────────┘             └─────────────┘
           ▲                           ▲                           ▲
           │                           │                           │
           └───────────────────────────┼───────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
             ┌──────┴──────┐    ┌──────┴──────┐    ┌──────┴──────┐
             │  Producer   │    │  Producer   │    │  Consumer   │
             │   Group     │    │   Group     │    │   Group     │
             └─────────────┘    └─────────────┘    └─────────────┘
```

#### 3.1.2 分区与副本机制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Topic 分区与副本分布                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Topic: market.ticker (3个分区, 副本因子=3)

Broker 1                    Broker 2                    Broker 3
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│                 │         │                 │         │                 │
│  ┌───────────┐  │         │  ┌───────────┐  │         │  ┌───────────┐  │
│  │Partition 0│  │         │  │Partition 0│  │         │  │Partition 0│  │
│  │  Leader   │◀─┼─────────┼─▶│  Follower │◀─┼─────────┼─▶│  Follower │  │
│  └───────────┘  │   ISR   │  └───────────┘  │   ISR   │  └───────────┘  │
│                 │  同步    │                 │  同步    │                 │
│  ┌───────────┐  │         │  ┌───────────┐  │         │  ┌───────────┐  │
│  │Partition 1│  │         │  │Partition 1│  │         │  │Partition 1│  │
│  │  Follower │◀─┼─────────┼─▶│  Leader   │◀─┼─────────┼─▶│  Follower │  │
│  └───────────┘  │         │  └───────────┘  │         │  └───────────┘  │
│                 │         │                 │         │                 │
│  ┌───────────┐  │         │  ┌───────────┐  │         │  ┌───────────┐  │
│  │Partition 2│  │         │  │Partition 2│  │         │  │Partition 2│  │
│  │  Follower │◀─┼─────────┼─▶│  Follower │◀─┼─────────┼─▶│  Leader   │  │
│  └───────────┘  │         │  └───────────┘  │         │  └───────────┘  │
│                 │         │                 │         │                 │
└─────────────────┘         └─────────────────┘         └─────────────────┘

ISR (In-Sync Replicas): 与 Leader 保持同步的副本集合
```

#### 3.1.3 日志存储结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Kafka 日志存储结构                                    │
└─────────────────────────────────────────────────────────────────────────────┘

/kafka-logs/
└── market.ticker-0/                    # Topic-Partition 目录
    ├── 00000000000000000000.log        # 日志段文件 (存储消息)
    ├── 00000000000000000000.index      # 偏移量索引
    ├── 00000000000000000000.timeindex  # 时间戳索引
    ├── 00000000000012345678.log        # 新的日志段
    ├── 00000000000012345678.index
    ├── 00000000000012345678.timeindex
    └── leader-epoch-checkpoint         # Leader epoch 信息

日志段文件内部结构:
┌───────────────────────────────────────────────────────────────────┐
│                         .log 文件                                  │
├───────────────────────────────────────────────────────────────────┤
│  Record Batch 1  │  Record Batch 2  │  Record Batch 3  │  ...     │
├──────────────────┼──────────────────┼──────────────────┼──────────┤
│                  │                  │                  │          │
│  ┌────────────┐  │  ┌────────────┐  │  ┌────────────┐  │          │
│  │Base Offset │  │  │Base Offset │  │  │Base Offset │  │          │
│  │Length      │  │  │Length      │  │  │Length      │  │          │
│  │CRC         │  │  │CRC         │  │  │CRC         │  │          │
│  │Magic       │  │  │Magic       │  │  │Magic       │  │          │
│  │Attributes  │  │  │Attributes  │  │  │Attributes  │  │          │
│  │Records[]   │  │  │Records[]   │  │  │Records[]   │  │          │
│  └────────────┘  │  └────────────┘  │  └────────────┘  │          │
└──────────────────┴──────────────────┴──────────────────┴──────────┘

索引文件结构 (.index):
┌────────────────────────────────────────────────┐
│  Relative Offset (4B) │ Physical Position (4B) │
├───────────────────────┼────────────────────────┤
│         0             │          0             │
│        100            │        4096            │
│        200            │        8192            │
│        ...            │         ...            │
└───────────────────────┴────────────────────────┘
```

#### 3.1.4 生产者写入流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Producer 消息发送流程                                  │
└─────────────────────────────────────────────────────────────────────────────┘

  应用层                      Producer 内部                         Broker
    │                            │                                   │
    │  send(record)              │                                   │
    ├───────────────────────────▶│                                   │
    │                            │                                   │
    │                    ┌───────┴───────┐                           │
    │                    │  Serializer   │ 序列化 Key/Value           │
    │                    └───────┬───────┘                           │
    │                            │                                   │
    │                    ┌───────┴───────┐                           │
    │                    │  Partitioner  │ 计算目标分区                │
    │                    │  hash(key) %  │                           │
    │                    │  partitions   │                           │
    │                    └───────┬───────┘                           │
    │                            │                                   │
    │                    ┌───────┴───────┐                           │
    │                    │Record         │                           │
    │                    │Accumulator    │ 消息累积器(按分区缓冲)       │
    │                    │               │                           │
    │                    │ ┌───────────┐ │                           │
    │                    │ │ Partition │ │                           │
    │                    │ │  Buffer 0 │ │                           │
    │                    │ ├───────────┤ │                           │
    │                    │ │ Partition │ │                           │
    │                    │ │  Buffer 1 │ │                           │
    │                    │ └───────────┘ │                           │
    │                    └───────┬───────┘                           │
    │                            │                                   │
    │                    ┌───────┴───────┐     批量发送               │
    │                    │    Sender     │─────────────────────────▶ │
    │                    │   Thread      │     ProduceRequest        │
    │                    └───────┬───────┘                           │
    │                            │                                   │
    │                            │◀────────────────────────────────  │
    │  Future<Metadata>          │     ProduceResponse               │
    │◀───────────────────────────│                                   │
    │                            │                                   │

批量发送条件 (满足任一即触发):
1. batch.size: 批次大小达到阈值 (默认 16KB)
2. linger.ms: 等待时间达到阈值 (默认 0ms)
3. buffer.memory: 缓冲区满
```

**Go 代码示例 - 高性能 Kafka Producer**：

```go
package kafka

import (
	"context"
	"crypto/tls"
	"time"

	"github.com/segmentio/kafka-go"
	"github.com/segmentio/kafka-go/compress"
)

// HighPerformanceProducer 高性能 Kafka 生产者
type HighPerformanceProducer struct {
	writer *kafka.Writer
	config *ProducerConfig
}

// ProducerConfig 生产者配置
type ProducerConfig struct {
	Brokers      []string
	Topic        string
	BatchSize    int           // 批次大小
	BatchBytes   int64         // 批次字节数
	BatchTimeout time.Duration // 批次超时
	MaxAttempts  int           // 最大重试次数
	RequiredAcks kafka.RequiredAcks
	Compression  compress.Codec
	Async        bool // 是否异步发送
	TLS          *tls.Config
}

// DefaultProducerConfig 默认配置（适合行情数据）
func DefaultProducerConfig(brokers []string, topic string) *ProducerConfig {
	return &ProducerConfig{
		Brokers:      brokers,
		Topic:        topic,
		BatchSize:    1000,                  // 1000条消息一批
		BatchBytes:   1 << 20,               // 1MB
		BatchTimeout: 10 * time.Millisecond, // 10ms 最大等待
		MaxAttempts:  3,
		RequiredAcks: kafka.RequireOne, // 只需 Leader 确认
		Compression:  compress.Lz4,     // LZ4 压缩
		Async:        true,
	}
}

// HighReliabilityConfig 高可靠配置（适合订单数据）
func HighReliabilityConfig(brokers []string, topic string) *ProducerConfig {
	return &ProducerConfig{
		Brokers:      brokers,
		Topic:        topic,
		BatchSize:    100,
		BatchBytes:   256 << 10, // 256KB
		BatchTimeout: 5 * time.Millisecond,
		MaxAttempts:  10,
		RequiredAcks: kafka.RequireAll, // 所有 ISR 确认
		Compression:  compress.Snappy,
		Async:        false, // 同步发送
	}
}

// NewHighPerformanceProducer 创建高性能生产者
func NewHighPerformanceProducer(config *ProducerConfig) *HighPerformanceProducer {
	transport := &kafka.Transport{
		DialTimeout: 3 * time.Second,
		IdleTimeout: 60 * time.Second,
	}
	if config.TLS != nil {
		transport.TLS = config.TLS
	}

	writer := &kafka.Writer{
		Addr:         kafka.TCP(config.Brokers...),
		Topic:        config.Topic,
		Balancer:     &kafka.Hash{}, // 按 Key Hash 分区
		BatchSize:    config.BatchSize,
		BatchBytes:   config.BatchBytes,
		BatchTimeout: config.BatchTimeout,
		MaxAttempts:  config.MaxAttempts,
		RequiredAcks: config.RequiredAcks,
		Compression:  config.Compression,
		Async:        config.Async,
		Transport:    transport,
	}

	return &HighPerformanceProducer{
		writer: writer,
		config: config,
	}
}

// SendMessages 批量发送消息
func (p *HighPerformanceProducer) SendMessages(ctx context.Context, messages []kafka.Message) error {
	return p.writer.WriteMessages(ctx, messages...)
}

// SendWithKey 发送带 Key 的消息（保证同 Key 消息顺序）
func (p *HighPerformanceProducer) SendWithKey(ctx context.Context, key, value []byte) error {
	return p.writer.WriteMessages(ctx, kafka.Message{
		Key:   key,
		Value: value,
		Time:  time.Now(),
	})
}

// Stats 获取生产者统计信息
func (p *HighPerformanceProducer) Stats() kafka.WriterStats {
	return p.writer.Stats()
}

// Close 关闭生产者
func (p *HighPerformanceProducer) Close() error {
	return p.writer.Close()
}

// 使用示例
func ExampleProducer() {
	config := DefaultProducerConfig(
		[]string{"kafka1:9092", "kafka2:9092", "kafka3:9092"},
		"market.ticker",
	)

	producer := NewHighPerformanceProducer(config)
	defer producer.Close()

	ctx := context.Background()

	// 批量发送行情数据
	messages := make([]kafka.Message, 0, 100)
	for i := 0; i < 100; i++ {
		messages = append(messages, kafka.Message{
			Key:   []byte("BTC-USDT"),
			Value: []byte(`{"price":"50000.00","volume":"1.5"}`),
			Time:  time.Now(),
		})
	}

	if err := producer.SendMessages(ctx, messages); err != nil {
		// 处理错误
	}
}
```

#### 3.1.5 消费者消费流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Consumer Group 消费机制                               │
└─────────────────────────────────────────────────────────────────────────────┘

                    Topic: market.ticker (6个分区)
          ┌─────┬─────┬─────┬─────┬─────┬─────┐
          │ P0  │ P1  │ P2  │ P3  │ P4  │ P5  │
          └──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┘
             │     │     │     │     │     │
             └─────┼─────┼─────┼─────┼─────┘
                   │     │     │     │
    Consumer Group: trading-strategy-group
    ┌──────────────┴─────┴─────┴─────┴──────────────┐
    │                                               │
    │  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
    │  │Consumer 1 │  │Consumer 2 │  │Consumer 3 │  │
    │  │  P0, P1   │  │  P2, P3   │  │  P4, P5   │  │
    │  └───────────┘  └───────────┘  └───────────┘  │
    │                                               │
    │          Group Coordinator (Broker)           │
    │  ┌─────────────────────────────────────────┐  │
    │  │  • 管理消费者成员                         │  │
    │  │  • 分区分配 (Range/RoundRobin/Sticky)    │  │
    │  │  • 心跳检测                              │  │
    │  │  • Offset 管理                          │  │
    │  └─────────────────────────────────────────┘  │
    └───────────────────────────────────────────────┘

Offset 提交机制:
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│   poll()        处理消息        commit()                            │
│     │              │              │                                │
│     ▼              ▼              ▼                                │
│  ┌─────┐       ┌───────┐       ┌──────┐                            │
│  │Fetch│──────▶│Process│────▶  │Commit│                            │
│  │ 100 │       │ 100   │       │ 100  │                            │
│  └─────┘       └───────┘       └──────┘                            │
│                                 │                                  │
│                                 ▼                                  │
│                    __consumer_offsets (内部Topic)                  │
│                    ┌────────────────────────┐                      │
│                    │ Group: xxx             │                      │
│                    │ Topic: market.ticker   │                      │
│                    │ Partition: 0           │                      │
│                    │ Offset: 100            │                      │
│                    └────────────────────────┘                      │
└────────────────────────────────────────────────────────────────────┘
```

**Go 代码示例 - 高性能 Kafka Consumer**：

```go
package kafka

import (
	"context"
	"log"
	"sync"
	"time"

	"github.com/segmentio/kafka-go"
)

// ConsumerConfig 消费者配置
type ConsumerConfig struct {
	Brokers        []string
	GroupID        string
	Topics         []string
	MinBytes       int           // 最小拉取字节数
	MaxBytes       int           // 最大拉取字节数
	MaxWait        time.Duration // 最大等待时间
	CommitInterval time.Duration // 自动提交间隔
	StartOffset    int64         // 起始偏移量
	WorkerCount    int           // 并发处理worker数
}

// DefaultConsumerConfig 默认消费者配置
func DefaultConsumerConfig(brokers []string, groupID string, topics []string) *ConsumerConfig {
	return &ConsumerConfig{
		Brokers:        brokers,
		GroupID:        groupID,
		Topics:         topics,
		MinBytes:       10 << 10, // 10KB
		MaxBytes:       10 << 20, // 10MB
		MaxWait:        100 * time.Millisecond,
		CommitInterval: time.Second,
		StartOffset:    kafka.LastOffset, // 从最新开始消费
		WorkerCount:    4,
	}
}

// MessageHandler 消息处理函数
type MessageHandler func(ctx context.Context, msg kafka.Message) error

// HighPerformanceConsumer 高性能消费者
type HighPerformanceConsumer struct {
	reader   *kafka.Reader
	config   *ConsumerConfig
	handler  MessageHandler
	wg       sync.WaitGroup
	msgChan  chan kafka.Message
	stopChan chan struct{}
}

// NewHighPerformanceConsumer 创建高性能消费者
func NewHighPerformanceConsumer(config *ConsumerConfig, handler MessageHandler) *HighPerformanceConsumer {
	reader := kafka.NewReader(kafka.ReaderConfig{
		Brokers:        config.Brokers,
		GroupID:        config.GroupID,
		GroupTopics:    config.Topics,
		MinBytes:       config.MinBytes,
		MaxBytes:       config.MaxBytes,
		MaxWait:        config.MaxWait,
		CommitInterval: config.CommitInterval,
		StartOffset:    config.StartOffset,
	})

	return &HighPerformanceConsumer{
		reader:   reader,
		config:   config,
		handler:  handler,
		msgChan:  make(chan kafka.Message, config.WorkerCount*100),
		stopChan: make(chan struct{}),
	}
}

// Start 启动消费者
func (c *HighPerformanceConsumer) Start(ctx context.Context) error {
	// 启动处理 workers
	for i := 0; i < c.config.WorkerCount; i++ {
		c.wg.Add(1)
		go c.worker(ctx, i)
	}

	// 启动拉取协程
	c.wg.Add(1)
	go c.fetcher(ctx)

	return nil
}

// fetcher 消息拉取协程
func (c *HighPerformanceConsumer) fetcher(ctx context.Context) {
	defer c.wg.Done()

	for {
		select {
		case <-ctx.Done():
			return
		case <-c.stopChan:
			return
		default:
		}

		msg, err := c.reader.FetchMessage(ctx)
		if err != nil {
			if ctx.Err() != nil {
				return
			}
			log.Printf("fetch error: %v", err)
			continue
		}

		select {
		case c.msgChan <- msg:
		case <-ctx.Done():
			return
		}
	}
}

// worker 消息处理 worker
func (c *HighPerformanceConsumer) worker(ctx context.Context, id int) {
	defer c.wg.Done()

	for {
		select {
		case <-ctx.Done():
			return
		case <-c.stopChan:
			return
		case msg := <-c.msgChan:
			if err := c.handler(ctx, msg); err != nil {
				log.Printf("worker %d handle error: %v", id, err)
				// 可以实现重试逻辑或发送到死信队列
				continue
			}

			// 提交偏移量
			if err := c.reader.CommitMessages(ctx, msg); err != nil {
				log.Printf("commit error: %v", err)
			}
		}
	}
}

// Stop 停止消费者
func (c *HighPerformanceConsumer) Stop() error {
	close(c.stopChan)
	c.wg.Wait()
	return c.reader.Close()
}

// Stats 获取消费者统计
func (c *HighPerformanceConsumer) Stats() kafka.ReaderStats {
	return c.reader.Stats()
}

// Lag 获取消费延迟
func (c *HighPerformanceConsumer) Lag() int64 {
	stats := c.reader.Stats()
	return stats.Lag
}
```

#### 3.1.6 Kafka 高可用机制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Kafka 高可用机制                                      │
└─────────────────────────────────────────────────────────────────────────────┘

1. Leader 选举流程:

   Broker 1 (Leader)         Broker 2 (Follower)       Broker 3 (Follower)
        │                            │                         │
        │  ──────── 正常同步 ────────▶│                         │
        │  ──────── 正常同步 ─────────────────────────────────▶ │
        │                           │                          │
        ╳ (宕机)                     │                          │
                                    │                          │
                           Controller 检测到 Leader 失效
                                    │
                           从 ISR 中选择新 Leader
                                    │
                                    ▼
                           ┌───────────────┐
                           │  Broker 2     │
                           │  成为新 Leader │
                           └───────────────┘
                                   │
                                   ▼
                           更新元数据,通知所有客户端

2. ISR (In-Sync Replicas) 机制:

   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                 │
   │  replica.lag.time.max.ms = 10000 (10秒)                        │
   │                                                                 │
   │  Leader Offset: 1000                                           │
   │  ┌────────────────────────────────────────────────────────┐    │
   │  │ Follower 1 Offset: 995  │ Lag: 5   │ 在 ISR 中 ✓       │    │
   │  │ Follower 2 Offset: 980  │ Lag: 20  │ 在 ISR 中 ✓       │    │
   │  │ Follower 3 Offset: 800  │ Lag: 200 │ 移出 ISR ✗        │    │
   │  └────────────────────────────────────────────────────────┘    │
   │                                                                 │
   │  min.insync.replicas = 2                                       │
   │  当 ISR 数量 < 2 时，拒绝写入 (保证数据不丢失)                   │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

3. 数据可靠性级别:

   acks=0: 不等待确认 (最快，可能丢数据)
   ┌──────────┐         ┌──────────┐
   │ Producer │────────▶│  Broker  │  发完即返回
   └──────────┘         └──────────┘

   acks=1: Leader 确认 (平衡)
   ┌──────────┐         ┌──────────┐
   │ Producer │────────▶│  Leader  │  Leader 写入即返回
   └──────────┘    ◀────└──────────┘

   acks=all: 所有 ISR 确认 (最慢，最可靠)
   ┌──────────┐         ┌──────────┐         ┌───────────┐
   │ Producer │────────▶│  Leader  │────────▶│ Followers │
   └──────────┘    ◀────└──────────┘    ◀────└───────────┘
                         所有 ISR 同步完成后返回
```

---

### 3.2 RabbitMQ 深度剖析

RabbitMQ 是基于 AMQP 协议的消息中间件，以其灵活的路由机制和可靠的消息传递著称，特别适合订单处理、通知系统等需要复杂路由的场景。

#### 3.2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RabbitMQ 架构概览                                     │
└─────────────────────────────────────────────────────────────────────────────┘

  Producer                    RabbitMQ Broker                      Consumer
     │                              │                                  │
     │    AMQP Connection           │                                  │
     │◀════════════════════════════▶│                                  │
     │         Channel              │                                  │
     │    ┌─────────────────────────┼──────────────────────────┐       │
     │    │                         │                          │       │
     ▼    ▼                         ▼                          ▼       ▼
┌─────────────┐              ┌─────────────┐              ┌─────────────┐
│  Publisher  │──────────────│  Exchange   │──────────────│   Queue     │────▶ Consumer
└─────────────┘   Routing    └─────────────┘   Binding    └─────────────┘
                   Key              │              Key
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
              ┌──────────┐  ┌──────────┐  ┌──────────┐
              │  Queue 1 │  │  Queue 2 │  │  Queue 3 │
              └──────────┘  └──────────┘  └──────────┘


Connection vs Channel:
┌─────────────────────────────────────────────────────────────────────┐
│                         TCP Connection                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │   ┌───────────┐  ┌───────────┐  ┌───────────┐              │   │
│  │   │ Channel 1 │  │ Channel 2 │  │ Channel 3 │  ...         │   │
│  │   │           │  │           │  │           │              │   │
│  │   │ • Publish │  │ • Consume │  │ • Publish │              │   │
│  │   │ • Confirm │  │ • Ack     │  │ • Declare │              │   │
│  │   └───────────┘  └───────────┘  └───────────┘              │   │
│  │                                                             │   │
│  │   轻量级逻辑连接，复用同一 TCP 连接                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3.2.2 Exchange 类型与路由机制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Exchange 类型详解                                     │
└─────────────────────────────────────────────────────────────────────────────┘

1. Direct Exchange (直连交换机)
   精确匹配 Routing Key

   Producer ──▶ [order.created] ──▶ ┌─────────────┐
                                    │   Direct    │
                                    │  Exchange   │
                                    └──────┬──────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
            Binding Key:           Binding Key:           Binding Key:
            order.created          order.paid             order.shipped
                    │                      │                      │
                    ▼                      ▼                      ▼
              ┌──────────┐          ┌──────────┐          ┌──────────┐
              │ Queue A  │          │ Queue B  │          │ Queue C  │
              │ (创建)    │          │ (支付)    │          │ (发货)    │
              └──────────┘          └──────────┘          └──────────┘

2. Fanout Exchange (扇出交换机)
   广播到所有绑定队列，忽略 Routing Key

   Producer ──▶ [any key] ──▶ ┌─────────────┐
                              │   Fanout    │
                              │  Exchange   │
                              └──────┬──────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
              ┌──────────┐    ┌──────────┐    ┌──────────┐
              │ Queue A  │    │ Queue B  │    │ Queue C  │
              │ (日志)    │    │ (监控)    │    │ (告警)    │
              └──────────┘    └──────────┘    └──────────┘

3. Topic Exchange (主题交换机)
   模式匹配 (* 匹配一个词, # 匹配零或多个词)

   Producer ──▶ [order.btc.created] ──▶ ┌─────────────┐
                                        │   Topic     │
                                        │  Exchange   │
                                        └──────┬──────┘
                                               │
                    ┌──────────────────────────┼──────────────────────────┐
                    │                          │                          │
            Binding Key:               Binding Key:               Binding Key:
            order.*.created            order.btc.*                *.*.created
                    │                          │                          │
                    ▼                          ▼                          ▼
              ┌──────────┐              ┌──────────┐              ┌──────────┐
              │ Queue A  │              │ Queue B  │              │ Queue C  │
              │(所有创建) │              │ (BTC订单) │              │(创建事件) │
              └──────────┘              └──────────┘              └──────────┘

4. Headers Exchange (头交换机)
   基于消息头属性匹配

   Producer ──▶ headers: {type: "order", priority: "high"}
                                        │
                              ┌─────────▼─────────┐
                              │    Headers        │
                              │    Exchange       │
                              └─────────┬─────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
            x-match: all          x-match: any        x-match: all
            type=order            type=order          priority=high
            priority=high         priority=low
                    │                   │                   │
                    ▼                   ▼                   ▼
              ┌──────────┐        ┌──────────┐        ┌──────────┐
              │ Queue A  │        │ Queue B  │        │ Queue C  │
              └──────────┘        └──────────┘        └──────────┘
```

#### 3.2.3 消息确认机制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        消息确认机制                                          │
└─────────────────────────────────────────────────────────────────────────────┘

1. Publisher Confirms (生产者确认)

   Producer                         Broker
      │                               │
      │  ──── publish msg (seq=1) ───▶│
      │  ──── publish msg (seq=2) ───▶│  持久化到磁盘
      │  ──── publish msg (seq=3) ───▶│
      │                               │
      │  ◀─── basic.ack (seq=3) ─────│  批量确认 (multiple=true)
      │                               │
      │  或者                          │
      │  ◀─── basic.nack (seq=2) ────│  消息被拒绝
      │                               │

   确认模式:
   ┌────────────────────────────────────────────────────────────────┐
   │  • 单条确认: 每条消息单独确认                                    │
   │  • 批量确认: multiple=true，确认 seq 及之前所有消息              │
   │  • 异步确认: 使用 confirm callback，非阻塞                      │
   └────────────────────────────────────────────────────────────────┘

2. Consumer Acknowledgements (消费者确认)

   Broker                          Consumer
      │                               │
      │  ──── deliver msg ──────────▶│
      │                               │  处理消息...
      │                               │
      │  ◀─── basic.ack ─────────────│  确认成功
      │  或                            │
      │  ◀─── basic.nack ────────────│  拒绝，可选 requeue
      │  或                            │
      │  ◀─── basic.reject ──────────│  拒绝单条消息
      │                               │

   QoS (prefetch) 设置:
   ┌────────────────────────────────────────────────────────────────┐
   │                                                                │
   │  prefetch_count = 10                                           │
   │                                                                │
   │  Broker                    Consumer                            │
   │     │                         │                                │
   │     │ ─── msg 1 ────────────▶│  unacked: 1                    │
   │     │ ─── msg 2 ────────────▶│  unacked: 2                    │
   │     │ ─── ...  ─────────────▶│  unacked: ...                  │
   │     │ ─── msg 10 ───────────▶│  unacked: 10                   │
   │     │                         │                                │
   │     │  (等待 ack 才发送更多)    │                                │
   │     │                         │                                │
   │     │ ◀─── ack msg 1 ────────│  unacked: 9                    │
   │     │ ─── msg 11 ───────────▶│  unacked: 10                   │
   │     │                         │                                │
   └────────────────────────────────────────────────────────────────┘
```

#### 3.2.4 消息持久化与存储

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RabbitMQ 存储架构                                     │
└─────────────────────────────────────────────────────────────────────────────┘

消息存储流程:

  持久化消息                              非持久化消息
       │                                       │
       ▼                                       ▼
┌─────────────┐                         ┌─────────────┐
│  msg_store  │                         │   RAM only  │
│  (磁盘存储)  │                         │  (内存存储)  │
└──────┬──────┘                         └─────────────┘
       │
       ├──▶ .rdq 文件 (消息数据)
       │    ┌────────────────────────────────────────┐
       │    │  msg_id | size | content | ...        │
       │    └────────────────────────────────────────┘
       │
       └──▶ .idx 文件 (索引)
            ┌────────────────────────────────────────┐
            │  msg_id | file_pos | ref_count | ...  │
            └────────────────────────────────────────┘

队列索引结构:
┌─────────────────────────────────────────────────────────────────────┐
│                          Queue Index                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Segment Files (.idx)                                              │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │
│   │  Segment 0    │  │  Segment 1    │  │  Segment 2    │  ...     │
│   │  seq 0-16383  │  │ seq 16384-... │  │     ...       │          │
│   └───────────────┘  └───────────────┘  └───────────────┘          │
│                                                                     │
│   每个 Segment 包含:                                                 │
│   • 发布记录 (publish)                                              │
│   • 投递记录 (deliver)                                              │
│   • 确认记录 (ack)                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

内存管理:
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  vm_memory_high_watermark = 0.4 (默认占用系统 40% 内存)              │
│                                                                     │
│  内存使用                  触发机制                                   │
│       │                                                             │
│  100% ├─────────────────────────────────────────                    │
│       │                                                             │
│   40% ├───────────────────── 阻塞生产者，开始 paging                 │
│       │                      (消息写入磁盘)                          │
│       │                                                             │
│    0% └─────────────────────────────────────────                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3.2.5 集群与高可用

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RabbitMQ 集群架构                                     │
└─────────────────────────────────────────────────────────────────────────────┘

1. 普通集群 (元数据同步，队列数据不复制)

   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │   Node 1    │◀────▶│   Node 2    │◀────▶│   Node 3    │
   │             │      │             │      │             │
   │  Queue A    │      │  Queue B    │      │  Queue C    │
   │  (master)   │      │  (master)   │      │  (master)   │
   │             │      │             │      │             │
   │ 元数据: ABC │      │ 元数据: ABC │      │ 元数据: ABC │
   └─────────────┘      └─────────────┘      └─────────────┘

   特点: 队列数据只在一个节点，其他节点存元数据指针

2. 镜像队列 (Classic Mirrored Queues) - 已废弃

   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │   Node 1    │      │   Node 2    │      │   Node 3    │
   │             │      │             │      │             │
   │  Queue A    │─────▶│  Queue A    │─────▶│  Queue A    │
   │  (master)   │ sync │  (mirror)   │ sync │  (mirror)   │
   └─────────────┘      └─────────────┘      └─────────────┘

3. Quorum Queues (仲裁队列) - 推荐使用

   基于 Raft 协议的强一致性队列

   ┌─────────────────────────────────────────────────────────────────┐
   │                      Quorum Queue 架构                          │
   ├─────────────────────────────────────────────────────────────────┤
   │                                                                 │
   │   ┌─────────┐      ┌─────────┐      ┌─────────┐                │
   │   │ Node 1  │      │ Node 2  │      │ Node 3  │                │
   │   │         │      │         │      │         │                │
   │   │ Leader  │◀────▶│Follower │◀────▶│Follower │                │
   │   │         │ Raft │         │ Raft │         │                │
   │   └────┬────┘      └────┬────┘      └────┬────┘                │
   │        │                │                │                      │
   │        ▼                ▼                ▼                      │
   │   ┌─────────┐      ┌─────────┐      ┌─────────┐                │
   │   │  WAL    │      │  WAL    │      │  WAL    │                │
   │   │  Log    │      │  Log    │      │  Log    │                │
   │   └─────────┘      └─────────┘      └─────────┘                │
   │                                                                 │
   │   写入流程:                                                      │
   │   1. Client 发送消息到 Leader                                    │
   │   2. Leader 追加到本地 WAL                                       │
   │   3. Leader 复制到 Followers                                    │
   │   4. 多数节点 (2/3) 确认后，提交并返回                            │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

   Quorum Queue vs Classic Queue:
   ┌──────────────────┬───────────────────┬───────────────────┐
   │      特性        │   Classic Queue   │   Quorum Queue    │
   ├──────────────────┼───────────────────┼───────────────────┤
   │ 数据安全性       │ 可能丢失          │ 强一致性          │
   │ 性能             │ 较高              │ 略低              │
   │ 内存使用         │ 较少              │ 较多              │
   │ 消息顺序         │ 保证              │ 保证              │
   │ 适用场景         │ 高吞吐非关键      │ 金融交易等        │
   └──────────────────┴───────────────────┴───────────────────┘
```

#### 3.2.6 Go 代码示例 - RabbitMQ 生产者/消费者

```go
package rabbitmq

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"sync"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

// ConnectionConfig RabbitMQ 连接配置
type ConnectionConfig struct {
	URL              string
	Vhost            string
	HeartbeatTimeout time.Duration
	ConnectionName   string
}

// Connection RabbitMQ 连接包装器
type Connection struct {
	conn   *amqp.Connection
	config *ConnectionConfig
	mu     sync.RWMutex
	closed bool
}

// NewConnection 创建新连接
func NewConnection(config *ConnectionConfig) (*Connection, error) {
	amqpConfig := amqp.Config{
		Vhost:     config.Vhost,
		Heartbeat: config.HeartbeatTimeout,
		Properties: amqp.Table{
			"connection_name": config.ConnectionName,
		},
	}

	conn, err := amqp.DialConfig(config.URL, amqpConfig)
	if err != nil {
		return nil, fmt.Errorf("dial rabbitmq: %w", err)
	}

	return &Connection{
		conn:   conn,
		config: config,
	}, nil
}

// OrderProducer 订单消息生产者
type OrderProducer struct {
	conn         *Connection
	channel      *amqp.Channel
	exchangeName string
	confirms     chan amqp.Confirmation
	mu           sync.Mutex
}

// ProducerConfig 生产者配置
type ProducerConfig struct {
	ExchangeName string
	ExchangeType string // direct, fanout, topic, headers
	Durable      bool
	AutoDelete   bool
}

// NewOrderProducer 创建订单生产者
func NewOrderProducer(conn *Connection, config *ProducerConfig) (*OrderProducer, error) {
	ch, err := conn.conn.Channel()
	if err != nil {
		return nil, fmt.Errorf("create channel: %w", err)
	}

	// 声明交换机
	err = ch.ExchangeDeclare(
		config.ExchangeName,
		config.ExchangeType,
		config.Durable,
		config.AutoDelete,
		false, // internal
		false, // no-wait
		nil,   // arguments
	)
	if err != nil {
		return nil, fmt.Errorf("declare exchange: %w", err)
	}

	// 开启 Publisher Confirms
	if err := ch.Confirm(false); err != nil {
		return nil, fmt.Errorf("enable confirms: %w", err)
	}

	confirms := ch.NotifyPublish(make(chan amqp.Confirmation, 100))

	producer := &OrderProducer{
		conn:         conn,
		channel:      ch,
		exchangeName: config.ExchangeName,
		confirms:     confirms,
	}

	// 启动确认处理协程
	go producer.handleConfirms()

	return producer, nil
}

// OrderMessage 订单消息
type OrderMessage struct {
	OrderID   string    `json:"order_id"`
	UserID    string    `json:"user_id"`
	Symbol    string    `json:"symbol"`
	Side      string    `json:"side"`
	Price     string    `json:"price"`
	Quantity  string    `json:"quantity"`
	Timestamp time.Time `json:"timestamp"`
}

// PublishOrder 发布订单消息
func (p *OrderProducer) PublishOrder(ctx context.Context, routingKey string, order *OrderMessage) error {
	p.mu.Lock()
	defer p.mu.Unlock()

	body, err := json.Marshal(order)
	if err != nil {
		return fmt.Errorf("marshal order: %w", err)
	}

	msg := amqp.Publishing{
		ContentType:  "application/json",
		DeliveryMode: amqp.Persistent, // 持久化消息
		Timestamp:    time.Now(),
		MessageId:    order.OrderID,
		Body:         body,
		Headers: amqp.Table{
			"order_type": order.Side,
			"symbol":     order.Symbol,
		},
	}

	return p.channel.PublishWithContext(
		ctx,
		p.exchangeName,
		routingKey,
		true,  // mandatory: 如果无法路由则返回
		false, // immediate: 已废弃
		msg,
	)
}

// PublishBatch 批量发布消息
func (p *OrderProducer) PublishBatch(ctx context.Context, routingKey string, orders []*OrderMessage) error {
	p.mu.Lock()
	defer p.mu.Unlock()

	for _, order := range orders {
		body, err := json.Marshal(order)
		if err != nil {
			return fmt.Errorf("marshal order %s: %w", order.OrderID, err)
		}

		msg := amqp.Publishing{
			ContentType:  "application/json",
			DeliveryMode: amqp.Persistent,
			Timestamp:    time.Now(),
			MessageId:    order.OrderID,
			Body:         body,
		}

		if err := p.channel.PublishWithContext(ctx, p.exchangeName, routingKey, true, false, msg); err != nil {
			return fmt.Errorf("publish order %s: %w", order.OrderID, err)
		}
	}

	return nil
}

func (p *OrderProducer) handleConfirms() {
	for confirm := range p.confirms {
		if confirm.Ack {
			log.Printf("Message %d confirmed", confirm.DeliveryTag)
		} else {
			log.Printf("Message %d nacked, need retry", confirm.DeliveryTag)
			// 实现重试逻辑
		}
	}
}

// Close 关闭生产者
func (p *OrderProducer) Close() error {
	return p.channel.Close()
}

// ─────────────────────────────────────────────────────────────────────────────
// 消费者实现
// ─────────────────────────────────────────────────────────────────────────────

// ConsumerConfig 消费者配置
type ConsumerConfig struct {
	QueueName     string
	ExchangeName  string
	RoutingKeys   []string
	PrefetchCount int
	PrefetchSize  int
	AutoAck       bool
	Exclusive     bool
	ConsumerTag   string
	Durable       bool
	QuorumQueue   bool // 是否使用 Quorum Queue
}

// OrderConsumer 订单消费者
type OrderConsumer struct {
	conn     *Connection
	channel  *amqp.Channel
	config   *ConsumerConfig
	handler  OrderHandler
	stopChan chan struct{}
	wg       sync.WaitGroup
}

// OrderHandler 订单处理函数
type OrderHandler func(ctx context.Context, order *OrderMessage) error

// NewOrderConsumer 创建订单消费者
func NewOrderConsumer(conn *Connection, config *ConsumerConfig, handler OrderHandler) (*OrderConsumer, error) {
	ch, err := conn.conn.Channel()
	if err != nil {
		return nil, fmt.Errorf("create channel: %w", err)
	}

	// 设置 QoS
	if err := ch.Qos(config.PrefetchCount, config.PrefetchSize, false); err != nil {
		return nil, fmt.Errorf("set qos: %w", err)
	}

	// 队列参数
	args := amqp.Table{}
	if config.QuorumQueue {
		args["x-queue-type"] = "quorum"
	}

	// 声明队列
	queue, err := ch.QueueDeclare(
		config.QueueName,
		config.Durable,
		false, // auto-delete
		config.Exclusive,
		false, // no-wait
		args,
	)
	if err != nil {
		return nil, fmt.Errorf("declare queue: %w", err)
	}

	// 绑定队列到交换机
	for _, key := range config.RoutingKeys {
		if err := ch.QueueBind(queue.Name, key, config.ExchangeName, false, nil); err != nil {
			return nil, fmt.Errorf("bind queue with key %s: %w", key, err)
		}
	}

	return &OrderConsumer{
		conn:     conn,
		channel:  ch,
		config:   config,
		handler:  handler,
		stopChan: make(chan struct{}),
	}, nil
}

// Start 启动消费者
func (c *OrderConsumer) Start(ctx context.Context) error {
	deliveries, err := c.channel.Consume(
		c.config.QueueName,
		c.config.ConsumerTag,
		c.config.AutoAck,
		c.config.Exclusive,
		false, // no-local
		false, // no-wait
		nil,
	)
	if err != nil {
		return fmt.Errorf("start consume: %w", err)
	}

	c.wg.Add(1)
	go c.consume(ctx, deliveries)

	return nil
}

func (c *OrderConsumer) consume(ctx context.Context, deliveries <-chan amqp.Delivery) {
	defer c.wg.Done()

	for {
		select {
		case <-ctx.Done():
			return
		case <-c.stopChan:
			return
		case delivery, ok := <-deliveries:
			if !ok {
				log.Println("Delivery channel closed")
				return
			}

			c.processDelivery(ctx, delivery)
		}
	}
}

func (c *OrderConsumer) processDelivery(ctx context.Context, delivery amqp.Delivery) {
	var order OrderMessage
	if err := json.Unmarshal(delivery.Body, &order); err != nil {
		log.Printf("Unmarshal error: %v, rejecting message", err)
		// 解析失败，不重新入队
		_ = delivery.Reject(false)
		return
	}

	// 处理消息
	if err := c.handler(ctx, &order); err != nil {
		log.Printf("Handle order %s error: %v", order.OrderID, err)

		// 检查重试次数
		retryCount := getRetryCount(delivery.Headers)
		if retryCount < 3 {
			// 重新入队，增加重试计数
			_ = delivery.Nack(false, true)
		} else {
			// 超过重试次数，发送到死信队列
			_ = delivery.Reject(false)
		}
		return
	}

	// 确认消息
	if err := delivery.Ack(false); err != nil {
		log.Printf("Ack error: %v", err)
	}
}

func getRetryCount(headers amqp.Table) int {
	if headers == nil {
		return 0
	}
	if count, ok := headers["x-retry-count"].(int); ok {
		return count
	}
	return 0
}

// Stop 停止消费者
func (c *OrderConsumer) Stop() error {
	close(c.stopChan)
	c.wg.Wait()
	return c.channel.Close()
}

// ─────────────────────────────────────────────────────────────────────────────
// 死信队列配置示例
// ─────────────────────────────────────────────────────────────────────────────

// SetupDeadLetterQueue 配置死信队列
func SetupDeadLetterQueue(ch *amqp.Channel) error {
	// 1. 声明死信交换机
	if err := ch.ExchangeDeclare(
		"orders.dlx",
		"direct",
		true,
		false,
		false,
		false,
		nil,
	); err != nil {
		return err
	}

	// 2. 声明死信队列
	if _, err := ch.QueueDeclare(
		"orders.dead",
		true,
		false,
		false,
		false,
		nil,
	); err != nil {
		return err
	}

	// 3. 绑定死信队列
	if err := ch.QueueBind("orders.dead", "dead", "orders.dlx", false, nil); err != nil {
		return err
	}

	// 4. 声明主队列，配置死信
	args := amqp.Table{
		"x-dead-letter-exchange":    "orders.dlx",
		"x-dead-letter-routing-key": "dead",
		"x-message-ttl":             int32(60000), // 60秒TTL
	}

	if _, err := ch.QueueDeclare(
		"orders.main",
		true,
		false,
		false,
		false,
		args,
	); err != nil {
		return err
	}

	return nil
}
```

---

### 3.3 Apache RocketMQ 深度剖析

RocketMQ 是阿里巴巴开源的分布式消息中间件，专为金融级场景设计，具有高吞吐、低延迟、高可用的特点，在国内金融交易系统中广泛应用。

#### 3.3.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RocketMQ 架构概览                                     │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────┐
                              │   NameServer    │
                              │   Cluster       │
                              │  (路由注册中心)  │
                              └────────┬────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │ 注册 Broker                 │ 查询路由                    │ 注册 Broker
          │                            │                            │
          ▼                            ▼                            ▼
   ┌─────────────┐              ┌─────────────┐              ┌─────────────┐
   │  Broker-a   │              │  Producer   │              │  Broker-b   │
   │  Master     │◀─────────────│   Group     │─────────────▶│  Master     │
   ├─────────────┤   发送消息    └─────────────┘   发送消息    ├─────────────┤
   │  Broker-a   │                    │                      │  Broker-b   │
   │  Slave      │                    │                      │  Slave      │
   └─────────────┘                    │                      └─────────────┘
          ▲                           │                            ▲
          │                           ▼                            │
          │                    ┌─────────────┐                     │
          │                    │  Consumer   │                     │
          └────────────────────│   Group     │─────────────────────┘
                 拉取消息        └─────────────┘        拉取消息


NameServer vs ZooKeeper:
┌────────────────────────┬─────────────────────┬─────────────────────┐
│         特性           │     NameServer      │     ZooKeeper       │
├────────────────────────┼─────────────────────┼─────────────────────┤
│ 一致性模型             │ 最终一致性(AP)      │ 强一致性(CP)        │
│ 复杂度                 │ 简单轻量            │ 功能丰富但复杂      │
│ 节点间通信             │ 无                  │ Leader选举/同步     │
│ 性能                   │ 高                  │ 中等                │
│ 依赖                   │ 无外部依赖          │ 需要单独集群        │
└────────────────────────┴─────────────────────┴─────────────────────┘
```

#### 3.3.2 消息存储模型

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RocketMQ 存储架构                                     │
└─────────────────────────────────────────────────────────────────────────────┘

核心设计: 所有消息顺序写入 CommitLog，ConsumeQueue 作为索引

                         Broker 存储结构
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   CommitLog (消息主体存储)                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  所有 Topic 的消息顺序追加写入                                        │  │
│   │  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐           │  │
│   │  │Msg1 │Msg2 │Msg3 │Msg4 │Msg5 │Msg6 │Msg7 │Msg8 │ ... │           │  │
│   │  │TopicA│TopicB│TopicA│TopicC│TopicA│TopicB│TopicA│TopicC│           │  │
│   │  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘           │  │
│   │  文件大小: 1GB, 文件名: 起始偏移量                                    │  │
│   │  00000000000000000000  (0-1GB)                                       │  │
│   │  00000000001073741824  (1GB-2GB)                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                    │                                                        │
│                    │ 异步构建索引                                            │
│                    ▼                                                        │
│   ConsumeQueue (消费队列索引)                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  Topic: orders                                                       │  │
│   │  ├── Queue 0                                                         │  │
│   │  │   ┌──────────────┬──────────────┬──────────────┐                 │  │
│   │  │   │CommitLog偏移 │ 消息大小     │ Tag HashCode │  (20字节/条)    │  │
│   │  │   │   8 bytes    │  4 bytes     │   8 bytes    │                 │  │
│   │  │   └──────────────┴──────────────┴──────────────┘                 │  │
│   │  ├── Queue 1                                                         │  │
│   │  ├── Queue 2                                                         │  │
│   │  └── Queue 3                                                         │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   IndexFile (消息索引，可选)                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  支持按 Message Key 或时间范围查询消息                                │  │
│   │  Hash 索引结构: Key -> CommitLog Offset                              │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

消息写入流程:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│  1. Producer 发送消息                                                       │
│         │                                                                  │
│         ▼                                                                  │
│  2. Broker 接收，写入 CommitLog (顺序写，PageCache)                         │
│         │                                                                  │
│         ▼                                                                  │
│  3. ReputMessageService 异步构建 ConsumeQueue 和 IndexFile                 │
│         │                                                                  │
│         ▼                                                                  │
│  4. FlushCommitLogService 异步/同步刷盘                                     │
│                                                                            │
│  刷盘策略:                                                                  │
│  ┌─────────────────┬─────────────────────────────────────────────────┐    │
│  │ SYNC_FLUSH      │ 同步刷盘，消息写入磁盘后返回，可靠但慢           │    │
│  │ ASYNC_FLUSH     │ 异步刷盘，写入 PageCache 即返回，快但可能丢失    │    │
│  └─────────────────┴─────────────────────────────────────────────────┘    │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

#### 3.3.3 消息类型与特性

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RocketMQ 消息类型                                     │
└─────────────────────────────────────────────────────────────────────────────┘

1. 普通消息 (Normal Message)
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │ Producer │────────▶│  Broker  │────────▶│ Consumer │
   └──────────┘         └──────────┘         └──────────┘

2. 顺序消息 (Ordered Message)
   同一 Sharding Key 的消息发送到同一队列，保证顺序

   Producer                          Broker                    Consumer
      │                                │                           │
      │ OrderID=1001, Msg1 ──────────▶│ Queue 0: [Msg1,Msg2,Msg3] │
      │ OrderID=1001, Msg2 ──────────▶│           ↓               │
      │ OrderID=1001, Msg3 ──────────▶│    顺序消费                │───▶ 顺序处理
      │                                │                           │

3. 延迟消息 (Delayed Message)
   消息发送后不立即可见，延迟指定时间后才能消费

   Producer ──▶ Broker ──▶ SCHEDULE_TOPIC_XXXX ──(延迟)──▶ 目标 Topic
                                    │
                           延迟级别: 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m
                                   9m 10m 20m 30m 1h 2h

4. 事务消息 (Transactional Message)

   ┌─────────────────────────────────────────────────────────────────────┐
   │                     事务消息流程                                     │
   │                                                                     │
   │  Producer                    Broker                  本地事务       │
   │      │                          │                        │          │
   │  (1) │──── Half Message ───────▶│                        │          │
   │      │                          │ 存入 Half Topic        │          │
   │      │◀─── 发送成功 ────────────│ (对消费者不可见)        │          │
   │      │                          │                        │          │
   │  (2) │ 执行本地事务 ────────────────────────────────────▶│          │
   │      │                          │                        │          │
   │  (3) │──── Commit/Rollback ────▶│                        │          │
   │      │                          │                        │          │
   │      │     如果没收到确认:       │                        │          │
   │  (4) │◀─── 回查本地事务状态 ────│                        │          │
   │      │──── 返回状态 ───────────▶│                        │          │
   │                                                                     │
   └─────────────────────────────────────────────────────────────────────┘

5. 批量消息 (Batch Message)
   多条消息打包发送，提高吞吐量

   限制条件:
   • 同一 Topic
   • 同一 waitStoreMsgOK
   • 不能是延迟消息或事务消息
   • 总大小不超过 4MB
```

#### 3.3.4 高可用架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RocketMQ 高可用部署                                   │
└─────────────────────────────────────────────────────────────────────────────┘

1. 主从同步模式 (Master-Slave)

   ┌─────────────────────────────────────────────────────────────────────┐
   │                                                                     │
   │   ┌─────────────┐           ┌─────────────┐                        │
   │   │   Master    │           │   Master    │                        │
   │   │  Broker-a   │           │  Broker-b   │                        │
   │   │  brokerId=0 │           │  brokerId=0 │                        │
   │   └──────┬──────┘           └──────┬──────┘                        │
   │          │ 同步/异步复制            │ 同步/异步复制                  │
   │          ▼                         ▼                               │
   │   ┌─────────────┐           ┌─────────────┐                        │
   │   │   Slave     │           │   Slave     │                        │
   │   │  Broker-a   │           │  Broker-b   │                        │
   │   │  brokerId=1 │           │  brokerId=1 │                        │
   │   └─────────────┘           └─────────────┘                        │
   │                                                                     │
   │   复制模式:                                                         │
   │   • SYNC_MASTER: 同步复制，Master 等待 Slave 写入成功后返回         │
   │   • ASYNC_MASTER: 异步复制，Master 写入成功即返回                   │
   │                                                                     │
   └─────────────────────────────────────────────────────────────────────┘

2. Dledger 模式 (自动主从切换)

   基于 Raft 协议实现自动选主

   ┌─────────────────────────────────────────────────────────────────────┐
   │                      Dledger 集群                                   │
   │                                                                     │
   │   ┌───────────┐      ┌───────────┐      ┌───────────┐              │
   │   │  Node 1   │      │  Node 2   │      │  Node 3   │              │
   │   │  Leader   │◀────▶│ Follower  │◀────▶│ Follower  │              │
   │   │           │ Raft │           │ Raft │           │              │
   │   └─────┬─────┘      └─────┬─────┘      └─────┬─────┘              │
   │         │                  │                  │                    │
   │         ▼                  ▼                  ▼                    │
   │   ┌───────────┐      ┌───────────┐      ┌───────────┐              │
   │   │ CommitLog │      │ CommitLog │      │ CommitLog │              │
   │   │ (Dledger) │      │ (Dledger) │      │ (Dledger) │              │
   │   └───────────┘      └───────────┘      └───────────┘              │
   │                                                                     │
   │   特点:                                                             │
   │   • 自动 Leader 选举                                                │
   │   • 多数派写入确认                                                  │
   │   • 故障自动恢复                                                    │
   │                                                                     │
   └─────────────────────────────────────────────────────────────────────┘

故障转移流程:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│   Master 故障                                                               │
│       │                                                                    │
│       ▼                                                                    │
│   NameServer 检测心跳超时 (默认 120s)                                       │
│       │                                                                    │
│       ▼                                                                    │
│   从路由表移除该 Broker                                                     │
│       │                                                                    │
│       ├──▶ 传统模式: 需手动切换 Slave 为 Master                            │
│       │                                                                    │
│       └──▶ Dledger: 自动选举新 Leader (秒级)                               │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

#### 3.3.5 消费模式与负载均衡

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        消费模式                                              │
└─────────────────────────────────────────────────────────────────────────────┘

1. 集群消费 (CLUSTERING) - 默认模式
   同一 Consumer Group 内消费者分摊消息

   Topic: orders (4个队列)
   ┌─────┬─────┬─────┬─────┐
   │ Q0  │ Q1  │ Q2  │ Q3  │
   └──┬──┴──┬──┴──┬──┴──┬──┘
      │     │     │     │
      └─────┼─────┼─────┘
            │     │
   Consumer Group: order-processor
   ┌────────┴─────┴────────┐
   │  Consumer1  Consumer2  │
   │   Q0, Q1     Q2, Q3    │
   └────────────────────────┘

2. 广播消费 (BROADCASTING)
   每个 Consumer 都收到全量消息

   Topic: config-update
   ┌─────────────────────┐
   │      Message        │
   └──────────┬──────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
Consumer1  Consumer2  Consumer3
  全量       全量       全量


负载均衡策略:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│  1. AllocateMessageQueueAveragely (平均分配)                               │
│     Queue: [Q0, Q1, Q2, Q3, Q4, Q5]                                       │
│     Consumer: [C1, C2]                                                     │
│     结果: C1 -> [Q0, Q1, Q2], C2 -> [Q3, Q4, Q5]                          │
│                                                                            │
│  2. AllocateMessageQueueAveragelyByCircle (环形分配)                       │
│     Queue: [Q0, Q1, Q2, Q3, Q4, Q5]                                       │
│     Consumer: [C1, C2]                                                     │
│     结果: C1 -> [Q0, Q2, Q4], C2 -> [Q1, Q3, Q5]                          │
│                                                                            │
│  3. AllocateMessageQueueByConfig (配置分配)                                │
│     手动指定消费者消费哪些队列                                              │
│                                                                            │
│  4. AllocateMessageQueueConsistentHash (一致性哈希)                        │
│     Consumer 变化时最小化重新分配                                           │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘

Rebalance 触发条件:
• Consumer 数量变化 (上线/下线)
• Queue 数量变化 (扩容/缩容)
• 定时触发 (默认 20s)
```

#### 3.3.6 Go 代码示例 - RocketMQ 客户端

```go
package rocketmq

import (
	"context"
	"fmt"
	"log"
	"sync"
	"time"

	"github.com/apache/rocketmq-client-go/v2"
	"github.com/apache/rocketmq-client-go/v2/consumer"
	"github.com/apache/rocketmq-client-go/v2/primitive"
	"github.com/apache/rocketmq-client-go/v2/producer"
)

// ─────────────────────────────────────────────────────────────────────────────
// 生产者实现
// ─────────────────────────────────────────────────────────────────────────────

// ProducerConfig 生产者配置
type ProducerConfig struct {
	NameServers    []string
	GroupName      string
	RetryTimes     int
	SendTimeout    time.Duration
	CompressLevel  int // 压缩级别 0-9
	MaxMessageSize int // 最大消息大小
}

// OrderProducer 订单生产者
type OrderProducer struct {
	producer rocketmq.Producer
	config   *ProducerConfig
}

// NewOrderProducer 创建订单生产者
func NewOrderProducer(config *ProducerConfig) (*OrderProducer, error) {
	p, err := rocketmq.NewProducer(
		producer.WithNameServer(config.NameServers),
		producer.WithGroupName(config.GroupName),
		producer.WithRetry(config.RetryTimes),
		producer.WithSendMsgTimeout(config.SendTimeout),
	)
	if err != nil {
		return nil, fmt.Errorf("create producer: %w", err)
	}

	if err := p.Start(); err != nil {
		return nil, fmt.Errorf("start producer: %w", err)
	}

	return &OrderProducer{
		producer: p,
		config:   config,
	}, nil
}

// OrderMessage 订单消息
type OrderMessage struct {
	OrderID   string `json:"order_id"`
	UserID    string `json:"user_id"`
	Symbol    string `json:"symbol"`
	Side      string `json:"side"`
	Price     string `json:"price"`
	Quantity  string `json:"quantity"`
	Timestamp int64  `json:"timestamp"`
}

// SendSync 同步发送
func (p *OrderProducer) SendSync(ctx context.Context, topic string, order *OrderMessage) (*primitive.SendResult, error) {
	body, err := json.Marshal(order)
	if err != nil {
		return nil, fmt.Errorf("marshal order: %w", err)
	}

	msg := &primitive.Message{
		Topic: topic,
		Body:  body,
	}

	// 设置消息属性
	msg.WithKeys([]string{order.OrderID})
	msg.WithTag(order.Side) // BUY/SELL 作为 Tag

	// 设置用户属性
	msg.WithProperty("user_id", order.UserID)
	msg.WithProperty("symbol", order.Symbol)

	return p.producer.SendSync(ctx, msg)
}

// SendOrderly 顺序发送 (同一 OrderID 发送到同一队列)
func (p *OrderProducer) SendOrderly(ctx context.Context, topic string, order *OrderMessage) (*primitive.SendResult, error) {
	body, err := json.Marshal(order)
	if err != nil {
		return nil, fmt.Errorf("marshal order: %w", err)
	}

	msg := &primitive.Message{
		Topic: topic,
		Body:  body,
	}
	msg.WithKeys([]string{order.OrderID})
	msg.WithTag(order.Side)

	// 使用 OrderID 作为 sharding key，保证同一订单顺序
	selector := producer.NewHashQueueSelector()

	return p.producer.SendSync(ctx, msg, selector)
}

// SendAsync 异步发送
func (p *OrderProducer) SendAsync(ctx context.Context, topic string, order *OrderMessage, callback func(*primitive.SendResult, error)) error {
	body, err := json.Marshal(order)
	if err != nil {
		return fmt.Errorf("marshal order: %w", err)
	}

	msg := &primitive.Message{
		Topic: topic,
		Body:  body,
	}
	msg.WithKeys([]string{order.OrderID})

	return p.producer.SendAsync(ctx, func(ctx context.Context, result *primitive.SendResult, err error) {
		callback(result, err)
	}, msg)
}

// SendDelay 延迟发送
func (p *OrderProducer) SendDelay(ctx context.Context, topic string, order *OrderMessage, delayLevel int) (*primitive.SendResult, error) {
	body, err := json.Marshal(order)
	if err != nil {
		return nil, fmt.Errorf("marshal order: %w", err)
	}

	msg := &primitive.Message{
		Topic: topic,
		Body:  body,
	}
	msg.WithKeys([]string{order.OrderID})
	msg.WithDelayTimeLevel(delayLevel) // 1-18 对应不同延迟时间

	return p.producer.SendSync(ctx, msg)
}

// SendBatch 批量发送
func (p *OrderProducer) SendBatch(ctx context.Context, topic string, orders []*OrderMessage) (*primitive.SendResult, error) {
	msgs := make([]*primitive.Message, 0, len(orders))

	for _, order := range orders {
		body, err := json.Marshal(order)
		if err != nil {
			return nil, fmt.Errorf("marshal order %s: %w", order.OrderID, err)
		}

		msg := &primitive.Message{
			Topic: topic,
			Body:  body,
		}
		msg.WithKeys([]string{order.OrderID})
		msgs = append(msgs, msg)
	}

	return p.producer.SendSync(ctx, msgs...)
}

// Close 关闭生产者
func (p *OrderProducer) Close() error {
	return p.producer.Shutdown()
}

// ─────────────────────────────────────────────────────────────────────────────
// 事务生产者实现
// ─────────────────────────────────────────────────────────────────────────────

// TransactionProducer 事务生产者
type TransactionProducer struct {
	producer rocketmq.TransactionProducer
	checker  TransactionChecker
}

// TransactionChecker 事务回查接口
type TransactionChecker interface {
	CheckLocalTransaction(orderID string) primitive.LocalTransactionState
}

// NewTransactionProducer 创建事务生产者
func NewTransactionProducer(config *ProducerConfig, checker TransactionChecker) (*TransactionProducer, error) {
	p, err := rocketmq.NewTransactionProducer(
		&transactionListener{checker: checker},
		producer.WithNameServer(config.NameServers),
		producer.WithGroupName(config.GroupName),
		producer.WithRetry(config.RetryTimes),
	)
	if err != nil {
		return nil, fmt.Errorf("create transaction producer: %w", err)
	}

	if err := p.Start(); err != nil {
		return nil, fmt.Errorf("start transaction producer: %w", err)
	}

	return &TransactionProducer{
		producer: p,
		checker:  checker,
	}, nil
}

type transactionListener struct {
	checker TransactionChecker
}

// ExecuteLocalTransaction 执行本地事务
func (l *transactionListener) ExecuteLocalTransaction(msg *primitive.Message) primitive.LocalTransactionState {
	// 这里执行本地事务逻辑
	// 返回 CommitMessageState, RollbackMessageState, 或 UnknowState
	return primitive.UnknowState
}

// CheckLocalTransaction 回查本地事务状态
func (l *transactionListener) CheckLocalTransaction(msg *primitive.MessageExt) primitive.LocalTransactionState {
	orderID := msg.GetKeys()
	return l.checker.CheckLocalTransaction(orderID)
}

// SendTransaction 发送事务消息
func (p *TransactionProducer) SendTransaction(ctx context.Context, topic string, order *OrderMessage, localTx func() error) (*primitive.SendResult, error) {
	body, err := json.Marshal(order)
	if err != nil {
		return nil, fmt.Errorf("marshal order: %w", err)
	}

	msg := &primitive.Message{
		Topic: topic,
		Body:  body,
	}
	msg.WithKeys([]string{order.OrderID})

	// 发送半消息并执行本地事务
	result, err := p.producer.SendMessageInTransaction(ctx, msg)
	if err != nil {
		return nil, err
	}

	// 执行本地事务
	if localTxErr := localTx(); localTxErr != nil {
		// 本地事务失败，回滚消息
		return result, localTxErr
	}

	return result, nil
}

// ─────────────────────────────────────────────────────────────────────────────
// 消费者实现
// ─────────────────────────────────────────────────────────────────────────────

// ConsumerConfig 消费者配置
type ConsumerConfig struct {
	NameServers       []string
	GroupName         string
	Topic             string
	Tag               string                // 订阅的 Tag，"*" 表示全部
	ConsumeMode       consumer.MessageModel // Clustering 或 BroadCasting
	ConsumeFromWhere  consumer.ConsumeFromWhere
	MaxReconsumeTimes int32         // 最大重试次数
	ConsumeTimeout    time.Duration // 消费超时
	PullBatchSize     int32         // 批量拉取数量
}

// OrderConsumer 订单消费者
type OrderConsumer struct {
	pushConsumer rocketmq.PushConsumer
	config       *ConsumerConfig
	handler      OrderHandler
	wg           sync.WaitGroup
}

// OrderHandler 订单处理函数
type OrderHandler func(ctx context.Context, order *OrderMessage) error

// NewOrderConsumer 创建订单消费者
func NewOrderConsumer(config *ConsumerConfig, handler OrderHandler) (*OrderConsumer, error) {
	c, err := rocketmq.NewPushConsumer(
		consumer.WithNameServer(config.NameServers),
		consumer.WithGroupName(config.GroupName),
		consumer.WithConsumeFromWhere(config.ConsumeFromWhere),
		consumer.WithConsumerModel(config.ConsumeMode),
		consumer.WithMaxReconsumeTimes(config.MaxReconsumeTimes),
		consumer.WithConsumeTimeout(config.ConsumeTimeout),
		consumer.WithPullBatchSize(config.PullBatchSize),
	)
	if err != nil {
		return nil, fmt.Errorf("create consumer: %w", err)
	}

	oc := &OrderConsumer{
		pushConsumer: c,
		config:       config,
		handler:      handler,
	}

	// 订阅 Topic
	selector := consumer.MessageSelector{
		Type:       consumer.TAG,
		Expression: config.Tag,
	}

	err = c.Subscribe(config.Topic, selector, oc.handleMessage)
	if err != nil {
		return nil, fmt.Errorf("subscribe topic: %w", err)
	}

	return oc, nil
}

// handleMessage 消息处理
func (c *OrderConsumer) handleMessage(ctx context.Context, msgs ...*primitive.MessageExt) (consumer.ConsumeResult, error) {
	for _, msg := range msgs {
		var order OrderMessage
		if err := json.Unmarshal(msg.Body, &order); err != nil {
			log.Printf("Unmarshal error: %v, msgId: %s", err, msg.MsgId)
			// 解析失败，不重试
			continue
		}

		if err := c.handler(ctx, &order); err != nil {
			log.Printf("Handle order %s error: %v, reconsume times: %d",
				order.OrderID, err, msg.ReconsumeTimes)

			// 返回重试
			return consumer.ConsumeRetryLater, nil
		}

		log.Printf("Consumed order %s successfully", order.OrderID)
	}

	return consumer.ConsumeSuccess, nil
}

// Start 启动消费者
func (c *OrderConsumer) Start() error {
	return c.pushConsumer.Start()
}

// Stop 停止消费者
func (c *OrderConsumer) Stop() error {
	return c.pushConsumer.Shutdown()
}

// ─────────────────────────────────────────────────────────────────────────────
// 顺序消费者实现
// ─────────────────────────────────────────────────────────────────────────────

// NewOrderlyConsumer 创建顺序消费者
func NewOrderlyConsumer(config *ConsumerConfig, handler OrderHandler) (*OrderConsumer, error) {
	c, err := rocketmq.NewPushConsumer(
		consumer.WithNameServer(config.NameServers),
		consumer.WithGroupName(config.GroupName),
		consumer.WithConsumeFromWhere(config.ConsumeFromWhere),
		consumer.WithConsumerModel(config.ConsumeMode),
		consumer.WithConsumeMessageBatchMaxSize(1), // 顺序消费建议设为1
		consumer.WithConsumerOrder(true),           // 开启顺序消费
	)
	if err != nil {
		return nil, fmt.Errorf("create orderly consumer: %w", err)
	}

	oc := &OrderConsumer{
		pushConsumer: c,
		config:       config,
		handler:      handler,
	}

	selector := consumer.MessageSelector{
		Type:       consumer.TAG,
		Expression: config.Tag,
	}

	err = c.Subscribe(config.Topic, selector, oc.handleOrderlyMessage)
	if err != nil {
		return nil, fmt.Errorf("subscribe topic: %w", err)
	}

	return oc, nil
}

// handleOrderlyMessage 顺序消息处理
func (c *OrderConsumer) handleOrderlyMessage(ctx context.Context, msgs ...*primitive.MessageExt) (consumer.ConsumeResult, error) {
	for _, msg := range msgs {
		var order OrderMessage
		if err := json.Unmarshal(msg.Body, &order); err != nil {
			log.Printf("Unmarshal error: %v", err)
			// 顺序消费解析失败也需要返回成功，否则会阻塞后续消息
			continue
		}

		// 顺序处理，失败会阻塞该队列
		if err := c.handler(ctx, &order); err != nil {
			log.Printf("Handle order %s error: %v, will retry", order.OrderID, err)
			// 返回 SuspendCurrentQueueAMoment 暂停当前队列消费
			return consumer.SuspendCurrentQueueAMoment, nil
		}
	}

	return consumer.ConsumeSuccess, nil
}

// ─────────────────────────────────────────────────────────────────────────────
// 使用示例
// ─────────────────────────────────────────────────────────────────────────────

func ExampleUsage() {
	ctx := context.Background()

	// 创建生产者
	producerConfig := &ProducerConfig{
		NameServers: []string{"127.0.0.1:9876"},
		GroupName:   "order-producer-group",
		RetryTimes:  3,
		SendTimeout: 3 * time.Second,
	}

	producer, err := NewOrderProducer(producerConfig)
	if err != nil {
		log.Fatal(err)
	}
	defer producer.Close()

	// 发送订单消息
	order := &OrderMessage{
		OrderID:   "ORD-20240101-001",
		UserID:    "USER-001",
		Symbol:    "BTC-USDT",
		Side:      "BUY",
		Price:     "50000.00",
		Quantity:  "0.1",
		Timestamp: time.Now().UnixNano(),
	}

	result, err := producer.SendSync(ctx, "ORDERS", order)
	if err != nil {
		log.Printf("Send error: %v", err)
	} else {
		log.Printf("Send success: msgId=%s, queue=%d", result.MsgID, result.MessageQueue.QueueId)
	}

	// 创建消费者
	consumerConfig := &ConsumerConfig{
		NameServers:       []string{"127.0.0.1:9876"},
		GroupName:         "order-consumer-group",
		Topic:             "ORDERS",
		Tag:               "*",
		ConsumeMode:       consumer.Clustering,
		ConsumeFromWhere:  consumer.ConsumeFromLastOffset,
		MaxReconsumeTimes: 3,
		ConsumeTimeout:    time.Minute,
		PullBatchSize:     32,
	}

	consumer, err := NewOrderConsumer(consumerConfig, func(ctx context.Context, order *OrderMessage) error {
		log.Printf("Processing order: %s", order.OrderID)
		// 处理订单逻辑
		return nil
	})
	if err != nil {
		log.Fatal(err)
	}

	if err := consumer.Start(); err != nil {
		log.Fatal(err)
	}
	defer consumer.Stop()

	// 等待信号...
	select {}
}
```

---

## 4. 常见交易系统架构

### 4.1 中心化交易所 (CEX) 架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     中心化交易所整体架构                                      │
└─────────────────────────────────────────────────────────────────────────────┘

                                   用户层
    ┌─────────────────────────────────────────────────────────────────────┐
    │   Web/App Client    │    API Client    │    WebSocket Client        │
    └──────────┬──────────┴────────┬─────────┴────────────┬───────────────┘
               │                   │                      │
               ▼                   ▼                      ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                         接入层 (Gateway)                             │
    │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
    │  │  API Gateway  │  │   WebSocket   │  │    Load Balancer      │   │
    │  │  (Kong/Nginx) │  │    Gateway    │  │    (HAProxy/LVS)      │   │
    │  └───────┬───────┘  └───────┬───────┘  └───────────┬───────────┘   │
    └──────────┼──────────────────┼──────────────────────┼───────────────┘
               │                  │                      │
               ▼                  ▼                      ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                         业务层 (Service)                             │
    │                                                                     │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐  │
    │  │ Order       │  │ Account     │  │ Market Data │  │ Risk      │  │
    │  │ Service     │  │ Service     │  │ Service     │  │ Service   │  │
    │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘  │
    │         │                │                │               │        │
    │         └────────────────┼────────────────┼───────────────┘        │
    │                          │                │                        │
    └──────────────────────────┼────────────────┼────────────────────────┘
                               │                │
                               ▼                ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                       消息中间件层                                    │
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐   │
    │  │                      Kafka Cluster                          │   │
    │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │   │
    │  │  │orders.submit│  │orders.match │  │ market.ticker       │  │   │
    │  │  │orders.cancel│  │orders.result│  │ market.depth        │  │   │
    │  │  │orders.update│  │trades.exec  │  │ market.trade        │  │   │
    │  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │   │
    │  └─────────────────────────────────────────────────────────────┘   │
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐   │
    │  │                    RabbitMQ Cluster                          │   │
    │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │   │
    │  │  │notification │  │  user.event │  │  system.alert       │  │   │
    │  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │   │
    │  └─────────────────────────────────────────────────────────────┘   │
    │                                                                     │
    └─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                         核心层 (Core)                                │
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐   │
    │  │                   Matching Engine (撮合引擎)                  │   │
    │  │                                                             │   │
    │  │   ┌───────────┐  ┌───────────┐  ┌───────────┐              │   │
    │  │   │ BTC/USDT  │  │ ETH/USDT  │  │ SOL/USDT  │  ...         │   │
    │  │   │  Engine   │  │  Engine   │  │  Engine   │              │   │
    │  │   └───────────┘  └───────────┘  └───────────┘              │   │
    │  │                                                             │   │
    │  │   每个交易对独立引擎，内存撮合，延迟 < 100μs                   │   │
    │  └─────────────────────────────────────────────────────────────┘   │
    │                                                                     │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐     │
    │  │ Clearing    │  │ Settlement  │  │    Asset Management     │     │
    │  │ (清算)      │  │ (结算)      │  │    (资产管理)            │     │
    │  └─────────────┘  └─────────────┘  └─────────────────────────┘     │
    │                                                                     │
    └─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                         存储层 (Storage)                             │
    │                                                                     │
    │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
    │  │    MySQL      │  │    Redis      │  │    ClickHouse         │   │
    │  │  (账户/订单)   │  │  (缓存/会话)   │  │    (行情历史)          │   │
    │  └───────────────┘  └───────────────┘  └───────────────────────┘   │
    │                                                                     │
    │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
    │  │  TiDB/CockroachDB │ │  Elasticsearch │ │     S3/OSS          │   │
    │  │  (分布式订单)   │  │  (日志检索)    │  │    (冷数据)          │   │
    │  └───────────────┘  └───────────────┘  └───────────────────────┘   │
    │                                                                     │
    └─────────────────────────────────────────────────────────────────────┘
```

#### 4.1.1 订单处理流程与消息流转

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         订单完整生命周期                                      │
└─────────────────────────────────────────────────────────────────────────────┘

用户下单                                                              用户收到通知
   │                                                                      ▲
   ▼                                                                      │
┌──────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│Client│───▶│API Gateway│───▶│Order Svc │───▶│  Kafka   │───▶│   Risk   │  │
└──────┘    └──────────┘    └────┬─────┘    │orders.new│    │  Check   │  │
                                 │          └──────────┘    └────┬─────┘  │
                                 │                               │        │
                          ┌──────┴──────┐                        │        │
                          │ 参数校验     │              ┌─────────┴────┐   │
                          │ 签名验证     │              │ 风控通过?    │   │
                          │ 频率限制     │              └──────┬───────┘   │
                          └─────────────┘                     │           │
                                                              ▼           │
                                                     ┌────────────────┐   │
                                                     │  Kafka         │   │
                                                     │ orders.checked │   │
                                                     └───────┬────────┘   │
                                                             │            │
                                                             ▼            │
┌─────────────────────────────────────────────────────────────────────────┤
│                        Matching Engine                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │   Order Book (BTC/USDT)                                         │   │
│  │   ┌─────────────────────┬─────────────────────┐                │   │
│  │   │      Bids (买单)     │      Asks (卖单)    │                │   │
│  │   ├─────────────────────┼─────────────────────┤                │   │
│  │   │ 50000.00 | 1.5 BTC  │ 50001.00 | 0.8 BTC │                │   │
│  │   │ 49999.00 | 2.0 BTC  │ 50002.00 | 1.2 BTC │                │   │
│  │   │ 49998.00 | 0.5 BTC  │ 50003.00 | 2.0 BTC │                │   │
│  │   │    ...              │     ...             │                │   │
│  │   └─────────────────────┴─────────────────────┘                │   │
│  │                                                                 │   │
│  │   撮合逻辑: Price-Time Priority                                  │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
           ┌───────────────┐         ┌───────────────┐
           │    成交       │         │    未成交      │
           └───────┬───────┘         └───────┬───────┘
                   │                         │
                   ▼                         ▼
           ┌───────────────┐         ┌───────────────┐
           │ Kafka         │         │ Order Book    │
           │ trades.exec   │         │ 挂单等待      │
           └───────┬───────┘         └───────────────┘
                   │
      ┌────────────┼────────────┐
      │            │            │
      ▼            ▼            ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│Clearing  │ │Account   │ │Market    │
│Service   │ │Service   │ │Data Svc  │
│(清算)    │ │(账户更新) │ │(行情推送) │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │
     ▼            ▼            ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│RabbitMQ  │ │  Redis   │ │WebSocket │
│settlement│ │  MySQL   │ │  Push    │───────────────────────────────────▶│
└──────────┘ └──────────┘ └──────────┘
```

#### 4.1.2 行情数据分发架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         行情数据流架构                                        │
└─────────────────────────────────────────────────────────────────────────────┘

数据源层
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │Matching Eng │  │External CEX │  │  DEX Data   │  │ Oracle Feed │        │
│  │ (内部撮合)   │  │(Binance等)  │  │ (链上数据)   │  │ (预言机)    │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │               │
└─────────┼────────────────┼────────────────┼────────────────┼───────────────┘
          │                │                │                │
          └────────────────┼────────────────┼────────────────┘
                           │                │
                           ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        数据聚合层                                            │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                   Market Data Aggregator                            │   │
│  │                                                                     │   │
│  │   • 数据清洗 (去重、校验、标准化)                                     │   │
│  │   • 价格聚合 (加权平均、中位数)                                       │   │
│  │   • 指数计算 (综合指数、波动率)                                       │   │
│  │   • K线生成 (1m, 5m, 15m, 1h, 4h, 1d)                               │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Kafka 消息层                                          │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐     │
│  │ market.ticker   │  │ market.depth    │  │ market.kline            │     │
│  │ (实时报价)       │  │ (深度数据)       │  │ (K线数据)               │     │
│  │                 │  │                 │  │                         │     │
│  │ 分区: 按交易对   │  │ 分区: 按交易对   │  │ 分区: 按交易对+周期      │     │
│  │ 保留: 1小时     │  │ 保留: 1小时     │  │ 保留: 7天               │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘     │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                                  │
│  │ market.trade    │  │ market.index    │                                  │
│  │ (逐笔成交)       │  │ (指数数据)       │                                  │
│  └─────────────────┘  └─────────────────┘                                  │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│ WebSocket Push  │      │  REST API       │      │   Storage       │
│    Service      │      │   Service       │      │   Service       │
│                 │      │                 │      │                 │
│ • 增量推送       │      │ • 快照查询       │      │ • ClickHouse    │
│ • 压缩传输       │      │ • 历史数据       │      │ • InfluxDB      │
│ • 心跳保活       │      │ • 批量接口       │      │ • Redis Cache   │
└────────┬────────┘      └─────────────────┘      └─────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          客户端分发                                          │
│                                                                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │  Web App  │  │Mobile App │  │  Trading  │  │   Bot     │               │
│  │           │  │           │  │   Bot     │  │  (做市商)  │               │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 去中心化交易所 (DEX) 架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     DEX 交易系统架构                                          │
└─────────────────────────────────────────────────────────────────────────────┘

                              用户交互层
    ┌─────────────────────────────────────────────────────────────────────┐
    │                                                                     │
    │   ┌───────────────┐     ┌───────────────┐     ┌───────────────┐    │
    │   │   Web dApp    │     │  Mobile dApp  │     │   API Client  │    │
    │   │ (React/Vue)   │     │  (RN/Flutter) │     │   (SDK)       │    │
    │   └───────┬───────┘     └───────┬───────┘     └───────┬───────┘    │
    │           │                     │                     │            │
    │           └─────────────────────┼─────────────────────┘            │
    │                                 │                                  │
    │                    ┌────────────┴────────────┐                     │
    │                    │   Wallet Connection     │                     │
    │                    │  (MetaMask/WalletConnect)│                    │
    │                    └────────────┬────────────┘                     │
    │                                 │                                  │
    └─────────────────────────────────┼──────────────────────────────────┘
                                      │
                                      ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                        链下服务层 (Off-chain)                        │
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐   │
    │  │                    Backend Services                          │   │
    │  │                                                             │   │
    │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐ │   │
    │  │  │  Price    │  │  Route    │  │ Analytics │  │ Gas Est. │ │   │
    │  │  │  Oracle   │  │  Finder   │  │  Service  │  │ Service  │ │   │
    │  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └────┬─────┘ │   │
    │  │        │              │              │             │       │   │
    │  └────────┼──────────────┼──────────────┼─────────────┼───────┘   │
    │           │              │              │             │           │
    │           ▼              ▼              ▼             ▼           │
    │  ┌─────────────────────────────────────────────────────────────┐   │
    │  │                    消息队列层                                │   │
    │  │                                                             │   │
    │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │   │
    │  │  │   Kafka     │  │   Redis     │  │    Event Stream     │ │   │
    │  │  │ • tx.pending│  │  Pub/Sub    │  │  • block.new        │ │   │
    │  │  │ • tx.confirm│  │ • price.upd │  │  • swap.executed    │ │   │
    │  │  │ • pool.sync │  │ • pool.upd  │  │  • liquidity.change │ │   │
    │  │  └─────────────┘  └─────────────┘  └─────────────────────┘ │   │
    │  │                                                             │   │
    │  └─────────────────────────────────────────────────────────────┘   │
    │                                                                     │
    └─────────────────────────────────┬───────────────────────────────────┘
                                      │
                                      ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                        链上层 (On-chain)                             │
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐   │
    │  │                  Smart Contracts                             │   │
    │  │                                                             │   │
    │  │  ┌─────────────────┐  ┌─────────────────┐                   │   │
    │  │  │   Router        │  │    Factory      │                   │   │
    │  │  │   Contract      │  │    Contract     │                   │   │
    │  │  │                 │  │                 │                   │   │
    │  │  │ • swap()        │  │ • createPair()  │                   │   │
    │  │  │ • addLiquidity()│  │ • getPair()     │                   │   │
    │  │  │ • removeLiq()   │  │ • allPairs()    │                   │   │
    │  │  └────────┬────────┘  └────────┬────────┘                   │   │
    │  │           │                    │                            │   │
    │  │           └─────────┬──────────┘                            │   │
    │  │                     │                                       │   │
    │  │                     ▼                                       │   │
    │  │  ┌──────────────────────────────────────────────────────┐  │   │
    │  │  │              Liquidity Pools                          │  │   │
    │  │  │                                                       │  │   │
    │  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │  │   │
    │  │  │  │ETH/USDC  │  │WBTC/ETH  │  │UNI/ETH   │   ...      │  │   │
    │  │  │  │Pool      │  │Pool      │  │Pool      │            │  │   │
    │  │  │  │          │  │          │  │          │            │  │   │
    │  │  │  │x*y=k     │  │x*y=k     │  │x*y=k     │            │  │   │
    │  │  │  └──────────┘  └──────────┘  └──────────┘            │  │   │
    │  │  │                                                       │  │   │
    │  │  └──────────────────────────────────────────────────────┘  │   │
    │  │                                                             │   │
    │  └─────────────────────────────────────────────────────────────┘   │
    │                                                                     │
    │  ┌─────────────────────────────────────────────────────────────┐   │
    │  │                    Blockchain Network                        │   │
    │  │  Ethereum / BSC / Polygon / Arbitrum / Optimism / ...       │   │
    │  └─────────────────────────────────────────────────────────────┘   │
    │                                                                     │
    └─────────────────────────────────────────────────────────────────────┘
```

#### 4.2.1 DEX 聚合器架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     DEX 聚合器 (如 1inch, Paraswap)                          │
└─────────────────────────────────────────────────────────────────────────────┘

用户请求: 用 1 ETH 换取最多 USDC
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         路由计算服务                                         │
│                                                                             │
│  输入: { fromToken: ETH, toToken: USDC, amount: 1 ETH }                    │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                     实时价格数据 (Kafka Consumer)                   │     │
│  │                                                                   │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │     │
│  │  │  Uniswap    │  │  Sushiswap  │  │   Curve     │    ...        │     │
│  │  │  V2: 3010   │  │  V1: 3008   │  │  3Pool:3012 │               │     │
│  │  │  V3: 3015   │  │             │  │             │               │     │
│  │  └─────────────┘  └─────────────┘  └─────────────┘               │     │
│  │                                                                   │     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                 │                                          │
│                                 ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                        路由算法                                    │     │
│  │                                                                   │     │
│  │   1. 单路径计算                                                    │     │
│  │      ETH -> USDC (Uniswap V3): 3015 USDC                         │     │
│  │      ETH -> USDC (Curve): 3012 USDC                              │     │
│  │                                                                   │     │
│  │   2. 多跳路径计算                                                  │     │
│  │      ETH -> WBTC -> USDC: 3018 USDC                              │     │
│  │      ETH -> DAI -> USDC: 3014 USDC                               │     │
│  │                                                                   │     │
│  │   3. 拆单计算 (Split)                                             │     │
│  │      50% Uniswap V3 + 50% Curve: 3016 USDC                       │     │
│  │      60% Uniswap V3 + 40% Sushi: 3013 USDC                       │     │
│  │                                                                   │     │
│  │   4. Gas 成本计算                                                 │     │
│  │      单路径 Gas: ~150,000                                         │     │
│  │      多跳 Gas: ~300,000                                           │     │
│  │      拆单 Gas: ~250,000                                           │     │
│  │                                                                   │     │
│  │   5. 最优解选择 (考虑 Gas 后净收益)                                │     │
│  │      最优: Uniswap V3 单路径, 净收益: 3010 USDC                   │     │
│  │                                                                   │     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │        返回最优路由           │
                    │                              │
                    │  {                           │
                    │    route: [Uniswap V3],      │
                    │    expectedOutput: 3015,     │
                    │    minOutput: 2985 (1% slip),│
                    │    gasEstimate: 150000,      │
                    │    calldata: "0x..."         │
                    │  }                           │
                    │                              │
                    └──────────────────────────────┘
```

### 4.3 量化交易系统架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     量化交易系统架构                                          │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          数据采集层                                          │
│                                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐ │
│  │ Exchange API  │  │ Blockchain    │  │  News Feed   │  │ Social Data │ │
│  │  Connectors   │  │  Indexer      │  │  (Reuters)   │  │ (Twitter)   │ │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘  └──────┬──────┘ │
│          │                  │                  │                 │        │
│          └──────────────────┴──────────────────┴─────────────────┘        │
│                                      │                                     │
└──────────────────────────────────────┼─────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       消息中间件层 (Kafka)                                   │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐ │
│  │ market.ticker   │  │ market.orderbook│  │ market.trades               │ │
│  │ (Tick 行情)      │  │ (订单簿快照)     │  │ (逐笔成交)                   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘ │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐ │
│  │ signal.alpha    │  │ order.request   │  │ order.execution             │ │
│  │ (Alpha 信号)     │  │ (订单请求)       │  │ (执行回报)                   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘ │
│                                                                             │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
          ▼                            ▼                            ▼
┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
│    策略引擎层        │    │    风控引擎层        │    │    执行引擎层        │
│                     │    │                     │    │                     │
│  ┌───────────────┐  │    │  ┌───────────────┐  │    │  ┌───────────────┐  │
│  │ Alpha 因子    │  │    │  │ 仓位检查      │  │    │  │ 订单路由      │  │
│  │ • 动量因子    │  │    │  │ • 最大持仓    │  │    │  │ • 交易所选择  │  │
│  │ • 均值回归    │  │    │  │ • 集中度限制  │  │    │  │ • 最优价格    │  │
│  │ • 套利因子    │  │    │  │ • 相关性控制  │  │    │  │ • 滑点控制    │  │
│  └───────────────┘  │    │  └───────────────┘  │    │  └───────────────┘  │
│                     │    │                     │    │                     │
│  ┌───────────────┐  │    │  ┌───────────────┐  │    │  ┌───────────────┐  │
│  │ 信号生成      │  │    │  │ 风险度量      │  │    │  │ 订单拆分      │  │
│  │ • ML 模型    │  │    │  │ • VaR         │  │    │  │ • TWAP        │  │
│  │ • 规则引擎   │  │    │  │ • 最大回撤    │  │    │  │ • VWAP        │  │
│  │ • 组合优化   │  │    │  │ • 夏普比率    │  │    │  │ • Iceberg     │  │
│  └───────────────┘  │    │  └───────────────┘  │    │  └───────────────┘  │
│                     │    │                     │    │                     │
│  ┌───────────────┐  │    │  ┌───────────────┐  │    │  ┌───────────────┐  │
│  │ 回测框架      │  │    │  │ 实时监控      │  │    │  │ 成交管理      │  │
│  │ • 历史回放    │  │    │  │ • P&L 监控    │  │    │  │ • 部分成交    │  │
│  │ • 滑点模拟    │  │    │  │ • 异常告警    │  │    │  │ • 撤单重发    │  │
│  │ • 绩效分析    │  │    │  │ • 熔断机制    │  │    │  │ • 状态同步    │  │
│  └───────────────┘  │    │  └───────────────┘  │    │  └───────────────┘  │
│                     │    │                     │    │                     │
└─────────────────────┘    └─────────────────────┘    └─────────────────────┘
          │                            │                            │
          └────────────────────────────┼────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          存储与分析层                                        │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐ │
│  │   TimescaleDB   │  │   ClickHouse    │  │        Redis               │ │
│  │  (时序数据)      │  │  (OLAP 分析)    │  │   (实时状态缓存)            │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘ │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐ │
│  │    PostgreSQL   │  │   Grafana       │  │       Jupyter              │ │
│  │  (交易记录)      │  │  (监控面板)      │  │   (策略研究)               │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 4.3.1 量化策略消息流示例

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     套利策略消息流                                           │
└─────────────────────────────────────────────────────────────────────────────┘

时间线 ─────────────────────────────────────────────────────────────────────▶

T0: 行情数据到达
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Kafka Topic: market.ticker                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ { exchange: "binance", symbol: "BTC/USDT", bid: 50000, ask: 50001 } │   │
│  │ { exchange: "okx",     symbol: "BTC/USDT", bid: 50010, ask: 50011 } │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  价差: OKX 买一 (50010) - Binance 卖一 (50001) = 9 USDT                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
T1: 策略计算 (延迟 < 1ms)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Alpha 信号生成:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 价差 9 USDT > 阈值 5 USDT                                           │   │
│  │ 预期利润: 9 - 手续费(2*0.1%) - 滑点(0.01%) ≈ 7 USDT                  │   │
│  │ 信号: ARBITRAGE_OPPORTUNITY                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  发送到 Kafka Topic: signal.alpha                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ {                                                                   │   │
│  │   signal_id: "arb-001",                                             │   │
│  │   strategy: "cross_exchange_arb",                                   │   │
│  │   symbol: "BTC/USDT",                                               │   │
│  │   legs: [                                                           │   │
│  │     { exchange: "binance", side: "buy",  price: 50001, qty: 0.1 },  │   │
│  │     { exchange: "okx",     side: "sell", price: 50010, qty: 0.1 }   │   │
│  │   ],                                                                │   │
│  │   expected_profit: 7,                                               │   │
│  │   timestamp: 1704067200000                                          │   │
│  │ }                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
T2: 风控检查 (延迟 < 0.5ms)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  风控规则检查:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ✓ 持仓限制: 当前 BTC 持仓 0.5, 限制 10, 新增 0.1 → 通过              │   │
│  │ ✓ 单笔限额: 0.1 BTC ≈ 5000 USDT < 限制 10000 USDT → 通过            │   │
│  │ ✓ 频率限制: 最近1分钟交易 5 次 < 限制 100 次 → 通过                  │   │
│  │ ✓ 价格偏离: 50000 在 ±5% 范围内 → 通过                               │   │
│  │                                                                     │   │
│  │ 风控结果: APPROVED                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
T3: 订单生成与执行 (延迟 < 1ms)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  发送到 Kafka Topic: order.request                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Order 1 (Binance Buy):                                              │   │
│  │ { order_id: "ord-001", exchange: "binance", symbol: "BTCUSDT",      │   │
│  │   side: "buy", type: "limit", price: "50001", quantity: "0.1" }     │   │
│  │                                                                     │   │
│  │ Order 2 (OKX Sell):                                                 │   │
│  │ { order_id: "ord-002", exchange: "okx", symbol: "BTC-USDT",         │   │
│  │   side: "sell", type: "limit", price: "50010", quantity: "0.1" }    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  执行引擎: 同时向两个交易所发送订单 (并行执行)                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
T4: 执行回报 (延迟取决于交易所)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Kafka Topic: order.execution                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ { order_id: "ord-001", status: "filled", filled_qty: "0.1",         │   │
│  │   filled_price: "50001", fee: "0.05", timestamp: ... }              │   │
│  │                                                                     │   │
│  │ { order_id: "ord-002", status: "filled", filled_qty: "0.1",         │   │
│  │   filled_price: "50010", fee: "0.05", timestamp: ... }              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  实际利润计算:                                                              │
│  收入: 50010 * 0.1 = 5001 USDT                                             │
│  支出: 50001 * 0.1 = 5000.1 USDT                                           │
│  手续费: 0.05 + 0.05 = 0.1 USDT                                            │
│  净利润: 5001 - 5000.1 - 0.1 = 0.8 USDT ✓                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 中间件在交易系统中的应用场景

### 5.1 场景一：订单流处理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         订单流处理场景                                        │
└─────────────────────────────────────────────────────────────────────────────┘

问题场景:
• 开盘/收盘时订单量激增 (10x-100x 平时流量)
• 订单处理需要顺序保证 (同一用户/交易对)
• 撮合引擎处理能力有限
• 需要持久化保证订单不丢失

解决方案: Kafka 作为订单缓冲队列

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   高峰期流量                                                                 │
│       │                                                                     │
│       │  100,000 orders/sec                                                │
│       ▼                                                                     │
│  ┌─────────────┐                                                           │
│  │ API Gateway │                                                           │
│  └──────┬──────┘                                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Kafka: orders.submit                              │   │
│  │                                                                     │   │
│  │   Partition 策略: hash(user_id + symbol) % partition_count          │   │
│  │   → 同一用户同一交易对的订单进入同一分区，保证顺序                      │   │
│  │                                                                     │   │
│  │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │   │
│  │   │  P0     │ │  P1     │ │  P2     │ │  P3     │ │  P4     │ ...  │   │
│  │   │ User A  │ │ User B  │ │ User C  │ │ User D  │ │ User E  │      │   │
│  │   │ BTC/USD │ │ ETH/USD │ │ BTC/USD │ │ SOL/USD │ │ ETH/USD │      │   │
│  │   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘      │   │
│  │        │          │          │          │          │              │   │
│  └────────┼──────────┼──────────┼──────────┼──────────┼──────────────┘   │
│           │          │          │          │          │                  │
│           ▼          ▼          ▼          ▼          ▼                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                   Order Processing Consumers                         │   │
│  │                                                                     │   │
│  │   每个 Consumer 按顺序处理分配到的分区                                 │   │
│  │   处理速度: 10,000 orders/sec per consumer                          │   │
│  │   Consumer 数量可动态扩展                                             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────┐      10,000 orders/sec (稳定输出)                         │
│  │  Matching   │                                                           │
│  │   Engine    │                                                           │
│  └─────────────┘                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

配置建议:
┌────────────────────────┬─────────────────────────────────────────────────────┐
│ 参数                    │ 推荐值                                              │
├────────────────────────┼─────────────────────────────────────────────────────┤
│ 分区数                  │ 撮合引擎并行度的 2-3 倍 (如 32-64)                   │
│ 副本因子                │ 3 (保证高可用)                                       │
│ acks                   │ all (保证不丢失)                                     │
│ min.insync.replicas    │ 2                                                   │
│ 消息保留时间            │ 7 天 (用于审计和回溯)                                │
│ 消息压缩                │ lz4 (平衡压缩率和速度)                               │
└────────────────────────┴─────────────────────────────────────────────────────┘
```

### 5.2 场景二：行情数据分发

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         行情分发场景                                          │
└─────────────────────────────────────────────────────────────────────────────┘

问题场景:
• 行情数据量大 (每秒数十万条 tick)
• 多种消费者有不同需求 (全量/增量/聚合)
• 延迟敏感 (毫秒级要求)
• 可以容忍少量数据丢失

解决方案: Kafka + Redis Pub/Sub 组合

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   行情源                                                                     │
│      │                                                                      │
│      ▼                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                      Kafka: market.raw                                │ │
│  │                                                                       │ │
│  │   原始行情数据，高吞吐，用于存储和回放                                   │ │
│  │   配置: acks=1, compression=lz4, retention=24h                        │ │
│  │                                                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│         │                                                                   │
│         │                                                                   │
│    ┌────┴────┬─────────────┬─────────────┐                                 │
│    │         │             │             │                                 │
│    ▼         ▼             ▼             ▼                                 │
│ ┌──────┐ ┌──────┐    ┌──────────┐  ┌──────────┐                           │
│ │Ticker│ │Depth │    │  K-Line  │  │  Index   │                           │
│ │Agg   │ │Agg   │    │Generator │  │Calculator│                           │
│ └──┬───┘ └──┬───┘    └────┬─────┘  └────┬─────┘                           │
│    │        │             │             │                                  │
│    ▼        ▼             ▼             ▼                                  │
│ ┌────────────────────────────────────────────────────────────────────────┐│
│ │                        Redis Pub/Sub                                   ││
│ │                                                                        ││
│ │  实时推送，最低延迟，允许丢失                                            ││
│ │                                                                        ││
│ │  Channels:                                                             ││
│ │  • ticker:{symbol}     → 实时价格                                       ││
│ │  • depth:{symbol}      → 深度更新 (增量)                                ││
│ │  • kline:{symbol}:{tf} → K线更新                                        ││
│ │  • index:{name}        → 指数更新                                       ││
│ │                                                                        ││
│ └────────────────────────────────────────────────────────────────────────┘│
│         │                                                                  │
│    ┌────┴────┬─────────────┬─────────────┐                                │
│    │         │             │             │                                │
│    ▼         ▼             ▼             ▼                                │
│ ┌──────┐ ┌──────┐    ┌──────────┐  ┌──────────┐                          │
│ │WebSocket│ │Mobile│  │ Trading  │  │  做市   │                          │
│ │ Server │ │ Push │  │   Bot    │  │  系统   │                          │
│ └────────┘ └──────┘  └──────────┘  └──────────┘                          │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘

延迟对比:
┌────────────────────────┬──────────────────┬──────────────────┐
│ 方案                    │ 平均延迟          │ P99 延迟          │
├────────────────────────┼──────────────────┼──────────────────┤
│ 纯 Kafka               │ 5-10ms           │ 50ms             │
│ Redis Pub/Sub          │ 0.5-1ms          │ 5ms              │
│ Kafka + Redis 组合     │ 1-2ms            │ 10ms             │
└────────────────────────┴──────────────────┴──────────────────┘
```

### 5.3 场景三：风控事件处理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         风控事件处理场景                                      │
└─────────────────────────────────────────────────────────────────────────────┘

问题场景:
• 风控检查必须在订单执行前完成
• 不同风控规则有不同优先级
• 需要支持实时规则更新
• 审计要求所有风控事件可追溯

解决方案: RabbitMQ 优先级队列 + Kafka 审计日志

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   交易事件                                                                   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    RabbitMQ Priority Queue                           │   │
│  │                                                                     │   │
│  │   Exchange: risk.events (Topic)                                     │   │
│  │                                                                     │   │
│  │   Queue: risk.critical (priority: 10)                               │   │
│  │   • 大额交易 (> $100,000)                                            │   │
│  │   • 异常价格 (偏离 > 5%)                                              │   │
│  │   • 黑名单检查                                                       │   │
│  │                                                                     │   │
│  │   Queue: risk.high (priority: 5)                                    │   │
│  │   • 频率检查                                                         │   │
│  │   • 仓位检查                                                         │   │
│  │   • 资金检查                                                         │   │
│  │                                                                     │   │
│  │   Queue: risk.normal (priority: 1)                                  │   │
│  │   • 常规风控                                                         │   │
│  │   • 统计类检查                                                       │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Risk Engine Cluster                             │   │
│  │                                                                     │   │
│  │   ┌─────────────────────────────────────────────────────────────┐  │   │
│  │   │                   Rule Engine (Drools/Aviator)              │  │   │
│  │   │                                                             │  │   │
│  │   │   规则示例:                                                  │  │   │
│  │   │   • IF amount > 100000 AND user.level < 3 THEN REJECT      │  │   │
│  │   │   • IF freq_1min > 100 THEN THROTTLE                       │  │   │
│  │   │   • IF price_deviation > 0.05 THEN ALERT                   │  │   │
│  │   │                                                             │  │   │
│  │   └─────────────────────────────────────────────────────────────┘  │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│               ┌──────────────────┼──────────────────┐                      │
│               │                  │                  │                      │
│               ▼                  ▼                  ▼                      │
│        ┌───────────┐      ┌───────────┐      ┌───────────┐                │
│        │  PASSED   │      │ REJECTED  │      │  PENDING  │                │
│        └─────┬─────┘      └─────┬─────┘      └─────┬─────┘                │
│              │                  │                  │                       │
│              ▼                  ▼                  ▼                       │
│        ┌───────────┐      ┌───────────┐      ┌───────────┐                │
│        │ Continue  │      │  Notify   │      │  Manual   │                │
│        │ to Match  │      │   User    │      │  Review   │                │
│        └───────────┘      └───────────┘      └───────────┘                │
│                                                                            │
│   同时记录到 Kafka (审计日志):                                               │
│   ┌────────────────────────────────────────────────────────────────────┐  │
│   │  Kafka: risk.audit.log                                             │  │
│   │  { event_id, timestamp, user_id, order_id, rules_checked,          │  │
│   │    result, reason, latency_ms }                                    │  │
│   └────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 场景四：交易通知与推送

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         通知推送场景                                          │
└─────────────────────────────────────────────────────────────────────────────┘

问题场景:
• 多种通知渠道 (App Push, SMS, Email, WebSocket)
• 不同优先级 (成交通知 > 营销通知)
• 需要去重和频率控制
• 失败需要重试

解决方案: RabbitMQ 多队列 + 死信队列

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   通知事件源                                                                 │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                       │
│   │ 订单成交 │  │ 价格预警 │  │ 充提完成 │  │ 系统公告 │                       │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘                       │
│        │           │           │           │                               │
│        └───────────┴───────────┴───────────┘                               │
│                          │                                                  │
│                          ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    RabbitMQ Exchange                                 │   │
│  │                                                                     │   │
│  │   Exchange: notification (Headers)                                  │   │
│  │                                                                     │   │
│  │   路由规则:                                                          │   │
│  │   • x-priority: critical → queue.critical                          │   │
│  │   • x-priority: high     → queue.high                              │   │
│  │   • x-priority: normal   → queue.normal                            │   │
│  │   • x-channel: sms       → queue.sms                               │   │
│  │   • x-channel: email     → queue.email                             │   │
│  │   • x-channel: push      → queue.push                              │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                          │                                                  │
│        ┌─────────────────┼─────────────────┐                               │
│        │                 │                 │                               │
│        ▼                 ▼                 ▼                               │
│  ┌───────────┐     ┌───────────┐     ┌───────────┐                        │
│  │queue.push │     │queue.sms  │     │queue.email│                        │
│  │           │     │           │     │           │                        │
│  │ prefetch  │     │ prefetch  │     │ prefetch  │                        │
│  │ = 100     │     │ = 10      │     │ = 50      │                        │
│  │           │     │ (限流)     │     │           │                        │
│  └─────┬─────┘     └─────┬─────┘     └─────┬─────┘                        │
│        │                 │                 │                               │
│        ▼                 ▼                 ▼                               │
│  ┌───────────┐     ┌───────────┐     ┌───────────┐                        │
│  │Push Worker│     │SMS Worker │     │Email Worker                        │
│  │  (FCM/    │     │ (Twilio/  │     │ (SendGrid/│                        │
│  │   APNs)   │     │  阿里云)   │     │   SES)    │                        │
│  └─────┬─────┘     └─────┬─────┘     └─────┬─────┘                        │
│        │                 │                 │                               │
│        │    失败重试      │    失败重试      │    失败重试                   │
│        ▼                 ▼                 ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Dead Letter Queue                               │   │
│  │                                                                     │   │
│  │   queue.notification.dlq                                            │   │
│  │   • 3次重试后进入                                                    │   │
│  │   • 人工处理或定时重试                                               │   │
│  │   • 保留7天                                                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

通知消息结构:
{
  "notification_id": "notif-001",
  "user_id": "user-123",
  "type": "ORDER_FILLED",
  "priority": "critical",
  "channels": ["push", "sms"],
  "data": {
    "order_id": "ord-456",
    "symbol": "BTC/USDT",
    "side": "BUY",
    "price": "50000",
    "quantity": "0.1"
  },
  "template": "order_filled_v1",
  "language": "zh-CN",
  "created_at": "2024-01-01T00:00:00Z",
  "expire_at": "2024-01-01T00:05:00Z"  // 5分钟后过期
}
```

### 5.5 场景五：事件溯源与审计

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         事件溯源场景                                          │
└─────────────────────────────────────────────────────────────────────────────┘

问题场景:
• 合规要求所有交易行为可追溯
• 需要支持任意时间点状态重建
• 海量数据长期存储
• 支持实时和离线分析

解决方案: Kafka 作为事件存储 + 流处理

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                          Event Sourcing 架构                                │
│                                                                             │
│   业务操作                                                                   │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                       │
│   │ 下单    │  │ 撤单    │  │  成交   │  │ 结算    │                       │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘                       │
│        │           │           │           │                               │
│        └───────────┴───────────┴───────────┘                               │
│                          │                                                  │
│                    转换为事件                                                │
│                          │                                                  │
│                          ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Kafka Event Store                                 │   │
│  │                                                                     │   │
│  │   Topic: trading.events                                             │   │
│  │   分区: 按 aggregate_id (如 order_id) 哈希                           │   │
│  │   保留: 永久 (或根据合规要求)                                         │   │
│  │   压缩: 关闭 (保留所有历史事件)                                       │   │
│  │                                                                     │   │
│  │   事件格式:                                                          │   │
│  │   {                                                                 │   │
│  │     "event_id": "evt-001",                                          │   │
│  │     "event_type": "OrderCreated",                                   │   │
│  │     "aggregate_type": "Order",                                      │   │
│  │     "aggregate_id": "ord-123",                                      │   │
│  │     "version": 1,                                                   │   │
│  │     "timestamp": "2024-01-01T00:00:00.000Z",                        │   │
│  │     "data": { ... },                                                │   │
│  │     "metadata": {                                                   │   │
│  │       "user_id": "user-456",                                        │   │
│  │       "ip": "1.2.3.4",                                              │   │
│  │       "trace_id": "trace-789"                                       │   │
│  │     }                                                               │   │
│  │   }                                                                 │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│         ┌────────────────────────┼────────────────────────────┐            │
│         │                        │                            │            │
│         ▼                        ▼                            ▼            │
│  ┌─────────────┐         ┌─────────────┐             ┌─────────────┐      │
│  │  实时投影    │         │  流式处理    │             │  批量归档    │      │
│  │  (CQRS)     │         │  (Flink)    │             │  (Spark)    │      │
│  │             │         │             │             │             │      │
│  │ 更新读模型   │         │ 实时聚合    │             │ 冷数据存储   │      │
│  │ Redis/PG   │         │ 风控计算    │             │ S3/HDFS    │      │
│  └─────────────┘         └─────────────┘             └─────────────┘      │
│                                                                            │
│                                                                            │
│   状态重建示例 (订单生命周期):                                               │
│   ┌────────────────────────────────────────────────────────────────────┐  │
│   │                                                                    │  │
│   │   Event 1: OrderCreated     → 状态: CREATED                        │  │
│   │   Event 2: OrderValidated   → 状态: VALIDATED                      │  │
│   │   Event 3: OrderMatched     → 状态: PARTIALLY_FILLED               │  │
│   │   Event 4: OrderMatched     → 状态: FILLED                         │  │
│   │   Event 5: OrderSettled     → 状态: SETTLED                        │  │
│   │                                                                    │  │
│   │   任意时间点重建:                                                    │  │
│   │   replay(events, until=Event3) → 状态: PARTIALLY_FILLED            │  │
│   │                                                                    │  │
│   └────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.6 中间件选型决策矩阵

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         中间件选型决策矩阵                                    │
└─────────────────────────────────────────────────────────────────────────────┘

根据场景特点选择合适的中间件:

┌────────────────────┬────────────┬────────────┬────────────┬────────────────┐
│      场景          │   Kafka    │  RabbitMQ  │  RocketMQ  │  Redis Pub/Sub │
├────────────────────┼────────────┼────────────┼────────────┼────────────────┤
│ 订单流处理         │    ★★★     │    ★★      │    ★★★     │       ★        │
│ 行情数据分发       │    ★★★     │     ★      │    ★★      │      ★★★       │
│ 风控事件处理       │    ★★      │    ★★★     │    ★★★     │       ★        │
│ 交易通知推送       │     ★      │    ★★★     │    ★★      │      ★★        │
│ 事件溯源审计       │    ★★★     │     ★      │    ★★      │       -        │
│ 系统内部通信       │    ★★      │    ★★      │    ★★      │      ★★★       │
└────────────────────┴────────────┴────────────┴────────────┴────────────────┘

★★★ = 最佳选择  ★★ = 适合  ★ = 可用但非最优  - = 不适合

推荐组合:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   交易系统推荐中间件组合:                                                    │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   Kafka (核心数据流)                                                 │  │
│   │   ├── 订单流: orders.submit, orders.result                          │  │
│   │   ├── 行情流: market.ticker, market.depth, market.trade            │  │
│   │   ├── 交易流: trades.execution, trades.settlement                   │  │
│   │   └── 审计流: audit.orders, audit.trades, audit.risk               │  │
│   │                                                                     │  │
│   │   RabbitMQ (业务消息)                                                │  │
│   │   ├── 通知: notification.push, notification.sms, notification.email│  │
│   │   ├── 风控: risk.check, risk.alert                                 │  │
│   │   └── 任务: task.report, task.settlement                           │  │
│   │                                                                     │  │
│   │   Redis Pub/Sub (实时推送)                                           │  │
│   │   ├── 行情推送: ticker:*, depth:*, trade:*                          │  │
│   │   ├── 用户推送: user:{id}:orders, user:{id}:balance                 │  │
│   │   └── 系统广播: system:maintenance, system:config                   │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 高性能交易系统中间件实践

### 6.1 性能优化策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         性能优化策略                                          │
└─────────────────────────────────────────────────────────────────────────────┘

1. 批量处理优化
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   单条发送 vs 批量发送:                                                      │
│                                                                             │
│   // 低效: 单条发送                                                         │
│   for msg := range messages {                                              │
│       producer.Send(msg)  // 每次网络往返                                   │
│   }                                                                        │
│   // 1000条消息 = 1000次网络往返 ≈ 1000ms                                   │
│                                                                             │
│   // 高效: 批量发送                                                         │
│   batch := make([]Message, 0, batchSize)                                   │
│   for msg := range messages {                                              │
│       batch = append(batch, msg)                                           │
│       if len(batch) >= batchSize {                                         │
│           producer.SendBatch(batch)                                        │
│           batch = batch[:0]                                                │
│       }                                                                    │
│   }                                                                        │
│   // 1000条消息 = 10次网络往返 (batch=100) ≈ 100ms                          │
│                                                                             │
│   性能提升: 10倍                                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

2. 压缩策略
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   压缩算法对比:                                                              │
│                                                                             │
│   ┌────────────┬──────────────┬──────────────┬──────────────────────────┐  │
│   │   算法      │   压缩率     │   CPU 占用   │   适用场景               │  │
│   ├────────────┼──────────────┼──────────────┼──────────────────────────┤  │
│   │   None     │     1.0x     │     最低     │   低延迟优先             │  │
│   │   LZ4      │   2.0-3.0x   │     低       │   推荐 (平衡)            │  │
│   │   Snappy   │   2.0-2.5x   │     低       │   Google 系统常用         │  │
│   │   ZSTD     │   3.0-4.0x   │     中       │   存储优先               │  │
│   │   GZIP     │   3.5-4.5x   │     高       │   归档场景               │  │
│   └────────────┴──────────────┴──────────────┴──────────────────────────┘  │
│                                                                             │
│   行情数据推荐: LZ4 (延迟敏感)                                               │
│   日志数据推荐: ZSTD (存储敏感)                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

3. 分区策略
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   分区数计算公式:                                                            │
│                                                                             │
│   目标吞吐量 (MB/s)                                                         │
│   ─────────────────────  =  最小分区数                                      │
│   单分区吞吐量 (MB/s)                                                       │
│                                                                             │
│   示例:                                                                     │
│   目标: 100 MB/s                                                           │
│   单分区: 10 MB/s (SSD)                                                    │
│   最小分区数: 100 / 10 = 10                                                │
│   推荐分区数: 10 * 2 = 20 (预留扩展空间)                                    │
│                                                                             │
│   分区键选择:                                                               │
│   ┌────────────────────┬──────────────────────────────────────────────┐   │
│   │      场景          │           分区键                              │   │
│   ├────────────────────┼──────────────────────────────────────────────┤   │
│   │   订单处理         │   user_id + symbol (保证用户订单顺序)          │   │
│   │   行情数据         │   symbol (同交易对数据集中)                    │   │
│   │   用户事件         │   user_id (同用户事件顺序)                     │   │
│   │   系统日志         │   random/round-robin (负载均衡)               │   │
│   └────────────────────┴──────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Go 实现：高性能消息处理框架

```go
package middleware

import (
	"context"
	"sync"
	"sync/atomic"
	"time"
)

// ─────────────────────────────────────────────────────────────────────────────
// 高性能批量处理器
// ─────────────────────────────────────────────────────────────────────────────

// BatchProcessor 批量处理器
type BatchProcessor[T any] struct {
	batchSize   int
	flushPeriod time.Duration
	handler     func([]T) error

	buffer    []T
	bufferMu  sync.Mutex
	flushChan chan struct{}

	metrics *BatchMetrics
	wg      sync.WaitGroup
	ctx     context.Context
	cancel  context.CancelFunc
}

// BatchMetrics 批量处理指标
type BatchMetrics struct {
	TotalMessages int64
	TotalBatches  int64
	FailedBatches int64
	AvgBatchSize  float64
	AvgLatencyMs  float64
}

// NewBatchProcessor 创建批量处理器
func NewBatchProcessor[T any](
	batchSize int,
	flushPeriod time.Duration,
	handler func([]T) error,
) *BatchProcessor[T] {
	ctx, cancel := context.WithCancel(context.Background())

	bp := &BatchProcessor[T]{
		batchSize:   batchSize,
		flushPeriod: flushPeriod,
		handler:     handler,
		buffer:      make([]T, 0, batchSize),
		flushChan:   make(chan struct{}, 1),
		metrics:     &BatchMetrics{},
		ctx:         ctx,
		cancel:      cancel,
	}

	bp.wg.Add(1)
	go bp.flushLoop()

	return bp
}

// Add 添加消息到缓冲区
func (bp *BatchProcessor[T]) Add(item T) {
	bp.bufferMu.Lock()
	bp.buffer = append(bp.buffer, item)
	shouldFlush := len(bp.buffer) >= bp.batchSize
	bp.bufferMu.Unlock()

	if shouldFlush {
		select {
		case bp.flushChan <- struct{}{}:
		default:
		}
	}
}

// flushLoop 定时刷新循环
func (bp *BatchProcessor[T]) flushLoop() {
	defer bp.wg.Done()

	ticker := time.NewTicker(bp.flushPeriod)
	defer ticker.Stop()

	for {
		select {
		case <-bp.ctx.Done():
			bp.flush()
			return
		case <-bp.flushChan:
			bp.flush()
		case <-ticker.C:
			bp.flush()
		}
	}
}

// flush 执行刷新
func (bp *BatchProcessor[T]) flush() {
	bp.bufferMu.Lock()
	if len(bp.buffer) == 0 {
		bp.bufferMu.Unlock()
		return
	}

	batch := bp.buffer
	bp.buffer = make([]T, 0, bp.batchSize)
	bp.bufferMu.Unlock()

	start := time.Now()
	err := bp.handler(batch)
	latency := time.Since(start).Milliseconds()

	// 更新指标
	atomic.AddInt64(&bp.metrics.TotalMessages, int64(len(batch)))
	atomic.AddInt64(&bp.metrics.TotalBatches, 1)
	if err != nil {
		atomic.AddInt64(&bp.metrics.FailedBatches, 1)
	}

	// 更新平均延迟 (简化版)
	bp.metrics.AvgLatencyMs = float64(latency)
}

// Close 关闭处理器
func (bp *BatchProcessor[T]) Close() {
	bp.cancel()
	bp.wg.Wait()
}

// Metrics 获取指标
func (bp *BatchProcessor[T]) Metrics() BatchMetrics {
	return BatchMetrics{
		TotalMessages: atomic.LoadInt64(&bp.metrics.TotalMessages),
		TotalBatches:  atomic.LoadInt64(&bp.metrics.TotalBatches),
		FailedBatches: atomic.LoadInt64(&bp.metrics.FailedBatches),
		AvgLatencyMs:  bp.metrics.AvgLatencyMs,
	}
}

// ─────────────────────────────────────────────────────────────────────────────
// 消息路由器
// ─────────────────────────────────────────────────────────────────────────────

// MessageRouter 消息路由器
type MessageRouter[T any] struct {
	routes   map[string][]MessageHandler[T]
	routesMu sync.RWMutex

	defaultHandler MessageHandler[T]

	// 中间件
	middlewares []Middleware[T]
}

// MessageHandler 消息处理器
type MessageHandler[T any] func(ctx context.Context, msg T) error

// Middleware 中间件
type Middleware[T any] func(next MessageHandler[T]) MessageHandler[T]

// NewMessageRouter 创建消息路由器
func NewMessageRouter[T any]() *MessageRouter[T] {
	return &MessageRouter[T]{
		routes: make(map[string][]MessageHandler[T]),
	}
}

// Use 添加中间件
func (r *MessageRouter[T]) Use(mw Middleware[T]) {
	r.middlewares = append(r.middlewares, mw)
}

// Register 注册路由
func (r *MessageRouter[T]) Register(topic string, handler MessageHandler[T]) {
	r.routesMu.Lock()
	defer r.routesMu.Unlock()

	// 应用中间件
	for i := len(r.middlewares) - 1; i >= 0; i-- {
		handler = r.middlewares[i](handler)
	}

	r.routes[topic] = append(r.routes[topic], handler)
}

// SetDefault 设置默认处理器
func (r *MessageRouter[T]) SetDefault(handler MessageHandler[T]) {
	r.defaultHandler = handler
}

// Route 路由消息
func (r *MessageRouter[T]) Route(ctx context.Context, topic string, msg T) error {
	r.routesMu.RLock()
	handlers, ok := r.routes[topic]
	r.routesMu.RUnlock()

	if !ok || len(handlers) == 0 {
		if r.defaultHandler != nil {
			return r.defaultHandler(ctx, msg)
		}
		return nil
	}

	for _, handler := range handlers {
		if err := handler(ctx, msg); err != nil {
			return err
		}
	}

	return nil
}

// ─────────────────────────────────────────────────────────────────────────────
// 常用中间件
// ─────────────────────────────────────────────────────────────────────────────

// LoggingMiddleware 日志中间件
func LoggingMiddleware[T any](logger Logger) Middleware[T] {
	return func(next MessageHandler[T]) MessageHandler[T] {
		return func(ctx context.Context, msg T) error {
			start := time.Now()
			err := next(ctx, msg)
			logger.Info("message processed",
				"duration", time.Since(start),
				"error", err,
			)
			return err
		}
	}
}

// RetryMiddleware 重试中间件
func RetryMiddleware[T any](maxRetries int, backoff time.Duration) Middleware[T] {
	return func(next MessageHandler[T]) MessageHandler[T] {
		return func(ctx context.Context, msg T) error {
			var err error
			for i := 0; i <= maxRetries; i++ {
				err = next(ctx, msg)
				if err == nil {
					return nil
				}

				if i < maxRetries {
					select {
					case <-ctx.Done():
						return ctx.Err()
					case <-time.After(backoff * time.Duration(i+1)):
					}
				}
			}
			return err
		}
	}
}

// TimeoutMiddleware 超时中间件
func TimeoutMiddleware[T any](timeout time.Duration) Middleware[T] {
	return func(next MessageHandler[T]) MessageHandler[T] {
		return func(ctx context.Context, msg T) error {
			ctx, cancel := context.WithTimeout(ctx, timeout)
			defer cancel()

			done := make(chan error, 1)
			go func() {
				done <- next(ctx, msg)
			}()

			select {
			case err := <-done:
				return err
			case <-ctx.Done():
				return ctx.Err()
			}
		}
	}
}

// MetricsMiddleware 指标中间件
func MetricsMiddleware[T any](metrics *ProcessingMetrics) Middleware[T] {
	return func(next MessageHandler[T]) MessageHandler[T] {
		return func(ctx context.Context, msg T) error {
			start := time.Now()
			err := next(ctx, msg)

			atomic.AddInt64(&metrics.TotalProcessed, 1)
			if err != nil {
				atomic.AddInt64(&metrics.TotalFailed, 1)
			}

			latency := time.Since(start).Microseconds()
			atomic.StoreInt64(&metrics.LastLatencyUs, latency)

			return err
		}
	}
}

// ProcessingMetrics 处理指标
type ProcessingMetrics struct {
	TotalProcessed int64
	TotalFailed    int64
	LastLatencyUs  int64
}

// Logger 日志接口
type Logger interface {
	Info(msg string, keysAndValues ...interface{})
	Error(msg string, keysAndValues ...interface{})
}

// ─────────────────────────────────────────────────────────────────────────────
// 消费者池
// ─────────────────────────────────────────────────────────────────────────────

// ConsumerPool 消费者池
type ConsumerPool[T any] struct {
	size    int
	handler MessageHandler[T]
	msgChan chan T
	wg      sync.WaitGroup
	ctx     context.Context
	cancel  context.CancelFunc
}

// NewConsumerPool 创建消费者池
func NewConsumerPool[T any](size int, bufferSize int, handler MessageHandler[T]) *ConsumerPool[T] {
	ctx, cancel := context.WithCancel(context.Background())

	pool := &ConsumerPool[T]{
		size:    size,
		handler: handler,
		msgChan: make(chan T, bufferSize),
		ctx:     ctx,
		cancel:  cancel,
	}

	return pool
}

// Start 启动消费者池
func (p *ConsumerPool[T]) Start() {
	for i := 0; i < p.size; i++ {
		p.wg.Add(1)
		go p.worker(i)
	}
}

// worker 工作协程
func (p *ConsumerPool[T]) worker(id int) {
	defer p.wg.Done()

	for {
		select {
		case <-p.ctx.Done():
			return
		case msg, ok := <-p.msgChan:
			if !ok {
				return
			}
			_ = p.handler(p.ctx, msg)
		}
	}
}

// Submit 提交消息
func (p *ConsumerPool[T]) Submit(msg T) bool {
	select {
	case p.msgChan <- msg:
		return true
	default:
		return false
	}
}

// SubmitBlocking 阻塞提交消息
func (p *ConsumerPool[T]) SubmitBlocking(ctx context.Context, msg T) error {
	select {
	case p.msgChan <- msg:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

// Stop 停止消费者池
func (p *ConsumerPool[T]) Stop() {
	p.cancel()
	close(p.msgChan)
	p.wg.Wait()
}

// Pending 获取待处理消息数
func (p *ConsumerPool[T]) Pending() int {
	return len(p.msgChan)
}
```

---

## 7. 最佳实践与注意事项

### 7.1 消息可靠性保障

#### 7.1.1 端到端可靠性模型

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        消息可靠性保障全链路                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────┐          │
│  │ Producer │───▶│   Broker    │───▶│   Consumer  │───▶│ Business │          │
│  └────┬────┘    └──────┬──────┘    └──────┬──────┘    └────┬────┘          │
│       │                │                   │                │               │
│       ▼                ▼                   ▼                ▼               │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────┐          │
│  │ 发送确认 │    │  持久化存储  │    │   消费确认   │    │ 幂等处理 │          │
│  │ (ACK)   │    │  (Flush)    │    │   (ACK)     │    │ (Dedup) │          │
│  └─────────┘    └─────────────┘    └─────────────┘    └─────────┘          │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│  可靠性级别：                                                                │
│                                                                             │
│  Level 0: At Most Once  (最多一次)                                          │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Producer ──fire&forget──▶ Broker ──auto-commit──▶ Consumer        │    │
│  │ 特点：可能丢失，不重复，性能最高                                        │    │
│  │ 场景：日志采集、监控指标                                               │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Level 1: At Least Once (至少一次)                                          │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Producer ──sync+ack──▶ Broker ──manual-commit──▶ Consumer         │    │
│  │ 特点：不丢失，可能重复，需幂等                                         │    │
│  │ 场景：订单处理、资金变动（配合幂等）                                    │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Level 2: Exactly Once  (精确一次)                                          │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Producer ──transaction──▶ Broker ──transaction──▶ Consumer        │    │
│  │ 特点：不丢失，不重复，性能最低                                         │    │
│  │ 场景：资金清算、关键业务流程                                           │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 7.1.2 生产者可靠性配置

```go
package reliability

import (
	"context"
	"encoding/json"
	"fmt"
	"sync"
	"time"

	"github.com/google/uuid"
)

// ═══════════════════════════════════════════════════════════════════════════
// 可靠消息生产者
// ═══════════════════════════════════════════════════════════════════════════

// ReliableMessage 可靠消息
type ReliableMessage struct {
	ID         string            `json:"id"`
	Topic      string            `json:"topic"`
	Key        string            `json:"key"`
	Payload    []byte            `json:"payload"`
	Headers    map[string]string `json:"headers"`
	CreatedAt  time.Time         `json:"created_at"`
	RetryCount int               `json:"retry_count"`
	MaxRetries int               `json:"max_retries"`
	Status     MessageStatus     `json:"status"`
}

type MessageStatus int

const (
	StatusPending MessageStatus = iota
	StatusSending
	StatusSent
	StatusFailed
	StatusConfirmed
)

// MessageStore 消息持久化存储接口
type MessageStore interface {
	Save(ctx context.Context, msg *ReliableMessage) error
	UpdateStatus(ctx context.Context, id string, status MessageStatus) error
	GetPending(ctx context.Context, limit int) ([]*ReliableMessage, error)
	GetFailed(ctx context.Context, limit int) ([]*ReliableMessage, error)
	Delete(ctx context.Context, id string) error
}

// MessageSender 消息发送接口
type MessageSender interface {
	Send(ctx context.Context, msg *ReliableMessage) error
}

// ReliableProducer 可靠消息生产者
type ReliableProducer struct {
	store      MessageStore
	sender     MessageSender
	maxRetries int
	retryDelay time.Duration
	batchSize  int

	ctx    context.Context
	cancel context.CancelFunc
	wg     sync.WaitGroup
}

// NewReliableProducer 创建可靠消息生产者
func NewReliableProducer(store MessageStore, sender MessageSender) *ReliableProducer {
	ctx, cancel := context.WithCancel(context.Background())

	return &ReliableProducer{
		store:      store,
		sender:     sender,
		maxRetries: 3,
		retryDelay: time.Second * 5,
		batchSize:  100,
		ctx:        ctx,
		cancel:     cancel,
	}
}

// Send 发送可靠消息（先存储后发送）
func (p *ReliableProducer) Send(ctx context.Context, topic, key string, payload []byte) (string, error) {
	msg := &ReliableMessage{
		ID:         uuid.New().String(),
		Topic:      topic,
		Key:        key,
		Payload:    payload,
		Headers:    make(map[string]string),
		CreatedAt:  time.Now(),
		MaxRetries: p.maxRetries,
		Status:     StatusPending,
	}

	// Step 1: 先持久化到本地存储
	if err := p.store.Save(ctx, msg); err != nil {
		return "", fmt.Errorf("save message failed: %w", err)
	}

	// Step 2: 尝试发送
	if err := p.trySend(ctx, msg); err != nil {
		// 发送失败，保持pending状态，等待重试
		return msg.ID, nil // 返回ID，不返回错误，消息会被后台重试
	}

	return msg.ID, nil
}

// trySend 尝试发送消息
func (p *ReliableProducer) trySend(ctx context.Context, msg *ReliableMessage) error {
	// 更新状态为发送中
	_ = p.store.UpdateStatus(ctx, msg.ID, StatusSending)

	// 发送消息
	if err := p.sender.Send(ctx, msg); err != nil {
		msg.RetryCount++
		if msg.RetryCount >= msg.MaxRetries {
			_ = p.store.UpdateStatus(ctx, msg.ID, StatusFailed)
		} else {
			_ = p.store.UpdateStatus(ctx, msg.ID, StatusPending)
		}
		return err
	}

	// 发送成功，删除消息或标记为已确认
	_ = p.store.UpdateStatus(ctx, msg.ID, StatusConfirmed)
	return nil
}

// Start 启动后台重试任务
func (p *ReliableProducer) Start() {
	p.wg.Add(1)
	go p.retryLoop()
}

// retryLoop 重试循环
func (p *ReliableProducer) retryLoop() {
	defer p.wg.Done()

	ticker := time.NewTicker(p.retryDelay)
	defer ticker.Stop()

	for {
		select {
		case <-p.ctx.Done():
			return
		case <-ticker.C:
			p.retryPendingMessages()
		}
	}
}

// retryPendingMessages 重试待发送消息
func (p *ReliableProducer) retryPendingMessages() {
	ctx, cancel := context.WithTimeout(p.ctx, time.Minute)
	defer cancel()

	messages, err := p.store.GetPending(ctx, p.batchSize)
	if err != nil {
		return
	}

	for _, msg := range messages {
		select {
		case <-p.ctx.Done():
			return
		default:
			_ = p.trySend(ctx, msg)
		}
	}
}

// Stop 停止生产者
func (p *ReliableProducer) Stop() {
	p.cancel()
	p.wg.Wait()
}

// ═══════════════════════════════════════════════════════════════════════════
// 事务消息模式
// ═══════════════════════════════════════════════════════════════════════════

// TransactionalMessage 事务消息
type TransactionalMessage struct {
	ID            string          `json:"id"`
	Topic         string          `json:"topic"`
	Payload       json.RawMessage `json:"payload"`
	LocalTxID     string          `json:"local_tx_id"`
	Status        TxMessageStatus `json:"status"`
	CheckCount    int             `json:"check_count"`
	MaxCheckCount int             `json:"max_check_count"`
	CreatedAt     time.Time       `json:"created_at"`
}

type TxMessageStatus int

const (
	TxStatusPrepare   TxMessageStatus = iota // 半消息
	TxStatusCommit                           // 提交
	TxStatusRollback                         // 回滚
	TxStatusUnknown                          // 未知
)

// LocalTransactionChecker 本地事务检查器
type LocalTransactionChecker interface {
	CheckLocalTransaction(ctx context.Context, txID string) (TxMessageStatus, error)
}

// TransactionalProducer 事务消息生产者
type TransactionalProducer struct {
	store   MessageStore
	sender  MessageSender
	checker LocalTransactionChecker
}

// SendTransactional 发送事务消息
func (p *TransactionalProducer) SendTransactional(
	ctx context.Context,
	topic string,
	payload []byte,
	localTx func(ctx context.Context, txID string) error,
) error {
	txID := uuid.New().String()

	// Step 1: 发送半消息（Prepare）
	msg := &TransactionalMessage{
		ID:            uuid.New().String(),
		Topic:         topic,
		Payload:       payload,
		LocalTxID:     txID,
		Status:        TxStatusPrepare,
		MaxCheckCount: 15,
		CreatedAt:     time.Now(),
	}

	if err := p.sendHalfMessage(ctx, msg); err != nil {
		return fmt.Errorf("send half message failed: %w", err)
	}

	// Step 2: 执行本地事务
	txErr := localTx(ctx, txID)

	// Step 3: 提交或回滚
	if txErr != nil {
		_ = p.rollbackMessage(ctx, msg.ID)
		return fmt.Errorf("local transaction failed: %w", txErr)
	}

	if err := p.commitMessage(ctx, msg.ID); err != nil {
		// 提交失败，依赖事务回查机制
		return fmt.Errorf("commit message failed: %w", err)
	}

	return nil
}

func (p *TransactionalProducer) sendHalfMessage(ctx context.Context, msg *TransactionalMessage) error {
	// 发送prepare消息到broker
	return nil
}

func (p *TransactionalProducer) commitMessage(ctx context.Context, msgID string) error {
	// 发送commit确认
	return nil
}

func (p *TransactionalProducer) rollbackMessage(ctx context.Context, msgID string) error {
	// 发送rollback确认
	return nil
}
```

### 7.2 幂等性设计

#### 7.2.1 幂等性实现策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           幂等性实现策略                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  策略一：唯一ID + 去重表                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │   Message ──▶ [Check ID in DB] ──exists──▶ Skip (Return Success)     │  │
│  │                     │                                                 │  │
│  │                not exists                                             │  │
│  │                     │                                                 │  │
│  │                     ▼                                                 │  │
│  │              [Insert ID + Process] ──success──▶ Commit               │  │
│  │                     │                                                 │  │
│  │                  duplicate                                            │  │
│  │                     │                                                 │  │
│  │                     ▼                                                 │  │
│  │              [Concurrent Request] ──▶ Skip (Return Success)          │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  策略二：状态机 + 版本号                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │   Order状态机:                                                        │  │
│  │   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐          │  │
│  │   │ Created  │──▶│ Pending  │──▶│ Matched  │──▶│ Settled  │          │  │
│  │   │ (v=1)    │   │ (v=2)    │   │ (v=3)    │   │ (v=4)    │          │  │
│  │   └──────────┘   └──────────┘   └──────────┘   └──────────┘          │  │
│  │                                                                       │  │
│  │   UPDATE orders SET status=?, version=version+1                       │  │
│  │   WHERE id=? AND version=? AND status=?                               │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  策略三：Redis + Lua 原子操作                                                │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │   EVAL "                                                              │  │
│  │     if redis.call('SETNX', KEYS[1], ARGV[1]) == 1 then               │  │
│  │       redis.call('EXPIRE', KEYS[1], ARGV[2])                         │  │
│  │       return 1  -- 首次处理                                           │  │
│  │     else                                                              │  │
│  │       return 0  -- 重复请求                                           │  │
│  │     end                                                               │  │
│  │   " 1 "dedup:order:123" "processed" 86400                            │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 7.2.2 幂等消费者实现

```go
package idempotent

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

// ═══════════════════════════════════════════════════════════════════════════
// 幂等消费者
// ═══════════════════════════════════════════════════════════════════════════

// IdempotentConsumer 幂等消费者
type IdempotentConsumer[T any] struct {
	deduplicator Deduplicator
	handler      func(ctx context.Context, msg T) error
	keyExtractor func(msg T) string
	ttl          time.Duration
}

// Deduplicator 去重器接口
type Deduplicator interface {
	// IsDuplicate 检查是否重复，如果不重复则标记
	IsDuplicate(ctx context.Context, key string, ttl time.Duration) (bool, error)
	// MarkComplete 标记处理完成
	MarkComplete(ctx context.Context, key string) error
	// Remove 移除标记（用于处理失败时）
	Remove(ctx context.Context, key string) error
}

// NewIdempotentConsumer 创建幂等消费者
func NewIdempotentConsumer[T any](
	dedup Deduplicator,
	handler func(ctx context.Context, msg T) error,
	keyExtractor func(msg T) string,
) *IdempotentConsumer[T] {
	return &IdempotentConsumer[T]{
		deduplicator: dedup,
		handler:      handler,
		keyExtractor: keyExtractor,
		ttl:          24 * time.Hour,
	}
}

// Process 幂等处理消息
func (c *IdempotentConsumer[T]) Process(ctx context.Context, msg T) error {
	key := c.keyExtractor(msg)

	// Step 1: 检查是否重复
	isDup, err := c.deduplicator.IsDuplicate(ctx, key, c.ttl)
	if err != nil {
		return fmt.Errorf("check duplicate failed: %w", err)
	}

	if isDup {
		// 重复消息，直接返回成功
		return nil
	}

	// Step 2: 处理消息
	if err := c.handler(ctx, msg); err != nil {
		// 处理失败，移除标记以允许重试
		_ = c.deduplicator.Remove(ctx, key)
		return err
	}

	// Step 3: 标记处理完成
	_ = c.deduplicator.MarkComplete(ctx, key)

	return nil
}

// ═══════════════════════════════════════════════════════════════════════════
// Redis 去重器实现
// ═══════════════════════════════════════════════════════════════════════════

// RedisDeduplicator Redis去重器
type RedisDeduplicator struct {
	client *redis.Client
	prefix string
}

// NewRedisDeduplicator 创建Redis去重器
func NewRedisDeduplicator(client *redis.Client, prefix string) *RedisDeduplicator {
	return &RedisDeduplicator{
		client: client,
		prefix: prefix,
	}
}

// IsDuplicate 检查是否重复
func (d *RedisDeduplicator) IsDuplicate(ctx context.Context, key string, ttl time.Duration) (bool, error) {
	fullKey := d.prefix + key

	// 使用 SETNX + EXPIRE 原子操作
	script := redis.NewScript(`
		if redis.call('SETNX', KEYS[1], ARGV[1]) == 1 then
			redis.call('EXPIRE', KEYS[1], ARGV[2])
			return 0
		else
			return 1
		end
	`)

	result, err := script.Run(ctx, d.client, []string{fullKey}, "processing", int(ttl.Seconds())).Int()
	if err != nil {
		return false, err
	}

	return result == 1, nil
}

// MarkComplete 标记处理完成
func (d *RedisDeduplicator) MarkComplete(ctx context.Context, key string) error {
	fullKey := d.prefix + key
	return d.client.Set(ctx, fullKey, "completed", 0).Err()
}

// Remove 移除标记
func (d *RedisDeduplicator) Remove(ctx context.Context, key string) error {
	fullKey := d.prefix + key
	return d.client.Del(ctx, fullKey).Err()
}

// ═══════════════════════════════════════════════════════════════════════════
// 数据库去重器实现（带事务）
// ═══════════════════════════════════════════════════════════════════════════

// DBDeduplicator 数据库去重器
type DBDeduplicator struct {
	db Database
}

// Database 数据库接口
type Database interface {
	ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error)
	QueryRowContext(ctx context.Context, query string, args ...interface{}) Row
}

type Result interface {
	RowsAffected() (int64, error)
}

type Row interface {
	Scan(dest ...interface{}) error
}

// NewDBDeduplicator 创建数据库去重器
func NewDBDeduplicator(db Database) *DBDeduplicator {
	return &DBDeduplicator{db: db}
}

// IsDuplicate 检查是否重复（使用INSERT IGNORE或ON CONFLICT）
func (d *DBDeduplicator) IsDuplicate(ctx context.Context, key string, ttl time.Duration) (bool, error) {
	// PostgreSQL 示例
	query := `
		INSERT INTO message_dedup (message_key, status, created_at, expires_at)
		VALUES ($1, 'processing', NOW(), NOW() + $2::interval)
		ON CONFLICT (message_key) DO NOTHING
		RETURNING message_key
	`

	var insertedKey string
	err := d.db.QueryRowContext(ctx, query, key, ttl.String()).Scan(&insertedKey)

	if err != nil {
		// 没有返回行，说明记录已存在
		return true, nil
	}

	return false, nil
}

// MarkComplete 标记处理完成
func (d *DBDeduplicator) MarkComplete(ctx context.Context, key string) error {
	query := `UPDATE message_dedup SET status = 'completed' WHERE message_key = $1`
	_, err := d.db.ExecContext(ctx, query, key)
	return err
}

// Remove 移除标记
func (d *DBDeduplicator) Remove(ctx context.Context, key string) error {
	query := `DELETE FROM message_dedup WHERE message_key = $1`
	_, err := d.db.ExecContext(ctx, query, key)
	return err
}

// ═══════════════════════════════════════════════════════════════════════════
// 幂等订单处理器（状态机模式）
// ═══════════════════════════════════════════════════════════════════════════

// OrderStatus 订单状态
type OrderStatus int

const (
	OrderCreated OrderStatus = iota
	OrderPending
	OrderMatched
	OrderPartialFilled
	OrderFilled
	OrderCancelled
)

// Order 订单
type Order struct {
	ID        string
	Status    OrderStatus
	Version   int64
	Amount    float64
	Filled    float64
	UpdatedAt time.Time
}

// OrderRepository 订单仓库
type OrderRepository interface {
	GetByID(ctx context.Context, id string) (*Order, error)
	UpdateWithVersion(ctx context.Context, order *Order, expectedVersion int64) error
}

// IdempotentOrderProcessor 幂等订单处理器
type IdempotentOrderProcessor struct {
	repo OrderRepository
}

// ProcessMatch 处理撮合结果（幂等）
func (p *IdempotentOrderProcessor) ProcessMatch(ctx context.Context, orderID string, matchAmount float64) error {
	// 获取当前订单
	order, err := p.repo.GetByID(ctx, orderID)
	if err != nil {
		return fmt.Errorf("get order failed: %w", err)
	}

	// 检查状态是否允许此操作
	if !p.canMatch(order.Status) {
		// 状态不允许，可能是重复消息，返回成功
		return nil
	}

	// 计算新的成交量
	newFilled := order.Filled + matchAmount
	if newFilled > order.Amount {
		// 超过订单数量，可能是重复消息
		return nil
	}

	// 更新订单状态
	order.Filled = newFilled
	if newFilled >= order.Amount {
		order.Status = OrderFilled
	} else {
		order.Status = OrderPartialFilled
	}
	order.UpdatedAt = time.Now()

	// 使用乐观锁更新
	if err := p.repo.UpdateWithVersion(ctx, order, order.Version); err != nil {
		if errors.Is(err, ErrVersionConflict) {
			// 版本冲突，重试
			return p.ProcessMatch(ctx, orderID, matchAmount)
		}
		return err
	}

	return nil
}

// canMatch 检查是否可以进行撮合
func (p *IdempotentOrderProcessor) canMatch(status OrderStatus) bool {
	return status == OrderPending || status == OrderPartialFilled
}

var ErrVersionConflict = errors.New("version conflict")

// ═══════════════════════════════════════════════════════════════════════════
// 消息指纹生成
// ═══════════════════════════════════════════════════════════════════════════

// MessageFingerprint 消息指纹生成器
type MessageFingerprint struct{}

// Generate 生成消息指纹
func (f *MessageFingerprint) Generate(topic string, key string, payload []byte) string {
	h := sha256.New()
	h.Write([]byte(topic))
	h.Write([]byte(":"))
	h.Write([]byte(key))
	h.Write([]byte(":"))
	h.Write(payload)
	return hex.EncodeToString(h.Sum(nil))[:32]
}

// GenerateWithBizKey 基于业务键生成指纹
func (f *MessageFingerprint) GenerateWithBizKey(bizType string, bizKey string) string {
	return fmt.Sprintf("%s:%s", bizType, bizKey)
}
```

### 7.3 监控与告警

#### 7.3.1 监控指标体系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          消息中间件监控指标体系                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        生产者指标                                    │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  • producer_messages_total      - 发送消息总数                      │   │
│  │  • producer_messages_failed     - 发送失败消息数                    │   │
│  │  • producer_latency_seconds     - 发送延迟分布                      │   │
│  │  • producer_batch_size          - 批次大小分布                      │   │
│  │  • producer_buffer_available    - 可用缓冲区大小                    │   │
│  │  • producer_retry_total         - 重试次数                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Broker 指标                                   │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  • broker_messages_in_rate      - 消息流入速率                      │   │
│  │  • broker_messages_out_rate     - 消息流出速率                      │   │
│  │  • broker_bytes_in_rate         - 字节流入速率                      │   │
│  │  • broker_bytes_out_rate        - 字节流出速率                      │   │
│  │  • broker_partition_count       - 分区数量                          │   │
│  │  • broker_replica_lag           - 副本延迟                          │   │
│  │  • broker_isr_shrink_rate       - ISR 收缩率                        │   │
│  │  • broker_disk_usage            - 磁盘使用率                        │   │
│  │  • broker_network_idle_percent  - 网络空闲率                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        消费者指标                                    │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  • consumer_messages_total      - 消费消息总数                      │   │
│  │  • consumer_lag                 - 消费延迟（消息数）                │   │
│  │  • consumer_lag_seconds         - 消费延迟（时间）                  │   │
│  │  • consumer_processing_time     - 处理时间分布                      │   │
│  │  • consumer_rebalance_total     - 重平衡次数                        │   │
│  │  • consumer_commit_latency      - 提交延迟                          │   │
│  │  • consumer_errors_total        - 处理错误数                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        业务指标                                      │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  • order_processing_latency     - 订单处理延迟                      │   │
│  │  • order_end_to_end_latency     - 端到端延迟                        │   │
│  │  • dead_letter_messages         - 死信消息数                        │   │
│  │  • duplicate_messages           - 重复消息数                        │   │
│  │  • message_retry_rate           - 消息重试率                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 7.3.2 监控实现

```go
package monitoring

import (
	"context"
	"sync"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

// ═══════════════════════════════════════════════════════════════════════════
// Prometheus 指标定义
// ═══════════════════════════════════════════════════════════════════════════

var (
	// 生产者指标
	producerMessagesTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "mq_producer_messages_total",
			Help: "Total number of messages produced",
		},
		[]string{"topic", "status"},
	)

	producerLatency = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "mq_producer_latency_seconds",
			Help:    "Producer latency distribution",
			Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1},
		},
		[]string{"topic"},
	)

	// 消费者指标
	consumerMessagesTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "mq_consumer_messages_total",
			Help: "Total number of messages consumed",
		},
		[]string{"topic", "consumer_group", "status"},
	)

	consumerLag = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "mq_consumer_lag",
			Help: "Consumer lag in messages",
		},
		[]string{"topic", "partition", "consumer_group"},
	)

	consumerProcessingTime = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "mq_consumer_processing_seconds",
			Help:    "Consumer message processing time",
			Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
		},
		[]string{"topic", "consumer_group"},
	)

	// 业务指标
	endToEndLatency = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "mq_end_to_end_latency_seconds",
			Help:    "End-to-end message latency",
			Buckets: []float64{.01, .05, .1, .25, .5, 1, 2.5, 5, 10},
		},
		[]string{"topic", "message_type"},
	)

	deadLetterMessages = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "mq_dead_letter_messages_total",
			Help: "Total number of dead letter messages",
		},
		[]string{"topic", "reason"},
	)
)

// ═══════════════════════════════════════════════════════════════════════════
// 指标收集器
// ═══════════════════════════════════════════════════════════════════════════

// MetricsCollector 指标收集器
type MetricsCollector struct {
	producerStats map[string]*ProducerStats
	consumerStats map[string]*ConsumerStats
	mu            sync.RWMutex
}

// ProducerStats 生产者统计
type ProducerStats struct {
	Topic          string
	MessagesSent   int64
	MessagesFailed int64
	BytesSent      int64
	LastSendTime   time.Time
	AvgLatencyMs   float64
}

// ConsumerStats 消费者统计
type ConsumerStats struct {
	Topic          string
	Group          string
	MessagesRecv   int64
	MessagesProc   int64
	MessagesFailed int64
	CurrentLag     int64
	AvgProcTimeMs  float64
}

// NewMetricsCollector 创建指标收集器
func NewMetricsCollector() *MetricsCollector {
	return &MetricsCollector{
		producerStats: make(map[string]*ProducerStats),
		consumerStats: make(map[string]*ConsumerStats),
	}
}

// RecordProducerSend 记录生产者发送
func (c *MetricsCollector) RecordProducerSend(topic string, success bool, latency time.Duration) {
	if success {
		producerMessagesTotal.WithLabelValues(topic, "success").Inc()
	} else {
		producerMessagesTotal.WithLabelValues(topic, "failed").Inc()
	}
	producerLatency.WithLabelValues(topic).Observe(latency.Seconds())
}

// RecordConsumerProcess 记录消费者处理
func (c *MetricsCollector) RecordConsumerProcess(topic, group string, success bool, procTime time.Duration) {
	status := "success"
	if !success {
		status = "failed"
	}
	consumerMessagesTotal.WithLabelValues(topic, group, status).Inc()
	consumerProcessingTime.WithLabelValues(topic, group).Observe(procTime.Seconds())
}

// UpdateConsumerLag 更新消费者延迟
func (c *MetricsCollector) UpdateConsumerLag(topic string, partition int, group string, lag int64) {
	consumerLag.WithLabelValues(topic, string(rune(partition)), group).Set(float64(lag))
}

// RecordEndToEndLatency 记录端到端延迟
func (c *MetricsCollector) RecordEndToEndLatency(topic, msgType string, latency time.Duration) {
	endToEndLatency.WithLabelValues(topic, msgType).Observe(latency.Seconds())
}

// RecordDeadLetter 记录死信消息
func (c *MetricsCollector) RecordDeadLetter(topic, reason string) {
	deadLetterMessages.WithLabelValues(topic, reason).Inc()
}

// ═══════════════════════════════════════════════════════════════════════════
// 告警规则
// ═══════════════════════════════════════════════════════════════════════════

// AlertRule 告警规则
type AlertRule struct {
	Name        string
	Description string
	Expr        string        // PromQL 表达式
	For         time.Duration // 持续时间
	Severity    string        // critical, warning, info
	Labels      map[string]string
	Annotations map[string]string
}

// GetAlertRules 获取告警规则
func GetAlertRules() []AlertRule {
	return []AlertRule{
		// 消费延迟告警
		{
			Name:        "HighConsumerLag",
			Description: "Consumer lag is too high",
			Expr:        "mq_consumer_lag > 10000",
			For:         5 * time.Minute,
			Severity:    "warning",
			Labels: map[string]string{
				"category": "mq",
			},
			Annotations: map[string]string{
				"summary":     "High consumer lag detected",
				"description": "Consumer group {{ $labels.consumer_group }} has lag > 10000 messages",
			},
		},
		// 严重消费延迟告警
		{
			Name:        "CriticalConsumerLag",
			Description: "Consumer lag is critically high",
			Expr:        "mq_consumer_lag > 100000",
			For:         5 * time.Minute,
			Severity:    "critical",
			Labels: map[string]string{
				"category": "mq",
			},
			Annotations: map[string]string{
				"summary":     "Critical consumer lag",
				"description": "Consumer group {{ $labels.consumer_group }} has lag > 100000 messages",
			},
		},
		// 生产者失败率告警
		{
			Name:        "HighProducerFailRate",
			Description: "Producer failure rate is high",
			Expr: `
				rate(mq_producer_messages_total{status="failed"}[5m]) /
				rate(mq_producer_messages_total[5m]) > 0.01
			`,
			For:      5 * time.Minute,
			Severity: "warning",
			Labels: map[string]string{
				"category": "mq",
			},
			Annotations: map[string]string{
				"summary":     "High producer failure rate",
				"description": "Producer failure rate > 1% for topic {{ $labels.topic }}",
			},
		},
		// 死信消息告警
		{
			Name:        "DeadLetterMessages",
			Description: "Dead letter messages detected",
			Expr:        "increase(mq_dead_letter_messages_total[1h]) > 10",
			For:         0,
			Severity:    "warning",
			Labels: map[string]string{
				"category": "mq",
			},
			Annotations: map[string]string{
				"summary":     "Dead letter messages detected",
				"description": "{{ $value }} dead letter messages in the last hour",
			},
		},
		// 端到端延迟告警
		{
			Name:        "HighEndToEndLatency",
			Description: "End-to-end latency is high",
			Expr:        "histogram_quantile(0.99, mq_end_to_end_latency_seconds_bucket) > 1",
			For:         5 * time.Minute,
			Severity:    "warning",
			Labels: map[string]string{
				"category": "mq",
			},
			Annotations: map[string]string{
				"summary":     "High end-to-end latency",
				"description": "P99 latency > 1s for {{ $labels.topic }}",
			},
		},
		// ISR 收缩告警 (Kafka)
		{
			Name:        "KafkaISRShrink",
			Description: "Kafka ISR is shrinking",
			Expr:        "kafka_server_replicamanager_isrshrinks_total > 0",
			For:         0,
			Severity:    "warning",
			Labels: map[string]string{
				"category": "kafka",
			},
			Annotations: map[string]string{
				"summary":     "Kafka ISR shrinking",
				"description": "ISR shrinking detected, potential data loss risk",
			},
		},
	}
}

// ═══════════════════════════════════════════════════════════════════════════
// 消费延迟监控器
// ═══════════════════════════════════════════════════════════════════════════

// LagMonitor 消费延迟监控器
type LagMonitor struct {
	kafkaAdmin    KafkaAdmin
	metrics       *MetricsCollector
	checkInterval time.Duration
	ctx           context.Context
	cancel        context.CancelFunc
}

// KafkaAdmin Kafka管理接口
type KafkaAdmin interface {
	GetConsumerGroupLag(ctx context.Context, group string) (map[string]map[int32]int64, error)
	ListConsumerGroups(ctx context.Context) ([]string, error)
}

// NewLagMonitor 创建延迟监控器
func NewLagMonitor(admin KafkaAdmin, metrics *MetricsCollector) *LagMonitor {
	ctx, cancel := context.WithCancel(context.Background())
	return &LagMonitor{
		kafkaAdmin:    admin,
		metrics:       metrics,
		checkInterval: 30 * time.Second,
		ctx:           ctx,
		cancel:        cancel,
	}
}

// Start 启动监控
func (m *LagMonitor) Start() {
	go m.monitorLoop()
}

// monitorLoop 监控循环
func (m *LagMonitor) monitorLoop() {
	ticker := time.NewTicker(m.checkInterval)
	defer ticker.Stop()

	for {
		select {
		case <-m.ctx.Done():
			return
		case <-ticker.C:
			m.checkAllGroups()
		}
	}
}

// checkAllGroups 检查所有消费组
func (m *LagMonitor) checkAllGroups() {
	groups, err := m.kafkaAdmin.ListConsumerGroups(m.ctx)
	if err != nil {
		return
	}

	for _, group := range groups {
		lagData, err := m.kafkaAdmin.GetConsumerGroupLag(m.ctx, group)
		if err != nil {
			continue
		}

		for topic, partitions := range lagData {
			for partition, lag := range partitions {
				m.metrics.UpdateConsumerLag(topic, int(partition), group, lag)
			}
		}
	}
}

// Stop 停止监控
func (m *LagMonitor) Stop() {
	m.cancel()
}
```

### 7.4 灾备与容灾

#### 7.4.1 多活架构设计

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          消息中间件多活架构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  方案一：双写双读（强一致）                                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │             ┌─────────────────────────────────────────┐              │  │
│  │             │           Producer                      │              │  │
│  │             └───────────────┬─────────────────────────┘              │  │
│  │                             │                                         │  │
│  │              ┌──────────────┴──────────────┐                         │  │
│  │              │         双写代理             │                         │  │
│  │              └──────────────┬──────────────┘                         │  │
│  │                    ┌────────┴────────┐                               │  │
│  │                    ▼                 ▼                               │  │
│  │  ┌────────────────────┐    ┌────────────────────┐                   │  │
│  │  │  Kafka Cluster A   │    │  Kafka Cluster B   │                   │  │
│  │  │    (Region A)      │    │    (Region B)      │                   │  │
│  │  └─────────┬──────────┘    └──────────┬─────────┘                   │  │
│  │            │                          │                              │  │
│  │            └──────────┬───────────────┘                              │  │
│  │                       │                                              │  │
│  │              ┌────────┴────────┐                                     │  │
│  │              │    Consumer     │                                     │  │
│  │              │  (去重消费)     │                                     │  │
│  │              └─────────────────┘                                     │  │
│  │                                                                       │  │
│  │  优点：强一致，无消息丢失                                              │  │
│  │  缺点：延迟高，成本高                                                  │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  方案二：异步复制（最终一致）                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  Region A (Primary)              Region B (Standby)                  │  │
│  │  ┌─────────────────────┐         ┌─────────────────────┐            │  │
│  │  │   Kafka Cluster A   │ ──────▶ │   Kafka Cluster B   │            │  │
│  │  │                     │  Mirror │                     │            │  │
│  │  │  ┌───┐ ┌───┐ ┌───┐ │  Maker  │  ┌───┐ ┌───┐ ┌───┐ │            │  │
│  │  │  │P0 │ │P1 │ │P2 │ │ ──────▶ │  │P0 │ │P1 │ │P2 │ │            │  │
│  │  │  └───┘ └───┘ └───┘ │         │  └───┘ └───┘ └───┘ │            │  │
│  │  └─────────────────────┘         └─────────────────────┘            │  │
│  │          │                                   │                       │  │
│  │          ▼                                   ▼                       │  │
│  │    ┌──────────┐                        ┌──────────┐                  │  │
│  │    │ Consumer │                        │ Consumer │ (Standby)       │  │
│  │    │ (Active) │                        │          │                  │  │
│  │    └──────────┘                        └──────────┘                  │  │
│  │                                                                       │  │
│  │  优点：延迟低，成本低                                                  │  │
│  │  缺点：存在复制延迟，可能丢失数据                                      │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  方案三：分区级多活                                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │        Topic: orders (12 partitions)                                 │  │
│  │                                                                       │  │
│  │  Region A                           Region B                         │  │
│  │  ┌───────────────────────┐         ┌───────────────────────┐        │  │
│  │  │ P0  P1  P2  P3  P4  P5│         │ P6  P7  P8  P9 P10 P11│        │  │
│  │  │ (Leader)              │         │ (Leader)              │        │  │
│  │  │                       │         │                       │        │  │
│  │  │ P6  P7  P8  P9 P10 P11│         │ P0  P1  P2  P3  P4  P5│        │  │
│  │  │ (Follower)            │         │ (Follower)            │        │  │
│  │  └───────────────────────┘         └───────────────────────┘        │  │
│  │                                                                       │  │
│  │  优点：读写分离，延迟和可用性平衡                                      │  │
│  │  缺点：实现复杂，运维成本高                                            │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 7.4.2 故障切换实现

```go
package disaster_recovery

import (
	"context"
	"errors"
	"sync"
	"sync/atomic"
	"time"
)

// ═══════════════════════════════════════════════════════════════════════════
// 故障切换管理器
// ═══════════════════════════════════════════════════════════════════════════

// ClusterStatus 集群状态
type ClusterStatus int

const (
	ClusterHealthy ClusterStatus = iota
	ClusterDegraded
	ClusterDown
)

// Cluster 集群信息
type Cluster struct {
	Name      string
	Endpoints []string
	Status    ClusterStatus
	Priority  int
	LastCheck time.Time
	FailCount int
}

// FailoverManager 故障切换管理器
type FailoverManager struct {
	clusters       []*Cluster
	currentCluster atomic.Value // *Cluster
	healthChecker  HealthChecker
	switchCallback func(from, to *Cluster)

	mu            sync.RWMutex
	checkInterval time.Duration
	failThreshold int

	ctx    context.Context
	cancel context.CancelFunc
}

// HealthChecker 健康检查接口
type HealthChecker interface {
	Check(ctx context.Context, cluster *Cluster) (ClusterStatus, error)
}

// NewFailoverManager 创建故障切换管理器
func NewFailoverManager(clusters []*Cluster, checker HealthChecker) *FailoverManager {
	ctx, cancel := context.WithCancel(context.Background())

	fm := &FailoverManager{
		clusters:      clusters,
		healthChecker: checker,
		checkInterval: 10 * time.Second,
		failThreshold: 3,
		ctx:           ctx,
		cancel:        cancel,
	}

	// 初始选择最高优先级的健康集群
	for _, c := range clusters {
		if c.Status == ClusterHealthy {
			fm.currentCluster.Store(c)
			break
		}
	}

	return fm
}

// Start 启动故障切换管理
func (m *FailoverManager) Start() {
	go m.healthCheckLoop()
}

// healthCheckLoop 健康检查循环
func (m *FailoverManager) healthCheckLoop() {
	ticker := time.NewTicker(m.checkInterval)
	defer ticker.Stop()

	for {
		select {
		case <-m.ctx.Done():
			return
		case <-ticker.C:
			m.checkAndFailover()
		}
	}
}

// checkAndFailover 检查并执行故障切换
func (m *FailoverManager) checkAndFailover() {
	m.mu.Lock()
	defer m.mu.Unlock()

	current := m.GetCurrentCluster()

	// 检查所有集群状态
	for _, cluster := range m.clusters {
		status, err := m.healthChecker.Check(m.ctx, cluster)
		if err != nil {
			cluster.FailCount++
			if cluster.FailCount >= m.failThreshold {
				cluster.Status = ClusterDown
			}
		} else {
			cluster.FailCount = 0
			cluster.Status = status
		}
		cluster.LastCheck = time.Now()
	}

	// 检查当前集群是否需要切换
	if current != nil && current.Status != ClusterHealthy {
		// 寻找新的健康集群
		newCluster := m.findHealthyCluster()
		if newCluster != nil && newCluster != current {
			m.switchTo(newCluster)
		}
	}
}

// findHealthyCluster 找到健康的集群
func (m *FailoverManager) findHealthyCluster() *Cluster {
	var best *Cluster
	for _, c := range m.clusters {
		if c.Status == ClusterHealthy {
			if best == nil || c.Priority > best.Priority {
				best = c
			}
		}
	}
	return best
}

// switchTo 切换到新集群
func (m *FailoverManager) switchTo(newCluster *Cluster) {
	old := m.GetCurrentCluster()
	m.currentCluster.Store(newCluster)

	if m.switchCallback != nil {
		m.switchCallback(old, newCluster)
	}
}

// GetCurrentCluster 获取当前集群
func (m *FailoverManager) GetCurrentCluster() *Cluster {
	if v := m.currentCluster.Load(); v != nil {
		return v.(*Cluster)
	}
	return nil
}

// OnSwitch 注册切换回调
func (m *FailoverManager) OnSwitch(callback func(from, to *Cluster)) {
	m.switchCallback = callback
}

// Stop 停止管理器
func (m *FailoverManager) Stop() {
	m.cancel()
}

// ═══════════════════════════════════════════════════════════════════════════
// 双写代理
// ═══════════════════════════════════════════════════════════════════════════

// DualWriteProxy 双写代理
type DualWriteProxy struct {
	primary   MessageProducer
	secondary MessageProducer
	mode      DualWriteMode
	timeout   time.Duration
}

type DualWriteMode int

const (
	ModeSync  DualWriteMode = iota // 同步双写，两边都成功才返回
	ModeAsync                      // 异步双写，主成功即返回
	ModeBest                       // 尽力双写，主失败切备
)

// MessageProducer 消息生产者接口
type MessageProducer interface {
	Send(ctx context.Context, topic string, key string, value []byte) error
}

// NewDualWriteProxy 创建双写代理
func NewDualWriteProxy(primary, secondary MessageProducer, mode DualWriteMode) *DualWriteProxy {
	return &DualWriteProxy{
		primary:   primary,
		secondary: secondary,
		mode:      mode,
		timeout:   5 * time.Second,
	}
}

// Send 发送消息
func (p *DualWriteProxy) Send(ctx context.Context, topic, key string, value []byte) error {
	switch p.mode {
	case ModeSync:
		return p.sendSync(ctx, topic, key, value)
	case ModeAsync:
		return p.sendAsync(ctx, topic, key, value)
	case ModeBest:
		return p.sendBestEffort(ctx, topic, key, value)
	default:
		return errors.New("unknown mode")
	}
}

// sendSync 同步双写
func (p *DualWriteProxy) sendSync(ctx context.Context, topic, key string, value []byte) error {
	ctx, cancel := context.WithTimeout(ctx, p.timeout)
	defer cancel()

	errCh := make(chan error, 2)

	go func() {
		errCh <- p.primary.Send(ctx, topic, key, value)
	}()

	go func() {
		errCh <- p.secondary.Send(ctx, topic, key, value)
	}()

	var errs []error
	for i := 0; i < 2; i++ {
		if err := <-errCh; err != nil {
			errs = append(errs, err)
		}
	}

	if len(errs) > 0 {
		return errors.Join(errs...)
	}

	return nil
}

// sendAsync 异步双写
func (p *DualWriteProxy) sendAsync(ctx context.Context, topic, key string, value []byte) error {
	// 异步写入备集群
	go func() {
		asyncCtx, cancel := context.WithTimeout(context.Background(), p.timeout)
		defer cancel()
		_ = p.secondary.Send(asyncCtx, topic, key, value)
	}()

	// 同步写入主集群
	return p.primary.Send(ctx, topic, key, value)
}

// sendBestEffort 尽力双写
func (p *DualWriteProxy) sendBestEffort(ctx context.Context, topic, key string, value []byte) error {
	// 先尝试主集群
	if err := p.primary.Send(ctx, topic, key, value); err == nil {
		// 主成功，异步写备
		go func() {
			asyncCtx, cancel := context.WithTimeout(context.Background(), p.timeout)
			defer cancel()
			_ = p.secondary.Send(asyncCtx, topic, key, value)
		}()
		return nil
	}

	// 主失败，尝试备集群
	return p.secondary.Send(ctx, topic, key, value)
}

// ═══════════════════════════════════════════════════════════════════════════
// 消息补偿器
// ═══════════════════════════════════════════════════════════════════════════

// MessageCompensator 消息补偿器（用于故障恢复后的数据同步）
type MessageCompensator struct {
	source     MessageReader
	target     MessageProducer
	checkpoint CheckpointStore
	batchSize  int
}

// MessageReader 消息读取接口
type MessageReader interface {
	Read(ctx context.Context, topic string, offset int64, limit int) ([]Message, error)
	GetLatestOffset(ctx context.Context, topic string) (int64, error)
}

// Message 消息
type Message struct {
	Topic     string
	Partition int
	Offset    int64
	Key       []byte
	Value     []byte
	Timestamp time.Time
}

// CheckpointStore 检查点存储
type CheckpointStore interface {
	Get(ctx context.Context, key string) (int64, error)
	Set(ctx context.Context, key string, offset int64) error
}

// NewMessageCompensator 创建消息补偿器
func NewMessageCompensator(source MessageReader, target MessageProducer, checkpoint CheckpointStore) *MessageCompensator {
	return &MessageCompensator{
		source:     source,
		target:     target,
		checkpoint: checkpoint,
		batchSize:  1000,
	}
}

// Compensate 执行补偿
func (c *MessageCompensator) Compensate(ctx context.Context, topic string) error {
	// 获取已同步的offset
	checkpointKey := "compensate:" + topic
	lastOffset, err := c.checkpoint.Get(ctx, checkpointKey)
	if err != nil {
		lastOffset = 0
	}

	// 获取最新offset
	latestOffset, err := c.source.GetLatestOffset(ctx, topic)
	if err != nil {
		return err
	}

	// 批量同步
	currentOffset := lastOffset
	for currentOffset < latestOffset {
		messages, err := c.source.Read(ctx, topic, currentOffset, c.batchSize)
		if err != nil {
			return err
		}

		for _, msg := range messages {
			if err := c.target.Send(ctx, msg.Topic, string(msg.Key), msg.Value); err != nil {
				return err
			}
			currentOffset = msg.Offset + 1
		}

		// 更新检查点
		if err := c.checkpoint.Set(ctx, checkpointKey, currentOffset); err != nil {
			return err
		}
	}

	return nil
}
```

### 7.5 常见问题与解决方案

#### 7.5.1 问题排查指南

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          常见问题排查指南                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  问题一：消息堆积                                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  症状：consumer lag 持续增长，消费速度跟不上生产速度                    │  │
│  │                                                                       │  │
│  │  排查步骤：                                                            │  │
│  │  1. 检查消费者实例数量是否足够                                         │  │
│  │     → kafka-consumer-groups.sh --describe --group <group>             │  │
│  │                                                                       │  │
│  │  2. 检查单条消息处理耗时                                               │  │
│  │     → 查看 consumer_processing_time 指标                              │  │
│  │                                                                       │  │
│  │  3. 检查是否有慢消费者阻塞                                             │  │
│  │     → 查看是否有线程阻塞或死锁                                         │  │
│  │                                                                       │  │
│  │  解决方案：                                                            │  │
│  │  • 增加消费者实例数（不超过分区数）                                    │  │
│  │  • 增加分区数量                                                        │  │
│  │  • 优化消息处理逻辑                                                    │  │
│  │  • 使用批量消费                                                        │  │
│  │  • 异步处理非关键逻辑                                                  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  问题二：消息重复消费                                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  症状：同一消息被处理多次，数据重复                                    │  │
│  │                                                                       │  │
│  │  原因分析：                                                            │  │
│  │  1. 消费者处理完成但offset提交失败                                     │  │
│  │  2. 消费者Rebalance导致offset回退                                     │  │
│  │  3. 生产者重试导致消息重复发送                                         │  │
│  │                                                                       │  │
│  │  解决方案：                                                            │  │
│  │  • 实现幂等消费（见7.2节）                                             │  │
│  │  • 使用唯一消息ID + 去重表                                             │  │
│  │  • 启用生产者幂等性 (enable.idempotence=true)                         │  │
│  │  • 使用事务消息                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  问题三：消息丢失                                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  症状：生产的消息在消费端找不到                                        │  │
│  │                                                                       │  │
│  │  排查步骤：                                                            │  │
│  │  1. 检查生产者是否收到确认                                             │  │
│  │     → 查看 producer acks 配置和回调结果                               │  │
│  │                                                                       │  │
│  │  2. 检查broker是否正常存储                                             │  │
│  │     → kafka-console-consumer.sh --from-beginning                      │  │
│  │                                                                       │  │
│  │  3. 检查消费者是否正常消费                                             │  │
│  │     → 查看消费者日志和offset                                          │  │
│  │                                                                       │  │
│  │  预防措施：                                                            │  │
│  │  • 生产者：acks=all, retries>0, enable.idempotence=true              │  │
│  │  • Broker：min.insync.replicas>=2, unclean.leader.election=false     │  │
│  │  • 消费者：auto.commit=false, 处理后手动提交                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  问题四：消息乱序                                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  症状：消息消费顺序与发送顺序不一致                                    │  │
│  │                                                                       │  │
│  │  原因分析：                                                            │  │
│  │  1. 消息发送到不同分区                                                 │  │
│  │  2. 生产者重试导致顺序变化                                             │  │
│  │  3. 多消费者并行处理                                                   │  │
│  │                                                                       │  │
│  │  解决方案：                                                            │  │
│  │  • 使用相同的partition key确保相关消息在同一分区                       │  │
│  │  • 设置 max.in.flight.requests.per.connection=1                       │  │
│  │  • 单分区单消费者                                                      │  │
│  │  • 业务层面处理乱序（使用版本号或时间戳）                              │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  问题五：Rebalance 频繁                                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  症状：消费组频繁触发Rebalance，消费暂停                               │  │
│  │                                                                       │  │
│  │  原因分析：                                                            │  │
│  │  1. 消费者处理时间过长，超过max.poll.interval.ms                      │  │
│  │  2. 消费者频繁加入/退出                                                │  │
│  │  3. 网络不稳定导致心跳超时                                             │  │
│  │                                                                       │  │
│  │  解决方案：                                                            │  │
│  │  • 增加 max.poll.interval.ms（如10分钟）                              │  │
│  │  • 减少 max.poll.records（每次拉取更少消息）                          │  │
│  │  • 使用 Static Group Membership（固定成员）                           │  │
│  │  • 优化消息处理速度                                                    │  │
│  │  • 使用 Cooperative Sticky Assignor                                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 7.5.2 性能调优检查清单

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          性能调优检查清单                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  □ 生产者调优                                                               │
│    ├─ □ batch.size: 根据消息大小调整（默认16KB，建议64-256KB）             │
│    ├─ □ linger.ms: 允许适当延迟以增加批次大小（5-100ms）                   │
│    ├─ □ buffer.memory: 足够的发送缓冲区（默认32MB）                        │
│    ├─ □ compression.type: 启用压缩（lz4/zstd）                             │
│    ├─ □ max.in.flight.requests: 平衡吞吐和顺序（1-5）                      │
│    └─ □ acks: 根据可靠性需求选择（1/all）                                  │
│                                                                             │
│  □ Broker调优                                                               │
│    ├─ □ num.network.threads: CPU核心数                                     │
│    ├─ □ num.io.threads: CPU核心数 * 2                                      │
│    ├─ □ socket.send.buffer.bytes: 增加网络缓冲区（128KB-1MB）              │
│    ├─ □ socket.receive.buffer.bytes: 增加网络缓冲区（128KB-1MB）           │
│    ├─ □ log.flush.interval.messages: 批量刷盘（10000+）                    │
│    ├─ □ replica.fetch.max.bytes: 足够大以避免副本延迟                      │
│    └─ □ log.segment.bytes: 合适的段大小（1GB）                             │
│                                                                             │
│  □ 消费者调优                                                               │
│    ├─ □ fetch.min.bytes: 增加批量拉取效率（1KB-1MB）                       │
│    ├─ □ fetch.max.wait.ms: 允许等待更多数据（100-500ms）                   │
│    ├─ □ max.poll.records: 每次拉取的记录数（500-2000）                     │
│    ├─ □ 消费者数量: 等于或小于分区数                                       │
│    ├─ □ 批量处理: 减少数据库/网络调用次数                                  │
│    └─ □ 异步处理: 非关键路径异步化                                         │
│                                                                             │
│  □ 系统级调优                                                               │
│    ├─ □ 文件描述符限制: ulimit -n 100000+                                  │
│    ├─ □ 内存映射限制: vm.max_map_count=262144+                             │
│    ├─ □ TCP参数优化: tcp_wmem/tcp_rmem                                     │
│    ├─ □ 磁盘IO: 使用SSD，避免RAID5                                         │
│    ├─ □ 网络: 万兆网卡，禁用swap                                           │
│    └─ □ JVM调优: G1GC，适当堆大小（6-8GB）                                 │
│                                                                             │
│  □ 架构级优化                                                               │
│    ├─ □ 分区策略: 根据业务均匀分布                                         │
│    ├─ □ 副本因子: 生产环境至少3副本                                        │
│    ├─ □ Topic设计: 避免过多小Topic                                         │
│    ├─ □ 消息设计: 合适的消息大小（<1MB）                                   │
│    └─ □ 序列化: 高效序列化（Protobuf/Avro）                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.6 总结

#### 7.6.1 中间件选型建议

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          中间件选型决策树                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                        ┌─────────────────┐                                  │
│                        │   交易系统需求   │                                  │
│                        └────────┬────────┘                                  │
│                                 │                                           │
│              ┌──────────────────┼──────────────────┐                       │
│              │                  │                  │                       │
│              ▼                  ▼                  ▼                       │
│     ┌────────────────┐ ┌────────────────┐ ┌────────────────┐              │
│     │ 高吞吐量需求   │ │ 复杂路由需求   │ │ 事务消息需求   │              │
│     │ (>100K/s)      │ │ 灵活订阅       │ │ 顺序消息       │              │
│     └───────┬────────┘ └───────┬────────┘ └───────┬────────┘              │
│             │                  │                  │                       │
│             ▼                  ▼                  ▼                       │
│     ┌────────────────┐ ┌────────────────┐ ┌────────────────┐              │
│     │     Kafka      │ │   RabbitMQ     │ │   RocketMQ     │              │
│     └────────────────┘ └────────────────┘ └────────────────┘              │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│  场景推荐：                                                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CEX（中心化交易所）                                                 │   │
│  │  ├─ 行情分发：Kafka（高吞吐、分区有序）                              │   │
│  │  ├─ 订单处理：RocketMQ（事务消息、顺序消息）                         │   │
│  │  └─ 通知系统：RabbitMQ（灵活路由、延迟消息）                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  DEX（去中心化交易所）                                               │   │
│  │  ├─ 链上事件：Kafka（高吞吐、实时处理）                              │   │
│  │  ├─ 交易路由：Kafka Streams（流处理）                                │   │
│  │  └─ 价格聚合：Redis Pub/Sub + Kafka                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  量化交易系统                                                        │   │
│  │  ├─ 行情订阅：Kafka/自研内存队列（超低延迟）                         │   │
│  │  ├─ 信号分发：Redis Pub/Sub（毫秒级）                                │   │
│  │  └─ 订单审计：Kafka（持久化、可回放）                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 7.6.2 核心原则总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       消息中间件最佳实践核心原则                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 可靠性优先                                                              │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • 资金相关消息必须保证不丢失（At Least Once + 幂等）              │    │
│     │ • 使用持久化存储和同步复制                                        │    │
│     │ • 实现完善的重试和补偿机制                                        │    │
│     │ • 关键路径启用事务消息                                            │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  2. 性能与可靠性平衡                                                        │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • 根据业务重要性选择不同的可靠性级别                              │    │
│     │ • 非关键消息可适当降低可靠性换取性能                              │    │
│     │ • 使用批量处理提升吞吐量                                          │    │
│     │ • 合理设置超时和重试策略                                          │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  3. 幂等性设计                                                              │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • 所有消费者必须实现幂等处理                                      │    │
│     │ • 使用唯一ID + 去重机制                                           │    │
│     │ • 利用状态机防止重复状态转换                                      │    │
│     │ • 乐观锁处理并发更新                                              │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  4. 监控告警完善                                                            │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • 监控生产者发送成功率和延迟                                      │    │
│     │ • 监控消费者lag和处理延迟                                         │    │
│     │ • 监控死信队列消息数量                                            │    │
│     │ • 设置合理的告警阈值和升级机制                                    │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  5. 容灾高可用                                                              │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • 至少3副本部署，跨机架/可用区                                    │    │
│     │ • 实现故障自动切换                                                │    │
│     │ • 制定数据同步和补偿策略                                          │    │
│     │ • 定期进行容灾演练                                                │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  6. 运维最佳实践                                                            │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • 版本升级采用滚动方式                                            │    │
│     │ • 扩容前评估影响范围                                              │    │
│     │ • 保留足够的日志用于问题排查                                      │    │
│     │ • 建立标准的运维手册和应急预案                                    │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

                            ╔═══════════════════════╗
                            ║   文档完成 - v1.0     ║
                            ╚═══════════════════════╝
```

---

## 附录

### 附录A：常用命令速查

```bash
# ═══════════════════════════════════════════════════════════════════════════
# Kafka 常用命令
# ═══════════════════════════════════════════════════════════════════════════

# 创建Topic
kafka-topics.sh --create --topic orders --partitions 12 --replication-factor 3 --bootstrap-server localhost:9092

# 查看Topic列表
kafka-topics.sh --list --bootstrap-server localhost:9092

# 查看Topic详情
kafka-topics.sh --describe --topic orders --bootstrap-server localhost:9092

# 查看消费组
kafka-consumer-groups.sh --list --bootstrap-server localhost:9092

# 查看消费组详情（包含lag）
kafka-consumer-groups.sh --describe --group my-group --bootstrap-server localhost:9092

# 重置offset
kafka-consumer-groups.sh --reset-offsets --group my-group --topic orders --to-earliest --execute --bootstrap-server localhost:9092

# 消费消息（调试）
kafka-console-consumer.sh --topic orders --from-beginning --bootstrap-server localhost:9092

# 生产消息（调试）
kafka-console-producer.sh --topic orders --bootstrap-server localhost:9092

# ═══════════════════════════════════════════════════════════════════════════
# RabbitMQ 常用命令
# ═══════════════════════════════════════════════════════════════════════════

# 查看队列
rabbitmqctl list_queues name messages consumers

# 查看交换机
rabbitmqctl list_exchanges name type

# 查看绑定
rabbitmqctl list_bindings

# 查看连接
rabbitmqctl list_connections

# 查看通道
rabbitmqctl list_channels

# 清空队列
rabbitmqctl purge_queue queue_name

# ═══════════════════════════════════════════════════════════════════════════
# RocketMQ 常用命令
# ═══════════════════════════════════════════════════════════════════════════

# 查看集群信息
mqadmin clusterList -n localhost:9876

# 查看Topic列表
mqadmin topicList -n localhost:9876

# 查看Topic详情
mqadmin topicStatus -n localhost:9876 -t orders

# 查看消费进度
mqadmin consumerProgress -n localhost:9876 -g my-group

# 查看消息
mqadmin queryMsgByOffset -n localhost:9876 -t orders -b broker-a -i 0 -o 0

# 重置消费位点
mqadmin resetOffsetByTime -n localhost:9876 -g my-group -t orders -s now
```

### 附录B：配置模板

```yaml
# ═══════════════════════════════════════════════════════════════════════════
# Kafka 生产环境配置模板
# ═══════════════════════════════════════════════════════════════════════════
# server.properties

broker.id=1
listeners=PLAINTEXT://:9092
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

log.dirs=/data/kafka-logs
num.partitions=12
num.recovery.threads.per.data.dir=4

offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
zookeeper.connection.timeout.ms=18000

group.initial.rebalance.delay.ms=3000

# 可靠性配置
min.insync.replicas=2
unclean.leader.election.enable=false
auto.create.topics.enable=false
default.replication.factor=3
```

```yaml
# ═══════════════════════════════════════════════════════════════════════════
# RabbitMQ 生产环境配置模板
# ═══════════════════════════════════════════════════════════════════════════
# rabbitmq.conf

# 网络配置
listeners.tcp.default = 5672
management.tcp.port = 15672

# 内存配置
vm_memory_high_watermark.relative = 0.6
vm_memory_high_watermark_paging_ratio = 0.75

# 磁盘配置
disk_free_limit.absolute = 5GB

# 集群配置
cluster_partition_handling = autoheal
cluster_keepalive_interval = 10000

# 队列配置
queue_master_locator = min-masters

# 消息配置
channel_max = 2047
heartbeat = 60

# 日志配置
log.file.level = info
log.console = true
log.console.level = info
```

---

**文档版本**：v1.0
**最后更新**：2024年
