# 🏨 Hotel Booking System - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Hotel Booking System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Room` | Represent a room and its status | Room properties change |
| `Guest` | Store guest information | Guest data model changes |
| `Booking` | Track reservation details | Booking features change |
| `Hotel` | Manage hotel configuration | Hotel setup changes |
| `PricingStrategy` | Calculate prices | Pricing rules change |
| `CancellationPolicy` | Calculate refunds | Refund rules change |
| `BookingService` | Coordinate booking operations | Business logic changes |
| `Invoice` | Generate billing | Invoice format changes |

**SRP in Action:**

```java
// Room ONLY manages its own state
public class Room {
    public void reserve() { status = RESERVED; }
    public void checkIn() { status = OCCUPIED; }
    public void checkOut() { status = CLEANING; }
}

// PricingStrategy ONLY calculates prices
public interface PricingStrategy {
    BigDecimal calculatePrice(Room room, DateRange dates, int guests);
}

// CancellationPolicy ONLY calculates refunds
public interface CancellationPolicy {
    BigDecimal calculateRefund(Booking booking);
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Pricing Strategies:**

```java
// No changes to existing code
public class PromotionalPricingStrategy implements PricingStrategy {
    private final String promoCode;
    private final BigDecimal discountPercent;
    
    @Override
    public BigDecimal calculatePrice(Room room, DateRange dateRange, int guests) {
        BigDecimal basePrice = standardStrategy.calculatePrice(room, dateRange, guests);
        return basePrice.multiply(BigDecimal.ONE.subtract(discountPercent));
    }
}

public class LoyaltyPricingStrategy implements PricingStrategy {
    @Override
    public BigDecimal calculatePrice(Room room, DateRange dateRange, int guests) {
        // Apply loyalty discount based on guest tier
    }
}
```

**Adding New Cancellation Policies:**

```java
public class NonRefundableCancellationPolicy implements CancellationPolicy {
    @Override
    public BigDecimal calculateRefund(Booking booking) {
        return BigDecimal.ZERO;  // No refunds ever
    }
}

public class PartialRefundPolicy implements CancellationPolicy {
    private final BigDecimal refundPercent;
    
    @Override
    public BigDecimal calculateRefund(Booking booking) {
        return booking.getTotalPrice().multiply(refundPercent);
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All PricingStrategies are interchangeable:**

```java
public class BookingService {
    private final PricingStrategy pricingStrategy;
    
    public Booking makeBooking(...) {
        BigDecimal price = pricingStrategy.calculatePrice(room, dates, guests);
        // Works with any strategy
    }
}

// All these work correctly
new BookingService(hotel, new StandardPricingStrategy(), policy);
new BookingService(hotel, new SeasonalPricingStrategy(), policy);
new BookingService(hotel, new PromotionalPricingStrategy("SUMMER20"), policy);
```

**All CancellationPolicies are interchangeable:**

```java
public BigDecimal cancelBooking(String bookingId) {
    BigDecimal refund = cancellationPolicy.calculateRefund(booking);
    // Works with any policy
}
```

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

```java
// Minimal interfaces
public interface PricingStrategy {
    BigDecimal calculatePrice(Room room, DateRange dateRange, int guests);
}

public interface CancellationPolicy {
    BigDecimal calculateRefund(Booking booking);
    String getDescription();
}
```

**Could be extended with ISP:**

```java
public interface PriceCalculator {
    BigDecimal calculatePrice(Room room, DateRange dateRange, int guests);
}

public interface TaxCalculator {
    BigDecimal calculateTax(BigDecimal subtotal);
}

public interface DiscountCalculator {
    BigDecimal calculateDiscount(BigDecimal price, Guest guest);
}
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
// BookingService depends on abstractions (good!)
public class BookingService {
    private final PricingStrategy pricingStrategy;      // Interface
    private final CancellationPolicy cancellationPolicy; // Interface
    
    public BookingService(Hotel hotel, 
                          PricingStrategy pricingStrategy,
                          CancellationPolicy cancellationPolicy) {
        this.pricingStrategy = pricingStrategy;
        this.cancellationPolicy = cancellationPolicy;
    }
}
```

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Room represents room state, Booking tracks reservation, PricingStrategy calculates prices, CancellationPolicy calculates refunds, BookingService coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new pricing strategies, cancellation policies) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All PricingStrategy and CancellationPolicy implementations properly implement their interface contracts. They are substitutable. | N/A | - |
| **ISP** | PASS | PricingStrategy and CancellationPolicy interfaces are minimal and focused. Clients only depend on what they need. Could be extended further but current design is good. | N/A | - |
| **DIP** | PASS | BookingService depends on PricingStrategy and CancellationPolicy interfaces (abstractions), not concrete implementations. DIP is well applied. | N/A | - |

---

## Design Patterns Used

### 1. Strategy Pattern

**Where:** Pricing and Cancellation

```java
// Pricing strategies
public interface PricingStrategy {
    BigDecimal calculatePrice(Room room, DateRange dateRange, int guests);
}

public class StandardPricingStrategy implements PricingStrategy { }
public class SeasonalPricingStrategy implements PricingStrategy { }

// Cancellation strategies
public interface CancellationPolicy {
    BigDecimal calculateRefund(Booking booking);
}

public class FlexibleCancellationPolicy implements CancellationPolicy { }
public class StrictCancellationPolicy implements CancellationPolicy { }

// Context switches strategies
BookingService service = new BookingService(hotel, 
    new SeasonalPricingStrategy(), 
    new ModerateCancellationPolicy());
```

---

### 2. Builder Pattern

**Where:** Invoice and SearchCriteria

```java
// Invoice builder
Invoice invoice = Invoice.builder(booking)
    .addCharge("Room", roomCharge)
    .addCharge("Minibar", minibarCharge)
    .addCharge("Room Service", roomServiceCharge)
    .build();

// SearchCriteria fluent builder
SearchCriteria criteria = new SearchCriteria(dateRange, 2)
    .withRoomType(RoomType.SUITE)
    .withMaxPrice(new BigDecimal("500"))
    .withAmenity("City View")
    .withPreferredFloor(10);
```

---

### 3. State Pattern (Implicit)

**Where:** Room and Booking status

```java
// Room state transitions
public class Room {
    private RoomStatus status;
    
    public void reserve() {
        if (status != AVAILABLE) throw new IllegalStateException();
        status = RESERVED;
    }
    
    public void checkIn() {
        if (status != RESERVED && status != AVAILABLE) 
            throw new IllegalStateException();
        status = OCCUPIED;
    }
}
```

**State diagram:**

```
Room States:
AVAILABLE ──reserve()──► RESERVED ──checkIn()──► OCCUPIED
    ▲                        │                      │
    │                        │ cancel()             │ checkOut()
    │                        ▼                      ▼
    └─────────────────── AVAILABLE ◄─── CLEANING ◄──┘
                              ▲            │
                              │ cleaned()  │
                              └────────────┘
```

---

### 4. Repository Pattern (Implicit)

**Where:** BookingService data management

```java
public class BookingService {
    private final Map<String, Booking> bookings;
    private final Map<Room, List<DateRange>> roomBookings;
    
    public Booking getBooking(String id) {
        return bookings.get(id);
    }
    
    public List<Booking> getBookingsForDate(LocalDate date) {
        return bookings.values().stream()
            .filter(b -> b.getDateRange().contains(date))
            .collect(Collectors.toList());
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single Booking Class with All Logic

```java
// Rejected
public class Booking {
    public BigDecimal calculatePrice() { /* pricing logic */ }
    public BigDecimal calculateRefund() { /* refund logic */ }
    public boolean isRoomAvailable() { /* availability logic */ }
}
```

**Why rejected:**
- Violates SRP
- Can't change pricing without changing Booking
- Hard to test pricing independently

### Alternative 2: Room Manages Own Bookings

```java
// Rejected
public class Room {
    private List<Booking> bookings;
    
    public boolean isAvailable(DateRange dates) {
        // Check against all bookings
    }
}
```

**Why rejected:**
- Room shouldn't know about bookings
- Distributed state
- Hard to query across rooms

### Alternative 3: Inheritance for Room Types

```java
// Rejected
public class SingleRoom extends Room { }
public class DoubleRoom extends Room { }
public class Suite extends Room { }
```

**Why rejected:**
- Explosion of subclasses
- Behavior differences are minimal
- Enum is simpler

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle concurrent bookings?

**Answer:**

```java
// Option 1: Synchronized method (current)
public synchronized Booking makeBooking(...) { }

// Option 2: Per-room locking
private final Map<String, ReentrantLock> roomLocks;

public Booking makeBooking(Guest guest, Room room, DateRange dates, int guests) {
    ReentrantLock lock = roomLocks.computeIfAbsent(
        room.getRoomNumber(), k -> new ReentrantLock());
    
    lock.lock();
    try {
        if (!isRoomAvailable(room, dates)) {
            throw new IllegalStateException("Room not available");
        }
        // Create booking
        return createBooking(...);
    } finally {
        lock.unlock();
    }
}

// Option 3: Optimistic locking with version
public void makeBooking(...) {
    while (true) {
        long version = getRoomVersion(room);
        if (isRoomAvailable(room, dates)) {
            try {
                bookWithVersion(room, dates, version);
                return;
            } catch (OptimisticLockException e) {
                // Retry
            }
        }
    }
}
```

---

### Q2: How would you implement room inventory management?

**Answer:**

```java
public class InventoryManager {
    
    public Map<RoomType, InventoryStatus> getInventoryStatus(LocalDate date) {
        Map<RoomType, InventoryStatus> status = new EnumMap<>(RoomType.class);
        
        for (RoomType type : RoomType.values()) {
            List<Room> rooms = hotel.getRoomsByType(type);
            int total = rooms.size();
            int available = (int) rooms.stream()
                .filter(r -> isRoomAvailable(r, new DateRange(date, date.plusDays(1))))
                .count();
            
            status.put(type, new InventoryStatus(total, available));
        }
        
        return status;
    }
    
    public void blockRooms(List<Room> rooms, DateRange dates, String reason) {
        for (Room room : rooms) {
            roomBookings.computeIfAbsent(room, k -> new ArrayList<>())
                       .add(new BlockedDateRange(dates, reason));
        }
    }
}
```

---

### Q3: How would you implement early check-in / late check-out?

**Answer:**

```java
public class ExtendedStayService {
    private static final BigDecimal EARLY_CHECKIN_FEE = new BigDecimal("50.00");
    private static final BigDecimal LATE_CHECKOUT_FEE = new BigDecimal("50.00");
    
    public boolean requestEarlyCheckIn(Booking booking, LocalTime requestedTime) {
        Room room = booking.getRoom();
        LocalDate checkInDate = booking.getDateRange().getCheckIn();
        
        // Check if room is available early
        if (room.getStatus() == RoomStatus.AVAILABLE) {
            booking.setEarlyCheckIn(requestedTime);
            booking.addCharge("Early Check-in", EARLY_CHECKIN_FEE);
            return true;
        }
        return false;
    }
    
    public boolean requestLateCheckOut(Booking booking, LocalTime requestedTime) {
        Room room = booking.getRoom();
        LocalDate checkOutDate = booking.getDateRange().getCheckOut();
        
        // Check if room is not needed by another guest
        if (!hasBookingStarting(room, checkOutDate)) {
            booking.setLateCheckOut(requestedTime);
            booking.addCharge("Late Check-out", LATE_CHECKOUT_FEE);
            return true;
        }
        return false;
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add room service** - Track orders and charges
2. **Add housekeeping management** - Clean room scheduling
3. **Add loyalty program** - Points and rewards
4. **Add multi-property support** - Hotel chains
5. **Add rate management** - Dynamic pricing based on demand
6. **Add payment processing** - Credit card handling
7. **Add guest preferences** - Remember room preferences, amenities
8. **Add event management** - Conference rooms, event bookings

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `searchRooms` | O(r × b) | r = rooms, b = bookings per room |
| `makeBooking` | O(b) | Check against existing bookings |
| `cancelBooking` | O(b) | Find and remove date range |
| `checkIn` | O(1) | Direct lookup |
| `checkOut` | O(b) | Remove date range |

### Space Complexity

| Component | Space |
|-----------|-------|
| Bookings | O(n) |
| Room bookings | O(r × b) | r = rooms, b = avg bookings |
| Hotel rooms | O(r) |

### Bottlenecks at Scale

**10x Usage (10 → 100 hotels, 1K → 10K rooms):**
- Problem: Room availability search becomes slow (O(r × d × s)), concurrent booking conflicts increase, date range overlap checks expensive
- Solution: Index rooms by hotel/type, use distributed locking (Redis) for bookings, optimize date range queries
- Tradeoff: Additional infrastructure (Redis), more complex locking logic

**100x Usage (10 → 1K hotels, 1K → 100K rooms):**
- Problem: Single instance can't handle all hotels, database becomes bottleneck, real-time availability queries too slow
- Solution: Shard hotels by region, use read replicas for availability queries, implement caching layer (Redis) for hot room data
- Tradeoff: Distributed system complexity, need shard routing and data consistency across shards


### Q1: How would you handle room upgrades?

```java
public class UpgradeService {
    public Room findUpgrade(Booking booking) {
        RoomType currentType = booking.getRoom().getType();
        RoomType upgradeType = getNextTier(currentType);
        
        return bookingService.searchRooms(new SearchCriteria(
            booking.getDateRange(), 
            booking.getNumberOfGuests()
        ).withRoomType(upgradeType)).stream().findFirst().orElse(null);
    }
    
    public Booking applyUpgrade(Booking booking, Room newRoom, BigDecimal additionalCost) {
        // Release old room
        releaseRoom(booking);
        
        // Update booking with new room
        booking.setRoom(newRoom);
        booking.setTotalPrice(booking.getTotalPrice().add(additionalCost));
        
        // Reserve new room
        reserveRoom(newRoom, booking.getDateRange());
        
        return booking;
    }
}
```

### Q2: How would you implement group bookings?

```java
public class GroupBooking {
    private final String id;
    private final String groupName;
    private final List<Booking> individualBookings;
    private final BigDecimal groupDiscount;
    private final Guest coordinator;
    
    public BigDecimal getTotalPrice() {
        BigDecimal total = individualBookings.stream()
            .map(Booking::getTotalPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        return total.multiply(BigDecimal.ONE.subtract(groupDiscount));
    }
    
    public void addRoom(Guest guest, Room room) {
        Booking booking = new Booking(guest, room, dateRange, 2, 
            pricingStrategy.calculatePrice(room, dateRange, 2));
        individualBookings.add(booking);
    }
}
```

### Q3: How would you implement room preferences?

```java
public class GuestPreferences {
    private final String guestId;
    private RoomType preferredType;
    private Integer preferredFloor;
    private Set<String> requiredAmenities;
    private String bedPreference;  // "King", "Two Queens"
    private boolean smokingRoom;
    private boolean accessibleRoom;
}

public class PreferenceMatchingService {
    public List<Room> findMatchingRooms(Guest guest, SearchCriteria criteria) {
        GuestPreferences prefs = getPreferences(guest);
        
        return bookingService.searchRooms(criteria).stream()
            .map(room -> new ScoredRoom(room, calculateMatchScore(room, prefs)))
            .sorted(Comparator.comparingInt(ScoredRoom::getScore).reversed())
            .map(ScoredRoom::getRoom)
            .collect(Collectors.toList());
    }
    
    private int calculateMatchScore(Room room, GuestPreferences prefs) {
        int score = 0;
        if (room.getType() == prefs.getPreferredType()) score += 10;
        if (room.getFloor() == prefs.getPreferredFloor()) score += 5;
        for (String amenity : prefs.getRequiredAmenities()) {
            if (room.hasAmenity(amenity)) score += 3;
        }
        return score;
    }
}
```

### Q4: How would you handle overbooking?

```java
public class OverbookingManager {
    private final double overbookingRate = 0.05;  // 5% overbook
    
    public int getEffectiveCapacity(Hotel hotel, RoomType type) {
        int actual = hotel.getRoomsByType(type).size();
        return (int) (actual * (1 + overbookingRate));
    }
    
    public List<Guest> handleOverbooking(LocalDate date) {
        List<Booking> bookings = getConfirmedBookings(date);
        int capacity = getTotalRooms();
        
        if (bookings.size() <= capacity) {
            return Collections.emptyList();
        }
        
        // Walk guests who booked most recently
        List<Booking> toWalk = bookings.stream()
            .sorted(Comparator.comparing(Booking::getCreatedAt).reversed())
            .limit(bookings.size() - capacity)
            .collect(Collectors.toList());
        
        for (Booking booking : toWalk) {
            // Offer alternative accommodation
            offerAlternative(booking);
            // Provide compensation
            compensate(booking);
        }
        
        return toWalk.stream().map(Booking::getGuest).collect(Collectors.toList());
    }
}
```

