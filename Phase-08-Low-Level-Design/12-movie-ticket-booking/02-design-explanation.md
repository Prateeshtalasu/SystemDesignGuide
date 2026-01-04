# 🎬 Movie Ticket Booking System - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Movie Ticket Booking System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Seat` | Represent physical seat | Seat properties change |
| `ShowSeat` | Track seat status for a show | Status logic changes |
| `Screen` | Manage seat layout | Layout changes |
| `Show` | Represent a movie showing | Show model changes |
| `Movie` | Store movie information | Movie data changes |
| `Theater` | Manage screens and shows | Theater config changes |
| `SeatLock` | Handle temporary seat locks | Lock logic changes |
| `Booking` | Store confirmed booking | Booking model changes |
| `BookingService` | Coordinate booking operations | Business logic changes |

**SRP in Action:**

```java
// Seat ONLY represents physical seat
public class Seat {
    private final String row;
    private final int number;
    private final SeatType type;
}

// ShowSeat handles status for a specific show
public class ShowSeat {
    private final Seat seat;
    private SeatStatus status;
    
    public synchronized boolean lock(String userId, long duration) { }
    public synchronized boolean book() { }
}

// SeatLock manages temporary locks
public class SeatLock {
    private final List<ShowSeat> seats;
    private final LocalDateTime expiresAt;
    
    public boolean isExpired() { }
    public void release() { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Seat Types:**

```java
public enum SeatType {
    ECONOMY(new BigDecimal("8.00")),
    STANDARD(new BigDecimal("12.00")),
    PREMIUM(new BigDecimal("15.00")),
    VIP(new BigDecimal("25.00")),
    RECLINER(new BigDecimal("30.00")),  // New!
    COUPLE(new BigDecimal("40.00"));     // New!
}
```

**Adding New Pricing Strategies:**

```java
public interface PricingStrategy {
    BigDecimal calculatePrice(ShowSeat seat, Show show);
}

public class WeekendPricingStrategy implements PricingStrategy {
    @Override
    public BigDecimal calculatePrice(ShowSeat seat, Show show) {
        BigDecimal base = seat.getSeat().getType().getBasePrice();
        if (isWeekend(show.getStartTime())) {
            return base.multiply(new BigDecimal("1.25"));
        }
        return base;
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All seat types behave consistently:**

```java
// Any SeatType works in ShowSeat
ShowSeat economySeat = new ShowSeat(new Seat("J", 1, SeatType.ECONOMY), price);
ShowSeat vipSeat = new ShowSeat(new Seat("A", 1, SeatType.VIP), price);

// Both can be locked, booked, released
economySeat.lock(userId, duration);
vipSeat.lock(userId, duration);
```

---

### 4. Interface Segregation Principle (ISP)

**Could be improved:**

```java
public interface Lockable {
    boolean lock(String userId, long duration);
    boolean unlock(String userId);
    boolean isLocked();
}

public interface Bookable {
    boolean book();
    void release();
    boolean isBooked();
}

public class ShowSeat implements Lockable, Bookable { }
```

---

### 5. Dependency Inversion Principle (DIP)

**Better with DIP:**

```java
public interface SeatRepository {
    ShowSeat findById(String showId, String seatId);
    List<ShowSeat> findAvailable(String showId);
    void save(ShowSeat seat);
}

public interface BookingRepository {
    void save(Booking booking);
    Booking findById(String id);
    List<Booking> findByUser(String userId);
}

public class BookingService {
    private final SeatRepository seatRepo;
    private final BookingRepository bookingRepo;
}
```

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Seat represents physical seat, ShowSeat tracks status, SeatLock manages locks, Booking stores booking details, BookingService coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new seat types, pricing strategies) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All SeatType enums and PricingStrategy implementations properly implement their contracts. They are substitutable. | N/A | - |
| **ISP** | WEAK | Could benefit from Lockable and Bookable interfaces. Currently using concrete classes. Mentioned in ISP section but not fully implemented. | Extract Lockable and Bookable interfaces | More interfaces/files, but increases flexibility |
| **DIP** | WEAK | BookingService depends on concrete classes. Could depend on SeatRepository and BookingRepository interfaces. Mentioned in DIP section but not fully implemented. | Extract SeatRepository and BookingRepository interfaces | More abstraction layers, but improves testability |

---

## Design Patterns Used

### 1. Singleton Pattern (Potential)

**Where:** BookingService could be singleton

```java
public class BookingService {
    private static BookingService instance;
    
    private BookingService() { }
    
    public static synchronized BookingService getInstance() {
        if (instance == null) {
            instance = new BookingService();
        }
        return instance;
    }
}
```

---

### 2. Factory Pattern

**Where:** Creating shows with proper initialization

```java
public class ShowFactory {
    public Show createShow(Movie movie, Screen screen, LocalDateTime time,
                          PricingStrategy pricing) {
        Show show = new Show(movie, screen, time);
        
        // Initialize seats with pricing
        for (Seat seat : screen.getAllSeats()) {
            BigDecimal price = pricing.calculatePrice(seat, show);
            show.addShowSeat(new ShowSeat(seat, price));
        }
        
        return show;
    }
}
```

---

### 3. Strategy Pattern

**Where:** Pricing strategies

```java
public interface PricingStrategy {
    BigDecimal calculatePrice(Seat seat, Show show);
}

public class StandardPricing implements PricingStrategy { }
public class WeekendPricing implements PricingStrategy { }
public class HolidayPricing implements PricingStrategy { }
```

---

### 4. State Pattern (Implicit)

**Where:** ShowSeat status management

```java
public class ShowSeat {
    private SeatStatus status;  // AVAILABLE, LOCKED, BOOKED
    
    public boolean lock(...) {
        if (status != AVAILABLE) return false;
        status = LOCKED;
        return true;
    }
    
    public boolean book() {
        if (status != LOCKED) return false;
        status = BOOKED;
        return true;
    }
}
```

---

## Concurrency Handling

### Seat Locking Mechanism

```java
public synchronized boolean lock(String userId, long durationMs) {
    if (status != SeatStatus.AVAILABLE) {
        return false;
    }
    this.status = SeatStatus.LOCKED;
    this.lockedBy = userId;
    this.lockExpiry = System.currentTimeMillis() + durationMs;
    return true;
}
```

**Why synchronized?**
- Prevents race condition when two users try to lock same seat
- Ensures atomic check-and-set operation

### Lock Expiration

```java
public synchronized boolean isLockExpired() {
    return status == SeatStatus.LOCKED && 
           System.currentTimeMillis() > lockExpiry;
}

public synchronized void checkAndReleaseLock() {
    if (isLockExpired()) {
        release();
    }
}
```

**Why automatic expiration?**
- Prevents seats being locked indefinitely
- User abandons booking → seats become available again
- Background cleanup service releases expired locks

---

## Why Alternatives Were Rejected

### Alternative 1: Single Seat Status in Seat Class

```java
// Rejected
public class Seat {
    private SeatStatus status;  // AVAILABLE, BOOKED
    private String bookedBy;
    private String showId;
}
```

**Why rejected:**
- Can't handle same seat for multiple shows
- Status tied to specific show, not physical seat
- Hard to query availability across shows

**What breaks:**
- Same seat can't be available for Show A and booked for Show B
- Need separate status per show

### Alternative 2: No Locking Mechanism

```java
// Rejected
public Booking bookSeats(String showId, List<String> seatIds, String userId) {
    // Direct booking without lock
    for (String seatId : seatIds) {
        showSeat.book();  // Immediate booking
    }
}
```

**Why rejected:**
- No time for user to complete payment
- Seats booked immediately, can't abandon
- Race conditions when multiple users select same seat

**What breaks:**
- Users can't complete payment flow
- Seats locked even if user abandons
- Concurrent booking conflicts

### Alternative 3: Global Lock for All Bookings

```java
// Rejected
public class BookingService {
    private final Object globalLock = new Object();
    
    public Booking bookSeats(...) {
        synchronized (globalLock) {
            // All bookings serialized
        }
    }
}
```

**Why rejected:**
- All bookings blocked even for different shows
- Poor performance and scalability
- Unnecessary contention

**What breaks:**
- Users booking different shows block each other
- System throughput severely limited
- Can't scale to multiple theaters

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle partial booking failures?

**Answer:**

```java
public Booking confirmBookingWithRollback(String lockId, String userId) {
    SeatLock lock = validateLock(lockId, userId);
    List<ShowSeat> bookedSeats = new ArrayList<>();
    
    try {
        for (ShowSeat seat : lock.getSeats()) {
            if (!seat.book()) {
                throw new BookingException("Failed to book: " + seat);
            }
            bookedSeats.add(seat);
        }
        
        return createBooking(lock, userId);
        
    } catch (Exception e) {
        // Rollback: release all booked seats
        for (ShowSeat seat : bookedSeats) {
            seat.release();
        }
        throw e;
    }
}
```

---

### Q2: How would you implement waiting list?

**Answer:**

```java
public class WaitingListService {
    private final Map<String, Queue<WaitRequest>> waitingLists;
    
    public void addToWaitingList(String showId, String userId, int seatCount) {
        waitingLists.computeIfAbsent(showId, k -> new LinkedList<>())
            .offer(new WaitRequest(userId, seatCount, LocalDateTime.now()));
    }
    
    public void onBookingCancelled(String showId, int releasedSeats) {
        Queue<WaitRequest> queue = waitingLists.get(showId);
        if (queue == null) return;
        
        // Notify users who can now book
        for (WaitRequest request : queue) {
            if (request.getSeatCount() <= releasedSeats) {
                notifyUser(request.getUserId(), showId);
                break;
            }
        }
    }
}
```

---

### Q3: How would you implement seat hold for VIP users?

**Answer:**

```java
public class VIPSeatService {
    private final Map<String, Set<String>> vipHolds;  // showId -> seatIds
    
    public void holdSeatsForVIP(String showId, List<String> seatIds, 
                                LocalDateTime releaseTime) {
        vipHolds.computeIfAbsent(showId, k -> new HashSet<>())
            .addAll(seatIds);
        
        // Schedule release
        scheduler.schedule(() -> releaseVIPHold(showId, seatIds),
            Duration.between(LocalDateTime.now(), releaseTime).toMillis(),
            TimeUnit.MILLISECONDS);
    }
    
    public boolean isSeatAvailableForRegularUser(String showId, String seatId) {
        Set<String> held = vipHolds.get(showId);
        return held == null || !held.contains(seatId);
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add seat categories** - Wheelchair accessible, aisle seats
2. **Add group booking** - Book adjacent seats together
3. **Add loyalty points** - Earn/redeem points
4. **Add show reviews** - User ratings and reviews
5. **Add food ordering** - Pre-order snacks with tickets
6. **Add notifications** - Show reminders, booking confirmations
7. **Add dynamic pricing** - Adjust prices based on demand
8. **Add seat recommendations** - Suggest optimal seats based on preferences

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `searchShows` | O(t × s) | t = theaters, s = shows |
| `getAvailableSeats` | O(n) | n = seats in show |
| `lockSeats` | O(k) | k = seats to lock |
| `confirmBooking` | O(k) | k = seats to book |
| `cancelBooking` | O(k) | k = seats to release |

### Space Complexity

| Component | Space |
|-----------|-------|
| Shows | O(s) |
| ShowSeats per show | O(n) | n = seats |
| Bookings | O(b) |
| SeatLocks | O(l) active locks |

### Bottlenecks at Scale

**10x Usage (100 → 1K shows/day, 1K → 10K concurrent bookings):**
- Problem: Seat locking becomes contention bottleneck, concurrent seat selection conflicts increase, lock cleanup overhead grows
- Solution: Use distributed locking (Redis) with expiration, implement lock timeout optimization, batch lock operations
- Tradeoff: Additional infrastructure (Redis), network latency for lock operations

**100x Usage (100 → 10K shows/day, 1K → 100K concurrent bookings):**
- Problem: Single instance can't handle all shows, lock storage exceeds memory, real-time seat availability queries too slow
- Solution: Shard shows by theater/region, use distributed lock service (Redis cluster), implement caching layer for seat availability
- Tradeoff: Distributed system complexity, need lock coordination across shards


### Q1: How would you handle seat selection UI?

```java
public class SeatMap {
    public String[][] generateSeatMap(Show show) {
        List<List<Seat>> layout = show.getScreen().getSeatLayout();
        String[][] map = new String[layout.size()][];
        
        for (int r = 0; r < layout.size(); r++) {
            List<Seat> row = layout.get(r);
            map[r] = new String[row.size()];
            
            for (int s = 0; s < row.size(); s++) {
                ShowSeat showSeat = show.getShowSeat(row.get(s).getId());
                map[r][s] = getSeatSymbol(showSeat);
            }
        }
        return map;
    }
    
    private String getSeatSymbol(ShowSeat seat) {
        switch (seat.getStatus()) {
            case AVAILABLE: return "[ ]";
            case LOCKED: return "[⌛]";
            case BOOKED: return "[X]";
            default: return "[?]";
        }
    }
}
```

### Q2: How would you implement seat recommendations?

```java
public class SeatRecommendationService {
    public List<List<ShowSeat>> recommendSeats(Show show, int count) {
        List<List<ShowSeat>> recommendations = new ArrayList<>();
        
        // Strategy 1: Consecutive seats in middle rows
        recommendations.addAll(findConsecutiveSeats(show, count, "middle"));
        
        // Strategy 2: Best available (by score)
        recommendations.add(findBestAvailable(show, count));
        
        return recommendations;
    }
    
    private int calculateSeatScore(ShowSeat seat, Screen screen) {
        int score = 0;
        Seat s = seat.getSeat();
        
        // Middle rows are better
        int middleRow = screen.getSeatLayout().size() / 2;
        int rowDiff = Math.abs(s.getRow().charAt(0) - 'A' - middleRow);
        score -= rowDiff * 10;
        
        // Middle seats in row are better
        int middleSeat = screen.getSeatLayout().get(0).size() / 2;
        int seatDiff = Math.abs(s.getNumber() - middleSeat);
        score -= seatDiff * 5;
        
        return score;
    }
}
```

### Q3: How would you handle payment integration?

```java
public interface PaymentGateway {
    PaymentResult processPayment(PaymentRequest request);
    RefundResult processRefund(String transactionId, BigDecimal amount);
}

public class BookingService {
    private final PaymentGateway paymentGateway;
    
    public Booking confirmBooking(String lockId, String userId, 
                                  PaymentDetails payment) {
        SeatLock lock = validateLock(lockId, userId);
        
        // Process payment first
        PaymentResult result = paymentGateway.processPayment(
            new PaymentRequest(lock.getTotalAmount(), payment));
        
        if (!result.isSuccess()) {
            throw new PaymentFailedException(result.getError());
        }
        
        try {
            // Create booking
            return createBooking(lock, userId, result.getTransactionId());
        } catch (Exception e) {
            // Refund on failure
            paymentGateway.processRefund(result.getTransactionId(), 
                                        lock.getTotalAmount());
            throw e;
        }
    }
}
```

### Q4: How would you scale for high traffic?

```java
// Use distributed locking with Redis
public class DistributedSeatLock {
    private final RedisClient redis;
    
    public boolean lockSeat(String showId, String seatId, String userId, 
                           long durationMs) {
        String key = "lock:" + showId + ":" + seatId;
        String value = userId + ":" + System.currentTimeMillis();
        
        // SET key value NX PX duration
        return redis.set(key, value, SetParams.setParams()
            .nx()  // Only if not exists
            .px(durationMs));
    }
    
    public boolean unlockSeat(String showId, String seatId, String userId) {
        String key = "lock:" + showId + ":" + seatId;
        
        // Lua script for atomic check-and-delete
        String script = 
            "if redis.call('get', KEYS[1]):match('^' .. ARGV[1]) then " +
            "  return redis.call('del', KEYS[1]) " +
            "else return 0 end";
        
        return (Long) redis.eval(script, 1, key, userId) == 1;
    }
}
```

