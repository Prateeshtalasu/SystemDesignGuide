# 🚗 Car Rental System - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Car Rental System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Vehicle` | Represent a vehicle and its state | Vehicle properties change |
| `Location` | Manage a rental branch | Location features change |
| `Reservation` | Track booking details | Reservation model changes |
| `Rental` | Track active rental | Rental tracking changes |
| `Invoice` | Generate billing | Billing format changes |
| `PricingStrategy` | Calculate prices | Pricing rules change |
| `RentalService` | Coordinate rental operations | Business logic changes |

**SRP in Action:**

```java
// Vehicle ONLY manages its own state
public class Vehicle {
    public void reserve() { status = RESERVED; }
    public void pickup() { status = RENTED; }
    public void returnVehicle() { status = AVAILABLE; }
}

// PricingStrategy ONLY handles pricing
public interface PricingStrategy {
    BigDecimal calculateBasePrice(Reservation reservation);
    BigDecimal calculateLateFee(Rental rental);
}

// Invoice ONLY handles billing presentation
public class Invoice {
    public void print() { /* Format and display */ }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Vehicle Types:**

```java
// Just add to enum - no other changes needed
public enum VehicleType {
    ECONOMY("Economy", 1.0),
    // ... existing types
    ELECTRIC("Electric", 1.8),    // New!
    CONVERTIBLE("Convertible", 2.5);  // New!
}
```

**Adding New Pricing Strategies:**

```java
// No changes to existing code
public class WeekendPricingStrategy implements PricingStrategy {
    @Override
    public BigDecimal calculateBasePrice(Reservation reservation) {
        BigDecimal base = standardStrategy.calculateBasePrice(reservation);
        if (isWeekend(reservation.getPickupTime())) {
            return base.multiply(new BigDecimal("1.25"));  // 25% weekend surcharge
        }
        return base;
    }
}

public class LoyaltyPricingStrategy implements PricingStrategy {
    @Override
    public BigDecimal calculateBasePrice(Reservation reservation) {
        BigDecimal base = standardStrategy.calculateBasePrice(reservation);
        int loyaltyLevel = getLoyaltyLevel(reservation.getCustomer());
        return base.multiply(getDiscount(loyaltyLevel));
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**Any PricingStrategy works:**

```java
public class RentalService {
    private final PricingStrategy pricingStrategy;
    
    // Works with any pricing strategy
    private Invoice generateInvoice(Rental rental) {
        BigDecimal basePrice = pricingStrategy.calculateBasePrice(
            rental.getReservation());
        // ...
    }
}

// All these work correctly
new RentalService(new StandardPricingStrategy());
new RentalService(new WeekendPricingStrategy());
new RentalService(new LoyaltyPricingStrategy());
```

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

```java
public interface PricingStrategy {
    BigDecimal calculateBasePrice(Reservation reservation);
    BigDecimal calculateLateFee(Rental rental);
    BigDecimal calculateMileageFee(Rental rental, int includedMiles);
    BigDecimal calculateDropoffFee(Reservation reservation);
}
```

**Better with ISP:**

```java
public interface BasePriceCalculator {
    BigDecimal calculateBasePrice(Reservation reservation);
}

public interface FeeCalculator {
    BigDecimal calculateLateFee(Rental rental);
    BigDecimal calculateMileageFee(Rental rental, int includedMiles);
}

public interface SurchargeCalculator {
    BigDecimal calculateDropoffFee(Reservation reservation);
    BigDecimal calculateAirportSurcharge(Location location);
}
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
// RentalService depends on abstraction (good!)
public class RentalService {
    private final PricingStrategy pricingStrategy;  // Interface
    
    public RentalService(PricingStrategy pricingStrategy) {
        this.pricingStrategy = pricingStrategy;
    }
}
```

**Could be improved:**

```java
// Define more abstractions
public interface VehicleRepository {
    Vehicle findById(String id);
    List<Vehicle> findAvailable(Location location);
    void save(Vehicle vehicle);
}

public interface ReservationRepository {
    Reservation findById(String id);
    List<Reservation> findByVehicle(Vehicle vehicle);
    void save(Reservation reservation);
}

// RentalService depends on abstractions
public class RentalService {
    private final VehicleRepository vehicleRepo;
    private final ReservationRepository reservationRepo;
    private final PricingStrategy pricingStrategy;
}
```

**Why We Use Concrete Domain Entities in This LLD Implementation:**

For low-level design interviews, we intentionally use concrete domain entities (`Vehicle`, `Reservation`) instead of repository interfaces for the following reasons:

1. **Domain vs Infrastructure**: `Vehicle` and `Reservation` are core domain objects representing business concepts, not infrastructure concerns. The repository pattern is an infrastructure pattern that doesn't demonstrate core LLD skills.

2. **Focus on Business Logic**: LLD interviews focus on rental business logic, state transitions, and pricing calculations. Adding repository abstractions shifts focus away from these core concepts.

3. **In-Memory Context**: In LLD interviews, we typically work with in-memory data structures. Repository interfaces are more relevant when dealing with persistent storage, which is often out of scope for LLD.

4. **Production vs Interview**: In production systems, we would absolutely extract `VehicleRepository` and `ReservationRepository` interfaces for:
   - Testability (mock repositories in unit tests)
   - Data access flexibility (swap database implementations)
   - Separation of concerns (domain logic vs data access)

**The Trade-off:**
- **Interview Scope**: Concrete domain entities focus on business logic and state management
- **Production Scope**: Repository interfaces provide testability and data access flexibility

Note: We do use the `PricingStrategy` interface (good DIP!), showing we understand when interfaces add value.

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Vehicle represents vehicle state, Reservation tracks booking, Rental tracks active rental, PricingStrategy calculates prices, Invoice generates billing. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new vehicle types, pricing strategies) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All PricingStrategy implementations properly implement the PricingStrategy interface contract. They are substitutable. | N/A | - |
| **ISP** | PASS | PricingStrategy interface is minimal and focused. Clients only depend on what they need. No unused methods. | N/A | - |
| **DIP** | ACCEPTABLE (LLD Scope) | RentalService depends on PricingStrategy interface (good DIP!), but depends on concrete Vehicle and Reservation classes. For LLD interview scope, this is acceptable as it focuses on core rental logic. In production, we would use VehicleRepository and ReservationRepository interfaces. | See "Why We Use Concrete Domain Entities" section above for detailed justification. This is an intentional design decision for interview context, not an oversight. | Interview: Simpler, focuses on core LLD skills. Production: More abstraction layers, but improves testability and data access flexibility |

---

## Design Patterns Used

### 1. Strategy Pattern

**Where:** Pricing calculation

```java
public interface PricingStrategy {
    BigDecimal calculateBasePrice(Reservation reservation);
}

public class StandardPricingStrategy implements PricingStrategy { }
public class WeekendPricingStrategy implements PricingStrategy { }
public class PromotionalPricingStrategy implements PricingStrategy { }

// Context
public class RentalService {
    private PricingStrategy strategy;
    
    public void setPricingStrategy(PricingStrategy strategy) {
        this.strategy = strategy;
    }
}
```

---

### 2. Builder Pattern

**Where:** Invoice construction

```java
Invoice invoice = Invoice.builder(rental)
    .addCharge("Base Rental", basePrice)
    .addCharge("Late Fee", lateFee)
    .addCharge("Mileage Fee", mileageFee)
    .build();
```

**Benefits:**
- Flexible charge addition
- Clear, readable code
- Immutable result

---

### 3. State Pattern (Implicit)

**Where:** Vehicle and Reservation status

```java
public class Vehicle {
    private VehicleStatus status;
    
    public void reserve() {
        if (status != AVAILABLE) throw new IllegalStateException();
        status = RESERVED;
    }
    
    public void pickup() {
        if (status != RESERVED) throw new IllegalStateException();
        status = RENTED;
    }
    
    public void returnVehicle() {
        if (status != RENTED) throw new IllegalStateException();
        status = AVAILABLE;
    }
}
```

**State transitions:**

```
Vehicle States:
AVAILABLE ──reserve()──► RESERVED ──pickup()──► RENTED ──return()──► AVAILABLE
    │                       │
    │                       │ cancel()
    │                       ▼
    └───────────────────── AVAILABLE
```

---

### 4. Repository Pattern (Implicit)

**Where:** Data management in RentalService

```java
public class RentalService {
    private final Map<String, Vehicle> vehicles;
    private final Map<String, Reservation> reservations;
    private final Map<String, Rental> rentals;
    
    public Vehicle getVehicle(String id) {
        return vehicles.get(id);
    }
    
    public Reservation getReservation(String id) {
        return reservations.get(id);
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single RentalTransaction Class

```java
// Rejected
public class RentalTransaction {
    private Customer customer;
    private Vehicle vehicle;
    private LocalDateTime reservationTime;
    private LocalDateTime pickupTime;
    private LocalDateTime returnTime;
    private BigDecimal totalCost;
    // All states and logic in one class
}
```

**Why rejected:**
- Too many responsibilities
- Hard to track lifecycle stages
- Difficult to extend

### Alternative 2: Vehicle Manages Own Reservations

```java
// Rejected
public class Vehicle {
    private List<Reservation> reservations;
    
    public boolean isAvailable(LocalDateTime start, LocalDateTime end) {
        // Check against all reservations
    }
}
```

**Why rejected:**
- Vehicle shouldn't know about reservations
- Distributed state
- Hard to query across vehicles

### Alternative 3: Inheritance for Vehicle Types

```java
// Rejected
public class EconomyCar extends Vehicle { }
public class SUV extends Vehicle { }
public class LuxuryCar extends Vehicle { }
```

**Why rejected:**
- Explosion of subclasses
- Behavior differences are minimal
- Enum is simpler and sufficient

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle concurrent reservations?

**Answer:**

```java
// Option 1: Synchronized method (current)
public synchronized Reservation makeReservation(...) { }

// Option 2: Per-vehicle locking
private final Map<String, ReentrantLock> vehicleLocks;

public Reservation makeReservation(Customer customer, Vehicle vehicle, ...) {
    ReentrantLock lock = vehicleLocks.computeIfAbsent(
        vehicle.getId(), k -> new ReentrantLock());
    
    lock.lock();
    try {
        // Verify and reserve
        if (vehicle.getStatus() != VehicleStatus.AVAILABLE) {
            throw new IllegalStateException("Vehicle not available");
        }
        return createReservation(...);
    } finally {
        lock.unlock();
    }
}

// Option 3: Optimistic locking with version
public class Vehicle {
    private long version;
    
    public void reserve(long expectedVersion) {
        if (this.version != expectedVersion) {
            throw new OptimisticLockException();
        }
        this.status = RESERVED;
        this.version++;
    }
}
```

---

### Q2: How would you implement a loyalty program?

**Answer:**

```java
public class LoyaltyProgram {
    private final Map<String, LoyaltyMember> members;
    
    public void recordRental(Customer customer, Rental rental) {
        LoyaltyMember member = members.get(customer.getId());
        if (member != null) {
            int points = calculatePoints(rental);
            member.addPoints(points);
            member.incrementRentals();
        }
    }
    
    public BigDecimal applyDiscount(Customer customer, BigDecimal amount) {
        LoyaltyMember member = members.get(customer.getId());
        if (member == null) return amount;
        
        double discount = member.getTier().getDiscountRate();
        return amount.multiply(BigDecimal.valueOf(1 - discount));
    }
}

public enum LoyaltyTier {
    BRONZE(0, 0.05),    // 5% discount
    SILVER(10, 0.10),   // 10% discount
    GOLD(25, 0.15),     // 15% discount
    PLATINUM(50, 0.20); // 20% discount
    
    private final int requiredRentals;
    private final double discountRate;
}
```

---

### Q3: How would you handle damage reporting?

**Answer:**

```java
public class DamageReport {
    private final String id;
    private final Rental rental;
    private final List<DamageItem> damages;
    private final LocalDateTime reportedAt;
    private DamageStatus status;
}

public class DamageItem {
    private final DamageType type;
    private final String description;
    private final String location;  // e.g., "front bumper"
    private final BigDecimal estimatedCost;
    private final List<String> photoUrls;
}

public class DamageService {
    public DamageReport createReport(Rental rental, List<DamageItem> damages) {
        DamageReport report = new DamageReport(rental, damages);
        
        // Calculate total damage cost
        BigDecimal totalCost = damages.stream()
            .map(DamageItem::getEstimatedCost)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        // Check insurance coverage
        if (rental.hasInsurance()) {
            BigDecimal deductible = rental.getInsurance().getDeductible();
            totalCost = totalCost.subtract(deductible).max(BigDecimal.ZERO);
        }
        
        // Add to invoice
        addDamageCharge(rental, totalCost);
        
        return report;
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add vehicle inspection** - Pre/post rental condition checklist
2. **Add GPS tracking** - Real-time vehicle location
3. **Add dynamic pricing** - Surge pricing for high demand
4. **Add mobile app support** - Digital key, self-service
5. **Add analytics** - Popular vehicles, utilization rates
6. **Add partner integrations** - Airlines, hotels, insurance
7. **Add maintenance scheduling** - Track service intervals
8. **Add fleet management** - Optimize vehicle distribution across locations

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `searchVehicles` | O(v × r) | v = vehicles at location, r = reservations |
| `makeReservation` | O(r) | Check all reservations for conflicts |
| `pickupVehicle` | O(1) | Direct lookup |
| `returnVehicle` | O(1) | Direct lookup and updates |
| `cancelReservation` | O(1) | Direct lookup |

### Space Complexity

| Component | Space |
|-----------|-------|
| Vehicles | O(v) |
| Locations | O(l) |
| Reservations | O(r) |
| Rentals | O(n) active rentals |

### Bottlenecks at Scale

**10x Usage (10 → 100 locations, 1K → 10K vehicles):**
- Problem: Vehicle search across locations becomes slow, concurrent reservation conflicts increase, inventory management overhead grows
- Solution: Index vehicles by location/type, use distributed locking (Redis) for reservations, optimize search queries
- Tradeoff: Additional infrastructure (Redis), more complex locking logic

**100x Usage (10 → 1K locations, 1K → 100K vehicles):**
- Problem: Single instance can't handle all locations, database becomes bottleneck, real-time availability queries too slow
- Solution: Shard locations by region, use read replicas for availability queries, implement caching layer (Redis) for hot inventory data
- Tradeoff: Distributed system complexity, need shard routing and data consistency across shards


### Q1: How would you handle vehicle upgrades?

```java
public class UpgradeService {
    public Vehicle offerUpgrade(Reservation reservation) {
        VehicleType currentType = reservation.getVehicle().getType();
        VehicleType upgradeType = getNextTier(currentType);
        
        List<Vehicle> upgrades = findAvailable(
            reservation.getPickupLocation(),
            upgradeType,
            reservation.getPickupTime(),
            reservation.getDropoffTime()
        );
        
        if (!upgrades.isEmpty()) {
            return upgrades.get(0);
        }
        return null;
    }
    
    public Reservation applyUpgrade(Reservation reservation, Vehicle upgrade) {
        // Release original vehicle
        reservation.getVehicle().cancelReservation();
        
        // Book upgrade
        upgrade.reserve();
        
        // Create new reservation with upgrade
        return new Reservation(
            reservation.getCustomer(),
            upgrade,
            reservation.getPickupLocation(),
            reservation.getDropoffLocation(),
            reservation.getPickupTime(),
            reservation.getDropoffTime()
        );
    }
}
```

### Q2: How would you implement insurance options?

```java
public enum InsuranceType {
    NONE(0),
    BASIC(15),      // $15/day
    PREMIUM(25),    // $25/day
    FULL(40);       // $40/day
    
    private final int dailyRate;
}

public class InsuranceOption {
    private final InsuranceType type;
    private final BigDecimal deductible;
    private final List<String> coverage;
}

public class ReservationWithInsurance extends Reservation {
    private InsuranceOption insurance;
    
    public BigDecimal getInsuranceCost() {
        return BigDecimal.valueOf(insurance.getType().getDailyRate())
            .multiply(BigDecimal.valueOf(getDurationDays()));
    }
}
```

### Q3: How would you handle fuel policies?

```java
public enum FuelPolicy {
    FULL_TO_FULL,       // Return with full tank
    PREPAID,            // Pay upfront for full tank
    SAME_TO_SAME        // Return with same level
}

public class FuelTracker {
    private final FuelPolicy policy;
    private final double tankCapacity;
    private final BigDecimal fuelPrice;
    
    public BigDecimal calculateFuelCharge(Rental rental) {
        switch (policy) {
            case FULL_TO_FULL:
                double returnLevel = rental.getReturnFuelLevel();
                if (returnLevel < 1.0) {
                    double gallonsNeeded = tankCapacity * (1.0 - returnLevel);
                    // Charge premium for refueling service
                    return fuelPrice.multiply(BigDecimal.valueOf(gallonsNeeded))
                                   .multiply(new BigDecimal("1.5"));
                }
                return BigDecimal.ZERO;
                
            case PREPAID:
                return fuelPrice.multiply(BigDecimal.valueOf(tankCapacity));
                
            default:
                return BigDecimal.ZERO;
        }
    }
}
```

### Q4: How would you implement fleet management?

```java
public class FleetManager {
    private final Map<Location, Map<VehicleType, Integer>> targetInventory;
    
    public List<VehicleTransfer> calculateTransfers() {
        List<VehicleTransfer> transfers = new ArrayList<>();
        
        for (Location location : locations) {
            Map<VehicleType, Integer> current = getCurrentInventory(location);
            Map<VehicleType, Integer> target = targetInventory.get(location);
            
            for (VehicleType type : VehicleType.values()) {
                int diff = target.getOrDefault(type, 0) - 
                          current.getOrDefault(type, 0);
                
                if (diff > 0) {
                    // Need more vehicles
                    transfers.add(new VehicleTransfer(
                        findSurplusLocation(type), location, type, diff));
                }
            }
        }
        
        return optimizeTransfers(transfers);
    }
    
    public void scheduleMaintenance(Vehicle vehicle) {
        if (vehicle.getMileage() > 30000 || 
            vehicle.getLastMaintenance().plusMonths(3).isBefore(LocalDate.now())) {
            vehicle.sendToMaintenance();
            maintenanceQueue.add(vehicle);
        }
    }
}
```

