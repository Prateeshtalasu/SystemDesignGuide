# 🚗 Parking Lot System - Design Explanation

## SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

> **Definition**: A class should have only one reason to change.

Let's analyze each class:

| Class | Single Responsibility | Why It Works |
|-------|----------------------|--------------|
| `Vehicle` | Hold vehicle identity and type | Only changes if vehicle identification changes |
| `ParkingSpot` | Manage one parking space | Only changes if spot behavior changes |
| `ParkingTicket` | Track one parking session | Only changes if ticketing rules change |
| `ParkingFloor` | Manage spots on one floor | Only changes if floor management changes |
| `DisplayBoard` | Show availability | Only changes if display format changes |
| `PaymentService` | Process payments | Only changes if payment logic changes |
| `ParkingLot` | Orchestrate parking operations | This is the coordination point |

**Potential SRP Violation - ParkingTicket:**

```java
// ParkingTicket currently calculates fees
public double calculateFee() {
    // Pricing logic here
}
```

**Why it's acceptable:**
- Fee calculation is directly tied to the ticket (duration, spot type)
- Extracting to a separate `PricingService` would be over-engineering for this scope
- In a larger system, you'd extract this to support dynamic pricing, promotions, etc.

**If we needed to fix it:**

```java
// FeeCalculator.java - Separate class for pricing
public class FeeCalculator {
    private final PricingStrategy strategy;
    
    public double calculate(ParkingTicket ticket) {
        return strategy.calculateFee(ticket);
    }
}
```

---

### 2. Open/Closed Principle (OCP)

> **Definition**: Software entities should be open for extension but closed for modification.

**How our design follows OCP:**

#### Adding New Vehicle Types

```java
// Current: Abstract Vehicle class with canFitInSpot() method
// To add Electric Vehicle:

public class ElectricCar extends Vehicle {
    
    public ElectricCar(String licensePlate) {
        super(licensePlate, VehicleType.ELECTRIC_CAR);
    }
    
    @Override
    public boolean canFitInSpot(ParkingSpotType spotType) {
        // Electric cars need charging spots or regular spots
        return spotType == ParkingSpotType.ELECTRIC_CHARGING ||
               spotType == ParkingSpotType.COMPACT ||
               spotType == ParkingSpotType.LARGE;
    }
}
```

**No existing code modified!** We just:
1. Add new enum value: `VehicleType.ELECTRIC_CAR`
2. Add new spot type: `ParkingSpotType.ELECTRIC_CHARGING`
3. Create new classes

#### Adding New Payment Methods

```java
// To add Mobile Payment (Apple Pay, Google Pay):

public class MobilePayment extends Payment {
    private final String walletId;
    
    public MobilePayment(double amount, String walletId) {
        super(amount);
        this.walletId = walletId;
    }
    
    @Override
    public boolean processPayment() {
        // Mobile payment processing logic
        return true;
    }
}
```

**No changes to existing Payment classes!**

#### Adding New Spot Types

```java
// To add Electric Charging Spot:

public class ElectricChargingSpot extends ParkingSpot {
    private final double chargingRate; // kW
    
    public ElectricChargingSpot(String spotId, int floorNumber, double chargingRate) {
        super(spotId, ParkingSpotType.ELECTRIC_CHARGING, floorNumber);
        this.chargingRate = chargingRate;
    }
    
    public double getChargingRate() {
        return chargingRate;
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

> **Definition**: Subtypes must be substitutable for their base types without altering program correctness.

**Testing LSP in our design:**

```java
// This code should work with ANY Vehicle subtype
public void testLSP() {
    List<Vehicle> vehicles = Arrays.asList(
        new Car("CAR-001"),
        new Motorcycle("BIKE-001"),
        new Truck("TRUCK-001")
    );
    
    for (Vehicle vehicle : vehicles) {
        // All vehicles can be checked for spot compatibility
        boolean canFit = vehicle.canFitInSpot(ParkingSpotType.LARGE);
        
        // All vehicles have a license plate
        String plate = vehicle.getLicensePlate();
        
        // All vehicles have a type
        VehicleType type = vehicle.getType();
    }
}
```

**LSP holds because:**
- All `Vehicle` subclasses implement `canFitInSpot()` correctly
- No subclass throws unexpected exceptions
- No subclass has stricter preconditions
- All subclasses maintain the same contract

**Potential LSP Violation to Avoid:**

```java
// BAD: This would violate LSP
public class BrokenMotorcycle extends Vehicle {
    @Override
    public boolean canFitInSpot(ParkingSpotType spotType) {
        throw new UnsupportedOperationException("Motorcycles park differently!");
    }
}
```

This breaks LSP because code expecting a `Vehicle` would crash.

---

### 4. Interface Segregation Principle (ISP)

> **Definition**: Clients should not be forced to depend on interfaces they don't use.

**Current design analysis:**

We don't have explicit interfaces in the current design, but we could improve:

```java
// Current: ParkingSpot is an abstract class
// Better: Use interfaces for specific capabilities

public interface Parkable {
    boolean canFitVehicle(Vehicle vehicle);
    boolean parkVehicle(Vehicle vehicle);
    Vehicle removeVehicle();
}

public interface Reservable {
    boolean reserve(String reservationId, Duration duration);
    void cancelReservation(String reservationId);
}

public interface ChargingCapable {
    void startCharging(ElectricVehicle vehicle);
    void stopCharging();
    double getChargingRate();
}
```

**With ISP applied:**

```java
// Regular spot only implements Parkable
public class CompactSpot implements Parkable {
    // Only parking methods
}

// Electric spot implements both
public class ElectricChargingSpot implements Parkable, ChargingCapable {
    // Parking + charging methods
}

// VIP spot can be reserved
public class VIPSpot implements Parkable, Reservable {
    // Parking + reservation methods
}
```

**Why we didn't over-apply ISP:**
- For interview scope, abstract classes are sufficient
- Adding interfaces everywhere adds complexity without clear benefit
- ISP becomes important when you have clients that need different capabilities

---

### 5. Dependency Inversion Principle (DIP)

> **Definition**: High-level modules should not depend on low-level modules. Both should depend on abstractions.

**Current implementation:**

```java
// ExitPanel depends on concrete PaymentService
public class ExitPanel {
    private final PaymentService paymentService;  // Concrete class
}
```

**Better with DIP:**

```java
// Define abstraction
public interface PaymentProcessor {
    boolean processPayment(ParkingTicket ticket, double amount);
}

// Concrete implementation
public class PaymentService implements PaymentProcessor {
    @Override
    public boolean processPayment(ParkingTicket ticket, double amount) {
        // Implementation
    }
}

// High-level module depends on abstraction
public class ExitPanel {
    private final PaymentProcessor paymentProcessor;  // Interface!
    
    public ExitPanel(String panelId, ParkingLot parkingLot, 
                     PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }
}
```

**Benefits of DIP:**
- Easy to swap payment providers (Stripe → Square)
- Easy to mock in tests
- Decouples exit panel from payment implementation

**Where DIP is applied well:**

```java
// Vehicle is abstract, ParkingLot works with abstraction
public synchronized ParkingTicket parkVehicle(Vehicle vehicle) {
    // Works with ANY vehicle type
    ParkingSpot spot = findAvailableSpot(vehicle);
    // ...
}
```

---

## Design Patterns Used

### 1. Singleton Pattern

**Where:** `ParkingLot` class

**Why it fits:**
- Only one parking lot should exist in the system
- All entry/exit panels must share the same state
- Prevents inconsistent availability counts

```java
public class ParkingLot {
    private static ParkingLot instance;
    
    private ParkingLot(String name, String address) {
        // Private constructor
    }
    
    public static synchronized ParkingLot getInstance(String name, String address) {
        if (instance == null) {
            instance = new ParkingLot(name, address);
        }
        return instance;
    }
}
```

**Trade-offs:**
| Pros | Cons |
|------|------|
| Guaranteed single instance | Harder to unit test |
| Global access point | Hidden dependencies |
| Lazy initialization | Can become a god object |

**Interview tip:** Mention that in production, you might use dependency injection instead of Singleton for better testability.

---

### 2. Strategy Pattern

**Where:** `Payment` hierarchy

**Why it fits:**
- Different payment methods have different algorithms
- Need to switch payment methods at runtime
- Encapsulates payment-specific logic

```java
// Strategy interface (implicit via abstract class)
public abstract class Payment {
    public abstract boolean processPayment();
}

// Concrete strategies
public class CashPayment extends Payment {
    @Override
    public boolean processPayment() {
        // Cash-specific logic
    }
}

public class CardPayment extends Payment {
    @Override
    public boolean processPayment() {
        // Card-specific logic
    }
}
```

**How it's used:**

```java
public class PaymentService {
    public boolean processPayment(ParkingTicket ticket, double amount, 
                                  PaymentType type) {
        Payment payment;
        switch (type) {
            case CASH:
                payment = new CashPayment(amount, cashTendered);
                break;
            case CARD:
                payment = new CardPayment(amount, cardNumber);
                break;
            default:
                throw new IllegalArgumentException("Unknown payment type");
        }
        return payment.processPayment();
    }
}
```

---

### 3. Factory Pattern (Implicit)

**Where:** Spot creation in `ParkingFloor`

**Why it fits:**
- Encapsulates object creation logic
- Caller doesn't need to know concrete classes

**Current (simple) approach:**

```java
// Direct instantiation
floor.addParkingSpot(new CompactSpot("F0-C001", 0));
```

**With Factory Pattern:**

```java
public class ParkingSpotFactory {
    
    public static ParkingSpot createSpot(ParkingSpotType type, 
                                         String spotId, int floor) {
        switch (type) {
            case COMPACT:
                return new CompactSpot(spotId, floor);
            case LARGE:
                return new LargeSpot(spotId, floor);
            case HANDICAPPED:
                return new HandicappedSpot(spotId, floor);
            default:
                throw new IllegalArgumentException("Unknown spot type: " + type);
        }
    }
}

// Usage
floor.addParkingSpot(ParkingSpotFactory.createSpot(
    ParkingSpotType.COMPACT, "F0-C001", 0));
```

**When to use Factory:**
- When creation logic is complex
- When you need to hide concrete classes
- When creation depends on configuration

---

### 4. Composition over Inheritance

**Where:** Throughout the design

**Examples:**

```java
// ParkingFloor CONTAINS DisplayBoard (composition)
public class ParkingFloor {
    private final DisplayBoard displayBoard;  // Owns the display
}

// ParkingLot CONTAINS ParkingFloors (composition)
public class ParkingLot {
    private final List<ParkingFloor> floors;  // Owns the floors
}
```

**Why composition:**
- Floors can't exist without a parking lot
- Display boards are integral to floors
- Changes to one don't require changes to others

---

## Why Alternatives Were Rejected

### Alternative 1: Single ParkingSpot Class with Type Field

```java
// Rejected approach
public class ParkingSpot {
    private ParkingSpotType type;
    
    public boolean canFitVehicle(Vehicle vehicle) {
        // Giant switch statement
        switch (type) {
            case COMPACT:
                // Logic for compact
            case LARGE:
                // Logic for large
            // ... more cases
        }
    }
}
```

**Why rejected:**
- Violates OCP (must modify class to add new types)
- Giant switch statements are hard to maintain
- Can't add type-specific behavior easily

---

### Alternative 2: Vehicle Stores Its Parking Spot

```java
// Rejected approach
public class Vehicle {
    private ParkingSpot currentSpot;  // Vehicle knows where it's parked
}
```

**Why rejected:**
- Bidirectional dependency (Vehicle ↔ Spot)
- Harder to manage state consistency
- Vehicle shouldn't know about parking infrastructure

**Better approach:** Ticket connects Vehicle and Spot

---

### Alternative 3: Global Spot List Instead of Floors

```java
// Rejected approach
public class ParkingLot {
    private List<ParkingSpot> allSpots;  // No floor organization
}
```

**Why rejected:**
- Loses physical organization
- Can't show per-floor availability
- Harder to implement floor-specific features (e.g., "park near entrance")

---

### Alternative 4: Ticket Contains Payment Logic

```java
// Rejected approach
public class ParkingTicket {
    public boolean processPayment(String cardNumber) {
        // Payment processing in ticket
    }
}
```

**Why rejected:**
- Violates SRP (ticket handles both tracking AND payment)
- Payment logic should be separate
- Can't easily swap payment providers

---

## What Would Break Without Each Class

| Class | What Breaks Without It |
|-------|----------------------|
| `Vehicle` (abstract) | Can't enforce vehicle types, no polymorphism |
| `ParkingSpot` (abstract) | Can't have different spot behaviors |
| `ParkingTicket` | Can't track parking sessions or calculate fees |
| `ParkingFloor` | Can't organize spots by location |
| `DisplayBoard` | Users can't see availability |
| `EntryPanel` | No controlled entry point |
| `ExitPanel` | No controlled exit, no payment processing |
| `PaymentService` | Can't collect parking fees |
| `ParkingLot` | No central coordination, inconsistent state |

---

## Class Interaction at Runtime

### Entry Flow

```
User arrives → EntryPanel.processEntry(vehicle)
                    │
                    ▼
            ParkingLot.parkVehicle(vehicle)
                    │
                    ├──► Check: vehicleToTicket.containsKey(plate)?
                    │         Yes → Return null (already parked)
                    │
                    ├──► findAvailableSpot(vehicle)
                    │         │
                    │         ▼
                    │    For each ParkingFloor:
                    │         floor.findAvailableSpot(vehicle)
                    │              │
                    │              ▼
                    │         For each spot in searchOrder:
                    │              spot.canFitVehicle(vehicle)?
                    │                   │
                    │                   ▼
                    │              vehicle.canFitInSpot(spotType)?
                    │
                    ├──► spot.parkVehicle(vehicle)
                    │         │
                    │         ▼
                    │    spot.isAvailable = false
                    │    spot.parkedVehicle = vehicle
                    │
                    ├──► Create ParkingTicket
                    │
                    ├──► activeTickets.put(ticketId, ticket)
                    │    vehicleToTicket.put(plate, ticket)
                    │
                    └──► updateAllDisplayBoards()
                              │
                              ▼
                         For each floor:
                              floor.updateDisplayBoard()
                                   │
                                   ▼
                              displayBoard.update(counts)
```

### Exit Flow

```
User at exit → ExitPanel.processExit(ticket)
                    │
                    ├──► ticket.calculateFee()
                    │         │
                    │         ▼
                    │    Duration = now - entryTime
                    │    Hours = ceiling(minutes / 60)
                    │    Fee = hours × hourlyRate
                    │
                    ├──► paymentService.processPayment(ticket, fee)
                    │         │
                    │         ▼
                    │    Payment.processPayment()
                    │         │
                    │         ▼
                    │    ticket.markAsPaid(fee)
                    │
                    └──► parkingLot.releaseSpot(ticket)
                              │
                              ├──► spot.removeVehicle()
                              │
                              ├──► activeTickets.remove(ticketId)
                              │    vehicleToTicket.remove(plate)
                              │
                              └──► updateAllDisplayBoards()
```

---

## Thread Safety Considerations

### Where Synchronization is Needed

1. **Spot allocation** - Two cars can't get the same spot

```java
public synchronized ParkingTicket parkVehicle(Vehicle vehicle) {
    // Only one thread can allocate at a time
}
```

2. **Spot state changes** - Park/remove must be atomic

```java
public synchronized boolean parkVehicle(Vehicle vehicle) {
    if (!canFitVehicle(vehicle)) return false;
    this.parkedVehicle = vehicle;
    this.isAvailable = false;
    return true;
}
```

3. **Ticket tracking** - Use ConcurrentHashMap

```java
private final Map<String, ParkingTicket> activeTickets = 
    new ConcurrentHashMap<>();
```

### Race Condition Example (Without Sync)

```
Thread A: Check spot F0-C001 available? → Yes
Thread B: Check spot F0-C001 available? → Yes
Thread A: Park in F0-C001 → Success
Thread B: Park in F0-C001 → COLLISION! Two cars, one spot
```

### With Synchronization

```
Thread A: Acquire lock on parkVehicle()
Thread A: Check spot F0-C001 available? → Yes
Thread A: Park in F0-C001 → Success
Thread A: Release lock
Thread B: Acquire lock on parkVehicle()
Thread B: Check spot F0-C001 available? → No
Thread B: Find next available spot
```

---

## Extensibility Analysis

### Adding Valet Parking

```java
// New class
public class ValetService {
    private final Queue<Vehicle> waitingVehicles;
    private final List<ValetAttendant> attendants;
    
    public ParkingTicket parkWithValet(Vehicle vehicle) {
        // Attendant parks the vehicle
        // Returns ticket with valet flag
    }
    
    public Vehicle retrieveVehicle(ParkingTicket ticket) {
        // Attendant retrieves the vehicle
    }
}
```

**Changes needed:** None to existing classes!

### Adding Reservation System

```java
public class ReservationService {
    private final Map<String, Reservation> reservations;
    
    public Reservation reserve(ParkingSpotType type, 
                               LocalDateTime startTime,
                               Duration duration) {
        // Find and reserve a spot
    }
}

public class Reservation {
    private final String reservationId;
    private final ParkingSpot spot;
    private final LocalDateTime startTime;
    private final Duration duration;
}
```

**Changes needed:** 
- Add `reserved` flag to `ParkingSpot`
- Modify `findAvailableSpot()` to skip reserved spots

### Adding Monthly Pass

```java
public class MonthlyPass {
    private final String passId;
    private final Vehicle vehicle;
    private final ParkingSpot assignedSpot;
    private final YearMonth validMonth;
    
    public boolean isValid() {
        return YearMonth.now().equals(validMonth);
    }
}
```

**Changes needed:**
- Modify entry logic to check for valid pass
- Skip payment for pass holders

