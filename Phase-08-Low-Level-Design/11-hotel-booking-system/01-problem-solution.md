# 🏨 Hotel Booking System - Problem Solution

## STEP 0: REQUIREMENTS QUICKPASS

### Core Functional Requirements
- Manage different room types (single, double, suite, deluxe)
- Search for room availability by date range
- Handle booking management (create, modify, cancel)
- Support multiple pricing strategies (seasonal, weekend, dynamic)
- Implement cancellation policies with refund rules
- Track room status and housekeeping
- Support guest check-in and check-out

### Explicit Out-of-Scope Items
- Online payment processing
- Multi-hotel chain management
- Room service and amenities booking
- Loyalty program management
- External booking platform integration
- Revenue management analytics

### Assumptions and Constraints
- **Single Hotel**: One property
- **Room Types**: Single, Double, Suite, Deluxe
- **Check-in Time**: 3 PM, Check-out: 11 AM
- **Cancellation**: Free up to 24 hours before
- **Max Stay**: 30 consecutive nights

### Scale Assumptions (LLD Focus)
- Single hotel with ~200 rooms
- Peak bookings: ~500 per day
- Occupancy tracking: 365 days forward
- In-memory storage for LLD demonstration

### Concurrency Model
- **Synchronized room booking**: Prevents double-booking same room
- **ConcurrentHashMap** for bookings and room registry
- **Atomic availability check + reservation**: Single transaction
- **Room status updates**: Synchronized per room instance

### Public APIs
- `searchRooms(checkIn, checkOut, type, guests)`: Find available rooms
- `makeBooking(guest, room, dates)`: Create reservation
- `cancelBooking(bookingId)`: Cancel with refund calculation
- `checkIn(bookingId)`: Process arrival
- `checkOut(bookingId)`: Process departure
- `updateRoomStatus(roomId, status)`: Housekeeping updates

### Public API Usage Examples
```java
// Example 1: Basic usage
Hotel hotel = new Hotel("Grand Plaza", "123 Main St", "NYC", 4);
hotel.addRoom(new Room("101", RoomType.SINGLE, 1));
BookingService service = new BookingService(hotel);
DateRange dates = new DateRange(LocalDate.now().plusDays(7), LocalDate.now().plusDays(10));
SearchCriteria criteria = new SearchCriteria(dates, 1).withRoomType(RoomType.SINGLE);
List<Room> available = service.searchRooms(criteria);
Guest guest = new Guest("John", "Smith", "john@email.com", "555-1234", "DL123");
Booking booking = service.makeBooking(guest, available.get(0), dates, 1);

// Example 2: Typical workflow
service.checkIn(booking.getId());
// ... guest stays ...
Invoice invoice = service.checkOut(booking.getId());
invoice.print();

// Example 3: Edge case - cancellation with refund
BigDecimal refund = service.cancelBooking(booking.getId());
System.out.println("Refund amount: $" + refund);
```

### Invariants
- **No Double Booking**: Room available for one guest per night
- **Pricing Consistency**: Rate locked at booking time
- **Status Tracking**: Room status always reflects reality

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          HOTEL BOOKING SYSTEM                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                        BookingService                                     │   │
│  │                                                                           │   │
│  │  - hotel: Hotel                                                           │   │
│  │  - bookings: Map<String, Booking>                                        │   │
│  │  - pricingStrategy: PricingStrategy                                      │   │
│  │  - cancellationPolicy: CancellationPolicy                                │   │
│  │                                                                           │   │
│  │  + searchRooms(criteria): List<Room>                                     │   │
│  │  + makeBooking(request): Booking                                         │   │
│  │  + cancelBooking(id): Refund                                             │   │
│  │  + modifyBooking(id, request): Booking                                   │   │
│  │  + checkIn(bookingId): void                                              │   │
│  │  + checkOut(bookingId): Invoice                                          │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│                          │ manages                                               │
│                          ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                           Hotel                                           │   │
│  │                                                                           │   │
│  │  - name: String                                                           │   │
│  │  - rooms: List<Room>                                                     │   │
│  │  - amenities: List<Amenity>                                              │   │
│  │  - address: Address                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│                          │ contains                                              │
│                          ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                           Room                                            │   │
│  │                                                                           │   │
│  │  - roomNumber: String              - status: RoomStatus                  │   │
│  │  - type: RoomType                  - floor: int                          │   │
│  │  - basePrice: BigDecimal           - amenities: List<Amenity>            │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │    Booking      │  │     Guest       │  │    Invoice      │                 │
│  │                 │  │                 │  │                 │                 │
│  │ - id            │  │ - name          │  │ - charges       │                 │
│  │ - guest         │  │ - email         │  │ - taxes         │                 │
│  │ - room          │  │ - phone         │  │ - total         │                 │
│  │ - dates         │  │ - idDocument    │  │                 │                 │
│  │ - status        │  │                 │  │                 │                 │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                 │
│                                                                                  │
│  ┌─────────────────────────┐    ┌─────────────────────────┐                    │
│  │    PricingStrategy      │    │   CancellationPolicy    │                    │
│  │      (interface)        │    │      (interface)        │                    │
│  │                         │    │                         │                    │
│  │ + calculatePrice()      │    │ + calculateRefund()     │                    │
│  └─────────────────────────┘    └─────────────────────────┘                    │
│            △                              △                                     │
│     ┌──────┴──────┐              ┌────────┴────────┐                           │
│     │             │              │                 │                           │
│  Standard    Seasonal      Flexible          Strict                            │
│  Pricing     Pricing       Policy            Policy                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Booking Flow Visualization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BOOKING LIFECYCLE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐           │
│  │  Search  │────►│   Book   │────►│ Check-In │────►│Check-Out │           │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘           │
│       │                │                │                │                  │
│       ▼                ▼                ▼                ▼                  │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐           │
### Responsibilities Table

| Class | Owns | Why |
|-------|------|-----|
| `Hotel` | Hotel configuration (rooms, pricing, policies) | Manages hotel setup - encapsulates hotel-level configuration and room inventory |
| `Room` | Room properties and status management | Encapsulates room state - manages room status transitions (available/reserved/occupied/cleaning) |
| `Guest` | Guest information (name, contact, ID) | Stores guest data - separates guest information from booking logic |
| `Booking` | Booking details (guest, room, dates, status) | Encapsulates booking data - stores all booking information and status |
| `DateRange` | Date period representation (start, end) | Encapsulates date range - separates date handling from booking logic |
| `PricingStrategy` (interface) | Pricing calculation contract | Defines pricing interface - enables multiple pricing strategies (standard, seasonal) |
| `CancellationPolicy` (interface) | Refund calculation contract | Defines cancellation policy interface - enables multiple policies (flexible, strict) |
| `BookingService` | Booking operations coordination | Coordinates booking workflow - separates business logic from domain objects, handles search/book/cancel |
| `Invoice` | Billing information and calculation | Generates billing - separates billing presentation and calculation from booking logic |

---

│  │Available │     │ RESERVED │     │ OCCUPIED │     │ CLEANING │           │
│  │  Rooms   │     │   Room   │     │   Room   │     │   Room   │           │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘           │
│                        │                                   │                │
│                        │ cancel                            │ cleaned        │
│                        ▼                                   ▼                │
│                   ┌──────────┐                       ┌──────────┐           │
│                   │ Refund   │                       │AVAILABLE │           │
│                   │Calculated│                       │   Room   │           │
│                   └──────────┘                       └──────────┘           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## STEP 4: Code Walkthrough - Building From Scratch

This section explains how an engineer builds this system from scratch, in the order code should be written.

### Phase 1: Design the Room Model

```java
// Step 1: Room types and status
public enum RoomType {
    SINGLE("Single Room", 1, new BigDecimal("100.00")),
    DOUBLE("Double Room", 2, new BigDecimal("150.00")),
    SUITE("Suite", 4, new BigDecimal("350.00"));
}

public enum RoomStatus {
    AVAILABLE, RESERVED, OCCUPIED, CLEANING, MAINTENANCE
}

public class Room {
    private final String roomNumber;
    private final RoomType type;
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

---

### Phase 2: Design the Date Range

```java
// Step 2: DateRange class
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

---

### Phase 3: Design Pricing Strategy

```java
// Step 3: Pricing strategy interface
public interface PricingStrategy {
    BigDecimal calculatePrice(Room room, DateRange dateRange, int guests);
}

// Step 4: Standard pricing implementation
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

---

### Phase 4: Design the Booking Service

```java
// Step 5: Booking service
public class BookingService {
    private final Map<Room, List<DateRange>> roomBookings;
    private final Map<String, Booking> bookings;
    private final PricingStrategy pricingStrategy;
    
    public synchronized Booking makeBooking(Guest guest, Room room, 
                                            DateRange dateRange, int guests) {
        if (!isRoomAvailable(room, dateRange)) {
            throw new IllegalStateException("Room not available");
        }
        
        BigDecimal totalPrice = pricingStrategy.calculatePrice(
            room, dateRange, guests);
        
        Booking booking = new Booking(guest, room, dateRange, guests, totalPrice);
        roomBookings.computeIfAbsent(room, k -> new ArrayList<>()).add(dateRange);
        bookings.put(booking.getId(), booking);
        return booking;
    }
    
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

---

### Phase 5: Threading Model and Concurrency Control

**Threading Model:**

This hotel booking system handles **concurrent booking requests**:
- Multiple guests can try to book the same room simultaneously
- Booking operations must be atomic to prevent double-booking
- Room availability checks must be thread-safe

**Concurrency Control:**

```java
// Option 1: Synchronized method (simple but coarse-grained)
public synchronized Booking makeBooking(Guest guest, Room room, 
                                        DateRange dateRange, int guests) {
    // Entire method is synchronized - prevents concurrent bookings
    // However, blocks all bookings even for different rooms
}

// Option 2: Per-room locking (better granularity)
private final Map<String, ReentrantLock> roomLocks = new ConcurrentHashMap<>();

public Booking makeBooking(Guest guest, Room room, 
                          DateRange dateRange, int guests) {
    ReentrantLock lock = roomLocks.computeIfAbsent(
        room.getRoomNumber(), k -> new ReentrantLock());
    
    lock.lock();
    try {
        if (!isRoomAvailable(room, dateRange)) {
            throw new IllegalStateException("Room not available");
        }
        // ... create booking ...
        return booking;
    } finally {
        lock.unlock();
    }
}

// Option 3: Optimistic locking with version field
public class Room {
    private int version;  // Version field for optimistic locking
    
    public boolean tryBook(DateRange dateRange) {
        // Check version before booking
        // Increment version on successful booking
    }
}
```

**Why per-room locking?**
- Allows concurrent bookings for different rooms
- Only blocks bookings for the same room
- Better performance than global lock

---

## STEP 2: Complete Final Implementation

> **Verified:** This code compiles successfully with Java 11+.

### 2.1 RoomType Enum

```java
// RoomType.java
package com.hotel;

import java.math.BigDecimal;

/**
 * Types of rooms available in the hotel.
 */
public enum RoomType {
    SINGLE("Single Room", 1, new BigDecimal("100.00")),
    DOUBLE("Double Room", 2, new BigDecimal("150.00")),
    TWIN("Twin Room", 2, new BigDecimal("140.00")),
    QUEEN("Queen Room", 2, new BigDecimal("175.00")),
    KING("King Room", 2, new BigDecimal("200.00")),
    SUITE("Suite", 4, new BigDecimal("350.00")),
    PRESIDENTIAL("Presidential Suite", 6, new BigDecimal("1000.00"));
    
    private final String displayName;
    private final int maxOccupancy;
    private final BigDecimal basePrice;
    
    RoomType(String displayName, int maxOccupancy, BigDecimal basePrice) {
        this.displayName = displayName;
        this.maxOccupancy = maxOccupancy;
        this.basePrice = basePrice;
    }
    
    public String getDisplayName() { return displayName; }
    public int getMaxOccupancy() { return maxOccupancy; }
    public BigDecimal getBasePrice() { return basePrice; }
}
```

### 2.2 RoomStatus Enum

```java
// RoomStatus.java
package com.hotel;

/**
 * Status of a hotel room.
 */
public enum RoomStatus {
    AVAILABLE,      // Ready for guests
    RESERVED,       // Booked but guest hasn't arrived
    OCCUPIED,       // Guest checked in
    CLEANING,       // Being cleaned after checkout
    MAINTENANCE,    // Under maintenance
    OUT_OF_ORDER    // Not available
}
```

### 2.3 BookingStatus Enum

```java
// BookingStatus.java
package com.hotel;

public enum BookingStatus {
    PENDING,        // Awaiting confirmation
    CONFIRMED,      // Booking confirmed
    CHECKED_IN,     // Guest has arrived
    CHECKED_OUT,    // Guest has left
    CANCELLED,      // Booking cancelled
    NO_SHOW         // Guest didn't show up
}
```

### 2.4 Room Class

```java
// Room.java
package com.hotel;

import java.math.BigDecimal;
import java.util.*;

/**
 * Represents a hotel room.
 */
public class Room {
    
    private final String roomNumber;
    private final RoomType type;
    private final int floor;
    private final BigDecimal basePrice;
    private final Set<String> amenities;
    private RoomStatus status;
    private String notes;
    
    public Room(String roomNumber, RoomType type, int floor) {
        this(roomNumber, type, floor, type.getBasePrice());
    }
    
    public Room(String roomNumber, RoomType type, int floor, BigDecimal basePrice) {
        this.roomNumber = roomNumber;
        this.type = type;
        this.floor = floor;
        this.basePrice = basePrice;
        this.amenities = new HashSet<>();
        this.status = RoomStatus.AVAILABLE;
    }
    
    public void addAmenity(String amenity) {
        amenities.add(amenity);
    }
    
    public boolean hasAmenity(String amenity) {
        return amenities.contains(amenity);
    }
    
    public void reserve() {
        if (status != RoomStatus.AVAILABLE) {
            throw new IllegalStateException("Room not available for reservation");
        }
        this.status = RoomStatus.RESERVED;
    }
    
    public void checkIn() {
        if (status != RoomStatus.RESERVED && status != RoomStatus.AVAILABLE) {
            throw new IllegalStateException("Room not ready for check-in");
        }
        this.status = RoomStatus.OCCUPIED;
    }
    
    public void checkOut() {
        if (status != RoomStatus.OCCUPIED) {
            throw new IllegalStateException("Room not occupied");
        }
        this.status = RoomStatus.CLEANING;
    }
    
    public void markCleaned() {
        if (status != RoomStatus.CLEANING) {
            throw new IllegalStateException("Room not in cleaning status");
        }
        this.status = RoomStatus.AVAILABLE;
    }
    
    public void cancelReservation() {
        if (status != RoomStatus.RESERVED) {
            throw new IllegalStateException("Room not reserved");
        }
        this.status = RoomStatus.AVAILABLE;
    }
    
    public void setMaintenance(String notes) {
        this.status = RoomStatus.MAINTENANCE;
        this.notes = notes;
    }
    
    public void completeMaintenance() {
        this.status = RoomStatus.AVAILABLE;
        this.notes = null;
    }
    
    public boolean isAvailable() {
        return status == RoomStatus.AVAILABLE;
    }
    
    // Getters
    public String getRoomNumber() { return roomNumber; }
    public RoomType getType() { return type; }
    public int getFloor() { return floor; }
    public BigDecimal getBasePrice() { return basePrice; }
    public Set<String> getAmenities() { return Collections.unmodifiableSet(amenities); }
    public RoomStatus getStatus() { return status; }
    public String getNotes() { return notes; }
    
    @Override
    public String toString() {
        return String.format("Room %s (%s) - Floor %d - %s", 
            roomNumber, type.getDisplayName(), floor, status);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Room)) return false;
        return roomNumber.equals(((Room) o).roomNumber);
    }
    
    @Override
    public int hashCode() {
        return roomNumber.hashCode();
    }
}
```

### 2.5 Guest Class

```java
// Guest.java
package com.hotel;

import java.time.LocalDate;

/**
 * Represents a hotel guest.
 */
public class Guest {
    
    private final String id;
    private final String firstName;
    private final String lastName;
    private final String email;
    private final String phone;
    private final String idDocument;
    private final LocalDate dateOfBirth;
    private String loyaltyNumber;
    
    public Guest(String firstName, String lastName, String email, 
                 String phone, String idDocument) {
        this(firstName, lastName, email, phone, idDocument, null);
    }
    
    public Guest(String firstName, String lastName, String email, 
                 String phone, String idDocument, LocalDate dateOfBirth) {
        this.id = java.util.UUID.randomUUID().toString().substring(0, 8);
        this.firstName = firstName;
        this.lastName = lastName;
        this.email = email;
        this.phone = phone;
        this.idDocument = idDocument;
        this.dateOfBirth = dateOfBirth;
    }
    
    public String getFullName() {
        return firstName + " " + lastName;
    }
    
    public void setLoyaltyNumber(String loyaltyNumber) {
        this.loyaltyNumber = loyaltyNumber;
    }
    
    // Getters
    public String getId() { return id; }
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
    public String getIdDocument() { return idDocument; }
    public LocalDate getDateOfBirth() { return dateOfBirth; }
    public String getLoyaltyNumber() { return loyaltyNumber; }
    
    @Override
    public String toString() {
        return String.format("Guest[%s, %s]", getFullName(), email);
    }
}
```

### 2.6 DateRange Class

```java
// DateRange.java
package com.hotel;

import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

/**
 * Represents a date range for bookings.
 */
public class DateRange {
    
    private final LocalDate checkIn;
    private final LocalDate checkOut;
    
    public DateRange(LocalDate checkIn, LocalDate checkOut) {
        if (checkIn.isAfter(checkOut) || checkIn.isEqual(checkOut)) {
            throw new IllegalArgumentException("Check-out must be after check-in");
        }
        this.checkIn = checkIn;
        this.checkOut = checkOut;
    }
    
    public long getNights() {
        return ChronoUnit.DAYS.between(checkIn, checkOut);
    }
    
    public boolean overlaps(DateRange other) {
        return checkIn.isBefore(other.checkOut) && checkOut.isAfter(other.checkIn);
    }
    
    public boolean contains(LocalDate date) {
        return !date.isBefore(checkIn) && date.isBefore(checkOut);
    }
    
    // Getters
    public LocalDate getCheckIn() { return checkIn; }
    public LocalDate getCheckOut() { return checkOut; }
    
    @Override
    public String toString() {
        return String.format("%s to %s (%d nights)", checkIn, checkOut, getNights());
    }
}
```

### 2.7 Booking Class

```java
// Booking.java
package com.hotel;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.*;

/**
 * Represents a hotel booking/reservation.
 */
public class Booking {
    
    private final String id;
    private final Guest guest;
    private final Room room;
    private final DateRange dateRange;
    private final int numberOfGuests;
    private BookingStatus status;
    private BigDecimal totalPrice;
    private final List<String> specialRequests;
    private final LocalDateTime createdAt;
    private LocalDateTime checkInTime;
    private LocalDateTime checkOutTime;
    
    public Booking(Guest guest, Room room, DateRange dateRange, 
                   int numberOfGuests, BigDecimal totalPrice) {
        this.id = "BK-" + System.currentTimeMillis() % 100000;
        this.guest = guest;
        this.room = room;
        this.dateRange = dateRange;
        this.numberOfGuests = numberOfGuests;
        this.totalPrice = totalPrice;
        this.status = BookingStatus.CONFIRMED;
        this.specialRequests = new ArrayList<>();
        this.createdAt = LocalDateTime.now();
    }
    
    public void addSpecialRequest(String request) {
        specialRequests.add(request);
    }
    
    public void checkIn() {
        if (status != BookingStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot check in: status is " + status);
        }
        this.status = BookingStatus.CHECKED_IN;
        this.checkInTime = LocalDateTime.now();
    }
    
    public void checkOut() {
        if (status != BookingStatus.CHECKED_IN) {
            throw new IllegalStateException("Cannot check out: status is " + status);
        }
        this.status = BookingStatus.CHECKED_OUT;
        this.checkOutTime = LocalDateTime.now();
    }
    
    public void cancel() {
        if (status == BookingStatus.CHECKED_IN || 
            status == BookingStatus.CHECKED_OUT) {
            throw new IllegalStateException("Cannot cancel: guest already checked in");
        }
        this.status = BookingStatus.CANCELLED;
    }
    
    public void markNoShow() {
        if (status != BookingStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot mark no-show: status is " + status);
        }
        this.status = BookingStatus.NO_SHOW;
    }
    
    public boolean isActive() {
        return status == BookingStatus.CONFIRMED || status == BookingStatus.CHECKED_IN;
    }
    
    public long getDaysUntilCheckIn() {
        return java.time.temporal.ChronoUnit.DAYS.between(
            LocalDateTime.now().toLocalDate(), dateRange.getCheckIn());
    }
    
    // Getters
    public String getId() { return id; }
    public Guest getGuest() { return guest; }
    public Room getRoom() { return room; }
    public DateRange getDateRange() { return dateRange; }
    public int getNumberOfGuests() { return numberOfGuests; }
    public BookingStatus getStatus() { return status; }
    public BigDecimal getTotalPrice() { return totalPrice; }
    public List<String> getSpecialRequests() { return Collections.unmodifiableList(specialRequests); }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getCheckInTime() { return checkInTime; }
    public LocalDateTime getCheckOutTime() { return checkOutTime; }
    
    public void setTotalPrice(BigDecimal totalPrice) {
        this.totalPrice = totalPrice;
    }
    
    @Override
    public String toString() {
        return String.format("Booking[%s: %s, Room %s, %s, %s]",
            id, guest.getFullName(), room.getRoomNumber(), dateRange, status);
    }
}
```

### 2.8 PricingStrategy Interface and Implementations

```java
// PricingStrategy.java
package com.hotel;

import java.math.BigDecimal;

/**
 * Strategy for calculating room prices.
 */
public interface PricingStrategy {
    
    BigDecimal calculatePrice(Room room, DateRange dateRange, int guests);
}
```

```java
// StandardPricingStrategy.java
package com.hotel;

import java.math.BigDecimal;
import java.math.RoundingMode;

/**
 * Standard pricing based on room base price.
 */
public class StandardPricingStrategy implements PricingStrategy {
    
    private static final BigDecimal TAX_RATE = new BigDecimal("0.12");  // 12% tax
    
    @Override
    public BigDecimal calculatePrice(Room room, DateRange dateRange, int guests) {
        BigDecimal nightlyRate = room.getBasePrice();
        BigDecimal nights = BigDecimal.valueOf(dateRange.getNights());
        
        BigDecimal subtotal = nightlyRate.multiply(nights);
        BigDecimal tax = subtotal.multiply(TAX_RATE);
        
        return subtotal.add(tax).setScale(2, RoundingMode.HALF_UP);
    }
}
```

```java
// SeasonalPricingStrategy.java
package com.hotel;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.time.Month;
import java.util.*;

/**
 * Pricing that varies by season.
 */
public class SeasonalPricingStrategy implements PricingStrategy {
    
    private static final BigDecimal TAX_RATE = new BigDecimal("0.12");
    
    private final Map<Month, BigDecimal> seasonalMultipliers;
    private final Set<LocalDate> holidays;
    private final BigDecimal holidayMultiplier;
    private final BigDecimal weekendMultiplier;
    
    public SeasonalPricingStrategy() {
        this.seasonalMultipliers = new EnumMap<>(Month.class);
        this.holidays = new HashSet<>();
        this.holidayMultiplier = new BigDecimal("1.5");
        this.weekendMultiplier = new BigDecimal("1.2");
        
        // Default seasonal rates
        seasonalMultipliers.put(Month.JANUARY, new BigDecimal("0.8"));
        seasonalMultipliers.put(Month.FEBRUARY, new BigDecimal("0.85"));
        seasonalMultipliers.put(Month.MARCH, new BigDecimal("1.0"));
        seasonalMultipliers.put(Month.APRIL, new BigDecimal("1.1"));
        seasonalMultipliers.put(Month.MAY, new BigDecimal("1.2"));
        seasonalMultipliers.put(Month.JUNE, new BigDecimal("1.3"));
        seasonalMultipliers.put(Month.JULY, new BigDecimal("1.4"));
        seasonalMultipliers.put(Month.AUGUST, new BigDecimal("1.4"));
        seasonalMultipliers.put(Month.SEPTEMBER, new BigDecimal("1.1"));
        seasonalMultipliers.put(Month.OCTOBER, new BigDecimal("1.0"));
        seasonalMultipliers.put(Month.NOVEMBER, new BigDecimal("0.9"));
        seasonalMultipliers.put(Month.DECEMBER, new BigDecimal("1.3"));
    }
    
    public void addHoliday(LocalDate date) {
        holidays.add(date);
    }
    
    @Override
    public BigDecimal calculatePrice(Room room, DateRange dateRange, int guests) {
        BigDecimal total = BigDecimal.ZERO;
        LocalDate current = dateRange.getCheckIn();
        
        while (current.isBefore(dateRange.getCheckOut())) {
            BigDecimal nightlyRate = room.getBasePrice();
            
            // Apply seasonal multiplier
            BigDecimal seasonal = seasonalMultipliers.getOrDefault(
                current.getMonth(), BigDecimal.ONE);
            nightlyRate = nightlyRate.multiply(seasonal);
            
            // Apply holiday multiplier
            if (holidays.contains(current)) {
                nightlyRate = nightlyRate.multiply(holidayMultiplier);
            }
            // Apply weekend multiplier (if not holiday)
            else if (isWeekend(current)) {
                nightlyRate = nightlyRate.multiply(weekendMultiplier);
            }
            
            total = total.add(nightlyRate);
            current = current.plusDays(1);
        }
        
        // Add tax
        BigDecimal tax = total.multiply(TAX_RATE);
        return total.add(tax).setScale(2, RoundingMode.HALF_UP);
    }
    
    private boolean isWeekend(LocalDate date) {
        return date.getDayOfWeek() == java.time.DayOfWeek.FRIDAY ||
               date.getDayOfWeek() == java.time.DayOfWeek.SATURDAY;
    }
}
```

### 2.9 CancellationPolicy Interface and Implementations

```java
// CancellationPolicy.java
package com.hotel;

import java.math.BigDecimal;

/**
 * Policy for calculating refunds on cancellation.
 */
public interface CancellationPolicy {
    
    BigDecimal calculateRefund(Booking booking);
    
    String getDescription();
}
```

```java
// FlexibleCancellationPolicy.java
package com.hotel;

import java.math.BigDecimal;

/**
 * Flexible policy - full refund up to 24 hours before check-in.
 */
public class FlexibleCancellationPolicy implements CancellationPolicy {
    
    @Override
    public BigDecimal calculateRefund(Booking booking) {
        long daysUntilCheckIn = booking.getDaysUntilCheckIn();
        
        if (daysUntilCheckIn >= 1) {
            return booking.getTotalPrice();  // Full refund
        } else {
            return BigDecimal.ZERO;  // No refund
        }
    }
    
    @Override
    public String getDescription() {
        return "Full refund if cancelled at least 24 hours before check-in";
    }
}
```

```java
// ModerateCancellationPolicy.java
package com.hotel;

import java.math.BigDecimal;
import java.math.RoundingMode;

/**
 * Moderate policy - tiered refunds based on cancellation timing.
 */
public class ModerateCancellationPolicy implements CancellationPolicy {
    
    @Override
    public BigDecimal calculateRefund(Booking booking) {
        long daysUntilCheckIn = booking.getDaysUntilCheckIn();
        BigDecimal total = booking.getTotalPrice();
        
        if (daysUntilCheckIn >= 7) {
            return total;  // Full refund
        } else if (daysUntilCheckIn >= 3) {
            return total.multiply(new BigDecimal("0.50"))
                       .setScale(2, RoundingMode.HALF_UP);  // 50% refund
        } else if (daysUntilCheckIn >= 1) {
            return total.multiply(new BigDecimal("0.25"))
                       .setScale(2, RoundingMode.HALF_UP);  // 25% refund
        } else {
            return BigDecimal.ZERO;  // No refund
        }
    }
    
    @Override
    public String getDescription() {
        return "Full refund 7+ days, 50% 3-6 days, 25% 1-2 days before check-in";
    }
}
```

```java
// StrictCancellationPolicy.java
package com.hotel;

import java.math.BigDecimal;
import java.math.RoundingMode;

/**
 * Strict policy - limited refunds.
 */
public class StrictCancellationPolicy implements CancellationPolicy {
    
    @Override
    public BigDecimal calculateRefund(Booking booking) {
        long daysUntilCheckIn = booking.getDaysUntilCheckIn();
        BigDecimal total = booking.getTotalPrice();
        
        if (daysUntilCheckIn >= 14) {
            return total.multiply(new BigDecimal("0.50"))
                       .setScale(2, RoundingMode.HALF_UP);  // 50% refund
        } else {
            return BigDecimal.ZERO;  // No refund
        }
    }
    
    @Override
    public String getDescription() {
        return "50% refund if cancelled 14+ days before check-in, no refund otherwise";
    }
}
```

### 2.10 Hotel Class

```java
// Hotel.java
package com.hotel;

import java.util.*;
import java.util.stream.Collectors;

/**
 * Represents a hotel with rooms and amenities.
 */
public class Hotel {
    
    private final String id;
    private final String name;
    private final String address;
    private final String city;
    private final int starRating;
    private final List<Room> rooms;
    private final Set<String> amenities;
    
    public Hotel(String name, String address, String city, int starRating) {
        this.id = java.util.UUID.randomUUID().toString().substring(0, 8);
        this.name = name;
        this.address = address;
        this.city = city;
        this.starRating = starRating;
        this.rooms = new ArrayList<>();
        this.amenities = new HashSet<>();
    }
    
    public void addRoom(Room room) {
        rooms.add(room);
    }
    
    public void addAmenity(String amenity) {
        amenities.add(amenity);
    }
    
    public List<Room> getRoomsByType(RoomType type) {
        return rooms.stream()
            .filter(r -> r.getType() == type)
            .collect(Collectors.toList());
    }
    
    public List<Room> getAvailableRooms() {
        return rooms.stream()
            .filter(Room::isAvailable)
            .collect(Collectors.toList());
    }
    
    public Room getRoom(String roomNumber) {
        return rooms.stream()
            .filter(r -> r.getRoomNumber().equals(roomNumber))
            .findFirst()
            .orElse(null);
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public String getAddress() { return address; }
    public String getCity() { return city; }
    public int getStarRating() { return starRating; }
    public List<Room> getRooms() { return Collections.unmodifiableList(rooms); }
    public Set<String> getAmenities() { return Collections.unmodifiableSet(amenities); }
    
    @Override
    public String toString() {
        return String.format("%s (%d-star) - %s, %s", name, starRating, address, city);
    }
}
```

### 2.11 SearchCriteria Class

```java
// SearchCriteria.java
package com.hotel;

import java.math.BigDecimal;
import java.util.*;

/**
 * Criteria for searching available rooms.
 */
public class SearchCriteria {
    
    private final DateRange dateRange;
    private final int guests;
    private RoomType roomType;
    private BigDecimal maxPrice;
    private Set<String> requiredAmenities;
    private Integer preferredFloor;
    
    public SearchCriteria(DateRange dateRange, int guests) {
        this.dateRange = dateRange;
        this.guests = guests;
        this.requiredAmenities = new HashSet<>();
    }
    
    public SearchCriteria withRoomType(RoomType type) {
        this.roomType = type;
        return this;
    }
    
    public SearchCriteria withMaxPrice(BigDecimal maxPrice) {
        this.maxPrice = maxPrice;
        return this;
    }
    
    public SearchCriteria withAmenity(String amenity) {
        this.requiredAmenities.add(amenity);
        return this;
    }
    
    public SearchCriteria withPreferredFloor(int floor) {
        this.preferredFloor = floor;
        return this;
    }
    
    // Getters
    public DateRange getDateRange() { return dateRange; }
    public int getGuests() { return guests; }
    public RoomType getRoomType() { return roomType; }
    public BigDecimal getMaxPrice() { return maxPrice; }
    public Set<String> getRequiredAmenities() { return requiredAmenities; }
    public Integer getPreferredFloor() { return preferredFloor; }
}
```

### 2.12 Invoice Class

```java
// Invoice.java
package com.hotel;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.*;

/**
 * Invoice generated at checkout.
 */
public class Invoice {
    
    private final String id;
    private final Booking booking;
    private final Map<String, BigDecimal> charges;
    private final BigDecimal total;
    private final LocalDateTime generatedAt;
    
    private Invoice(Builder builder) {
        this.id = "INV-" + System.currentTimeMillis() % 100000;
        this.booking = builder.booking;
        this.charges = Collections.unmodifiableMap(builder.charges);
        this.total = builder.total;
        this.generatedAt = LocalDateTime.now();
    }
    
    public static Builder builder(Booking booking) {
        return new Builder(booking);
    }
    
    public void print() {
        System.out.println("\n╔════════════════════════════════════════╗");
        System.out.println("║           HOTEL INVOICE                ║");
        System.out.println("╠════════════════════════════════════════╣");
        System.out.printf("║ Invoice #: %-28s║%n", id);
        System.out.printf("║ Date: %-33s║%n", generatedAt.toLocalDate());
        System.out.println("╠════════════════════════════════════════╣");
        System.out.printf("║ Guest: %-32s║%n", booking.getGuest().getFullName());
        System.out.printf("║ Room: %-33s║%n", booking.getRoom().getRoomNumber());
        System.out.printf("║ Stay: %-33s║%n", booking.getDateRange());
        System.out.println("╠════════════════════════════════════════╣");
        System.out.println("║ CHARGES:                               ║");
        for (Map.Entry<String, BigDecimal> charge : charges.entrySet()) {
            System.out.printf("║   %-25s $%8.2f ║%n", charge.getKey(), charge.getValue());
        }
        System.out.println("╠════════════════════════════════════════╣");
        System.out.printf("║   %-25s $%8.2f ║%n", "TOTAL", total);
        System.out.println("╚════════════════════════════════════════╝\n");
    }
    
    // Getters
    public String getId() { return id; }
    public Booking getBooking() { return booking; }
    public Map<String, BigDecimal> getCharges() { return charges; }
    public BigDecimal getTotal() { return total; }
    
    public static class Builder {
        private final Booking booking;
        private final Map<String, BigDecimal> charges = new LinkedHashMap<>();
        private BigDecimal total = BigDecimal.ZERO;
        
        Builder(Booking booking) {
            this.booking = booking;
        }
        
        public Builder addCharge(String description, BigDecimal amount) {
            charges.put(description, amount);
            total = total.add(amount);
            return this;
        }
        
        public Invoice build() {
            return new Invoice(this);
        }
    }
}
```

### 2.13 BookingService Class

```java
// BookingService.java
package com.hotel;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * Main service for managing hotel bookings.
 */
public class BookingService {
    
    private final Hotel hotel;
    private final Map<String, Booking> bookings;
    private final Map<Room, List<DateRange>> roomBookings;
    private final PricingStrategy pricingStrategy;
    private final CancellationPolicy cancellationPolicy;
    
    public BookingService(Hotel hotel) {
        this(hotel, new StandardPricingStrategy(), new FlexibleCancellationPolicy());
    }
    
    public BookingService(Hotel hotel, PricingStrategy pricingStrategy, 
                          CancellationPolicy cancellationPolicy) {
        this.hotel = hotel;
        this.bookings = new ConcurrentHashMap<>();
        this.roomBookings = new ConcurrentHashMap<>();
        this.pricingStrategy = pricingStrategy;
        this.cancellationPolicy = cancellationPolicy;
    }
    
    // ==================== Search ====================
    
    public List<Room> searchRooms(SearchCriteria criteria) {
        return hotel.getRooms().stream()
            .filter(room -> matchesCriteria(room, criteria))
            .filter(room -> isRoomAvailable(room, criteria.getDateRange()))
            .sorted(Comparator.comparing(Room::getBasePrice))
            .collect(Collectors.toList());
    }
    
    private boolean matchesCriteria(Room room, SearchCriteria criteria) {
        // Check room type
        if (criteria.getRoomType() != null && room.getType() != criteria.getRoomType()) {
            return false;
        }
        
        // Check capacity
        if (room.getType().getMaxOccupancy() < criteria.getGuests()) {
            return false;
        }
        
        // Check price
        if (criteria.getMaxPrice() != null) {
            BigDecimal price = pricingStrategy.calculatePrice(
                room, criteria.getDateRange(), criteria.getGuests());
            if (price.compareTo(criteria.getMaxPrice()) > 0) {
                return false;
            }
        }
        
        // Check amenities
        for (String amenity : criteria.getRequiredAmenities()) {
            if (!room.hasAmenity(amenity)) {
                return false;
            }
        }
        
        // Check floor preference
        if (criteria.getPreferredFloor() != null && 
            room.getFloor() != criteria.getPreferredFloor()) {
            return false;
        }
        
        return true;
    }
    
    private boolean isRoomAvailable(Room room, DateRange dateRange) {
        List<DateRange> bookedRanges = roomBookings.get(room);
        if (bookedRanges == null) {
            return true;
        }
        
        for (DateRange booked : bookedRanges) {
            if (dateRange.overlaps(booked)) {
                return false;
            }
        }
        return true;
    }
    
    // ==================== Booking ====================
    
    public synchronized Booking makeBooking(Guest guest, Room room, 
                                            DateRange dateRange, int numberOfGuests) {
        // Validate
        if (!isRoomAvailable(room, dateRange)) {
            throw new IllegalStateException("Room not available for selected dates");
        }
        
        if (numberOfGuests > room.getType().getMaxOccupancy()) {
            throw new IllegalArgumentException("Too many guests for room type");
        }
        
        // Calculate price
        BigDecimal totalPrice = pricingStrategy.calculatePrice(
            room, dateRange, numberOfGuests);
        
        // Create booking
        Booking booking = new Booking(guest, room, dateRange, numberOfGuests, totalPrice);
        
        // Reserve room and dates
        roomBookings.computeIfAbsent(room, k -> new ArrayList<>()).add(dateRange);
        bookings.put(booking.getId(), booking);
        
        return booking;
    }
    
    // ==================== Cancellation ====================
    
    public BigDecimal cancelBooking(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) {
            throw new IllegalArgumentException("Booking not found");
        }
        
        if (!booking.isActive()) {
            throw new IllegalStateException("Booking is not active");
        }
        
        // Calculate refund
        BigDecimal refund = cancellationPolicy.calculateRefund(booking);
        
        // Cancel booking
        booking.cancel();
        
        // Release room dates
        List<DateRange> bookedRanges = roomBookings.get(booking.getRoom());
        if (bookedRanges != null) {
            bookedRanges.remove(booking.getDateRange());
        }
        
        return refund;
    }
    
    // ==================== Check-in/Check-out ====================
    
    public void checkIn(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) {
            throw new IllegalArgumentException("Booking not found");
        }
        
        // Verify check-in date
        if (LocalDate.now().isBefore(booking.getDateRange().getCheckIn())) {
            throw new IllegalStateException("Cannot check in before reservation date");
        }
        
        booking.checkIn();
        booking.getRoom().checkIn();
    }
    
    public Invoice checkOut(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) {
            throw new IllegalArgumentException("Booking not found");
        }
        
        booking.checkOut();
        booking.getRoom().checkOut();
        
        // Release room dates
        List<DateRange> bookedRanges = roomBookings.get(booking.getRoom());
        if (bookedRanges != null) {
            bookedRanges.remove(booking.getDateRange());
        }
        
        return generateInvoice(booking);
    }
    
    private Invoice generateInvoice(Booking booking) {
        return Invoice.builder(booking)
            .addCharge("Room (" + booking.getDateRange().getNights() + " nights)", 
                      booking.getTotalPrice())
            .build();
    }
    
    // ==================== Queries ====================
    
    public Booking getBooking(String id) {
        return bookings.get(id);
    }
    
    public List<Booking> getBookingsForDate(LocalDate date) {
        return bookings.values().stream()
            .filter(b -> b.getDateRange().contains(date))
            .filter(Booking::isActive)
            .collect(Collectors.toList());
    }
    
    public List<Booking> getGuestBookings(Guest guest) {
        return bookings.values().stream()
            .filter(b -> b.getGuest().getId().equals(guest.getId()))
            .collect(Collectors.toList());
    }
    
    public Hotel getHotel() {
        return hotel;
    }
    
    public CancellationPolicy getCancellationPolicy() {
        return cancellationPolicy;
    }
}
```

### 2.14 Demo Application

```java
// HotelBookingDemo.java
package com.hotel;

import java.math.BigDecimal;
import java.time.LocalDate;

public class HotelBookingDemo {
    
    public static void main(String[] args) {
        System.out.println("=== HOTEL BOOKING SYSTEM DEMO ===\n");
        
        // ==================== Setup Hotel ====================
        System.out.println("===== SETTING UP HOTEL =====\n");
        
        Hotel hotel = new Hotel("Grand Plaza Hotel", "123 Main Street", 
                               "New York", 4);
        
        // Add amenities
        hotel.addAmenity("Pool");
        hotel.addAmenity("Gym");
        hotel.addAmenity("Restaurant");
        hotel.addAmenity("Spa");
        
        // Add rooms
        for (int i = 1; i <= 3; i++) {
            Room single = new Room("10" + i, RoomType.SINGLE, 1);
            Room doubleRoom = new Room("20" + i, RoomType.DOUBLE, 2);
            doubleRoom.addAmenity("City View");
            Room suite = new Room("30" + i, RoomType.SUITE, 3);
            suite.addAmenity("City View");
            suite.addAmenity("Balcony");
            
            hotel.addRoom(single);
            hotel.addRoom(doubleRoom);
            hotel.addRoom(suite);
        }
        
        System.out.println("Hotel: " + hotel);
        System.out.println("Total rooms: " + hotel.getRooms().size());
        
        // Use seasonal pricing
        SeasonalPricingStrategy pricing = new SeasonalPricingStrategy();
        pricing.addHoliday(LocalDate.of(2024, 12, 25));
        pricing.addHoliday(LocalDate.of(2024, 12, 31));
        
        BookingService service = new BookingService(hotel, pricing, 
            new ModerateCancellationPolicy());
        
        System.out.println("Cancellation policy: " + 
                          service.getCancellationPolicy().getDescription());
        
        // ==================== Search Rooms ====================
        System.out.println("\n===== SEARCHING ROOMS =====\n");
        
        LocalDate checkIn = LocalDate.now().plusDays(7);
        LocalDate checkOut = checkIn.plusDays(3);
        DateRange dateRange = new DateRange(checkIn, checkOut);
        
        SearchCriteria criteria = new SearchCriteria(dateRange, 2)
            .withRoomType(RoomType.DOUBLE);
        
        System.out.println("Searching for: " + dateRange + ", 2 guests, Double room");
        
        for (Room room : service.searchRooms(criteria)) {
            BigDecimal price = pricing.calculatePrice(room, dateRange, 2);
            System.out.println("  " + room + " - $" + price);
        }
        
        // ==================== Make Booking ====================
        System.out.println("\n===== MAKING BOOKING =====\n");
        
        Guest guest = new Guest("John", "Smith", "john@email.com", 
                               "555-1234", "DL123456");
        
        Room selectedRoom = service.searchRooms(criteria).get(0);
        Booking booking = service.makeBooking(guest, selectedRoom, dateRange, 2);
        booking.addSpecialRequest("Late check-in (after 10 PM)");
        booking.addSpecialRequest("Extra pillows");
        
        System.out.println("Created: " + booking);
        System.out.println("Total price: $" + booking.getTotalPrice());
        System.out.println("Special requests: " + booking.getSpecialRequests());
        
        // ==================== Cancel Booking ====================
        System.out.println("\n===== CANCELLATION SCENARIO =====\n");
        
        // Create another booking to cancel
        DateRange futureRange = new DateRange(
            LocalDate.now().plusDays(14), 
            LocalDate.now().plusDays(17));
        
        Guest guest2 = new Guest("Jane", "Doe", "jane@email.com", 
                                "555-5678", "DL789012");
        
        Booking bookingToCancel = service.makeBooking(
            guest2, hotel.getRoom("201"), futureRange, 2);
        
        System.out.println("Created booking: " + bookingToCancel);
        System.out.println("Days until check-in: " + bookingToCancel.getDaysUntilCheckIn());
        
        BigDecimal refund = service.cancelBooking(bookingToCancel.getId());
        System.out.println("Cancelled. Refund: $" + refund);
        System.out.println("Status: " + bookingToCancel.getStatus());
        
        // ==================== Check-in / Check-out ====================
        System.out.println("\n===== CHECK-IN / CHECK-OUT =====\n");
        
        // Create booking for today
        DateRange todayRange = new DateRange(LocalDate.now(), LocalDate.now().plusDays(2));
        Booking todayBooking = service.makeBooking(
            guest, hotel.getRoom("301"), todayRange, 2);
        
        System.out.println("Booking for today: " + todayBooking);
        
        // Check in
        service.checkIn(todayBooking.getId());
        System.out.println("Checked in. Status: " + todayBooking.getStatus());
        System.out.println("Room status: " + todayBooking.getRoom().getStatus());
        
        // Check out
        Invoice invoice = service.checkOut(todayBooking.getId());
        invoice.print();
        
        // Mark room as cleaned
        todayBooking.getRoom().markCleaned();
        System.out.println("Room cleaned. Status: " + todayBooking.getRoom().getStatus());
        
        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## File Structure

```
com/hotel/
├── RoomType.java
├── RoomStatus.java
├── BookingStatus.java
├── Room.java
├── Guest.java
├── DateRange.java
├── Booking.java
├── PricingStrategy.java
├── StandardPricingStrategy.java
├── SeasonalPricingStrategy.java
├── CancellationPolicy.java
├── FlexibleCancellationPolicy.java
├── ModerateCancellationPolicy.java
├── StrictCancellationPolicy.java
├── Hotel.java
├── SearchCriteria.java
├── Invoice.java
├── BookingService.java
└── HotelBookingDemo.java
```

