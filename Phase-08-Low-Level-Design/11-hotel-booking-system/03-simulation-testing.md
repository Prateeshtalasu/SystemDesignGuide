# 🏨 Hotel Booking System - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Room Booking Flow

```
1. Search: Double room, Jan 15-17
   - Check availability for all double rooms
   - Room 201, 203 available
   - Calculate price: 2 nights × $150 = $300

2. Book: Room 201, Guest: John Doe
   - Create booking B-001
   - Mark dates as reserved
   - Send confirmation email

3. Check-in: B-001
   - Verify ID, collect payment
   - Issue room key
   - Update status to OCCUPIED

4. Check-out: B-001
   - Calculate final charges
   - Update status to NEEDS_CLEANING
```

**Final State:**
```
Room 201: NEEDS_CLEANING
Booking B-001: COMPLETED
All operations completed successfully
```

---

### Scenario 2: Failure/Invalid Input - Overlapping Bookings

**Initial State:**
```
Room 201: DOUBLE, AVAILABLE
Existing Booking: Room 201, Jan 15-17 (RESERVED)
New Booking Request: Room 201, Jan 16-18 (overlaps)
```

**Step-by-step:**

1. `bookingService.searchAvailableRooms(RoomType.DOUBLE, Jan 16-18)`
   - Check Room 201 availability
   - Existing booking: Jan 15-17
   - New request: Jan 16-18
   - Overlap detected: Jan 16-17 overlaps with existing
   - Room 201 NOT in available results

2. `bookingService.bookRoom("guest1", Room 201, Jan 16-18)` (attempt booking anyway)
   - Validate availability
   - `room.isAvailable(dateRange)` → checks overlaps
   - Overlap detected → throws IllegalStateException
   - Booking rejected
   - Room 201 status unchanged (RESERVED for Jan 15-17)

3. `bookingService.bookRoom(null, Room 201, Jan 20-22)` (invalid input)
   - Null guest ID → throws IllegalArgumentException
   - No state change

**Final State:**
```
Room 201: RESERVED (original booking intact)
New booking rejected due to overlap
Invalid inputs properly rejected
```

---

### Scenario 3: Concurrency/Race Condition - Double Booking

**Initial State:**
```
Room 201: DOUBLE, AVAILABLE
Thread A: Guest1 booking request
Thread B: Guest2 booking request (concurrent)
Both request: Room 201, Jan 15-17
```

**Step-by-step (simulating concurrent booking attempts):**

**Thread A:** `bookingService.bookRoom("guest1", Room 201, Jan 15-17)` at time T0
**Thread B:** `bookingService.bookRoom("guest2", Room 201, Jan 15-17)` at time T0 (concurrent)

1. **Thread A:** Enters `bookRoom()` method
   - Checks availability: Room 201 is AVAILABLE
   - Creates booking object
   - Calls `room.reserve(dateRange)`
   - Acquires lock on Room 201
   - Status change: AVAILABLE → RESERVED
   - Releases lock

2. **Thread B:** Enters `bookRoom()` method (concurrent)
   - Checks availability: Room 201 appears AVAILABLE (before Thread A's lock)
   - Creates booking object
   - Calls `room.reserve(dateRange)`
   - Waits for lock (Thread A holds it)
   - After Thread A releases, acquires lock
   - Checks status: RESERVED (Thread A already reserved)
   - `reserve()` throws IllegalStateException
   - Booking fails

**Final State:**
```
Room 201: RESERVED by Guest1 (Thread A succeeded)
Guest2 booking rejected (Thread B failed)
Only one booking succeeds, no double booking
Proper synchronization prevents race conditions
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions
- **Overlapping Bookings**: Prevent double-booking
- **Same-Day Booking**: Handle late check-in
- **Early Check-out**: Partial refund calculation
- **No-Show**: Handle after 6 PM

---

## Testing Approach

### Unit Tests

```java
- Guests book rooms for date ranges
- Different room types at different prices
- Handle check-in, check-out, cancellations
- Support various pricing and refund policies

**Key Entities:**
- **Room**: Physical room with type and status
- **Guest**: Person making the booking
- **Booking**: Reservation linking guest to room and dates
- **DateRange**: Check-in to check-out period

---

### Phase 2: Design the Room Model

```java
// Step 1: Room types with pricing
public enum RoomType {
    SINGLE("Single Room", 1, new BigDecimal("100.00")),
    DOUBLE("Double Room", 2, new BigDecimal("150.00")),
    SUITE("Suite", 4, new BigDecimal("350.00"));
    
    private final int maxOccupancy;
    private final BigDecimal basePrice;
}
```

```java
// Step 2: Room status
public enum RoomStatus {
    AVAILABLE,    // Ready for guests
    RESERVED,     // Booked but not checked in
    OCCUPIED,     // Guest staying
    CLEANING,     // Being cleaned
    MAINTENANCE   // Out of service
}
```

```java
// Step 3: Room class
public class Room {
    private final String roomNumber;
    private final RoomType type;
    private final int floor;
    private RoomStatus status;
    private Set<String> amenities;
    
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

---

### Phase 3: Design the Date Range

```java
// Step 4: DateRange class
public class DateRange {
    private final LocalDate checkIn;
    private final LocalDate checkOut;
    
    public long getNights() {
        return ChronoUnit.DAYS.between(checkIn, checkOut);
    }
    
    public boolean overlaps(DateRange other) {
        return checkIn.isBefore(other.checkOut) && 
               checkOut.isAfter(other.checkIn);
    }
}
```

**Overlap detection visualization:**

```
Existing booking:    |-------|
New booking:              |-------|
                     ↑     ↑ ↑     ↑
                    in    out in  out

Overlap if: new.checkIn < existing.checkOut 
        AND new.checkOut > existing.checkIn
```

---

### Phase 4: Design Pricing Strategy

```java
// Step 5: Pricing strategy interface
public interface PricingStrategy {
    BigDecimal calculatePrice(Room room, DateRange dateRange, int guests);
}
```

```java
// Step 6: Standard pricing
public class StandardPricingStrategy implements PricingStrategy {
    private static final BigDecimal TAX_RATE = new BigDecimal("0.12");
    
    @Override
    public BigDecimal calculatePrice(Room room, DateRange dateRange, int guests) {
        BigDecimal nightlyRate = room.getBasePrice();
        BigDecimal nights = BigDecimal.valueOf(dateRange.getNights());
        
        BigDecimal subtotal = nightlyRate.multiply(nights);
        BigDecimal tax = subtotal.multiply(TAX_RATE);
        
        return subtotal.add(tax);
    }
}
```

```java
// Step 7: Seasonal pricing
public class SeasonalPricingStrategy implements PricingStrategy {
    private final Map<Month, BigDecimal> seasonalMultipliers;
    
    @Override
    public BigDecimal calculatePrice(Room room, DateRange dateRange, int guests) {
        BigDecimal total = BigDecimal.ZERO;
        LocalDate current = dateRange.getCheckIn();
        
        while (current.isBefore(dateRange.getCheckOut())) {
            BigDecimal nightlyRate = room.getBasePrice();
            BigDecimal seasonal = seasonalMultipliers.get(current.getMonth());
            nightlyRate = nightlyRate.multiply(seasonal);
            
            if (isWeekend(current)) {
                nightlyRate = nightlyRate.multiply(weekendMultiplier);
            }
            
            total = total.add(nightlyRate);
            current = current.plusDays(1);
        }
        
        return total.add(total.multiply(TAX_RATE));
    }
}
```

**Seasonal pricing example:**

```
Room: Double ($150/night base)
Dates: July 15-18 (3 nights)
July multiplier: 1.4 (peak season)
Weekend multiplier: 1.2

Night 1 (Fri): $150 × 1.4 × 1.2 = $252
Night 2 (Sat): $150 × 1.4 × 1.2 = $252
Night 3 (Sun): $150 × 1.4 = $210

Subtotal: $714
Tax (12%): $85.68
Total: $799.68
```

---

### Phase 5: Design Cancellation Policy

```java
// Step 8: Cancellation policy interface
public interface CancellationPolicy {
    BigDecimal calculateRefund(Booking booking);
    String getDescription();
}
```

```java
// Step 9: Moderate policy implementation
public class ModerateCancellationPolicy implements CancellationPolicy {
    
    @Override
    public BigDecimal calculateRefund(Booking booking) {
        long daysUntilCheckIn = booking.getDaysUntilCheckIn();
        BigDecimal total = booking.getTotalPrice();
        
        if (daysUntilCheckIn >= 7) {
            return total;  // Full refund
        } else if (daysUntilCheckIn >= 3) {
            return total.multiply(new BigDecimal("0.50"));  // 50%
        } else if (daysUntilCheckIn >= 1) {
            return total.multiply(new BigDecimal("0.25"));  // 25%
        } else {
            return BigDecimal.ZERO;  // No refund
        }
    }
}
```

**Refund calculation:**

```
Booking total: $800
Days until check-in: 5

5 days → falls in 3-6 day range
Refund = $800 × 0.50 = $400
```

---

### Phase 6: Design the Booking Service

```java
// Step 10: Track room availability
public class BookingService {
    // Room → List of booked date ranges
    private final Map<Room, List<DateRange>> roomBookings;
    
    private boolean isRoomAvailable(Room room, DateRange dateRange) {
        List<DateRange> bookedRanges = roomBookings.get(room);
        if (bookedRanges == null) return true;
        
        for (DateRange booked : bookedRanges) {
            if (dateRange.overlaps(booked)) {
                return false;
            }
        }
        return true;
    }
}
```

```java
// Step 11: Search for available rooms
public List<Room> searchRooms(SearchCriteria criteria) {
    return hotel.getRooms().stream()
        .filter(room -> matchesCriteria(room, criteria))
        .filter(room -> isRoomAvailable(room, criteria.getDateRange()))
        .sorted(Comparator.comparing(Room::getBasePrice))
        .collect(Collectors.toList());
}

private boolean matchesCriteria(Room room, SearchCriteria criteria) {
    // Check room type
    if (criteria.getRoomType() != null && 
        room.getType() != criteria.getRoomType()) {
        return false;
    }
    
    // Check capacity
    if (room.getType().getMaxOccupancy() < criteria.getGuests()) {
        return false;
    }
    
    // Check amenities
    for (String amenity : criteria.getRequiredAmenities()) {
        if (!room.hasAmenity(amenity)) {
            return false;
        }
    }
    
    return true;
}
```

```java
// Step 12: Make booking
public synchronized Booking makeBooking(Guest guest, Room room, 
                                        DateRange dateRange, int guests) {
    // Validate availability
    if (!isRoomAvailable(room, dateRange)) {
        throw new IllegalStateException("Room not available");
    }
    
    // Calculate price
    BigDecimal totalPrice = pricingStrategy.calculatePrice(
        room, dateRange, guests);
    
    // Create booking
    Booking booking = new Booking(guest, room, dateRange, guests, totalPrice);
    
    // Reserve dates
    roomBookings.computeIfAbsent(room, k -> new ArrayList<>())
                .add(dateRange);
    bookings.put(booking.getId(), booking);
    
    return booking;
}
```

```java
// Step 13: Cancel booking
public BigDecimal cancelBooking(String bookingId) {
    Booking booking = bookings.get(bookingId);
    
    // Calculate refund
    BigDecimal refund = cancellationPolicy.calculateRefund(booking);
    
    // Cancel booking
    booking.cancel();
    
    // Release dates
    roomBookings.get(booking.getRoom()).remove(booking.getDateRange());
    
    return refund;
}
```

---

## Testing Approach

### Unit Tests

```java
// DateRangeTest.java
public class DateRangeTest {
    
    @Test
    void testOverlap() {
        DateRange range1 = new DateRange(
            LocalDate.of(2024, 1, 10), 
            LocalDate.of(2024, 1, 15));
        DateRange range2 = new DateRange(
            LocalDate.of(2024, 1, 13), 
            LocalDate.of(2024, 1, 18));
        DateRange range3 = new DateRange(
            LocalDate.of(2024, 1, 15), 
            LocalDate.of(2024, 1, 20));
        
        assertTrue(range1.overlaps(range2));   // Overlapping
        assertFalse(range1.overlaps(range3));  // Adjacent, not overlapping
    }
    
    @Test
    void testNights() {
        DateRange range = new DateRange(
            LocalDate.of(2024, 1, 10), 
            LocalDate.of(2024, 1, 13));
        
        assertEquals(3, range.getNights());
    }
}
```

```java
// PricingStrategyTest.java
public class PricingStrategyTest {
    
    @Test
    void testStandardPricing() {
        Room room = new Room("101", RoomType.DOUBLE, 1);
        DateRange dates = new DateRange(
            LocalDate.of(2024, 1, 10), 
            LocalDate.of(2024, 1, 13));
        
        PricingStrategy strategy = new StandardPricingStrategy();
        BigDecimal price = strategy.calculatePrice(room, dates, 2);
        
        // $150/night × 3 nights × 1.12 (tax) = $504
        assertEquals(new BigDecimal("504.00"), price);
    }
}
```

```java
// CancellationPolicyTest.java
public class CancellationPolicyTest {
    
    @Test
    void testModeratePolicy() {
        CancellationPolicy policy = new ModerateCancellationPolicy();
        
        // Booking 10 days away
        Booking booking = createBooking(LocalDate.now().plusDays(10), 
                                       new BigDecimal("500.00"));
        
        assertEquals(new BigDecimal("500.00"), policy.calculateRefund(booking));
        
        // Booking 5 days away
        Booking booking2 = createBooking(LocalDate.now().plusDays(5), 
                                        new BigDecimal("500.00"));
        
        assertEquals(new BigDecimal("250.00"), policy.calculateRefund(booking2));
    }
}
```

```java
// BookingServiceTest.java
public class BookingServiceTest {
    
    private Hotel hotel;
    private BookingService service;
    
    @BeforeEach
    void setUp() {
        hotel = new Hotel("Test Hotel", "123 Main", "City", 4);
        hotel.addRoom(new Room("101", RoomType.SINGLE, 1));
        hotel.addRoom(new Room("201", RoomType.DOUBLE, 2));
        
        service = new BookingService(hotel);
    }
    
    @Test
    void testSearchAndBook() {
        DateRange dates = new DateRange(
            LocalDate.now().plusDays(7), 
            LocalDate.now().plusDays(10));
        
        SearchCriteria criteria = new SearchCriteria(dates, 2)
            .withRoomType(RoomType.DOUBLE);
        
        List<Room> available = service.searchRooms(criteria);
        assertEquals(1, available.size());
        
        Guest guest = new Guest("John", "Smith", "john@email.com", 
                               "555-1234", "DL123");
        Booking booking = service.makeBooking(guest, available.get(0), dates, 2);
        
        assertNotNull(booking.getId());
        assertEquals(BookingStatus.CONFIRMED, booking.getStatus());
    }
    
    @Test
    void testDoubleBookingPrevented() {
        DateRange dates = new DateRange(
            LocalDate.now().plusDays(7), 
            LocalDate.now().plusDays(10));
        Room room = hotel.getRoom("201");
        
        Guest guest1 = new Guest("John", "Smith", "john@email.com", 
                                "555-1234", "DL123");
        service.makeBooking(guest1, room, dates, 2);
        
        Guest guest2 = new Guest("Jane", "Doe", "jane@email.com", 
                                "555-5678", "DL456");
        
        assertThrows(IllegalStateException.class, () -> 
            service.makeBooking(guest2, room, dates, 2));
    }
}
```


### searchRooms

```
Time: O(r × b)
  - r = total rooms
  - b = bookings per room (for availability check)
  
  For each room:
    - matchesCriteria: O(a) where a = amenities
    - isRoomAvailable: O(b) date range checks

Space: O(r) for result list
```

### makeBooking

```
Time: O(b)
  - Check availability: O(b)
  - Calculate price: O(n) for seasonal (n = nights)
  - Create and store: O(1)

Space: O(1)
```

### cancelBooking

```
Time: O(b)
  - Calculate refund: O(1)
  - Remove date range: O(b)

Space: O(1)
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.

