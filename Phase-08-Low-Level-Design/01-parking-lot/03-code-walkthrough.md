# 🚗 Parking Lot System - Code Walkthrough

## Building From Scratch: Step-by-Step

This section explains how an engineer would build this system from scratch, in the order a real developer would write the code.

---

## Phase 1: Start with the Data Model (Enums First)

### Why enums first?

Enums define the vocabulary of your system. Before writing any logic, you need to know:
- What types of vehicles exist?
- What types of spots exist?
- What states can a ticket be in?

```java
// Step 1: Define VehicleType enum
// File: com/parkinglot/enums/VehicleType.java

package com.parkinglot.enums;

public enum VehicleType {
    MOTORCYCLE,
    CAR,
    TRUCK
}
```

**Line-by-line:**
- `package com.parkinglot.enums;` - Organizes enums in their own package
- `public enum VehicleType` - Enum is public so all classes can use it
- Each value represents a distinct vehicle category

```java
// Step 2: Define ParkingSpotType enum
// File: com/parkinglot/enums/ParkingSpotType.java

package com.parkinglot.enums;

public enum ParkingSpotType {
    COMPACT,
    LARGE,
    HANDICAPPED
}
```

**Why separate enums for Vehicle and Spot types?**
- They're different concepts (vehicle IS a type, spot HAS a type)
- They change independently (might add ELECTRIC_CHARGING spot without new vehicle type)
- Keeps mapping logic in one place (Vehicle.canFitInSpot)

---

## Phase 2: Build the Core Domain Objects

### Step 3: Abstract Vehicle Class

```java
// File: com/parkinglot/models/Vehicle.java

package com.parkinglot.models;

import com.parkinglot.enums.VehicleType;
import com.parkinglot.enums.ParkingSpotType;

public abstract class Vehicle {
    
    // Fields are final because vehicle identity doesn't change
    private final String licensePlate;
    private final VehicleType type;
```

**Why `final` fields?**
- A car's license plate doesn't change while parked
- Immutability prevents bugs (no accidental modification)
- Thread-safe without synchronization

```java
    // Protected constructor - only subclasses call this
    protected Vehicle(String licensePlate, VehicleType type) {
        // Validate input immediately
        if (licensePlate == null || licensePlate.trim().isEmpty()) {
            throw new IllegalArgumentException(
                "License plate cannot be null or empty");
        }
        // Normalize the plate (uppercase, trimmed)
        this.licensePlate = licensePlate.toUpperCase().trim();
        this.type = type;
    }
```

**Why protected constructor?**
- Prevents direct instantiation of `Vehicle`
- Forces use of concrete subclasses (Car, Truck, etc.)
- Ensures every vehicle has a valid type

**Why normalize the license plate?**
- "abc-1234" and "ABC-1234" should be the same vehicle
- Prevents duplicate entries due to case differences
- Consistent format for lookups

```java
    // Abstract method - each vehicle type defines its own rules
    public abstract boolean canFitInSpot(ParkingSpotType spotType);
```

**Why abstract?**
- Different vehicles have different size constraints
- A motorcycle can fit anywhere, a truck needs large spots
- Polymorphism lets us treat all vehicles uniformly

```java
    // Override equals/hashCode for proper collection behavior
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Vehicle vehicle = (Vehicle) obj;
        return licensePlate.equals(vehicle.licensePlate);
    }
    
    @Override
    public int hashCode() {
        return licensePlate.hashCode();
    }
}
```

**Why override equals/hashCode?**
- Two Vehicle objects with same plate should be "equal"
- Required for correct HashMap/HashSet behavior
- Without this, `vehicleToTicket.get(vehicle)` might fail

### Step 4: Concrete Vehicle Classes

```java
// File: com/parkinglot/models/Car.java

package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;
import com.parkinglot.enums.VehicleType;

public class Car extends Vehicle {
    
    // Additional field specific to cars
    private final boolean hasHandicappedPermit;
    
    // Simple constructor for regular cars
    public Car(String licensePlate) {
        this(licensePlate, false);
    }
    
    // Full constructor
    public Car(String licensePlate, boolean hasHandicappedPermit) {
        super(licensePlate, VehicleType.CAR);
        this.hasHandicappedPermit = hasHandicappedPermit;
    }
    
    @Override
    public boolean canFitInSpot(ParkingSpotType spotType) {
        switch (spotType) {
            case COMPACT:
            case LARGE:
                return true;  // Cars fit in compact and large
            case HANDICAPPED:
                return hasHandicappedPermit;  // Only with permit
            default:
                return false;
        }
    }
}
```

**Design decisions explained:**
1. **Two constructors** - Most cars don't have permits, so provide a simple constructor
2. **`hasHandicappedPermit` field** - Only Car has this, not all vehicles
3. **Switch statement** - Clear, readable logic for spot compatibility

---

## Phase 3: Build the Parking Spot Hierarchy

### Step 5: Abstract ParkingSpot Class

```java
// File: com/parkinglot/models/ParkingSpot.java

package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;

public abstract class ParkingSpot {
    
    // Identity fields (immutable)
    private final String spotId;
    private final ParkingSpotType type;
    private final int floorNumber;
    
    // State fields (mutable)
    private Vehicle parkedVehicle;
    private boolean isAvailable;
```

**Why separate identity and state?**
- Identity never changes (spot F0-C001 is always F0-C001)
- State changes frequently (vehicle parks/leaves)
- Clear separation helps with debugging

```java
    // Core method: Can this vehicle park here?
    public boolean canFitVehicle(Vehicle vehicle) {
        // Two checks:
        // 1. Is the spot available?
        if (!isAvailable) {
            return false;
        }
        // 2. Does the vehicle fit this spot type?
        return vehicle.canFitInSpot(this.type);
    }
```

**Notice the delegation:**
- Spot checks availability
- Vehicle checks compatibility
- Each class handles its own concern

```java
    // Thread-safe parking operation
    public synchronized boolean parkVehicle(Vehicle vehicle) {
        if (!canFitVehicle(vehicle)) {
            return false;
        }
        this.parkedVehicle = vehicle;
        this.isAvailable = false;
        return true;
    }
    
    public synchronized Vehicle removeVehicle() {
        Vehicle vehicle = this.parkedVehicle;
        this.parkedVehicle = null;
        this.isAvailable = true;
        return vehicle;
    }
}
```

**Why synchronized?**
- Multiple threads (entry panels) might try to park simultaneously
- Without sync: Thread A checks available, Thread B checks available, both try to park
- With sync: Only one thread can modify state at a time

---

## Phase 4: Build the Ticket System

### Step 6: ParkingTicket Class

```java
// File: com/parkinglot/models/ParkingTicket.java

package com.parkinglot.models;

import com.parkinglot.enums.TicketStatus;
import java.time.LocalDateTime;
import java.time.Duration;
import java.util.UUID;

public class ParkingTicket {
    
    // Pricing constants (could be externalized to config)
    private static final double COMPACT_HOURLY_RATE = 2.0;
    private static final double LARGE_HOURLY_RATE = 3.0;
    private static final double HANDICAPPED_HOURLY_RATE = 1.5;
```

**Why static constants?**
- Rates are the same for all tickets
- Easy to find and modify
- Could be moved to a config file in production

```java
    // Generate unique ticket ID
    private String generateTicketId() {
        return "TKT-" + System.currentTimeMillis() + "-" + 
               UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
```

**ID format: TKT-1703234567890-A1B2C3D4**
- Prefix "TKT-" for easy identification
- Timestamp for rough ordering
- UUID suffix for uniqueness
- In production: Use Snowflake ID for distributed systems

```java
    // Fee calculation logic
    public double calculateFee() {
        // Use exit time if set, otherwise current time
        LocalDateTime endTime = (exitTime != null) ? exitTime : LocalDateTime.now();
        
        // Calculate duration
        Duration duration = Duration.between(entryTime, endTime);
        
        // Convert to hours (round up)
        long minutes = duration.toMinutes();
        long hours = (minutes + 59) / 60;  // Ceiling division
        if (hours == 0) hours = 1;  // Minimum 1 hour
        
        // Get rate and calculate
        double hourlyRate = getHourlyRate();
        return hours * hourlyRate;
    }
```

**Fee calculation walkthrough:**

| Entry | Exit | Minutes | Hours (rounded) | Rate | Fee |
|-------|------|---------|-----------------|------|-----|
| 10:00 | 10:30 | 30 | 1 | $2 | $2 |
| 10:00 | 11:01 | 61 | 2 | $2 | $4 |
| 10:00 | 14:45 | 285 | 5 | $2 | $10 |

---

## Phase 5: Build the Floor Management

### Step 7: ParkingFloor Class

```java
// File: com/parkinglot/models/ParkingFloor.java

package com.parkinglot.models;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class ParkingFloor {
    
    // Organize spots by type for efficient lookup
    private final Map<ParkingSpotType, List<ParkingSpot>> spotsByType;
    
    // Quick lookup by ID
    private final Map<String, ParkingSpot> spotsById;
```

**Two data structures, why?**
- `spotsByType`: Fast when searching for "any compact spot"
- `spotsById`: Fast when looking up specific spot "F0-C001"
- Trade-off: More memory, faster operations

```java
    // Find available spot with smart allocation
    public synchronized ParkingSpot findAvailableSpot(Vehicle vehicle) {
        // Get search order based on vehicle type
        List<ParkingSpotType> searchOrder = getSearchOrder(vehicle);
        
        // Try each spot type in order
        for (ParkingSpotType spotType : searchOrder) {
            List<ParkingSpot> spots = spotsByType.get(spotType);
            for (ParkingSpot spot : spots) {
                if (spot.canFitVehicle(vehicle)) {
                    return spot;
                }
            }
        }
        
        return null;  // No spot found
    }
    
    private List<ParkingSpotType> getSearchOrder(Vehicle vehicle) {
        List<ParkingSpotType> order = new ArrayList<>();
        
        switch (vehicle.getType()) {
            case MOTORCYCLE:
                // Small vehicles: prefer small spots first
                order.add(ParkingSpotType.COMPACT);
                order.add(ParkingSpotType.LARGE);
                break;
            case CAR:
                order.add(ParkingSpotType.COMPACT);
                order.add(ParkingSpotType.LARGE);
                // Handicapped only if permitted
                if (vehicle instanceof Car && ((Car) vehicle).hasHandicappedPermit()) {
                    order.add(ParkingSpotType.HANDICAPPED);
                }
                break;
            case TRUCK:
                // Large vehicles: only large spots
                order.add(ParkingSpotType.LARGE);
                break;
        }
        
        return order;
    }
```

**Why search order matters:**

Without search order:
```
Motorcycle arrives → Takes LARGE spot
Truck arrives → No LARGE spots left → Can't park!
```

With search order:
```
Motorcycle arrives → Takes COMPACT spot (preferred)
Truck arrives → LARGE spot available → Parks successfully
```

---

## Phase 6: Build the Main Orchestrator

### Step 8: ParkingLot Singleton

```java
// File: com/parkinglot/models/ParkingLot.java

package com.parkinglot.models;

public class ParkingLot {
    
    // Singleton instance
    private static ParkingLot instance;
    
    // Private constructor
    private ParkingLot(String name, String address) {
        this.name = name;
        this.address = address;
        // Initialize collections...
    }
    
    // Thread-safe singleton accessor
    public static synchronized ParkingLot getInstance(String name, String address) {
        if (instance == null) {
            instance = new ParkingLot(name, address);
        }
        return instance;
    }
```

**Singleton explained:**
1. `private static instance` - Only one instance exists
2. `private constructor` - Can't create with `new`
3. `synchronized getInstance` - Thread-safe creation

```java
    // Main parking operation
    public synchronized ParkingTicket parkVehicle(Vehicle vehicle) {
        // Step 1: Check if already parked
        if (vehicleToTicket.containsKey(vehicle.getLicensePlate())) {
            System.out.println("Vehicle already parked!");
            return null;
        }
        
        // Step 2: Find available spot
        ParkingSpot spot = findAvailableSpot(vehicle);
        if (spot == null) {
            System.out.println("No available spot!");
            return null;
        }
        
        // Step 3: Park the vehicle
        spot.parkVehicle(vehicle);
        
        // Step 4: Create ticket
        ParkingTicket ticket = new ParkingTicket(vehicle, spot);
        
        // Step 5: Track the ticket
        activeTickets.put(ticket.getTicketId(), ticket);
        vehicleToTicket.put(vehicle.getLicensePlate(), ticket);
        
        // Step 6: Update displays
        updateAllDisplayBoards();
        
        return ticket;
    }
```

**State changes traced:**

```
Before parkVehicle("ABC-1234"):
  activeTickets: {}
  vehicleToTicket: {}
  F0-C001.isAvailable: true
  F0-C001.parkedVehicle: null

After parkVehicle("ABC-1234"):
  activeTickets: {"TKT-xxx": ticket}
  vehicleToTicket: {"ABC-1234": ticket}
  F0-C001.isAvailable: false
  F0-C001.parkedVehicle: Car("ABC-1234")
```

---

## Phase 7: Build Entry/Exit Panels

### Step 9: EntryPanel and ExitPanel

```java
// File: com/parkinglot/models/EntryPanel.java

public class EntryPanel {
    
    private final String panelId;
    private final ParkingLot parkingLot;
    
    public ParkingTicket processEntry(Vehicle vehicle) {
        System.out.println("Entry Panel " + panelId + 
                          ": Processing entry for " + vehicle.getLicensePlate());
        
        // Delegate to parking lot
        ParkingTicket ticket = parkingLot.parkVehicle(vehicle);
        
        if (ticket != null) {
            System.out.println("Ticket issued: " + ticket.getTicketId());
            System.out.println("Assigned spot: " + ticket.getParkingSpot().getSpotId());
        } else {
            System.out.println("Parking lot is full!");
        }
        
        return ticket;
    }
}
```

**Why separate panels from ParkingLot?**
- Multiple entry points exist (North entrance, South entrance)
- Each panel has its own identity for logging
- Panels handle user interaction, ParkingLot handles logic

---

## Testing Approach

### Unit Tests for Each Class

```java
// VehicleTest.java
@Test
public void testCarCanFitInCompactSpot() {
    Car car = new Car("ABC-1234");
    assertTrue(car.canFitInSpot(ParkingSpotType.COMPACT));
}

@Test
public void testTruckCannotFitInCompactSpot() {
    Truck truck = new Truck("TRK-5678");
    assertFalse(truck.canFitInSpot(ParkingSpotType.COMPACT));
}

@Test
public void testCarWithoutPermitCannotUseHandicapped() {
    Car car = new Car("ABC-1234", false);
    assertFalse(car.canFitInSpot(ParkingSpotType.HANDICAPPED));
}

@Test
public void testCarWithPermitCanUseHandicapped() {
    Car car = new Car("ABC-1234", true);
    assertTrue(car.canFitInSpot(ParkingSpotType.HANDICAPPED));
}
```

```java
// ParkingSpotTest.java
@Test
public void testParkVehicleSuccess() {
    CompactSpot spot = new CompactSpot("F0-C001", 0);
    Car car = new Car("ABC-1234");
    
    assertTrue(spot.isAvailable());
    assertTrue(spot.parkVehicle(car));
    assertFalse(spot.isAvailable());
    assertEquals(car, spot.getParkedVehicle());
}

@Test
public void testParkVehicleFailsWhenOccupied() {
    CompactSpot spot = new CompactSpot("F0-C001", 0);
    Car car1 = new Car("ABC-1234");
    Car car2 = new Car("XYZ-5678");
    
    spot.parkVehicle(car1);
    assertFalse(spot.parkVehicle(car2));  // Should fail
}
```

```java
// ParkingTicketTest.java
@Test
public void testFeeCalculationMinimumOneHour() {
    // Park for 5 minutes, still charged 1 hour
    CompactSpot spot = new CompactSpot("F0-C001", 0);
    Car car = new Car("ABC-1234");
    ParkingTicket ticket = new ParkingTicket(car, spot);
    
    // Fee should be 1 hour * $2 = $2
    assertEquals(2.0, ticket.calculateFee(), 0.01);
}

@Test
public void testFeeCalculationRoundsUp() {
    // Simulate 61 minutes (should be 2 hours)
    // Would need to mock time or use a time provider
}
```

### Integration Tests

```java
// ParkingLotIntegrationTest.java
@Test
public void testFullParkingFlow() {
    ParkingLot.resetInstance();
    ParkingLot lot = ParkingLot.getInstance("Test Lot", "123 Test St");
    
    // Setup
    ParkingFloor floor = new ParkingFloor(0, "Ground");
    floor.addParkingSpot(new CompactSpot("F0-C001", 0));
    lot.addFloor(floor);
    
    // Entry
    Car car = new Car("ABC-1234");
    ParkingTicket ticket = lot.parkVehicle(car);
    
    assertNotNull(ticket);
    assertEquals("ABC-1234", ticket.getVehicle().getLicensePlate());
    assertEquals(1, lot.getActiveVehicleCount());
    
    // Exit
    lot.releaseSpot(ticket);
    assertEquals(0, lot.getActiveVehicleCount());
}

@Test
public void testDuplicateVehicleRejected() {
    ParkingLot.resetInstance();
    ParkingLot lot = ParkingLot.getInstance("Test Lot", "123 Test St");
    
    ParkingFloor floor = new ParkingFloor(0, "Ground");
    floor.addParkingSpot(new CompactSpot("F0-C001", 0));
    floor.addParkingSpot(new CompactSpot("F0-C002", 0));
    lot.addFloor(floor);
    
    Car car = new Car("ABC-1234");
    
    ParkingTicket ticket1 = lot.parkVehicle(car);
    ParkingTicket ticket2 = lot.parkVehicle(car);  // Same car
    
    assertNotNull(ticket1);
    assertNull(ticket2);  // Should be rejected
}
```

### Mocking for Tests

```java
// Mocking PaymentService
@Test
public void testExitWithFailedPayment() {
    PaymentService mockPaymentService = mock(PaymentService.class);
    when(mockPaymentService.processPayment(any(), anyDouble()))
        .thenReturn(false);  // Simulate payment failure
    
    ExitPanel exitPanel = new ExitPanel("EXIT-1", parkingLot, mockPaymentService);
    
    boolean result = exitPanel.processExit(ticket);
    
    assertFalse(result);  // Exit should fail
    // Verify spot is NOT released
    assertFalse(ticket.getParkingSpot().isAvailable());
}
```

---

## Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `parkVehicle()` | O(F × S) | F = floors, S = spots per floor |
| `findAvailableSpot()` | O(S) | Linear scan of spots |
| `releaseSpot()` | O(1) | Direct access via ticket |
| `getTicket()` | O(1) | HashMap lookup |
| `findTicketByVehicle()` | O(1) | HashMap lookup |
| `calculateFee()` | O(1) | Simple arithmetic |

### Space Complexity

| Data Structure | Space | Purpose |
|----------------|-------|---------|
| `floors` | O(F) | Store floor objects |
| `spotsByType` | O(S) | Organize spots by type |
| `spotsById` | O(S) | Quick spot lookup |
| `activeTickets` | O(V) | V = parked vehicles |
| `vehicleToTicket` | O(V) | Vehicle → ticket mapping |

### Bottlenecks at Scale

**10x Usage (100 → 1,000 vehicles):**
- Linear spot search becomes noticeable
- Solution: Index available spots separately

**100x Usage (100 → 10,000 vehicles):**
- Single ParkingLot becomes bottleneck
- Solution: Partition by floor, parallel processing

**Optimization: Available Spot Index**

```java
// Instead of scanning all spots:
private final Map<ParkingSpotType, Queue<ParkingSpot>> availableSpots;

// O(1) to get available spot
public ParkingSpot findAvailableSpot(Vehicle vehicle) {
    for (ParkingSpotType type : getSearchOrder(vehicle)) {
        ParkingSpot spot = availableSpots.get(type).poll();
        if (spot != null) return spot;
    }
    return null;
}

// When vehicle leaves, add back to queue
public void releaseSpot(ParkingSpot spot) {
    availableSpots.get(spot.getType()).offer(spot);
}
```

---

## Interview Follow-ups with Answers

### Q1: How would you scale this for a multi-building parking structure?

**Answer:**
```java
// Add Building layer above ParkingLot
public class ParkingComplex {
    private final List<ParkingLot> buildings;
    
    public ParkingTicket findParkingAcrossBuildings(Vehicle vehicle) {
        for (ParkingLot building : buildings) {
            if (!building.isFull(vehicle)) {
                return building.parkVehicle(vehicle);
            }
        }
        return null;
    }
}
```

### Q2: How would you handle peak hours with long entry queues?

**Answer:**
1. **Pre-allocation**: Reserve spots when vehicle detected approaching
2. **Multiple entry lanes**: Each with dedicated spot pools
3. **License plate recognition**: Auto-assign spot before arrival
4. **Overflow handling**: Direct to nearby lots when full

### Q3: How would you add dynamic pricing (surge pricing)?

**Answer:**
```java
public interface PricingStrategy {
    double calculateRate(ParkingSpotType type, LocalDateTime time);
}

public class DynamicPricingStrategy implements PricingStrategy {
    @Override
    public double calculateRate(ParkingSpotType type, LocalDateTime time) {
        double baseRate = getBaseRate(type);
        double occupancyMultiplier = getOccupancyMultiplier();
        double timeMultiplier = getTimeMultiplier(time);
        
        return baseRate * occupancyMultiplier * timeMultiplier;
    }
}
```

### Q4: What if we need to support electric vehicle charging?

**Answer:**
```java
// Add new spot type
public enum ParkingSpotType {
    COMPACT, LARGE, HANDICAPPED, ELECTRIC_CHARGING
}

// New spot class
public class ElectricChargingSpot extends ParkingSpot {
    private final double chargingRateKW;
    private ChargingSession currentSession;
    
    public void startCharging(ElectricVehicle vehicle) {
        this.currentSession = new ChargingSession(vehicle, chargingRateKW);
    }
}

// Extended ticket for charging
public class ChargingTicket extends ParkingTicket {
    private final ChargingSession chargingSession;
    
    @Override
    public double calculateFee() {
        double parkingFee = super.calculateFee();
        double chargingFee = chargingSession.calculateChargingFee();
        return parkingFee + chargingFee;
    }
}
```

### Q5: How would you handle concurrent access from multiple entry panels?

**Answer:**
Current design uses `synchronized` methods, which works but has limitations:

**Better approach for high concurrency:**
```java
// Use optimistic locking with CAS
public class ParkingSpot {
    private final AtomicReference<Vehicle> parkedVehicle = 
        new AtomicReference<>(null);
    
    public boolean parkVehicle(Vehicle vehicle) {
        // Compare-and-swap: only succeeds if currently null
        return parkedVehicle.compareAndSet(null, vehicle);
    }
}

// Or use fine-grained locks per floor
public class ParkingFloor {
    private final ReentrantLock floorLock = new ReentrantLock();
    
    public ParkingSpot findAvailableSpot(Vehicle vehicle) {
        floorLock.lock();
        try {
            // Find and allocate spot
        } finally {
            floorLock.unlock();
        }
    }
}
```

### Q6: What would you do differently with more time?

**Answer:**
1. **Add reservation system** - Book spots in advance
2. **Add analytics** - Track peak hours, popular spots
3. **Add notification service** - Alert when spot available
4. **Add admin dashboard** - Monitor in real-time
5. **Add audit logging** - Track all operations
6. **Add rate limiting** - Prevent abuse
7. **Extract pricing to service** - Support A/B testing

