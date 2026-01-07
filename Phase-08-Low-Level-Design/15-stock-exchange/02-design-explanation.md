# 📈 Stock Exchange / Order Matching - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Stock Exchange System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Order` | Store order data | Order model changes |
| `Trade` | Store trade data | Trade model changes |
| `PriceLevel` | Manage orders at one price | Price level logic changes |
| `OrderBook` | Match orders for one symbol | Matching algorithm changes |
| `StockExchange` | Coordinate all operations | Business rules change |

**SRP in Action:**

```java
// Order ONLY stores order data
public class Order {
    private int remainingQuantity;
    
    public void fill(int quantity) { }
    public void cancel() { }
}

// PriceLevel manages orders at one price
public class PriceLevel {
    private final LinkedList<Order> orders;
    
    public void addOrder(Order order) { }
    public Order getFirstOrder() { }  // FIFO
}

// OrderBook handles matching
public class OrderBook {
    public List<Trade> match(Order order) { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Order Types:**

```java
public enum OrderType {
    MARKET,
    LIMIT,
    STOP,           // New!
    STOP_LIMIT,     // New!
    ICEBERG         // New!
}

// OrderBook can handle new types without modification
public List<Trade> match(Order order) {
    if (order.getType() == OrderType.STOP) {
        return handleStopOrder(order);
    }
    // ... existing logic
}
```

**Adding New Matching Algorithms:**

```java
public interface MatchingEngine {
    List<Trade> match(Order order, OrderBook book);
}

public class PriceTimePriority implements MatchingEngine { }
public class ProRataMatching implements MatchingEngine { }
```

---

### 3. Liskov Substitution Principle (LSP)

**All order types work in the matching engine:**

```java
public List<Trade> match(Order order) {
    // Works for MARKET, LIMIT, etc.
    while (order.getRemainingQuantity() > 0 && !oppositeBook.isEmpty()) {
        if (!pricesCross(order, bestPrice)) break;
        // Match...
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Could be improved:**

```java
public interface Matchable {
    boolean canMatchAt(BigDecimal price);
    int getRemainingQuantity();
}

public interface Cancellable {
    void cancel();
    boolean isCancellable();
}

public class Order implements Matchable, Cancellable { }
```

---

### 5. Dependency Inversion Principle (DIP)

**Better with DIP:**

```java
public interface OrderRepository {
    void save(Order order);
    Order findById(String id);
}

public interface TradeRepository {
    void save(Trade trade);
    List<Trade> findBySymbol(String symbol);
}

public class StockExchange {
    private final OrderRepository orderRepo;
    private final TradeRepository tradeRepo;
    private final MatchingEngine matchingEngine;
}
```

**Why We Use Concrete Classes in This LLD Implementation:**

For low-level design interviews, we intentionally use concrete classes instead of repository interfaces for the following reasons:

1. **In-Memory Order Books**: The system operates on in-memory order books and trade logs. Repository interfaces are more relevant for persistent storage, which is often out of scope for LLD.

2. **Core Algorithm Focus**: LLD interviews focus on order matching algorithms and trade execution logic. Adding repository abstractions shifts focus away from these core concepts.

3. **Single Implementation**: There's no requirement for multiple data access implementations in the interview context. The abstraction doesn't provide value for demonstrating LLD skills.

4. **Production vs Interview**: In production systems, we would absolutely extract `OrderRepository` and `TradeRepository` interfaces for:
   - Testability (mock repositories in unit tests)
   - Data access flexibility (swap database implementations)
   - Separation of concerns (matching logic vs data access)

**The Trade-off:**
- **Interview Scope**: Concrete classes focus on matching algorithms and trade execution
- **Production Scope**: Repository interfaces provide testability and data access flexibility

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Order stores order data, Trade stores trade data, OrderBook matches orders, StockExchange coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new order types, matching strategies) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All order types properly implement the Order contract. They are substitutable in the order book. | N/A | - |
| **ISP** | PASS | Order interface is minimal and focused. Clients only depend on what they need. No unused methods. | N/A | - |
| **DIP** | ACCEPTABLE (LLD Scope) | StockExchange depends on concrete classes. For LLD interview scope, this is acceptable as it focuses on core order matching algorithms. In production, we would depend on OrderRepository and TradeRepository interfaces. | See "Why We Use Concrete Classes" section above for detailed justification. This is an intentional design decision for interview context, not an oversight. | Interview: Simpler, focuses on core LLD skills. Production: More abstraction layers, but improves testability and data access flexibility |

---

## Design Patterns Used

### 1. Strategy Pattern (Potential)

**Where:** Matching algorithms

```java
public interface MatchingStrategy {
    List<Trade> match(Order order, TreeMap<BigDecimal, PriceLevel> book);
}

public class PriceTimePriority implements MatchingStrategy {
    // First by price, then by time
}

public class ProRataMatching implements MatchingStrategy {
    // Proportional to order size
}
```

---

### 2. Observer Pattern (Potential)

**Where:** Trade notifications

```java
public interface TradeObserver {
    void onTrade(Trade trade);
    void onOrderFilled(Order order);
    void onOrderCancelled(Order order);
}

public class MarketDataPublisher implements TradeObserver {
    @Override
    public void onTrade(Trade trade) {
        publishLastPrice(trade.getSymbol(), trade.getPrice());
    }
}
```

---

### 3. Factory Pattern (Potential)

**Where:** Creating orders

```java
public class OrderFactory {
    public static Order createMarketOrder(String symbol, String traderId,
                                         OrderSide side, int quantity) {
        return new Order(symbol, traderId, side, OrderType.MARKET, null, quantity);
    }
    
    public static Order createLimitOrder(String symbol, String traderId,
                                        OrderSide side, BigDecimal price,
                                        int quantity) {
        return new Order(symbol, traderId, side, OrderType.LIMIT, price, quantity);
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single Order List with Linear Search

**What it is:**

```java
// Rejected approach
public class OrderBook {
    private List<Order> allOrders;  // Unsorted list
    
    public List<Trade> match(Order order) {
        // Linear search through all orders
        for (Order existingOrder : allOrders) {
            if (canMatch(order, existingOrder)) {
                // Match orders
            }
        }
    }
}
```

**Why rejected:**
- O(n) search complexity for each match attempt
- No price-time priority - orders processed in insertion order
- Inefficient for large order books
- Cannot efficiently find best bid/ask

**What breaks:**
- Performance degrades significantly with many orders
- Price-time priority rules violated
- Best bid/ask queries require full list scan (O(n))
- Real-time matching becomes too slow for production

---

### Alternative 2: HashMap for Order Storage

**What it is:**

```java
// Rejected approach
public class OrderBook {
    private Map<String, Order> ordersById;  // Order ID -> Order
    
    public List<Trade> match(Order order) {
        // How to find matching orders? Need to iterate all
        for (Order existingOrder : ordersById.values()) {
            if (canMatch(order, existingOrder)) {
                // Match
            }
        }
    }
}
```

**Why rejected:**
- No price ordering - cannot efficiently find best prices
- No time ordering at same price - FIFO requirement violated
- Still requires iterating all orders for matching
- Cannot efficiently get order book depth by price

**What breaks:**
- Price-time priority matching impossible
- Cannot maintain order book structure (bids/asks by price)
- Best bid/ask queries inefficient
- Order book visualization requires full scan and sorting

---

## Order Matching Algorithms

### Price-Time Priority Algorithm

**Definition:** Orders are matched first by price (best price wins), then by time (earliest order at same price wins).

**How It Works:**

```
Matching Algorithm:

1. For BUY order:
   - Look at ASKS (sell orders)
   - Match against lowest price first
   - At same price, match oldest order first (FIFO)

2. For SELL order:
   - Look at BIDS (buy orders)
   - Match against highest price first
   - At same price, match oldest order first (FIFO)

3. Price crossing:
   - BUY crosses if: buy_price >= ask_price
   - SELL crosses if: sell_price <= bid_price
   - MARKET orders always cross

4. Trade execution:
   - Price = resting order's price (maker's price)
   - Quantity = min(incoming_qty, resting_qty)
```

**Example: Price-Time Priority Matching**

```java
// Order Book State:
// ASKS (Sell Orders):
//   $150.10 - Order A (100 shares, timestamp: 10:00:00)
//   $150.10 - Order B (50 shares, timestamp: 10:00:05)
//   $150.20 - Order C (200 shares, timestamp: 10:00:10)

// Incoming BUY order: $150.15, 150 shares, timestamp: 10:00:15

// Matching Process:
// 1. Best ASK price: $150.10
// 2. At $150.10, oldest order first: Order A (10:00:00)
// 3. Match 100 shares with Order A at $150.10
// 4. Remaining 50 shares match with Order B at $150.10
// 5. Order B fully filled, Order A partially filled

// Result:
// - Trade 1: 100 shares @ $150.10 (with Order A)
// - Trade 2: 50 shares @ $150.10 (with Order B)
// - Incoming order: 0 shares remaining (fully filled)
```

**Implementation:**

```java
public class PriceTimePriorityMatcher implements MatchingStrategy {
    
    public List<Trade> match(Order incomingOrder, OrderBook orderBook) {
        List<Trade> trades = new ArrayList<>();
        
        if (incomingOrder.getSide() == OrderSide.BUY) {
            // Match against ASKS (sell orders)
            TreeMap<BigDecimal, PriceLevel> asks = orderBook.getAsks();
            
            while (incomingOrder.hasRemainingQuantity() && !asks.isEmpty()) {
                BigDecimal bestAskPrice = asks.firstKey();
                
                // Check if price crosses
                if (incomingOrder.getPrice().compareTo(bestAskPrice) < 0) {
                    break; // No more matches possible
                }
                
                PriceLevel bestLevel = asks.get(bestAskPrice);
                
                // Match with oldest order first (FIFO)
                while (!bestLevel.isEmpty() && incomingOrder.hasRemainingQuantity()) {
                    Order restingOrder = bestLevel.getFirstOrder();
                    
                    int matchQty = Math.min(
                        incomingOrder.getRemainingQuantity(),
                        restingOrder.getRemainingQuantity()
                    );
                    
                    // Execute trade at resting order's price
                    Trade trade = new Trade(
                        incomingOrder.getSymbol(),
                        incomingOrder,
                        restingOrder,
                        bestAskPrice,  // Maker's price
                        matchQty
                    );
                    trades.add(trade);
                    
                    // Update quantities
                    incomingOrder.fill(matchQty);
                    restingOrder.fill(matchQty);
                    
                    // Remove if fully filled
                    if (restingOrder.isFullyFilled()) {
                        bestLevel.removeOrder(restingOrder);
                    }
                }
                
                // Remove price level if empty
                if (bestLevel.isEmpty()) {
                    asks.remove(bestAskPrice);
                }
            }
        } else {
            // Similar logic for SELL orders matching against BIDS
            // Match against highest bid price first, then FIFO
        }
        
        return trades;
    }
}
```

**Use Cases:**
- Stock exchanges (NYSE, NASDAQ)
- Most equity markets
- Standard matching for retail trading

**Pros:**
- Fair: First-come-first-served at same price
- Predictable: Clear matching rules
- Simple: Easy to understand and implement

**Cons:**
- Large orders may not get filled completely
- Can favor small orders over large ones

---

### Pro-Rata Matching Algorithm

**Definition:** Orders at the same price are matched proportionally based on their quantities, rather than time priority.

**How It Works:**

```
Matching Algorithm:

1. Collect all orders at best price
2. Calculate total quantity at that price
3. Match incoming order proportionally:
   - Each resting order gets: (resting_qty / total_qty) × incoming_qty
4. Round down to avoid over-allocation
5. Remaining quantity allocated by time priority
```

**Example: Pro-Rata Matching**

```java
// Order Book State:
// ASKS (Sell Orders) at $150.10:
//   Order A: 100 shares
//   Order B: 200 shares
//   Order C: 100 shares
//   Total: 400 shares

// Incoming BUY order: 200 shares

// Pro-Rata Calculation:
// Order A: (100/400) × 200 = 50 shares
// Order B: (200/400) × 200 = 100 shares
// Order C: (100/400) × 200 = 50 shares
// Total: 200 shares (fully allocated)

// Result:
// - Trade 1: 50 shares @ $150.10 (with Order A)
// - Trade 2: 100 shares @ $150.10 (with Order B)
// - Trade 3: 50 shares @ $150.10 (with Order C)
```

**Implementation:**

```java
public class ProRataMatcher implements MatchingStrategy {
    
    public List<Trade> match(Order incomingOrder, OrderBook orderBook) {
        List<Trade> trades = new ArrayList<>();
        
        if (incomingOrder.getSide() == OrderSide.BUY) {
            TreeMap<BigDecimal, PriceLevel> asks = orderBook.getAsks();
            
            while (incomingOrder.hasRemainingQuantity() && !asks.isEmpty()) {
                BigDecimal bestAskPrice = asks.firstKey();
                
                if (incomingOrder.getPrice().compareTo(bestAskPrice) < 0) {
                    break;
                }
                
                PriceLevel bestLevel = asks.get(bestAskPrice);
                List<Order> ordersAtPrice = bestLevel.getAllOrders();
                
                // Calculate total quantity at this price
                int totalQty = ordersAtPrice.stream()
                    .mapToInt(Order::getRemainingQuantity)
                    .sum();
                
                int incomingQty = incomingOrder.getRemainingQuantity();
                
                // Pro-rata allocation
                Map<Order, Integer> allocations = new HashMap<>();
                int allocatedTotal = 0;
                
                for (Order restingOrder : ordersAtPrice) {
                    int restingQty = restingOrder.getRemainingQuantity();
                    // Proportional allocation
                    int allocated = (int) ((long) restingQty * incomingQty / totalQty);
                    allocations.put(restingOrder, allocated);
                    allocatedTotal += allocated;
                }
                
                // Handle rounding: allocate remaining by time priority
                int remaining = incomingQty - allocatedTotal;
                if (remaining > 0) {
                    // Allocate remaining to oldest orders first
                    ordersAtPrice.sort(Comparator.comparing(Order::getTimestamp));
                    for (Order order : ordersAtPrice) {
                        if (remaining <= 0) break;
                        int currentAlloc = allocations.get(order);
                        int maxPossible = order.getRemainingQuantity() - currentAlloc;
                        int additional = Math.min(remaining, maxPossible);
                        allocations.put(order, currentAlloc + additional);
                        remaining -= additional;
                    }
                }
                
                // Execute trades
                for (Map.Entry<Order, Integer> entry : allocations.entrySet()) {
                    Order restingOrder = entry.getKey();
                    int matchQty = entry.getValue();
                    
                    if (matchQty > 0) {
                        Trade trade = new Trade(
                            incomingOrder.getSymbol(),
                            incomingOrder,
                            restingOrder,
                            bestAskPrice,
                            matchQty
                        );
                        trades.add(trade);
                        
                        incomingOrder.fill(matchQty);
                        restingOrder.fill(matchQty);
                        
                        if (restingOrder.isFullyFilled()) {
                            bestLevel.removeOrder(restingOrder);
                        }
                    }
                }
                
                if (bestLevel.isEmpty()) {
                    asks.remove(bestAskPrice);
                }
            }
        }
        
        return trades;
    }
}
```

**Use Cases:**
- Futures exchanges
- Options markets
- Markets with large institutional orders

**Pros:**
- Fair for large orders: All orders at same price get proportional fill
- Reduces order splitting: Large orders don't need to split into many small orders

**Cons:**
- More complex: Requires careful rounding handling
- Less predictable: Fill quantity depends on other orders at same price

---

### Time-Slicing Algorithm

**Definition:** Orders are matched in time slices (e.g., every 100ms), with all orders in a slice matched simultaneously.

**How It Works:**

```
1. Collect all orders in time window (e.g., 100ms)
2. Group by price level
3. Match all orders at same price simultaneously
4. Pro-rata within price level
5. Time priority between price levels
```

**Use Cases:**
- High-frequency trading markets
- Markets requiring batch processing

**Pros:**
- Reduces latency advantage: All orders in slice treated equally
- Fair for high-frequency traders

**Cons:**
- Adds latency: Must wait for time slice
- More complex: Requires time window management

---

### Algorithm Comparison

| Algorithm | Priority | Use Case | Complexity |
|-----------|----------|----------|------------|
| **Price-Time** | Price → Time | Stock exchanges, retail trading | Simple |
| **Pro-Rata** | Price → Quantity proportion | Futures, options, large orders | Medium |
| **Time-Slicing** | Time slice → Price → Pro-rata | High-frequency trading | Complex |

**Choosing an Algorithm:**

- **Price-Time Priority**: Default choice for most markets, simple and fair
- **Pro-Rata**: Use when large orders need proportional fills
- **Time-Slicing**: Use in high-frequency trading to reduce latency advantages

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle high-frequency trading?

**Answer:**

```java
public class LowLatencyOrderBook {
    // Use primitive arrays instead of objects
    private final long[] bidPrices;
    private final int[] bidQuantities;
    
    // Lock-free data structures
    private final AtomicReference<OrderBook> currentBook;
    
    // Pre-allocated object pools
    private final ObjectPool<Order> orderPool;
    private final ObjectPool<Trade> tradePool;
}
```

---

### Q2: How would you implement auction matching?

**Answer:**

```java
public class AuctionMatcher {
    public BigDecimal calculateAuctionPrice(OrderBook book) {
        // Find price that maximizes volume
        BigDecimal bestPrice = null;
        int maxVolume = 0;
        
        for (BigDecimal price : getAllPrices(book)) {
            int buyVolume = getBuyVolumeAtOrAbove(book, price);
            int sellVolume = getSellVolumeAtOrBelow(book, price);
            int matchVolume = Math.min(buyVolume, sellVolume);
            
            if (matchVolume > maxVolume) {
                maxVolume = matchVolume;
                bestPrice = price;
            }
        }
        
        return bestPrice;
    }
}
```

---

### Q3: How would you persist order book state?

**Answer:**

```java
public class OrderBookPersistence {
    public void snapshot(OrderBook book) {
        // Write to file/database
        List<Order> allOrders = book.getAllOrders();
        for (Order order : allOrders) {
            persist(order);
        }
    }
    
    public OrderBook restore(String symbol) {
        OrderBook book = new OrderBook(symbol);
        List<Order> orders = loadOrders(symbol);
        
        // Replay in timestamp order
        orders.sort(Comparator.comparing(Order::getTimestamp));
        for (Order order : orders) {
            if (order.isActive()) {
                book.addToBook(order);
            }
        }
        
        return book;
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add order types** - Stop, stop-limit, iceberg, FOK, IOC
2. **Add circuit breakers** - Halt trading on large moves
3. **Add market data** - Last price, OHLC, volume
4. **Add multiple matching algorithms** - Price-time, pro-rata, time-slicing
5. **Add order routing** - Route to multiple exchanges
6. **Add position tracking** - Track user positions and P&L
7. **Add risk management** - Position limits, margin requirements
8. **Add real-time streaming** - WebSocket for market data

---

### Q5: How would you implement distributed order books?

**Answer:**

```java
public class DistributedOrderBook {
    private final Map<String, OrderBookShard> shards;
    private final OrderRouter router;
    
    public List<Trade> placeOrder(Order order) {
        String symbol = order.getSymbol();
        OrderBookShard shard = getShard(symbol);
        
        // If order matches locally, execute
        List<Trade> localTrades = shard.match(order);
        
        // If not fully filled, route to other shards
        if (order.getRemainingQuantity() > 0) {
            List<Trade> remoteTrades = router.routeOrder(order);
            localTrades.addAll(remoteTrades);
        }
        
        return localTrades;
    }
    
    private OrderBookShard getShard(String symbol) {
        int shardIndex = symbol.hashCode() % shards.size();
        return shards.get(shardIndex);
    }
}
```

**Considerations:**
- Shard order books by symbol hash
- Need distributed locking for cross-shard matching
- Trade reconciliation across shards
- Market data aggregation from all shards

---

### Q6: How would you handle order cancellation performance?

**Answer:**

```java
public class OptimizedOrderBook {
    // Index orders by ID for O(1) cancellation
    private final Map<String, OrderLocation> orderIndex;
    
    private static class OrderLocation {
        PriceLevel priceLevel;
        Order order;
    }
    
    public boolean cancelOrder(String orderId) {
        OrderLocation location = orderIndex.get(orderId);
        if (location == null || !location.order.isActive()) {
            return false;
        }
        
        // O(1) removal with location reference
        location.priceLevel.removeOrder(location.order);
        location.order.cancel();
        orderIndex.remove(orderId);
        
        return true;
    }
}
```

**Optimizations:**
- Maintain order ID to location mapping
- Direct access to price level containing order
- Avoid scanning all orders at price level
- Tradeoff: Additional memory for index

---

### Q7: How would you implement order book depth aggregation?

**Answer:**

```java
public class OrderBookDepth {
    public Map<BigDecimal, Integer> getDepth(String symbol, int levels) {
        OrderBook book = exchange.getOrderBook(symbol);
        Map<BigDecimal, Integer> depth = new TreeMap<>();
        
        // Aggregate bids (descending price)
        int count = 0;
        for (Map.Entry<BigDecimal, PriceLevel> entry : 
             book.getBids().descendingMap().entrySet()) {
            if (count >= levels) break;
            depth.put(entry.getKey(), entry.getValue().getTotalQuantity());
            count++;
        }
        
        // Aggregate asks (ascending price)
        count = 0;
        for (Map.Entry<BigDecimal, PriceLevel> entry : 
             book.getAsks().entrySet()) {
            if (count >= levels) break;
            depth.put(entry.getKey(), entry.getValue().getTotalQuantity());
            count++;
        }
        
        return depth;
    }
}
```

**Use cases:**
- Display top N price levels
- Market depth visualization
- Liquidity analysis
- Order sizing decisions

---

### Q8: How would you implement trade settlement?

**Answer:**

```java
public class TradeSettlementService {
    private final AccountService accountService;
    private final SettlementQueue settlementQueue;
    
    public void settleTrade(Trade trade) {
        Settlement settlement = new Settlement(
            trade.getId(),
            trade.getBuyOrder().getTraderId(),
            trade.getSellOrder().getTraderId(),
            trade.getQuantity(),
            trade.getPrice()
        );
        
        BigDecimal totalAmount = trade.getPrice()
            .multiply(new BigDecimal(trade.getQuantity()));
        
        // Debit buyer, credit seller
        accountService.debit(trade.getBuyOrder().getTraderId(), totalAmount);
        accountService.credit(trade.getSellOrder().getTraderId(), totalAmount);
        
        settlementQueue.add(settlement);
    }
    
    public void processSettlements() {
        while (!settlementQueue.isEmpty()) {
            Settlement s = settlementQueue.poll();
            // Execute settlement (T+2, T+0, etc.)
        }
    }
}
```

**Considerations:**
- Settlement cycles (T+0, T+1, T+2)
- Account balance management
- Settlement failure handling
- Regulatory reporting requirements

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `placeOrder` (no match) | O(log P) | P = price levels |
| `placeOrder` (with match) | O(M log P) | M = matched orders |
| `cancelOrder` | O(N) | N = orders at price level |
| `getBestBid/Ask` | O(1) | TreeMap firstKey |
| `getSpread` | O(1) | Two firstKey calls |

### Space Complexity

| Component | Space |
|-----------|-------|
| Order Book per symbol | O(P × N) | P = prices, N = orders |
| All Orders | O(O) | O = total orders |
| Trades | O(T) | T = total trades |

### Bottlenecks at Scale

**10x Usage (100 → 1K orders/second, 10 → 100 symbols):**
- Problem: Order matching becomes bottleneck (O(P × N) per order), order book storage grows, trade execution overhead increases
- Solution: Optimize matching algorithm, use specialized data structures (price-level queues), implement order batching
- Tradeoff: Algorithm complexity increases, batching adds latency

**100x Usage (100 → 10K orders/second, 10 → 1K symbols):**
- Problem: Single instance can't handle all symbols/orders, matching algorithm too slow, order book memory exceeds capacity
- Solution: Shard order books by symbol, use dedicated matching engines per symbol, implement distributed order routing
- Tradeoff: Distributed system complexity, need order routing and market data distribution


### Q1: How would you handle stop orders?

```java
public class StopOrder extends Order {
    private final BigDecimal stopPrice;
    private boolean triggered;
    
    public boolean shouldTrigger(BigDecimal lastPrice) {
        if (triggered) return false;
        
        if (getSide() == OrderSide.BUY) {
            return lastPrice.compareTo(stopPrice) >= 0;
        } else {
            return lastPrice.compareTo(stopPrice) <= 0;
        }
    }
    
    public void trigger() {
        this.triggered = true;
    }
}

public class OrderBook {
    private final List<StopOrder> stopOrders;
    
    public void checkStopOrders(BigDecimal lastPrice) {
        for (StopOrder stop : stopOrders) {
            if (stop.shouldTrigger(lastPrice)) {
                stop.trigger();
                match(stop);  // Convert to market/limit order
            }
        }
    }
}
```

### Q2: How would you handle iceberg orders?

```java
public class IcebergOrder extends Order {
    private final int displayQuantity;
    private final int totalQuantity;
    private int executedQuantity;
    
    @Override
    public int getRemainingQuantity() {
        // Only show display quantity
        return Math.min(displayQuantity, 
            totalQuantity - executedQuantity);
    }
    
    public void refill() {
        // After display quantity filled, refill from hidden
        if (executedQuantity < totalQuantity) {
            // Reset visible quantity
        }
    }
}
```

### Q3: How would you implement market data feeds?

```java
public class MarketDataFeed {
    private final Map<String, List<MarketDataSubscriber>> subscribers;
    
    public void subscribe(String symbol, MarketDataSubscriber subscriber) {
        subscribers.computeIfAbsent(symbol, k -> new ArrayList<>())
            .add(subscriber);
    }
    
    public void publishTrade(Trade trade) {
        List<MarketDataSubscriber> subs = subscribers.get(trade.getSymbol());
        if (subs != null) {
            for (MarketDataSubscriber sub : subs) {
                sub.onTrade(trade);
            }
        }
    }
    
    public void publishQuote(String symbol, BigDecimal bid, BigDecimal ask) {
        // Publish best bid/ask
    }
}
```

### Q4: How would you handle order modification?

```java
public class StockExchange {
    public boolean modifyOrder(String orderId, BigDecimal newPrice, 
                              int newQuantity) {
        Order order = getOrder(orderId);
        if (order == null || !order.isActive()) {
            return false;
        }
        
        // Cancel and replace (standard approach)
        OrderBook book = orderBooks.get(order.getSymbol());
        book.cancelOrder(orderId);
        
        Order newOrder = new Order(
            order.getSymbol(),
            order.getTraderId(),
            order.getSide(),
            order.getType(),
            newPrice,
            newQuantity);
        
        placeOrder(newOrder);
        return true;
    }
}
```

