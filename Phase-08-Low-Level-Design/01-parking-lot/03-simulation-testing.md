# 🚗 Parking Lot System - Simulation, Testing & Interview Q&A

## STEP 5: Simulation / Dry Run

### Scenario A: Normal Parking Flow

**Initial State:**

```
Floor 0: 5 compact, 3 large, 2 handicapped (all empty)
Floor 1: 5 compact, 3 large (all empty)
activeTickets: {}
vehicleToTicket: {}
```

**Step 1: Car "ABC-1234" enters via Entry Panel 1**

```
EntryPanel.processEntry(Car("ABC-1234"))
    │
    └──► ParkingLot.parkVehicle(car)
              │
              ├──► vehicleToTicket.containsKey("ABC-1234")? → false
              │
              ├──► findAvailableSpot(car)
              │         │
              │         └──► Floor 0: findAvailableSpot(car)
              │                   │
              │                   └──► Search order: COMPACT → LARGE
              │                         │
              │                         └──► F0-C001.canFitVehicle(car)? → true
              │
              ├──► F0-C001.parkVehicle(car)
              │         │
              │         └──► F0-C001.isAvailable = false
              │              F0-C001.parkedVehicle = car
              │
              ├──► Create ParkingTicket(car, F0-C001)
              │         │
              │         └──► ticketId = "TKT-1234567890-ABCD1234"
              │              entryTime = 10:00:00
              │              status = ACTIVE
              │
              ├──► activeTickets["TKT-xxx"] = ticket
              │    vehicleToTicket["ABC-1234"] = ticket
              │
              └──► updateAllDisplayBoards()
                        │
                        └──► Floor 0 COMPACT: 5 → 4
```

**State After Step 1:**

```
Floor 0: 4 compact, 3 large, 2 handicapped
activeTickets: {"TKT-xxx": ticket1}
vehicleToTicket: {"ABC-1234": ticket1}
F0-C001.isAvailable: false
F0-C001.parkedVehicle: Car("ABC-1234")
```

---

**Step 2: Truck "TRK-9999" enters via Entry Panel 1**

```
EntryPanel.processEntry(Truck("TRK-9999"))
    │
    └──► ParkingLot.parkVehicle(truck)
              │
              ├──► vehicleToTicket.containsKey("TRK-9999")? → false
              │
              ├──► findAvailableSpot(truck)
              │         │
              │         └──► Floor 0: findAvailableSpot(truck)
              │                   │
              │                   └──► Search order: LARGE only
              │                         │
              │                         └──► F0-L001.canFitVehicle(truck)? → true
              │
              ├──► F0-L001.parkVehicle(truck)
              │
              └──► Ticket created, display updated
```

**State After Step 2:**

```
Floor 0: 4 compact, 2 large, 2 handicapped
activeTickets: {"TKT-xxx": ticket1, "TKT-yyy": ticket2}
vehicleToTicket: {"ABC-1234": ticket1, "TRK-9999": ticket2}
```

---

**Step 3: Car "ABC-1234" tries to enter again (Duplicate)**

```
EntryPanel.processEntry(Car("ABC-1234"))
    │
    └──► ParkingLot.parkVehicle(car)
              │
              └──► vehicleToTicket.containsKey("ABC-1234")? → true
                        │
                        └──► Return null (entry denied)
                             Print: "Vehicle ABC-1234 is already parked!"
```

**Result:** Entry rejected, no state change

---

**Step 4: Car "ABC-1234" exits via Exit Panel 1**

```
ExitPanel.processExit(ticket1)
    │
    ├──► ticket1.calculateFee()
    │         │
    │         └──► Duration = 10:45 - 10:00 = 45 minutes
    │              Hours = ceiling(45/60) = 1 hour
    │              Rate = $2.00 (COMPACT)
    │              Fee = 1 × $2.00 = $2.00
    │
    ├──► paymentService.processPayment(ticket1, $2.00)
    │         │
    │         └──► CardPayment.processPayment()
    │                   │
    │                   └──► Success (95% probability)
    │                        ticket1.markAsPaid($2.00)
    │                        ticket1.status = PAID
    │                        ticket1.exitTime = 10:45:00
    │
    └──► parkingLot.releaseSpot(ticket1)
              │
              ├──► F0-C001.removeVehicle()
              │         │
              │         └──► F0-C001.isAvailable = true
              │              F0-C001.parkedVehicle = null
              │
              ├──► activeTickets.remove("TKT-xxx")
              │    vehicleToTicket.remove("ABC-1234")
              │
              └──► updateAllDisplayBoards()
                        │
                        └──► Floor 0 COMPACT: 4 → 5
```

**State After Step 4:**

```
Floor 0: 5 compact, 2 large, 2 handicapped
activeTickets: {"TKT-yyy": ticket2}
vehicleToTicket: {"TRK-9999": ticket2}
F0-C001.isAvailable: true
```

---

### Scenario B: Handicapped Parking

**Step 1: Car with permit "HND-0001" enters**

```
Car handicappedCar = new Car("HND-0001", true);  // hasHandicappedPermit = true

ParkingLot.parkVehicle(handicappedCar)
    │
    └──► findAvailableSpot(handicappedCar)
              │
              └──► Floor 0: findAvailableSpot(handicappedCar)
                        │
                        └──► Search order: COMPACT → LARGE → HANDICAPPED
                              │
                              └──► F0-C001 available? → Yes
                                   Use COMPACT (saves handicapped for those who need it)
```

**Key insight:** Even with permit, system prefers compact spots first.

**Step 2: All compact/large spots full, handicapped car enters**

```
Search order: COMPACT → LARGE → HANDICAPPED
    │
    ├──► COMPACT spots: All occupied → Skip
    ├──► LARGE spots: All occupied → Skip
    └──► HANDICAPPED: F0-H001 available
              │
              └──► handicappedCar.canFitInSpot(HANDICAPPED)?
                        │
                        └──► hasHandicappedPermit? → true → Return true
                             Park in F0-H001
```

**Step 3: Car WITHOUT permit tries handicapped spot**

```
Car regularCar = new Car("REG-0001", false);  // No permit

Search order: COMPACT → LARGE
    │
    └──► HANDICAPPED spots are NOT in search order for cars without permit
         Car cannot be assigned to handicapped spot
```

---

### Scenario C: Parking Lot Full

**State:** All spots occupied

```
Floor 0: 0 compact, 0 large, 0 handicapped
Floor 1: 0 compact, 0 large
```

**New car enters:**

```
ParkingLot.parkVehicle(newCar)
    │
    └──► findAvailableSpot(newCar)
              │
              ├──► Floor 0: findAvailableSpot(newCar) → null
              │
              └──► Floor 1: findAvailableSpot(newCar) → null
                        │
                        └──► Return null

Print: "No available spot for CAR"
Return null (no ticket issued)
```

---

### Scenario D: Concurrent Access (Race Condition Prevention)

**Without synchronization (PROBLEM):**

```
Thread A (Entry Panel 1)          Thread B (Entry Panel 2)
─────────────────────────         ─────────────────────────
Check F0-C001 available?
    → Yes
                                  Check F0-C001 available?
                                      → Yes (RACE!)
Park in F0-C001
    → Success
                                  Park in F0-C001
                                      → COLLISION! Two cars, one spot
```

**With synchronization (SOLUTION):**

```
Thread A (Entry Panel 1)          Thread B (Entry Panel 2)
─────────────────────────         ─────────────────────────
Acquire lock on parkVehicle()
Check F0-C001 available?
    → Yes
Park in F0-C001
    → Success
Release lock
                                  Acquire lock on parkVehicle()
                                  Check F0-C001 available?
                                      → No (already taken)
                                  Find next spot: F0-C002
                                  Park in F0-C002
                                      → Success
                                  Release lock
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions

| Condition           | Test Case                    | Expected Result                 |
| ------------------- | ---------------------------- | ------------------------------- |
| Empty lot           | Park first vehicle           | Success, ticket issued          |
| Full lot            | Park when all spots taken    | Return null, "lot full" message |
| Zero duration       | Exit immediately after entry | Minimum 1 hour charge           |
| Max duration        | Park for 24+ hours           | Calculate correct fee           |
| Empty license plate | `new Car("")`                | IllegalArgumentException        |
| Null license plate  | `new Car(null)`              | IllegalArgumentException        |
| Duplicate vehicle   | Park same car twice          | Second attempt returns null     |

### Invalid Inputs and Error Handling

```java
// Test: Empty license plate
@Test(expected = IllegalArgumentException.class)
public void testEmptyLicensePlate() {
    new Car("");  // Should throw
}

// Test: Null license plate
@Test(expected = IllegalArgumentException.class)
public void testNullLicensePlate() {
    new Car(null);  // Should throw
}

// Test: Truck in compact spot
@Test
public void testTruckCannotFitInCompact() {
    Truck truck = new Truck("TRK-001");
    assertFalse(truck.canFitInSpot(ParkingSpotType.COMPACT));
}
```

### Concurrent Access Scenarios

```java
// Test: Concurrent parking attempts
@Test
public void testConcurrentParking() throws InterruptedException {
    ParkingLot.resetInstance();
    ParkingLot lot = ParkingLot.getInstance("Test", "Address");

    ParkingFloor floor = new ParkingFloor(0, "Ground");
    floor.addParkingSpot(new CompactSpot("F0-C001", 0));  // Only 1 spot
    lot.addFloor(floor);

    Car car1 = new Car("CAR-001");
    Car car2 = new Car("CAR-002");

    AtomicReference<ParkingTicket> ticket1 = new AtomicReference<>();
    AtomicReference<ParkingTicket> ticket2 = new AtomicReference<>();

    Thread t1 = new Thread(() -> ticket1.set(lot.parkVehicle(car1)));
    Thread t2 = new Thread(() -> ticket2.set(lot.parkVehicle(car2)));

    t1.start();
    t2.start();
    t1.join();
    t2.join();

    // Exactly one should succeed
    int successCount = 0;
    if (ticket1.get() != null) successCount++;
    if (ticket2.get() != null) successCount++;

    assertEquals(1, successCount);
}
```

### Unit Test Strategy

**Vehicle Tests:**

```java
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

@Test
public void testMotorcycleCanFitAnywhere() {
    Motorcycle bike = new Motorcycle("BIKE-001");
    assertTrue(bike.canFitInSpot(ParkingSpotType.COMPACT));
    assertTrue(bike.canFitInSpot(ParkingSpotType.LARGE));
    assertTrue(bike.canFitInSpot(ParkingSpotType.HANDICAPPED));
}
```

**ParkingSpot Tests:**

```java
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

@Test
public void testRemoveVehicle() {
    CompactSpot spot = new CompactSpot("F0-C001", 0);
    Car car = new Car("ABC-1234");

    spot.parkVehicle(car);
    Vehicle removed = spot.removeVehicle();

    assertEquals(car, removed);
    assertTrue(spot.isAvailable());
    assertNull(spot.getParkedVehicle());
}
```

**ParkingTicket Tests:**

```java
@Test
public void testFeeCalculationMinimumOneHour() {
    CompactSpot spot = new CompactSpot("F0-C001", 0);
    Car car = new Car("ABC-1234");
    ParkingTicket ticket = new ParkingTicket(car, spot);

    // Immediately calculate fee (< 1 minute)
    double fee = ticket.calculateFee();

    // Should be minimum 1 hour × $2.00 = $2.00
    assertEquals(2.0, fee, 0.01);
}

@Test
public void testFeeCalculationRoundsUp() {
    // This would require mocking time
    // 61 minutes should be 2 hours
    // Fee = 2 × $2.00 = $4.00
}

@Test
public void testTicketStatusTransitions() {
    CompactSpot spot = new CompactSpot("F0-C001", 0);
    Car car = new Car("ABC-1234");
    ParkingTicket ticket = new ParkingTicket(car, spot);

    assertEquals(TicketStatus.ACTIVE, ticket.getStatus());

    ticket.markAsPaid(2.0);
    assertEquals(TicketStatus.PAID, ticket.getStatus());
}
```

### Integration Tests

```java
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

@Test
public void testParkingLotFull() {
    ParkingLot.resetInstance();
    ParkingLot lot = ParkingLot.getInstance("Test Lot", "123 Test St");

    ParkingFloor floor = new ParkingFloor(0, "Ground");
    floor.addParkingSpot(new CompactSpot("F0-C001", 0));  // Only 1 spot
    lot.addFloor(floor);

    Car car1 = new Car("CAR-001");
    Car car2 = new Car("CAR-002");

    ParkingTicket ticket1 = lot.parkVehicle(car1);
    ParkingTicket ticket2 = lot.parkVehicle(car2);

    assertNotNull(ticket1);
    assertNull(ticket2);  // Lot is full
    assertTrue(lot.isFull(car2));
}
```

### Mocking for Tests

```java
// Mocking PaymentService for testing exit flow
@Test
public void testExitWithFailedPayment() {
    PaymentService mockPaymentService = mock(PaymentService.class);
    when(mockPaymentService.processPayment(any(), anyDouble()))
        .thenReturn(false);  // Simulate payment failure

    ParkingLot lot = ParkingLot.getInstance();
    ExitPanel exitPanel = new ExitPanel("EXIT-1", lot, mockPaymentService);

    // Setup: park a car first
    Car car = new Car("ABC-1234");
    ParkingTicket ticket = lot.parkVehicle(car);

    boolean result = exitPanel.processExit(ticket);

    assertFalse(result);  // Exit should fail
    // Verify spot is NOT released
    assertFalse(ticket.getParkingSpot().isAvailable());
    assertEquals(1, lot.getActiveVehicleCount());
}
```

### Load and Stress Testing Approach

```java
// Simulate peak hour load
@Test
public void testPeakHourLoad() throws InterruptedException {
    ParkingLot.resetInstance();
    ParkingLot lot = ParkingLot.getInstance("Test", "Address");

    // Setup: 100 spots
    ParkingFloor floor = new ParkingFloor(0, "Ground");
    for (int i = 0; i < 100; i++) {
        floor.addParkingSpot(new CompactSpot("F0-C" + i, 0));
    }
    lot.addFloor(floor);

    // Simulate 50 concurrent entry attempts
    ExecutorService executor = Executors.newFixedThreadPool(50);
    CountDownLatch latch = new CountDownLatch(50);
    AtomicInteger successCount = new AtomicInteger(0);

    for (int i = 0; i < 50; i++) {
        final int carNum = i;
        executor.submit(() -> {
            try {
                Car car = new Car("CAR-" + carNum);
                ParkingTicket ticket = lot.parkVehicle(car);
                if (ticket != null) {
                    successCount.incrementAndGet();
                }
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await(10, TimeUnit.SECONDS);
    executor.shutdown();

    assertEquals(50, successCount.get());  // All should succeed
    assertEquals(50, lot.getActiveVehicleCount());
}
```

---

### Q1: How would you scale this for a multi-building parking structure?

**Answer:**

```java
// Add Building layer above ParkingLot
public class ParkingComplex {
    private final List<ParkingLot> buildings;
    private final LoadBalancer loadBalancer;

    public ParkingTicket findParkingAcrossBuildings(Vehicle vehicle) {
        // Strategy 1: Round-robin across buildings
        // Strategy 2: Nearest building with availability
        // Strategy 3: Building with most available spots

        for (ParkingLot building : loadBalancer.getOrderedBuildings()) {
            if (!building.isFull(vehicle)) {
                return building.parkVehicle(vehicle);
            }
        }
        return null;
    }
}
```

**Key considerations:**

- Each building is independent (own singleton or remove singleton pattern)
- Load balancer decides which building to try first
- Cross-building ticket lookup requires central registry

---

### Q2: How would you handle peak hours with long entry queues?

**Answer:**

1. **Pre-allocation**: Reserve spots when vehicle detected approaching

```java
public class SpotReservation {
    public String reserveSpot(VehicleType type, Duration holdTime) {
        ParkingSpot spot = findAndLockSpot(type);
        scheduleRelease(spot, holdTime);
        return spot.getSpotId();
    }
}
```

2. **Multiple entry lanes**: Each with dedicated spot pools

```java
public class EntryLane {
    private final Queue<ParkingSpot> dedicatedSpots;
    // Reduces contention by partitioning spots
}
```

3. **License plate recognition**: Auto-assign spot before arrival
4. **Overflow handling**: Direct to nearby lots when full
5. **Dynamic pricing**: Higher rates during peak to reduce demand

---

### Q3: How would you add dynamic pricing (surge pricing)?

**Answer:**

```java
public interface PricingStrategy {
    double calculateRate(ParkingSpotType type, LocalDateTime time);
}

public class DynamicPricingStrategy implements PricingStrategy {
    private final OccupancyTracker occupancyTracker;

    @Override
    public double calculateRate(ParkingSpotType type, LocalDateTime time) {
        double baseRate = getBaseRate(type);
        double occupancyMultiplier = getOccupancyMultiplier();
        double timeMultiplier = getTimeMultiplier(time);

        return baseRate * occupancyMultiplier * timeMultiplier;
    }

    private double getOccupancyMultiplier() {
        double occupancy = occupancyTracker.getCurrentOccupancy();
        if (occupancy > 0.9) return 2.0;      // 90%+ full: 2x price
        if (occupancy > 0.7) return 1.5;      // 70%+ full: 1.5x price
        return 1.0;
    }

    private double getTimeMultiplier(LocalDateTime time) {
        int hour = time.getHour();
        if (hour >= 9 && hour <= 11) return 1.5;   // Morning rush
        if (hour >= 17 && hour <= 19) return 1.5;  // Evening rush
        if (hour >= 22 || hour <= 6) return 0.5;   // Night discount
        return 1.0;
    }
}
```

---

### Q4: What if we need to support electric vehicle charging?

**Answer:**

```java
// Add new spot type
public enum ParkingSpotType {
    COMPACT, LARGE, HANDICAPPED, ELECTRIC_CHARGING
}

// New spot class with charging capability
public class ElectricChargingSpot extends ParkingSpot
        implements ChargingCapable {

    private final double chargingRateKW;
    private ChargingSession currentSession;

    public ElectricChargingSpot(String spotId, int floor, double chargingRate) {
        super(spotId, ParkingSpotType.ELECTRIC_CHARGING, floor);
        this.chargingRateKW = chargingRate;
    }

    @Override
    public void startCharging(ElectricVehicle vehicle) {
        this.currentSession = new ChargingSession(vehicle, chargingRateKW);
    }

    @Override
    public void stopCharging() {
        this.currentSession.end();
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

---

### Q5: How would you handle concurrent access from multiple entry panels?

**Answer:**

Current design uses `synchronized` methods, which works but has limitations.

**Better approach for high concurrency:**

```java
// Option 1: Optimistic locking with CAS
public class ParkingSpot {
    private final AtomicReference<Vehicle> parkedVehicle =
        new AtomicReference<>(null);

    public boolean parkVehicle(Vehicle vehicle) {
        // Compare-and-swap: only succeeds if currently null
        return parkedVehicle.compareAndSet(null, vehicle);
    }
}

// Option 2: Fine-grained locks per floor
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

// Option 3: Lock-free available spot queue
public class ParkingFloor {
    private final ConcurrentLinkedQueue<ParkingSpot> availableSpots;

    public ParkingSpot findAvailableSpot(Vehicle vehicle) {
        return availableSpots.poll();  // Atomic operation
    }

    public void releaseSpot(ParkingSpot spot) {
        availableSpots.offer(spot);  // Atomic operation
    }
}
```

---

### Q6: What would you do differently with more time?

**Answer:**

1. **Add reservation system** - Book spots in advance
2. **Add analytics** - Track peak hours, popular spots, revenue
3. **Add notification service** - Alert when spot available
4. **Add admin dashboard** - Monitor in real-time
5. **Add audit logging** - Track all operations for compliance
6. **Add rate limiting** - Prevent abuse of entry panels
7. **Extract pricing to service** - Support A/B testing of pricing strategies
8. **Add caching layer** - Cache availability counts for display boards
9. **Add event sourcing** - Full history of all parking events
10. **Add metrics/monitoring** - Prometheus/Grafana for operations

---

### Q7: What are the tradeoffs in your design?

**Answer:**

| Decision                       | Trade-off                                |
| ------------------------------ | ---------------------------------------- |
| Singleton ParkingLot           | Simple global access vs harder to test   |
| Synchronized methods           | Thread-safe vs potential bottleneck      |
| In-memory storage              | Fast access vs no persistence            |
| Linear spot search             | Simple implementation vs O(n) complexity |
| Fee calculation in Ticket      | Cohesion vs potential SRP violation      |
| Abstract classes vs Interfaces | Simpler hierarchy vs less flexibility    |

---

### Q8: Common interviewer challenges and responses

**Challenge:** "Your spot search is O(n). How would you optimize it?"

**Response:**

```java
// Maintain separate queue of available spots per type
private final Map<ParkingSpotType, Queue<ParkingSpot>> availableSpots;

// O(1) to get available spot
public ParkingSpot findAvailableSpot(Vehicle vehicle) {
    for (ParkingSpotType type : getSearchOrder(vehicle)) {
        ParkingSpot spot = availableSpots.get(type).poll();
        if (spot != null) return spot;
    }
    return null;
}
```

**Challenge:** "What if the singleton pattern makes testing difficult?"

**Response:**
"For production, I would use dependency injection instead. The ParkingLot would be created by a factory or IoC container and injected into panels. This allows easy mocking in tests while maintaining single-instance semantics in production."

**Challenge:** "How do you handle partial failures? What if payment succeeds but spot release fails?"

**Response:**
"I would implement the saga pattern or use a transactional outbox:

1. Record the intent to release spot
2. Process payment
3. Release spot
4. If step 3 fails, a background job retries
5. If payment fails, rollback by not releasing spot"

---

## Expected Output

```
Setting up parking lot...

Added floor: Ground Floor
Added floor: Level 1

==================================================
PARKING LOT STATUS: Downtown Parking
Address: 123 Main Street
==================================================

Active Vehicles: 0

Availability by Floor:

╔════════════════════════════════════╗
║        FLOOR 0 - AVAILABILITY       ║
╠════════════════════════════════════╣
║  COMPACT     :   5  [AVAILABLE]  ║
║  LARGE       :   3  [AVAILABLE]  ║
║  HANDICAPPED :   2  [AVAILABLE]  ║
╚════════════════════════════════════╝

╔════════════════════════════════════╗
║        FLOOR 1 - AVAILABILITY       ║
╠════════════════════════════════════╣
║  COMPACT     :   5  [AVAILABLE]  ║
║  LARGE       :   3  [AVAILABLE]  ║
║  HANDICAPPED :   0  [FULL]      ║
╚════════════════════════════════════╝

Total Availability:
  COMPACT: 10
  LARGE: 6
  HANDICAPPED: 2
==================================================


===== SCENARIO 1: Cars Entering =====

Entry Panel ENTRY-1: Processing entry for ABC-1234
Entry Panel ENTRY-1: Ticket issued - TKT-1234567890-ABCD1234
Entry Panel ENTRY-1: Assigned spot - F0-C001

Entry Panel ENTRY-1: Processing entry for XYZ-5678
Entry Panel ENTRY-1: Ticket issued - TKT-1234567891-EFGH5678
Entry Panel ENTRY-1: Assigned spot - F0-C002

Entry Panel ENTRY-2: Processing entry for HND-0001
Entry Panel ENTRY-2: Ticket issued - TKT-1234567892-IJKL9012
Entry Panel ENTRY-2: Assigned spot - F0-C003

... (status updates continue)
```
