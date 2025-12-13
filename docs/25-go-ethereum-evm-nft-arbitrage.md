# Go-Ethereum EVM 深入解析：NFT 套利

## 概述

NFT 市场的碎片化和信息不对称创造了独特的套利机会。本文介绍 NFT 地板价套利、稀有度套利和跨市场套利的实现。

## 1. NFT 市场基础

### 1.1 市场架构

```
┌─────────────────────────────────────────────────────────────┐
│                      NFT 市场生态                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  交易市场                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  OpenSea (Seaport)   - 最大市场                      │   │
│  │  Blur               - 专业交易者                     │   │
│  │  LooksRare          - 代币激励                       │   │
│  │  X2Y2               - 低费用                         │   │
│  │  Sudoswap           - AMM 模式                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  聚合器                                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Blur Aggregator    - 最优路由                       │   │
│  │  Gem (OpenSea)      - 批量购买                       │   │
│  │  Reservoir          - 统一 API                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  NFT-Fi                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  NFTfi              - NFT 抵押借贷                   │   │
│  │  BendDAO            - 蓝筹 NFT 借贷                  │   │
│  │  Blur Lending       - 即时借贷                       │   │
│  │  NFTX               - NFT 碎片化                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心数据结构

```go
package nft

import (
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

// NFTCollection NFT 集合
type NFTCollection struct {
	Address     common.Address `json:"address"`
	Name        string         `json:"name"`
	Symbol      string         `json:"symbol"`
	TotalSupply uint64         `json:"totalSupply"`
	FloorPrice  *big.Int       `json:"floorPrice"`
	Volume24h   *big.Int       `json:"volume24h"`
	Holders     uint64         `json:"holders"`
}

// NFTItem 单个 NFT
type NFTItem struct {
	Collection common.Address `json:"collection"`
	TokenID    *big.Int       `json:"tokenId"`
	Owner      common.Address `json:"owner"`
	Rarity     *RarityInfo    `json:"rarity"`
	LastSale   *SaleInfo      `json:"lastSale"`
	Listings   []*Listing     `json:"listings"`
	Offers     []*Offer       `json:"offers"`
}

// RarityInfo 稀有度信息
type RarityInfo struct {
	Rank        uint64             `json:"rank"`
	Score       float64            `json:"score"`
	Traits      map[string]string  `json:"traits"`
	TraitRarity map[string]float64 `json:"traitRarity"`
}

// Listing 挂单信息
type Listing struct {
	Marketplace string         `json:"marketplace"`
	Seller      common.Address `json:"seller"`
	Price       *big.Int       `json:"price"`
	Currency    common.Address `json:"currency"` // ETH 或 WETH
	Expiry      time.Time      `json:"expiry"`
	OrderHash   common.Hash    `json:"orderHash"`
}

// Offer 报价信息
type Offer struct {
	Marketplace  string         `json:"marketplace"`
	Buyer        common.Address `json:"buyer"`
	Price        *big.Int       `json:"price"`
	Currency     common.Address `json:"currency"`
	Expiry       time.Time      `json:"expiry"`
	IsCollection bool           `json:"isCollection"` // 集合报价
}

// SaleInfo 销售信息
type SaleInfo struct {
	Price     *big.Int       `json:"price"`
	Seller    common.Address `json:"seller"`
	Buyer     common.Address `json:"buyer"`
	Timestamp time.Time      `json:"timestamp"`
	TxHash    common.Hash    `json:"txHash"`
}

// NFTMarketplace NFT 市场接口
type NFTMarketplace interface {
	// 查询
	GetListings(ctx context.Context, collection common.Address) ([]*Listing, error)
	GetOffers(ctx context.Context, collection common.Address) ([]*Offer, error)
	GetFloorPrice(ctx context.Context, collection common.Address) (*big.Int, error)

	// 交易
	Buy(ctx context.Context, listing *Listing) (common.Hash, error)
	List(ctx context.Context, params *ListParams) (common.Hash, error)
	AcceptOffer(ctx context.Context, offer *Offer, tokenID *big.Int) (common.Hash, error)
}

type ListParams struct {
	Collection common.Address
	TokenID    *big.Int
	Price      *big.Int
	Duration   time.Duration
}
```

## 2. Seaport 协议集成

### 2.1 Seaport 客户端

```go
package seaport

import (
	"context"
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

// Seaport 合约地址
var (
	SeaportV15 = common.HexToAddress("0x00000000000000ADc04C56Bf30aC9d3c0aAF14dC")
	Conduit    = common.HexToAddress("0x1E0049783F008A0085193E00003D00cd54003c71")
)

// OrderType 订单类型
type OrderType uint8

const (
	FullOpen OrderType = iota
	PartialOpen
	FullRestricted
	PartialRestricted
)

// ItemType 物品类型
type ItemType uint8

const (
	Native ItemType = iota
	ERC20
	ERC721
	ERC1155
	ERC721WithCriteria
	ERC1155WithCriteria
)

// SeaportOrder Seaport 订单
type SeaportOrder struct {
	Parameters OrderParameters `json:"parameters"`
	Signature  []byte          `json:"signature"`
}

type OrderParameters struct {
	Offerer       common.Address      `json:"offerer"`
	Zone          common.Address      `json:"zone"`
	Offer         []OfferItem         `json:"offer"`
	Consideration []ConsiderationItem `json:"consideration"`
	OrderType     OrderType           `json:"orderType"`
	StartTime     *big.Int            `json:"startTime"`
	EndTime       *big.Int            `json:"endTime"`
	ZoneHash      [32]byte            `json:"zoneHash"`
	Salt          *big.Int            `json:"salt"`
	ConduitKey    [32]byte            `json:"conduitKey"`
	Counter       *big.Int            `json:"counter"`
}

type OfferItem struct {
	ItemType             ItemType       `json:"itemType"`
	Token                common.Address `json:"token"`
	IdentifierOrCriteria *big.Int       `json:"identifierOrCriteria"`
	StartAmount          *big.Int       `json:"startAmount"`
	EndAmount            *big.Int       `json:"endAmount"`
}

type ConsiderationItem struct {
	ItemType             ItemType       `json:"itemType"`
	Token                common.Address `json:"token"`
	IdentifierOrCriteria *big.Int       `json:"identifierOrCriteria"`
	StartAmount          *big.Int       `json:"startAmount"`
	EndAmount            *big.Int       `json:"endAmount"`
	Recipient            common.Address `json:"recipient"`
}

// SeaportClient Seaport 客户端
type SeaportClient struct {
	ethClient  *ethclient.Client
	seaportABI abi.ABI
	chainID    *big.Int
	privateKey *ecdsa.PrivateKey
}

// NewSeaportClient 创建客户端
func NewSeaportClient(ethClient *ethclient.Client, privateKey *ecdsa.PrivateKey) (*SeaportClient, error) {
	seaportABI, err := abi.JSON(strings.NewReader(SeaportABI))
	if err != nil {
		return nil, err
	}

	chainID, _ := ethClient.ChainID(context.Background())

	return &SeaportClient{
		ethClient:  ethClient,
		seaportABI: seaportABI,
		chainID:    chainID,
		privateKey: privateKey,
	}, nil
}

// FulfillBasicOrder 执行基础订单（单个 NFT 购买）
func (c *SeaportClient) FulfillBasicOrder(ctx context.Context, order *SeaportOrder) (common.Hash, error) {
	// 构建 BasicOrderParameters
	basicParams := c.orderToBasicParams(order)

	callData, err := c.seaportABI.Pack("fulfillBasicOrder", basicParams)
	if err != nil {
		return common.Hash{}, err
	}

	// 计算需要发送的 ETH
	value := c.calculatePayment(order)

	tx, err := c.sendTransaction(ctx, SeaportV15, callData, value)
	if err != nil {
		return common.Hash{}, err
	}

	return tx.Hash(), nil
}

// FulfillAdvancedOrder 执行高级订单
func (c *SeaportClient) FulfillAdvancedOrder(ctx context.Context, order *SeaportOrder, criteriaResolvers []CriteriaResolver) (common.Hash, error) {
	advancedOrder := AdvancedOrder{
		Parameters:  order.Parameters,
		Numerator:   big.NewInt(1),
		Denominator: big.NewInt(1),
		Signature:   order.Signature,
		ExtraData:   []byte{},
	}

	callData, err := c.seaportABI.Pack(
		"fulfillAdvancedOrder",
		advancedOrder,
		criteriaResolvers,
		[32]byte{}, // fulfillerConduitKey
		c.address(),
	)
	if err != nil {
		return common.Hash{}, err
	}

	value := c.calculatePayment(order)

	tx, err := c.sendTransaction(ctx, SeaportV15, callData, value)
	if err != nil {
		return common.Hash{}, err
	}

	return tx.Hash(), nil
}

// FulfillAvailableOrders 批量购买
func (c *SeaportClient) FulfillAvailableOrders(ctx context.Context, orders []*SeaportOrder) (common.Hash, error) {
	advancedOrders := make([]AdvancedOrder, len(orders))
	for i, order := range orders {
		advancedOrders[i] = AdvancedOrder{
			Parameters:  order.Parameters,
			Numerator:   big.NewInt(1),
			Denominator: big.NewInt(1),
			Signature:   order.Signature,
			ExtraData:   []byte{},
		}
	}

	// 计算 fulfillments
	offerFulfillments, considerationFulfillments := c.calculateFulfillments(orders)

	callData, err := c.seaportABI.Pack(
		"fulfillAvailableAdvancedOrders",
		advancedOrders,
		[]CriteriaResolver{},
		offerFulfillments,
		considerationFulfillments,
		[32]byte{},
		c.address(),
		big.NewInt(int64(len(orders))),
	)
	if err != nil {
		return common.Hash{}, err
	}

	value := c.calculateTotalPayment(orders)

	tx, err := c.sendTransaction(ctx, SeaportV15, callData, value)
	if err != nil {
		return common.Hash{}, err
	}

	return tx.Hash(), nil
}

// CreateOrder 创建挂单
func (c *SeaportClient) CreateOrder(ctx context.Context, params *CreateOrderParams) (*SeaportOrder, error) {
	order := &SeaportOrder{
		Parameters: OrderParameters{
			Offerer:   c.address(),
			Zone:      common.Address{},
			OrderType: FullOpen,
			StartTime: big.NewInt(time.Now().Unix()),
			EndTime:   big.NewInt(time.Now().Add(params.Duration).Unix()),
			Salt:      big.NewInt(time.Now().UnixNano()),
			Offer: []OfferItem{
				{
					ItemType:             ERC721,
					Token:                params.Collection,
					IdentifierOrCriteria: params.TokenID,
					StartAmount:          big.NewInt(1),
					EndAmount:            big.NewInt(1),
				},
			},
			Consideration: c.buildConsideration(params),
		},
	}

	// 签名订单
	signature, err := c.signOrder(order)
	if err != nil {
		return nil, err
	}
	order.Signature = signature

	return order, nil
}

func (c *SeaportClient) signOrder(order *SeaportOrder) ([]byte, error) {
	// EIP-712 签名
	orderHash := c.getOrderHash(&order.Parameters)

	domainSeparator := c.getDomainSeparator()

	data := []byte{0x19, 0x01}
	data = append(data, domainSeparator[:]...)
	data = append(data, orderHash[:]...)

	hash := crypto.Keccak256Hash(data)
	signature, err := crypto.Sign(hash.Bytes(), c.privateKey)
	if err != nil {
		return nil, err
	}

	signature[64] += 27
	return signature, nil
}

func (c *SeaportClient) buildConsideration(params *CreateOrderParams) []ConsiderationItem {
	// 计算各方分成
	// OpenSea: 2.5%, Creator: varies, Seller: remainder
	opensea := new(big.Int).Mul(params.Price, big.NewInt(250))
	opensea.Div(opensea, big.NewInt(10000))

	creator := new(big.Int).Mul(params.Price, big.NewInt(int64(params.CreatorFee)))
	creator.Div(creator, big.NewInt(10000))

	seller := new(big.Int).Sub(params.Price, opensea)
	seller.Sub(seller, creator)

	items := []ConsiderationItem{
		{
			ItemType:    Native,
			Token:       common.Address{},
			StartAmount: seller,
			EndAmount:   seller,
			Recipient:   c.address(),
		},
		{
			ItemType:    Native,
			Token:       common.Address{},
			StartAmount: opensea,
			EndAmount:   opensea,
			Recipient:   common.HexToAddress("0x0000a26b00c1F0DF003000390027140000fAa719"), // OpenSea fee recipient
		},
	}

	if creator.Sign() > 0 {
		items = append(items, ConsiderationItem{
			ItemType:    Native,
			Token:       common.Address{},
			StartAmount: creator,
			EndAmount:   creator,
			Recipient:   params.CreatorAddress,
		})
	}

	return items
}

type CreateOrderParams struct {
	Collection     common.Address
	TokenID        *big.Int
	Price          *big.Int
	Duration       time.Duration
	CreatorFee     uint64 // basis points
	CreatorAddress common.Address
}

type AdvancedOrder struct {
	Parameters  OrderParameters
	Numerator   *big.Int
	Denominator *big.Int
	Signature   []byte
	ExtraData   []byte
}

type CriteriaResolver struct {
	OrderIndex    *big.Int
	Side          uint8
	Index         *big.Int
	Identifier    *big.Int
	CriteriaProof [][32]byte
}
```

## 3. Blur 市场集成

### 3.1 Blur 客户端

```go
package blur

import (
	"context"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"time"

	"github.com/ethereum/go-ethereum/common"
)

var (
	BlurExchange = common.HexToAddress("0x000000000000Ad05Ccc4F10045630fb830B95127")
	BlurAPI      = "https://blur.io/api/v1"
)

// BlurClient Blur 客户端
type BlurClient struct {
	httpClient *http.Client
	ethClient  *ethclient.Client
	authToken  string
}

// NewBlurClient 创建客户端
func NewBlurClient(ethClient *ethclient.Client, authToken string) *BlurClient {
	return &BlurClient{
		httpClient: &http.Client{Timeout: 30 * time.Second},
		ethClient:  ethClient,
		authToken:  authToken,
	}
}

// GetCollectionStats 获取集合统计
func (c *BlurClient) GetCollectionStats(ctx context.Context, collection common.Address) (*BlurCollectionStats, error) {
	url := fmt.Sprintf("%s/collections/%s", BlurAPI, collection.Hex())

	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	req.Header.Set("Authorization", "Bearer "+c.authToken)

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result struct {
		Collection BlurCollectionStats `json:"collection"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	return &result.Collection, nil
}

// GetListings 获取挂单
func (c *BlurClient) GetListings(ctx context.Context, collection common.Address) ([]*BlurListing, error) {
	url := fmt.Sprintf("%s/collections/%s/tokens", BlurAPI, collection.Hex())

	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	req.Header.Set("Authorization", "Bearer "+c.authToken)

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result struct {
		Tokens []*BlurListing `json:"tokens"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	return result.Tokens, nil
}

// GetBids 获取集合报价
func (c *BlurClient) GetBids(ctx context.Context, collection common.Address) ([]*BlurBid, error) {
	url := fmt.Sprintf("%s/collections/%s/executable-bids", BlurAPI, collection.Hex())

	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	req.Header.Set("Authorization", "Bearer "+c.authToken)

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result struct {
		Bids []*BlurBid `json:"priceLevels"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}

	return result.Bids, nil
}

// BulkBuy 批量购买
func (c *BlurClient) BulkBuy(ctx context.Context, listings []*BlurListing) (common.Hash, error) {
	// 构建交易数据
	var executions []BlurExecution

	for _, listing := range listings {
		executions = append(executions, BlurExecution{
			Sell: listing.Order,
			Buy:  c.createMatchingBuy(listing),
		})
	}

	// 编码 calldata
	callData, err := c.encodeExecutions(executions)
	if err != nil {
		return common.Hash{}, err
	}

	// 计算总价
	totalPrice := big.NewInt(0)
	for _, l := range listings {
		totalPrice.Add(totalPrice, l.Price)
	}

	tx, err := c.sendTransaction(ctx, BlurExchange, callData, totalPrice)
	if err != nil {
		return common.Hash{}, err
	}

	return tx.Hash(), nil
}

// AcceptBid 接受报价
func (c *BlurClient) AcceptBid(ctx context.Context, bid *BlurBid, tokenID *big.Int) (common.Hash, error) {
	// 构建卖单
	sellOrder := c.createSellOrder(bid.Collection, tokenID, bid.Price)

	execution := BlurExecution{
		Sell: sellOrder,
		Buy:  bid.Order,
	}

	callData, err := c.encodeExecution(execution)
	if err != nil {
		return common.Hash{}, err
	}

	tx, err := c.sendTransaction(ctx, BlurExchange, callData, big.NewInt(0))
	if err != nil {
		return common.Hash{}, err
	}

	return tx.Hash(), nil
}

type BlurCollectionStats struct {
	Address       common.Address `json:"contractAddress"`
	Name          string         `json:"name"`
	FloorPrice    string         `json:"floorPrice"`
	FloorPriceWei *big.Int       `json:"floorPriceWei"`
	Volume24h     string         `json:"totalVolume"`
	NumOwners     uint64         `json:"numOwners"`
}

type BlurListing struct {
	TokenID    *big.Int       `json:"tokenId"`
	Collection common.Address `json:"contractAddress"`
	Price      *big.Int       `json:"price"`
	Owner      common.Address `json:"owner"`
	Order      *BlurOrder     `json:"order"`
}

type BlurBid struct {
	Collection common.Address `json:"contractAddress"`
	Price      *big.Int       `json:"price"`
	Amount     uint64         `json:"executableSize"`
	Order      *BlurOrder     `json:"order"`
}

type BlurOrder struct {
	Trader         common.Address `json:"trader"`
	Side           uint8          `json:"side"`
	MatchingPolicy common.Address `json:"matchingPolicy"`
	Collection     common.Address `json:"collection"`
	TokenId        *big.Int       `json:"tokenId"`
	Amount         *big.Int       `json:"amount"`
	PaymentToken   common.Address `json:"paymentToken"`
	Price          *big.Int       `json:"price"`
	ListingTime    *big.Int       `json:"listingTime"`
	ExpirationTime *big.Int       `json:"expirationTime"`
	Fees           []Fee          `json:"fees"`
	Salt           *big.Int       `json:"salt"`
	ExtraParams    []byte         `json:"extraParams"`
	Signature      []byte         `json:"signature"`
}

type Fee struct {
	Rate      uint16         `json:"rate"`
	Recipient common.Address `json:"recipient"`
}

type BlurExecution struct {
	Sell *BlurOrder
	Buy  *BlurOrder
}
```

## 4. NFT 套利策略

### 4.1 地板价套利

```go
package arbitrage

import (
	"context"
	"math/big"
	"sort"
	"sync"

	"github.com/ethereum/go-ethereum/common"
)

// FloorArbitrage 地板价套利
type FloorArbitrage struct {
	markets     map[string]NFTMarketplace
	collections []common.Address
	config      FloorArbConfig
}

type FloorArbConfig struct {
	MinProfitPct float64 // 最小利润百分比
	MaxGasCost   *big.Int
	BatchSize    int
}

// FloorArbOpportunity 地板价套利机会
type FloorArbOpportunity struct {
	Collection common.Address
	TokenID    *big.Int
	BuyMarket  string
	BuyPrice   *big.Int
	SellMarket string
	SellPrice  *big.Int // 最高出价
	Profit     *big.Int
	ProfitPct  float64
}

// FindOpportunities 寻找地板价套利机会
func (a *FloorArbitrage) FindOpportunities(ctx context.Context) ([]*FloorArbOpportunity, error) {
	var opportunities []*FloorArbOpportunity
	var mu sync.Mutex
	var wg sync.WaitGroup

	for _, collection := range a.collections {
		wg.Add(1)
		go func(col common.Address) {
			defer wg.Done()

			opps := a.findCollectionOpportunities(ctx, col)
			if len(opps) > 0 {
				mu.Lock()
				opportunities = append(opportunities, opps...)
				mu.Unlock()
			}
		}(collection)
	}

	wg.Wait()

	// 按利润排序
	sort.Slice(opportunities, func(i, j int) bool {
		return opportunities[i].ProfitPct > opportunities[j].ProfitPct
	})

	return opportunities, nil
}

// findCollectionOpportunities 寻找单个集合的机会
func (a *FloorArbitrage) findCollectionOpportunities(ctx context.Context, collection common.Address) []*FloorArbOpportunity {
	var opportunities []*FloorArbOpportunity

	// 获取所有市场的挂单
	allListings := make(map[string][]*Listing)
	allOffers := make(map[string][]*Offer)

	for name, market := range a.markets {
		listings, err := market.GetListings(ctx, collection)
		if err == nil {
			allListings[name] = listings
		}

		offers, err := market.GetOffers(ctx, collection)
		if err == nil {
			allOffers[name] = offers
		}
	}

	// 找出最低挂单价
	var lowestListing *Listing
	var lowestMarket string

	for market, listings := range allListings {
		for _, listing := range listings {
			if lowestListing == nil || listing.Price.Cmp(lowestListing.Price) < 0 {
				lowestListing = listing
				lowestMarket = market
			}
		}
	}

	if lowestListing == nil {
		return nil
	}

	// 找出最高出价
	var highestOffer *Offer
	var highestMarket string

	for market, offers := range allOffers {
		for _, offer := range offers {
			if offer.IsCollection { // 只考虑集合出价
				if highestOffer == nil || offer.Price.Cmp(highestOffer.Price) > 0 {
					highestOffer = offer
					highestMarket = market
				}
			}
		}
	}

	if highestOffer == nil {
		return nil
	}

	// 计算利润
	profit := new(big.Int).Sub(highestOffer.Price, lowestListing.Price)

	// 扣除预估 gas
	profit.Sub(profit, a.config.MaxGasCost)

	if profit.Sign() <= 0 {
		return nil
	}

	profitPct := float64(profit.Int64()) / float64(lowestListing.Price.Int64()) * 100

	if profitPct < a.config.MinProfitPct {
		return nil
	}

	return []*FloorArbOpportunity{
		{
			Collection: collection,
			TokenID:    lowestListing.TokenID,
			BuyMarket:  lowestMarket,
			BuyPrice:   lowestListing.Price,
			SellMarket: highestMarket,
			SellPrice:  highestOffer.Price,
			Profit:     profit,
			ProfitPct:  profitPct,
		},
	}
}

// ExecuteArbitrage 执行套利
func (a *FloorArbitrage) ExecuteArbitrage(ctx context.Context, opp *FloorArbOpportunity) error {
	buyMarket := a.markets[opp.BuyMarket]
	sellMarket := a.markets[opp.SellMarket]

	// 1. 购买 NFT
	listing := &Listing{
		Marketplace: opp.BuyMarket,
		Price:       opp.BuyPrice,
		TokenID:     opp.TokenID,
	}

	buyTx, err := buyMarket.Buy(ctx, listing)
	if err != nil {
		return fmt.Errorf("buy failed: %w", err)
	}

	// 等待确认
	if err := a.waitForTx(ctx, buyTx); err != nil {
		return err
	}

	// 2. 接受最高出价
	offer := &Offer{
		Marketplace: opp.SellMarket,
		Price:       opp.SellPrice,
	}

	sellTx, err := sellMarket.AcceptOffer(ctx, offer, opp.TokenID)
	if err != nil {
		// 如果卖出失败，尝试挂单
		return a.fallbackList(ctx, opp)
	}

	return a.waitForTx(ctx, sellTx)
}
```

### 4.2 稀有度套利

```go
package arbitrage

import (
	"context"
	"math/big"
	"sort"
)

// RarityArbitrage 稀有度套利
type RarityArbitrage struct {
	markets   map[string]NFTMarketplace
	rarityAPI RarityAPI
	config    RarityArbConfig
}

type RarityArbConfig struct {
	MinRarityDiscount float64 // 相对于稀有度的最小折扣
	MaxRankPct        float64 // 最大稀有度排名百分比 (如 top 10%)
	MinProfitETH      *big.Int
}

// RarityAPI 稀有度 API
type RarityAPI interface {
	GetRarity(collection common.Address, tokenID *big.Int) (*RarityInfo, error)
	GetCollectionRarityStats(collection common.Address) (*CollectionRarityStats, error)
}

type CollectionRarityStats struct {
	TotalSupply   uint64
	AveragePrice  *big.Int
	RarityPricing map[int]*big.Int // rank range -> avg price
}

// RarityOpportunity 稀有度套利机会
type RarityOpportunity struct {
	Collection   common.Address
	TokenID      *big.Int
	Rarity       *RarityInfo
	CurrentPrice *big.Int
	FairValue    *big.Int
	Discount     float64
	Market       string
}

// FindRarityDeals 寻找稀有度低估的 NFT
func (a *RarityArbitrage) FindRarityDeals(ctx context.Context, collection common.Address) ([]*RarityOpportunity, error) {
	var opportunities []*RarityOpportunity

	// 获取集合稀有度统计
	stats, err := a.rarityAPI.GetCollectionRarityStats(collection)
	if err != nil {
		return nil, err
	}

	// 获取所有挂单
	for name, market := range a.markets {
		listings, err := market.GetListings(ctx, collection)
		if err != nil {
			continue
		}

		for _, listing := range listings {
			// 获取稀有度信息
			rarity, err := a.rarityAPI.GetRarity(collection, listing.TokenID)
			if err != nil {
				continue
			}

			// 检查是否在目标稀有度范围内
			rankPct := float64(rarity.Rank) / float64(stats.TotalSupply) * 100
			if rankPct > a.config.MaxRankPct {
				continue
			}

			// 计算公允价值
			fairValue := a.calculateFairValue(rarity, stats)

			// 计算折扣
			discount := float64(fairValue.Int64()-listing.Price.Int64()) / float64(fairValue.Int64()) * 100

			if discount >= a.config.MinRarityDiscount {
				opportunities = append(opportunities, &RarityOpportunity{
					Collection:   collection,
					TokenID:      listing.TokenID,
					Rarity:       rarity,
					CurrentPrice: listing.Price,
					FairValue:    fairValue,
					Discount:     discount,
					Market:       name,
				})
			}
		}
	}

	// 按折扣排序
	sort.Slice(opportunities, func(i, j int) bool {
		return opportunities[i].Discount > opportunities[j].Discount
	})

	return opportunities, nil
}

// calculateFairValue 计算公允价值
func (a *RarityArbitrage) calculateFairValue(rarity *RarityInfo, stats *CollectionRarityStats) *big.Int {
	// 基于稀有度排名估算价值
	// 可以使用机器学习模型或历史数据

	// 简单方法：根据排名百分位定价
	percentile := float64(rarity.Rank) / float64(stats.TotalSupply)

	var multiplier float64
	switch {
	case percentile <= 0.01: // Top 1%
		multiplier = 5.0
	case percentile <= 0.05: // Top 5%
		multiplier = 3.0
	case percentile <= 0.10: // Top 10%
		multiplier = 2.0
	case percentile <= 0.25: // Top 25%
		multiplier = 1.5
	default:
		multiplier = 1.0
	}

	fairValue := new(big.Float).Mul(
		new(big.Float).SetInt(stats.AveragePrice),
		big.NewFloat(multiplier),
	)

	result, _ := fairValue.Int(nil)
	return result
}
```

### 4.3 跨市场套利

```go
package arbitrage

import (
	"context"
	"math/big"
	"sync"
)

// CrossMarketArbitrage 跨市场套利
type CrossMarketArbitrage struct {
	seaport  *SeaportClient
	blur     *BlurClient
	sudoswap *SudoswapClient
	config   CrossMarketConfig
}

type CrossMarketConfig struct {
	MinProfitETH *big.Int
	MaxGasPrice  *big.Int
	Collections  []common.Address
}

// CrossMarketOpportunity 跨市场套利机会
type CrossMarketOpportunity struct {
	Collection common.Address
	TokenID    *big.Int

	// 最佳购买
	BuyMarket string
	BuyPrice  *big.Int
	BuyOrder  interface{} // *SeaportOrder 或 *BlurOrder

	// 最佳出售
	SellMarket string
	SellPrice  *big.Int
	SellOrder  interface{}

	// 利润
	GrossProfit *big.Int
	GasCost     *big.Int
	NetProfit   *big.Int
}

// FindCrossMarketOpportunities 寻找跨市场套利
func (a *CrossMarketArbitrage) FindCrossMarketOpportunities(ctx context.Context) ([]*CrossMarketOpportunity, error) {
	var opportunities []*CrossMarketOpportunity
	var mu sync.Mutex
	var wg sync.WaitGroup

	for _, collection := range a.config.Collections {
		wg.Add(1)
		go func(col common.Address) {
			defer wg.Done()

			opp := a.findOpportunityForCollection(ctx, col)
			if opp != nil {
				mu.Lock()
				opportunities = append(opportunities, opp)
				mu.Unlock()
			}
		}(collection)
	}

	wg.Wait()
	return opportunities, nil
}

// findOpportunityForCollection 寻找单个集合的机会
func (a *CrossMarketArbitrage) findOpportunityForCollection(ctx context.Context, collection common.Address) *CrossMarketOpportunity {
	// 并行获取各市场数据
	var wg sync.WaitGroup
	var mu sync.Mutex

	type marketData struct {
		name     string
		listings []*Listing
		bids     []*Offer
	}

	data := make(map[string]*marketData)

	// OpenSea (Seaport)
	wg.Add(1)
	go func() {
		defer wg.Done()
		listings := a.getSeaportListings(ctx, collection)
		bids := a.getSeaportBids(ctx, collection)
		mu.Lock()
		data["opensea"] = &marketData{name: "opensea", listings: listings, bids: bids}
		mu.Unlock()
	}()

	// Blur
	wg.Add(1)
	go func() {
		defer wg.Done()
		listings := a.getBlurListings(ctx, collection)
		bids := a.getBlurBids(ctx, collection)
		mu.Lock()
		data["blur"] = &marketData{name: "blur", listings: listings, bids: bids}
		mu.Unlock()
	}()

	wg.Wait()

	// 找出最低购买价
	var bestBuy *Listing
	var bestBuyMarket string

	for market, md := range data {
		for _, listing := range md.listings {
			if bestBuy == nil || listing.Price.Cmp(bestBuy.Price) < 0 {
				bestBuy = listing
				bestBuyMarket = market
			}
		}
	}

	// 找出最高出价
	var bestSell *Offer
	var bestSellMarket string

	for market, md := range data {
		for _, bid := range md.bids {
			if bid.IsCollection {
				if bestSell == nil || bid.Price.Cmp(bestSell.Price) > 0 {
					bestSell = bid
					bestSellMarket = market
				}
			}
		}
	}

	if bestBuy == nil || bestSell == nil {
		return nil
	}

	// 计算利润
	grossProfit := new(big.Int).Sub(bestSell.Price, bestBuy.Price)

	// 估算 gas 成本
	gasCost := a.estimateGasCost(bestBuyMarket, bestSellMarket)

	netProfit := new(big.Int).Sub(grossProfit, gasCost)

	if netProfit.Cmp(a.config.MinProfitETH) < 0 {
		return nil
	}

	return &CrossMarketOpportunity{
		Collection:  collection,
		TokenID:     bestBuy.TokenID,
		BuyMarket:   bestBuyMarket,
		BuyPrice:    bestBuy.Price,
		SellMarket:  bestSellMarket,
		SellPrice:   bestSell.Price,
		GrossProfit: grossProfit,
		GasCost:     gasCost,
		NetProfit:   netProfit,
	}
}

// ExecuteAtomicArbitrage 原子化执行套利
func (a *CrossMarketArbitrage) ExecuteAtomicArbitrage(ctx context.Context, opp *CrossMarketOpportunity) error {
	// 使用闪电贷或自有资金
	// 在单笔交易中完成买入和卖出

	// 构建 multicall
	// 1. 购买 NFT
	// 2. 接受出价/挂单

	// 需要部署自定义合约来实现原子化执行
	return a.executeViaContract(ctx, opp)
}

func (a *CrossMarketArbitrage) estimateGasCost(buyMarket, sellMarket string) *big.Int {
	// 估算两笔交易的 gas 成本
	buyGas := uint64(200000)  // Seaport 约 200k
	sellGas := uint64(150000) // 接受出价约 150k

	gasPrice := big.NewInt(30e9) // 30 gwei

	totalGas := new(big.Int).SetUint64(buyGas + sellGas)
	return totalGas.Mul(totalGas, gasPrice)
}
```

## 5. 总结

### 5.1 NFT 市场对比

| 市场            | 费用   | 特点       | 适合   |
|---------------|------|----------|------|
| **OpenSea**   | 2.5% | 最大流动性    | 通用交易 |
| **Blur**      | 0%   | 专业交易者    | 高频交易 |
| **LooksRare** | 2%   | LOOKS 奖励 | 挖矿   |
| **Sudoswap**  | 0.5% | AMM 模式   | 批量交易 |

### 5.2 套利策略总结

| 策略           | 风险 | 收益 | 复杂度 |
|--------------|----|----|-----|
| 地板价套利        | 低  | 中  | 低   |
| 稀有度套利        | 中  | 高  | 中   |
| 跨市场套利        | 低  | 低  | 中   |
| Wash Trading | 高  | 高  | 高   |

### 5.3 注意事项

```
┌─────────────────────────────────────────────────────────────┐
│                     NFT 套利注意事项                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 流动性风险                                               │
│     └─ NFT 流动性低，买入后可能无法及时卖出                  │
│                                                             │
│  2. Gas 成本                                                 │
│     └─ NFT 交易 gas 较高，需计入成本                        │
│                                                             │
│  3. 版税                                                     │
│     └─ 部分集合有创作者版税 (2.5-10%)                       │
│                                                             │
│  4. 假货风险                                                 │
│     └─ 验证合约地址，防止购买假冒 NFT                       │
│                                                             │
│  5. 市场操纵                                                 │
│     └─ 警惕 wash trading 造成的虚假成交                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

NFT 套利需要结合市场理解、技术实现和风险管理，在流动性好的蓝筹项目上更容易找到机会。
