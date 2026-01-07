# 🍽️ Restaurant Reservation System - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Restaurant Reservation System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class                | Responsibility                  | Reason for Change         |
| -------------------- | ------------------------------- | ------------------------- |
| `Customer`           | Store customer information      | Customer data changes     |
| `Table`              | Represent a physical table      | Table properties change   |
| `TimeSlot`           | Represent a time period         | Time handling changes     |
| `Reservation`        | Store reservation details       | Reservation model changes |
| `Restaurant`         | Manage restaurant configuration | Restaurant setup changes  |
| `ReservationService` | Handle booking logic            | Booking rules change      |
| `WaitlistEntry`      | Track waitlist entries          | Waitlist features change  |

**SRP in Action:**

```java
// Table ONLY knows about its physical properties
public class Table {
    private final int capacity;
    private final Section section;

    public boolean canAccommodate(int partySize) {
        return partySize <= capacity;
    }
}

// ReservationService handles booking logic
public class ReservationService {
    public Reservation makeReservation(ReservationRequest request) {
        // Validation, availability check, booking
    }
}

// Restaurant manages configuration
public class Restaurant {
    public List<TimeSlot> getTimeSlots(LocalDate date) {
        // Generate time slots based on operating hours
    }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Table Types:**

```java
// No changes to existing code needed
public class BoothTable extends Table {
    private final boolean hasPrivacy;

    @Override
    public boolean canAccommodate(int partySize) {
        // Booths might have different rules
        return partySize >= 2 && partySize <= getCapacity();
    }
}

public class BarSeating extends Table {
    @Override
    public boolean canAccommodate(int partySize) {
        // Bar only for small parties
        return partySize <= 2;
    }
}
```

**Adding New Reservation Rules:**

```java
// Strategy pattern for allocation
public interface TableAllocationStrategy {
    Table findTable(List<Table> available, int partySize);
}

public class SmallestFitStrategy implements TableAllocationStrategy {
    @Override
    public Table findTable(List<Table> available, int partySize) {
        return available.stream()
            .filter(t -> t.canAccommodate(partySize))
            .min(Comparator.comparingInt(Table::getCapacity))
            .orElse(null);
    }
}

public class PreferredSectionStrategy implements TableAllocationStrategy {
    private final String preferredSection;

    @Override
    public Table findTable(List<Table> available, int partySize) {
        // Try preferred section first
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**Table hierarchy:**

```java
public abstract class Seating {
    public abstract int getCapacity();
    public abstract boolean canAccommodate(int partySize);
}

public class Table extends Seating { }
public class Booth extends Seating { }
public class BarSeat extends Seating { }

// All work in ReservationService
public Table findAvailableSeating(int partySize) {
    for (Seating seating : restaurant.getAllSeating()) {
        if (seating.canAccommodate(partySize)) {
            return seating;  // Works for any Seating type
        }
    }
    return null;
}
```

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

```java
// Could be improved with ISP
public interface Reservable {
    boolean isAvailable(LocalDate date, TimeSlot slot);
    void book(LocalDate date, TimeSlot slot);
    void release(LocalDate date, TimeSlot slot);
}

public interface Capacitated {
    int getCapacity();
    boolean canAccommodate(int partySize);
}

public class Table implements Reservable, Capacitated { }
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
// ReservationService depends on concrete Restaurant
public class ReservationService {
    private final Restaurant restaurant;
}
```

**Better with DIP:**

```java
// Define abstractions
public interface TableProvider {
    List<Table> getTablesForPartySize(int size);
}

public interface TimeSlotProvider {
    List<TimeSlot> getTimeSlots(LocalDate date);
    TimeSlot findTimeSlot(LocalDate date, LocalTime time);
}

// ReservationService depends on abstractions
public class ReservationService {
    private final TableProvider tableProvider;
    private final TimeSlotProvider timeSlotProvider;

    public ReservationService(TableProvider tp, TimeSlotProvider tsp) {
        this.tableProvider = tp;
        this.timeSlotProvider = tsp;
    }
}
```

**Why We Use Concrete Classes in This LLD Implementation:**

For low-level design interviews, we intentionally use concrete classes (`Restaurant` instead of `TableProvider`/`TimeSlotProvider` interfaces) for the following reasons:

1. **Scope Focus**: LLD interviews focus on reservation algorithms, state management, and data structures. Adding interface abstractions adds complexity without demonstrating core LLD skills.

2. **Single Domain Entity**: The system operates on a single, well-defined domain entity (Restaurant). There's no requirement for multiple table/time slot providers, so the abstraction doesn't provide value in the interview context.

3. **Interview Time Constraints**: Implementing full interface hierarchies takes time away from demonstrating more critical LLD concepts like reservation conflict resolution, time slot management, and state transitions.

4. **Production vs Interview**: In production systems, we would absolutely extract `TableProvider` and `TimeSlotProvider` interfaces for:
   - Testability (mock providers in unit tests)
   - Flexibility (swap implementations for different restaurant types)
   - Dependency injection (easier configuration and testing)

**The Trade-off:**
- **Interview Scope**: Concrete classes are simpler and focus on core LLD skills (algorithms, state management)
- **Production Scope**: Interfaces provide testability and flexibility at the cost of more abstraction layers

This is a conscious design decision for the interview context, not an oversight.

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Customer stores customer info, Table represents physical table, Reservation stores reservation details, ReservationService handles booking logic. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new table types like BoothTable, new allocation strategies) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All Table subclasses properly implement the Table contract. They are substitutable in reservation operations. | N/A | - |
| **ISP** | ACCEPTABLE (LLD Scope) | Could benefit from interface segregation (Reservable, Capacitated interfaces). For LLD interview scope, using concrete classes is acceptable as it focuses on core reservation algorithms. In production, we would extract Reservable and Capacitated interfaces. | See "Why We Use Concrete Classes" section above for detailed justification. This is an intentional design decision for interview context. | Interview: Simpler, focuses on core LLD skills. Production: More interfaces/files, but increases flexibility |
| **DIP** | ACCEPTABLE (LLD Scope) | ReservationService depends on concrete Restaurant. For LLD interview scope, this is acceptable as it focuses on core reservation logic. In production, we would depend on TableProvider and TimeSlotProvider interfaces. | See "Why We Use Concrete Classes" section above for detailed justification. This is an intentional design decision for interview context, not an oversight. | Interview: Simpler, focuses on core LLD skills. Production: More abstraction layers, but improves testability |

---

## Design Patterns Used

### 1. Builder Pattern

**Where:** ReservationRequest

```java
ReservationRequest request = new ReservationRequest(customer, date, time, partySize)
    .withSection("Patio")
    .withSpecialRequests("Birthday celebration");
```

**Benefits:**

- Optional parameters without constructor overloading
- Fluent, readable API
- Easy to extend with new options

---

### 2. Repository Pattern (Implicit)

**Where:** ReservationService manages reservations

```java
public class ReservationService {
    private final Map<String, Reservation> reservations;

    public Reservation getReservation(String id) {
        return reservations.get(id);
    }

    public List<Reservation> getReservationsForDate(LocalDate date) {
        return reservations.values().stream()
            .filter(r -> r.getDate().equals(date))
            .collect(Collectors.toList());
    }
}
```

---

### 3. Strategy Pattern (Potential)

**Where:** Table allocation could use strategies

```java
public interface AllocationStrategy {
    Table allocate(List<Table> available, ReservationRequest request);
}

public class FirstAvailableStrategy implements AllocationStrategy { }
public class SmallestTableStrategy implements AllocationStrategy { }
public class PreferredSectionStrategy implements AllocationStrategy { }

public class ReservationService {
    private AllocationStrategy strategy;

    public void setAllocationStrategy(AllocationStrategy strategy) {
        this.strategy = strategy;
    }
}
```

---

### 4. Observer Pattern (Potential)

**Where:** Waitlist notifications

```java
public interface ReservationObserver {
    void onCancellation(Reservation reservation);
    void onTableAvailable(Table table, TimeSlot slot);
}

public class WaitlistNotifier implements ReservationObserver {
    @Override
    public void onTableAvailable(Table table, TimeSlot slot) {
        // Notify waitlist entries
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single Reservation Class with All Logic

```java
// Rejected
public class Reservation {
    public static Reservation create(Customer c, Date d, Time t, int size) {
        // Find table
        // Check availability
        // Book table
        // Create reservation
    }
}
```

**Why rejected:**

- Violates SRP
- Hard to test
- Can't change booking logic without changing Reservation

### Alternative 2: Table Manages Its Own Bookings

```java
// Rejected
public class Table {
    private Map<LocalDate, Set<TimeSlot>> bookings;

    public boolean book(LocalDate date, TimeSlot slot) {
        // Table manages its own bookings
    }
}
```

**Why rejected:**

- Distributed state (hard to query across tables)
- Table shouldn't know about dates and time slots
- Difficult to implement cross-table queries

### Alternative 3: Time-Based Locking

```java
// Rejected
public class Table {
    private Lock lock;

    public void book(LocalDate date, TimeSlot slot) {
        lock.lock();
        try {
            // Book
        } finally {
            lock.unlock();
        }
    }
}
```

**Why rejected:**

- Per-table locking is complex
- Need to lock multiple tables for party splitting
- Centralized service with synchronized methods is simpler

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle concurrent reservations?

**Answer:**

```java
// Option 1: Synchronized method (current approach)
public synchronized Reservation makeReservation(...) { }

// Option 2: Optimistic locking
public Reservation makeReservation(...) {
    while (true) {
        long version = getBookingVersion(date, table, slot);
        if (isAvailable(date, table, slot)) {
            try {
                bookWithVersion(date, table, slot, version);
                return createReservation(...);
            } catch (OptimisticLockException e) {
                // Retry
            }
        }
    }
}

// Option 3: Per-table locks
private final Map<String, ReentrantLock> tableLocks;

public Reservation makeReservation(...) {
    Table table = findAvailableTable(...);
    ReentrantLock lock = tableLocks.get(table.getId());
    lock.lock();
    try {
        // Verify still available and book
        if (isAvailable(date, table, slot)) {
            return createReservation(...);
        }
    } finally {
        lock.unlock();
    }
}
```

---

### Q2: How would you handle walk-ins?

**Answer:**

```java
public class WalkInService {
    private final ReservationService reservationService;
    
    public Table findTableForWalkIn(int partySize) {
        LocalDate today = LocalDate.now();
        LocalTime now = LocalTime.now();
        
        // Find current time slot
        TimeSlot currentSlot = restaurant.findTimeSlot(today, now);
        
        // Find available table
        return reservationService.findAvailableTable(
            today, currentSlot, partySize, null);
    }
    
    public Reservation seatWalkIn(Customer customer, Table table) {
        // Create immediate reservation
        Reservation res = new Reservation(
            customer, LocalDate.now(), 
            restaurant.findTimeSlot(LocalDate.now(), LocalTime.now()),
            customer.getPartySize(), table);
        
        res.markSeated();
        return res;
    }
}
```

---

### Q3: How would you implement reminders?

**Answer:**

```java
public class ReminderService {
    private final ScheduledExecutorService scheduler;
    
    public void scheduleReminder(Reservation reservation) {
        LocalDateTime reminderTime = LocalDateTime.of(
            reservation.getDate(),
            reservation.getTimeSlot().getStartTime()
        ).minusHours(24);  // 24 hours before
        
        long delay = Duration.between(LocalDateTime.now(), reminderTime)
                            .toMillis();
        
        if (delay > 0) {
            scheduler.schedule(() -> sendReminder(reservation), 
                             delay, TimeUnit.MILLISECONDS);
        }
    }
    
    private void sendReminder(Reservation reservation) {
        if (reservation.isActive()) {
            // Send SMS/email
            notificationService.send(
                reservation.getCustomer(),
                "Reminder: Your reservation at " + restaurant.getName() +
                " is tomorrow at " + reservation.getTimeSlot().getStartTime()
            );
        }
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add deposit handling** - Require deposits for large parties
2. **Add loyalty program** - Track frequent customers, offer perks
3. **Add table preferences** - Allow customers to request specific tables
4. **Add cancellation policies** - Different policies for different time windows
5. **Add multi-restaurant support** - Chain restaurants with shared customer base
6. **Add analytics** - Track booking patterns, popular time slots
7. **Add dynamic pricing** - Adjust reservation fees based on demand
8. **Add table combination** - Combine tables for large parties

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation                | Complexity | Explanation                   |
| ------------------------ | ---------- | ----------------------------- |
| `makeReservation`        | O(t × s)   | t = tables, s = slots per day |
| `cancelReservation`      | O(w)       | w = waitlist size             |
| `findAvailableTable`     | O(t)       | t = suitable tables           |
| `getAvailableTimeSlots`  | O(s × t)   | s = slots, t = tables         |
| `getReservationsForDate` | O(r)       | r = total reservations        |

### Space Complexity

| Component      | Space        |
| -------------- | ------------ | ------------------------------- |
| Reservations   | O(r)         |
| Table bookings | O(d × t × s) | d = days, t = tables, s = slots |
| Waitlist       | O(w)         |

### Bottlenecks at Scale

**10x Usage (10 → 100 restaurants, 1K → 10K reservations/day):**
- Problem: Table availability search becomes slow (O(t × s)), concurrent reservation conflicts increase, waitlist management overhead grows
- Solution: Index available tables by date/slot, use distributed locking (Redis) for concurrent reservations, optimize search queries
- Tradeoff: Additional infrastructure (Redis), more complex locking logic

**100x Usage (10 → 1K restaurants, 1K → 100K reservations/day):**
- Problem: Single instance can't handle all restaurants, database becomes bottleneck, real-time availability queries too slow
- Solution: Shard restaurants by region, use read replicas for availability queries, implement caching layer (Redis) for hot data
- Tradeoff: Distributed system complexity, need shard routing and data consistency across shards


### Q1: How would you handle overbooking?

```java
public class OverbookingService {
    private final double overbookingRate = 0.1;  // 10% overbook

    public int getEffectiveCapacity(Table table) {
        return (int) (table.getCapacity() * (1 + overbookingRate));
    }

    public boolean canOverbook(LocalDate date, TimeSlot slot) {
        int totalCapacity = getTotalCapacity();
        int booked = getBookedCount(date, slot);
        int maxAllowed = (int) (totalCapacity * (1 + overbookingRate));
        return booked < maxAllowed;
    }
}
```

### Q2: How would you implement table combining?

```java
public class TableCombiner {
    public List<Table> findCombinableTables(List<Table> available, int partySize) {
        // Find adjacent tables that can be combined
        List<List<Table>> combinations = new ArrayList<>();

        for (Table t1 : available) {
            for (Table t2 : available) {
                if (areAdjacent(t1, t2) &&
                    t1.getCapacity() + t2.getCapacity() >= partySize) {
                    combinations.add(Arrays.asList(t1, t2));
                }
            }
        }

        // Return smallest combination
        return combinations.stream()
            .min(Comparator.comparingInt(this::totalCapacity))
            .orElse(null);
    }
}
```

### Q3: How would you handle recurring reservations?

```java
public class RecurringReservation {
    private final ReservationRequest template;
    private final RecurrencePattern pattern;
    private final LocalDate endDate;

    public List<Reservation> generateReservations(ReservationService service) {
        List<Reservation> reservations = new ArrayList<>();
        LocalDate current = template.getDate();

        while (!current.isAfter(endDate)) {
            try {
                ReservationRequest request = template.withDate(current);
                reservations.add(service.makeReservation(request));
            } catch (NoTableAvailableException e) {
                // Log and continue
            }
            current = pattern.nextOccurrence(current);
        }

        return reservations;
    }
}

public enum RecurrencePattern {
    DAILY, WEEKLY, BIWEEKLY, MONTHLY
}
```

### Q4: How would you implement dynamic pricing?

```java
public class DynamicPricing {
    private final Map<DayOfWeek, Double> dayMultipliers;
    private final Map<TimeSlot, Double> timeMultipliers;

    public double getDepositAmount(Reservation reservation) {
        double base = getBaseDeposit(reservation.getPartySize());

        double dayMultiplier = dayMultipliers.getOrDefault(
            reservation.getDate().getDayOfWeek(), 1.0);

        double timeMultiplier = timeMultipliers.getOrDefault(
            reservation.getTimeSlot(), 1.0);

        // Peak times (Friday/Saturday dinner) cost more
        return base * dayMultiplier * timeMultiplier;
    }

    private double getBaseDeposit(int partySize) {
        if (partySize >= 8) return 100.0;
        if (partySize >= 6) return 50.0;
        return 25.0;
    }
}
```
