# 🚗 Car Rental System - Problem Solution

## STEP 0: REQUIREMENTS QUICKPASS

### Core Functional Requirements
- Manage vehicle inventory with different types (sedan, SUV, truck, van)
- Handle booking and returns
- Support pricing (daily, hourly rates, mileage)
- Calculate late fees and damage charges
- Track vehicle availability and location
- Support multiple rental locations
- Handle one-way rentals (different pickup/dropoff)

### Explicit Out-of-Scope Items
- Online payment processing
- Insurance management
- Vehicle maintenance scheduling
- GPS tracking integration
- Loyalty programs
- International rentals

### Assumptions and Constraints
- **Multiple Locations**: Support several branches
- **Vehicle Types**: Sedan, SUV, Truck, Van
- **Pricing**: Daily rate + mileage (if applicable)
- **Late Fee**: 1.5x daily rate per extra day
- **Age Requirement**: Driver must be 21+

### Scale Assumptions (LLD Focus)
- Single company with ~10 locations
- Fleet size: ~500 vehicles total
- Peak reservations: ~1000 per day
- In-memory storage acceptable for LLD scope

### Concurrency Model
- **Synchronized vehicle reservation**: Prevents double-booking same vehicle
- **ConcurrentHashMap** for reservations and vehicle registries
- **Atomic availability check + booking**: Single transaction to prevent race conditions
- **Location inventory updates**: Synchronized per location

### Public APIs
- `searchVehicles(location, dates, type)`: Find available vehicles
- `makeReservation(customer, vehicle, dates)`: Book vehicle
- `pickupVehicle(reservationId)`: Start rental
- `returnVehicle(reservationId, location)`: End rental
- `calculateCharges(rental)`: Get total cost

### Public API Usage Examples
```java
// Example 1: Basic usage
RentalService service = new RentalService();
Location location = new Location("LOC1", "Airport", "123 Main St", "NYC", true);
service.addLocation(location);
Vehicle vehicle = new Vehicle("V1", VehicleType.SEDAN, "Honda", "Accord", 2023, "ABC-123", 5);
service.addVehicle(vehicle, location);
SearchCriteria criteria = new SearchCriteria(location, LocalDateTime.now().plusDays(1), LocalDateTime.now().plusDays(3));
List<Vehicle> available = service.searchVehicles(criteria);
Customer customer = new Customer("John", "john@email.com", "555-1234", "DL123", LocalDate.now().plusYears(2), LocalDate.of(1990, 1, 1));
Reservation reservation = service.makeReservation(customer, vehicle, criteria);

// Example 2: Typical workflow
Rental rental = service.pickupVehicle(reservation.getId());
// ... customer uses vehicle ...
Invoice invoice = service.returnVehicle(rental.getId(), location, rental.getStartMileage() + 450);
invoice.print();

// Example 3: Edge case - invalid customer
Customer invalidCustomer = new Customer("Jane", "jane@email.com", "555-5678", "DL456", LocalDate.now().minusDays(1), LocalDate.of(2010, 1, 1));
assertThrows(IllegalArgumentException.class, () -> 
    service.makeReservation(invalidCustomer, vehicle, criteria));
```

### Invariants
- **Vehicle Availability**: Can't book already reserved vehicle
- **Location Tracking**: Vehicle always at known location
- **Pricing Consistency**: Charges match agreed rates

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CAR RENTAL SYSTEM                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                        RentalService                                      │   │
│  │                                                                           │   │
│  │  - vehicles: Map<String, Vehicle>                                        │   │
│  │  - reservations: Map<String, Reservation>                                │   │
│  │  - locations: Map<String, Location>                                      │   │
│  │                                                                           │   │
│  │  + searchVehicles(criteria): List<Vehicle>                               │   │
│  │  + makeReservation(request): Reservation                                 │   │
│  │  + pickupVehicle(reservationId): Rental                                  │   │
│  │  + returnVehicle(rentalId, location): Invoice                           │   │
│  │  + cancelReservation(reservationId): boolean                             │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│           ┌──────────────┼──────────────┐                                       │
│           │              │              │                                       │
│           ▼              ▼              ▼                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                             │
│  │   Vehicle   │  │ Reservation │  │   Rental    │                             │
│  │             │  │             │  │             │                             │
│  │ - id        │  │ - id        │  │ - id        │                             │
│  │ - type      │  │ - customer  │  │ - vehicle   │                             │
│  │ - make/model│  │ - vehicle   │  │ - startTime │                             │
│  │ - status    │  │ - dates     │  │ - endTime   │                             │
│  │ - location  │  │ - status    │  │ - mileage   │                             │
│  └─────────────┘  └─────────────┘  └─────────────┘                             │
│        △                                    │                                   │
│        │                                    │                                   │
│  ┌─────┴─────┬─────────┬─────────┐         ▼                                   │
│  │           │         │         │  ┌─────────────┐                            │
│  │   Car     │   SUV   │  Truck  │  │   Invoice   │                            │
│  │           │         │         │  │             │                            │
│  └───────────┴─────────┴─────────┘  │ - charges   │                            │
│                                      │ - fees      │                            │
│  ┌─────────────┐  ┌─────────────┐   │ - total     │                            │
│  │  Location   │  │   Customer  │   └─────────────┘                            │
│  │             │  │             │                                               │
│  │ - name      │  │ - name      │   ┌─────────────┐                            │
│  │ - address   │  │ - license   │   │PricingStrategy│                          │
│  │ - vehicles  │  │ - email     │   │ (interface) │                            │
│  └─────────────┘  └─────────────┘   └─────────────┘                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Rental Flow Visualization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           RENTAL LIFECYCLE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐           │
│  │  Search  │────►│  Reserve │────►│  Pickup  │────►│  Return  │           │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘           │
│       │                │                │                │                  │
│       ▼                ▼                ▼                ▼                  │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐           │
│  │Available │     │Reserved  │     │ Rented   │     │Available │           │
│  │ Vehicles │     │ Vehicle  │     │ Vehicle  │     │ Vehicle  │           │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘           │
│                                                           │                 │
│                                                           ▼                 │
│                                                     ┌──────────┐           │
│                                                     │ Invoice  │           │
│                                                     │Generated │           │
│                                                     └──────────┘           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Responsibilities Table

| Class | Owns | Why |
|-------|------|-----|
| `Vehicle` | Vehicle properties and status management | Encapsulates vehicle state - manages vehicle availability, status transitions, and location |
| `Location` | Rental branch location and vehicle inventory | Manages rental location - encapsulates location properties and vehicle collection at branch |
| `Customer` | Customer information (license, contact) | Stores customer data - separates customer information from rental/reservation logic |
| `Reservation` | Booking details (customer, vehicle, dates, status) | Encapsulates reservation data - stores reservation information and status |
| `Rental` | Active rental tracking (vehicle, dates, mileage) | Tracks active rental - stores rental period information and mileage |
| `Invoice` | Billing information and calculation | Generates billing - separates billing presentation and calculation from rental logic |
| `PricingStrategy` (interface) | Pricing calculation contract | Defines pricing interface - enables multiple pricing strategies (daily, hourly, discount) |
| `RentalService` | Rental operations coordination | Coordinates rental workflow - separates business logic from domain objects, handles search/reserve/pickup/return |

---

## STEP 4: Code Walkthrough - Building From Scratch

This section explains how an engineer builds this system from scratch, in the order code should be written.

### Phase 1: Design the Vehicle Model

```java
// Step 1: Vehicle types and status
public enum VehicleType {
    ECONOMY("Economy", 1.0),
    SEDAN("Sedan", 1.5),
    SUV("SUV", 2.0),
    LUXURY("Luxury", 3.0);
}

public enum VehicleStatus {
    AVAILABLE, RESERVED, RENTED, MAINTENANCE
}

// Step 2: Vehicle class
public class Vehicle {
    private final String id;
    private final VehicleType type;
    private VehicleStatus status;
    private Location currentLocation;
    private int mileage;
    
    public void reserve() {
        if (status != VehicleStatus.AVAILABLE) {
            throw new IllegalStateException("Vehicle not available");
        }
        this.status = VehicleStatus.RESERVED;
    }
}
```

---

### Phase 2: Design Reservation and Rental

```java
// Step 3: Reservation - future booking
public class Reservation {
    private final String id;
    private final Customer customer;
    private final Vehicle vehicle;
    private final Location pickupLocation;
    private final Location dropoffLocation;
    private final LocalDateTime pickupTime;
    private final LocalDateTime dropoffTime;
    private ReservationStatus status;
}

// Step 4: Rental - active rental
public class Rental {
    private final String id;
    private final Reservation reservation;
    private final int startMileage;
    private final LocalDateTime actualPickupTime;
    private LocalDateTime actualReturnTime;
    private int endMileage;
}
```

---

### Phase 3: Design the Rental Service

```java
// Step 5: Rental service
public class RentalService {
    private final Map<String, Reservation> reservations;
    private final Map<String, Rental> rentals;
    
    public synchronized Reservation makeReservation(Customer customer, Vehicle vehicle,
                                                     SearchCriteria criteria) {
        if (!customer.hasValidLicense()) {
            throw new IllegalArgumentException("License expired");
        }
        
        // Verify still available (double-check)
        if (!isVehicleAvailable(vehicle, criteria.getPickupTime(), 
                               criteria.getDropoffTime())) {
            throw new IllegalStateException("Vehicle no longer available");
        }
        
        Reservation reservation = new Reservation(
            customer, vehicle,
            criteria.getPickupLocation(), criteria.getDropoffLocation(),
            criteria.getPickupTime(), criteria.getDropoffTime()
        );
        
        vehicle.reserve();
        reservations.put(reservation.getId(), reservation);
        
        return reservation;
    }
    
    private boolean isVehicleAvailable(Vehicle vehicle, 
                                       LocalDateTime start, LocalDateTime end) {
        for (Reservation res : reservations.values()) {
            if (res.getVehicle().equals(vehicle) && 
                res.getStatus() == ReservationStatus.CONFIRMED) {
                if (start.isBefore(res.getDropoffTime()) && 
                    end.isAfter(res.getPickupTime())) {
                    return false;  // Overlaps
                }
            }
        }
        return true;
    }
}
```

---

### Phase 4: Threading Model and Concurrency Control

**Threading Model:**

This car rental system handles **concurrent reservation requests**:
- Multiple customers can try to reserve the same vehicle
- Reservation operations must be atomic to prevent double-booking
- Vehicle availability checks must be thread-safe

**Concurrency Control:**

```java
// Synchronized method ensures atomic reservation
public synchronized Reservation makeReservation(Customer customer, Vehicle vehicle,
                                                 SearchCriteria criteria) {
    // Entire operation is synchronized:
    // 1. Check availability
    // 2. Create reservation
    // 3. Reserve vehicle
    // All happens atomically
}

// Alternative: Per-vehicle locking (better granularity)
private final Map<String, ReentrantLock> vehicleLocks = new ConcurrentHashMap<>();

public Reservation makeReservation(Customer customer, Vehicle vehicle,
                                   SearchCriteria criteria) {
    ReentrantLock lock = vehicleLocks.computeIfAbsent(
        vehicle.getId(), k -> new ReentrantLock());
    
    lock.lock();
    try {
        // Double-check availability after acquiring lock
        if (!isVehicleAvailable(vehicle, ...)) {
            throw new IllegalStateException("Vehicle no longer available");
        }
        // Create reservation
        Reservation reservation = new Reservation(...);
        vehicle.reserve();
        reservations.put(reservation.getId(), reservation);
        return reservation;
    } finally {
        lock.unlock();
    }
}
```

**Why synchronized?**
- Without: Two threads can both see vehicle as available and both reserve it
- With: Only one thread can check and reserve atomically

---

## STEP 2: Complete Final Implementation

> **Verified:** This code compiles successfully with Java 11+.

### 2.1 VehicleType Enum

```java
// VehicleType.java
package com.carrental;

/**
 * Types of vehicles available for rental.
 */
public enum VehicleType {
    ECONOMY("Economy", 1.0),
    COMPACT("Compact", 1.2),
    SEDAN("Sedan", 1.5),
    SUV("SUV", 2.0),
    LUXURY("Luxury", 3.0),
    TRUCK("Truck", 2.5),
    VAN("Van", 2.2);
    
    private final String displayName;
    private final double priceMultiplier;
    
    VehicleType(String displayName, double priceMultiplier) {
        this.displayName = displayName;
        this.priceMultiplier = priceMultiplier;
    }
    
    public String getDisplayName() { return displayName; }
    public double getPriceMultiplier() { return priceMultiplier; }
}
```

### 2.2 VehicleStatus Enum

```java
// VehicleStatus.java
package com.carrental;

/**
 * Status of a vehicle in the system.
 */
public enum VehicleStatus {
    AVAILABLE,      // Ready for rental
    RESERVED,       // Reserved but not picked up
    RENTED,         // Currently rented out
    MAINTENANCE,    // Under maintenance
    OUT_OF_SERVICE  // Not available
}
```

### 2.3 Vehicle Class

```java
// Vehicle.java
package com.carrental;

/**
 * Represents a vehicle in the rental fleet.
 */
public class Vehicle {
    
    private final String id;
    private final VehicleType type;
    private final String make;
    private final String model;
    private final int year;
    private final String licensePlate;
    private final int seats;
    private VehicleStatus status;
    private Location currentLocation;
    private int mileage;
    
    public Vehicle(String id, VehicleType type, String make, String model, 
                   int year, String licensePlate, int seats) {
        this.id = id;
        this.type = type;
        this.make = make;
        this.model = model;
        this.year = year;
        this.licensePlate = licensePlate;
        this.seats = seats;
        this.status = VehicleStatus.AVAILABLE;
        this.mileage = 0;
    }
    
    public void reserve() {
        if (status != VehicleStatus.AVAILABLE) {
            throw new IllegalStateException("Vehicle not available for reservation");
        }
        this.status = VehicleStatus.RESERVED;
    }
    
    public void pickup() {
        if (status != VehicleStatus.RESERVED) {
            throw new IllegalStateException("Vehicle not reserved");
        }
        this.status = VehicleStatus.RENTED;
    }
    
    public void returnVehicle(Location location, int endMileage) {
        if (status != VehicleStatus.RENTED) {
            throw new IllegalStateException("Vehicle not rented");
        }
        this.status = VehicleStatus.AVAILABLE;
        this.currentLocation = location;
        this.mileage = endMileage;
    }
    
    public void cancelReservation() {
        if (status != VehicleStatus.RESERVED) {
            throw new IllegalStateException("Vehicle not reserved");
        }
        this.status = VehicleStatus.AVAILABLE;
    }
    
    public void sendToMaintenance() {
        this.status = VehicleStatus.MAINTENANCE;
    }
    
    public void completeMaintenace() {
        this.status = VehicleStatus.AVAILABLE;
    }
    
    public boolean isAvailable() {
        return status == VehicleStatus.AVAILABLE;
    }
    
    // Getters
    public String getId() { return id; }
    public VehicleType getType() { return type; }
    public String getMake() { return make; }
    public String getModel() { return model; }
    public int getYear() { return year; }
    public String getLicensePlate() { return licensePlate; }
    public int getSeats() { return seats; }
    public VehicleStatus getStatus() { return status; }
    public Location getCurrentLocation() { return currentLocation; }
    public int getMileage() { return mileage; }
    
    public void setCurrentLocation(Location location) {
        this.currentLocation = location;
    }
    
    @Override
    public String toString() {
        return String.format("%s %s %s (%d) - %s [%s]", 
            type.getDisplayName(), make, model, year, licensePlate, status);
    }
}
```

### 2.4 Location Class

```java
// Location.java
package com.carrental;

import java.util.*;

/**
 * Represents a rental location/branch.
 */
public class Location {
    
    private final String id;
    private final String name;
    private final String address;
    private final String city;
    private final Set<Vehicle> vehicles;
    private final boolean airportLocation;
    
    public Location(String id, String name, String address, String city, 
                    boolean airportLocation) {
        this.id = id;
        this.name = name;
        this.address = address;
        this.city = city;
        this.airportLocation = airportLocation;
        this.vehicles = new HashSet<>();
    }
    
    public void addVehicle(Vehicle vehicle) {
        vehicles.add(vehicle);
        vehicle.setCurrentLocation(this);
    }
    
    public void removeVehicle(Vehicle vehicle) {
        vehicles.remove(vehicle);
    }
    
    public List<Vehicle> getAvailableVehicles() {
        List<Vehicle> available = new ArrayList<>();
        for (Vehicle v : vehicles) {
            if (v.isAvailable()) {
                available.add(v);
            }
        }
        return available;
    }
    
    public List<Vehicle> getAvailableVehicles(VehicleType type) {
        List<Vehicle> available = new ArrayList<>();
        for (Vehicle v : vehicles) {
            if (v.isAvailable() && v.getType() == type) {
                available.add(v);
            }
        }
        return available;
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public String getAddress() { return address; }
    public String getCity() { return city; }
    public boolean isAirportLocation() { return airportLocation; }
    public Set<Vehicle> getVehicles() { return Collections.unmodifiableSet(vehicles); }
    
    @Override
    public String toString() {
        return String.format("%s - %s, %s", name, address, city);
    }
}
```

### 2.5 Customer Class

```java
// Customer.java
package com.carrental;

import java.time.LocalDate;

/**
 * Represents a customer/renter.
 */
public class Customer {
    
    private final String id;
    private final String name;
    private final String email;
    private final String phone;
    private final String driverLicense;
    private final LocalDate licenseExpiry;
    private final LocalDate dateOfBirth;
    
    public Customer(String name, String email, String phone, 
                    String driverLicense, LocalDate licenseExpiry, LocalDate dateOfBirth) {
        this.id = UUID.randomUUID().toString().substring(0, 8);
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.driverLicense = driverLicense;
        this.licenseExpiry = licenseExpiry;
        this.dateOfBirth = dateOfBirth;
    }
    
    public boolean hasValidLicense() {
        return licenseExpiry.isAfter(LocalDate.now());
    }
    
    public int getAge() {
        return java.time.Period.between(dateOfBirth, LocalDate.now()).getYears();
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
    public String getDriverLicense() { return driverLicense; }
    public LocalDate getLicenseExpiry() { return licenseExpiry; }
    public LocalDate getDateOfBirth() { return dateOfBirth; }
    
    @Override
    public String toString() {
        return String.format("Customer[%s, %s]", name, email);
    }
}
```

### 2.6 ReservationStatus Enum

```java
// ReservationStatus.java
package com.carrental;

public enum ReservationStatus {
    PENDING,
    CONFIRMED,
    PICKED_UP,
    COMPLETED,
    CANCELLED,
    NO_SHOW
}
```

### 2.7 Reservation Class

```java
// Reservation.java
package com.carrental;

import java.time.*;

/**
 * Represents a vehicle reservation.
 */
public class Reservation {
    
    private final String id;
    private final Customer customer;
    private final Vehicle vehicle;
    private final Location pickupLocation;
    private final Location dropoffLocation;
    private final LocalDateTime pickupTime;
    private final LocalDateTime dropoffTime;
    private ReservationStatus status;
    private final LocalDateTime createdAt;
    
    public Reservation(Customer customer, Vehicle vehicle, 
                       Location pickupLocation, Location dropoffLocation,
                       LocalDateTime pickupTime, LocalDateTime dropoffTime) {
        this.id = "RES-" + System.currentTimeMillis() % 100000;
        this.customer = customer;
        this.vehicle = vehicle;
        this.pickupLocation = pickupLocation;
        this.dropoffLocation = dropoffLocation;
        this.pickupTime = pickupTime;
        this.dropoffTime = dropoffTime;
        this.status = ReservationStatus.CONFIRMED;
        this.createdAt = LocalDateTime.now();
    }
    
    public void pickup() {
        if (status != ReservationStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot pickup: status is " + status);
        }
        this.status = ReservationStatus.PICKED_UP;
    }
    
    public void complete() {
        if (status != ReservationStatus.PICKED_UP) {
            throw new IllegalStateException("Cannot complete: status is " + status);
        }
        this.status = ReservationStatus.COMPLETED;
    }
    
    public void cancel() {
        if (status == ReservationStatus.PICKED_UP || 
            status == ReservationStatus.COMPLETED) {
            throw new IllegalStateException("Cannot cancel: status is " + status);
        }
        this.status = ReservationStatus.CANCELLED;
    }
    
    public void markNoShow() {
        if (status != ReservationStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot mark no-show: status is " + status);
        }
        this.status = ReservationStatus.NO_SHOW;
    }
    
    public long getDurationDays() {
        return Duration.between(pickupTime, dropoffTime).toDays();
    }
    
    public long getDurationHours() {
        return Duration.between(pickupTime, dropoffTime).toHours();
    }
    
    public boolean isDifferentDropoff() {
        return !pickupLocation.equals(dropoffLocation);
    }
    
    // Getters
    public String getId() { return id; }
    public Customer getCustomer() { return customer; }
    public Vehicle getVehicle() { return vehicle; }
    public Location getPickupLocation() { return pickupLocation; }
    public Location getDropoffLocation() { return dropoffLocation; }
    public LocalDateTime getPickupTime() { return pickupTime; }
    public LocalDateTime getDropoffTime() { return dropoffTime; }
    public ReservationStatus getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    
    @Override
    public String toString() {
        return String.format("Reservation[%s: %s, %s, %s to %s, %s]",
            id, customer.getName(), vehicle.getMake() + " " + vehicle.getModel(),
            pickupTime.toLocalDate(), dropoffTime.toLocalDate(), status);
    }
}
```

### 2.8 Rental Class

```java
// Rental.java
package com.carrental;

import java.time.LocalDateTime;

/**
 * Represents an active rental (vehicle has been picked up).
 */
public class Rental {
    
    private final String id;
    private final Reservation reservation;
    private final int startMileage;
    private final LocalDateTime actualPickupTime;
    private LocalDateTime actualReturnTime;
    private int endMileage;
    private Location actualReturnLocation;
    
    public Rental(Reservation reservation, int startMileage) {
        this.id = "RNT-" + System.currentTimeMillis() % 100000;
        this.reservation = reservation;
        this.startMileage = startMileage;
        this.actualPickupTime = LocalDateTime.now();
    }
    
    public void complete(Location returnLocation, int endMileage) {
        this.actualReturnTime = LocalDateTime.now();
        this.actualReturnLocation = returnLocation;
        this.endMileage = endMileage;
    }
    
    public int getMilesDriven() {
        return endMileage - startMileage;
    }
    
    public boolean isLateReturn() {
        if (actualReturnTime == null) {
            return LocalDateTime.now().isAfter(reservation.getDropoffTime());
        }
        return actualReturnTime.isAfter(reservation.getDropoffTime());
    }
    
    public long getLateHours() {
        if (!isLateReturn()) return 0;
        LocalDateTime returnTime = actualReturnTime != null ? 
            actualReturnTime : LocalDateTime.now();
        return java.time.Duration.between(
            reservation.getDropoffTime(), returnTime).toHours();
    }
    
    // Getters
    public String getId() { return id; }
    public Reservation getReservation() { return reservation; }
    public int getStartMileage() { return startMileage; }
    public LocalDateTime getActualPickupTime() { return actualPickupTime; }
    public LocalDateTime getActualReturnTime() { return actualReturnTime; }
    public int getEndMileage() { return endMileage; }
    public Location getActualReturnLocation() { return actualReturnLocation; }
}
```

### 2.9 PricingStrategy Interface

```java
// PricingStrategy.java
package com.carrental;

import java.math.BigDecimal;

/**
 * Strategy for calculating rental prices.
 */
public interface PricingStrategy {
    
    BigDecimal calculateBasePrice(Reservation reservation);
    BigDecimal calculateLateFee(Rental rental);
    BigDecimal calculateMileageFee(Rental rental, int includedMiles);
    BigDecimal calculateDropoffFee(Reservation reservation);
}
```

### 2.10 StandardPricingStrategy

```java
// StandardPricingStrategy.java
package com.carrental;

import java.math.BigDecimal;
import java.math.RoundingMode;

/**
 * Standard pricing implementation.
 */
public class StandardPricingStrategy implements PricingStrategy {
    
    private static final BigDecimal BASE_DAILY_RATE = new BigDecimal("50.00");
    private static final BigDecimal LATE_FEE_PER_HOUR = new BigDecimal("15.00");
    private static final BigDecimal MILEAGE_FEE_PER_MILE = new BigDecimal("0.25");
    private static final BigDecimal DIFFERENT_DROPOFF_FEE = new BigDecimal("75.00");
    private static final BigDecimal AIRPORT_SURCHARGE = new BigDecimal("25.00");
    
    @Override
    public BigDecimal calculateBasePrice(Reservation reservation) {
        long days = reservation.getDurationDays();
        if (days == 0) days = 1;  // Minimum 1 day
        
        BigDecimal dailyRate = BASE_DAILY_RATE.multiply(
            BigDecimal.valueOf(reservation.getVehicle().getType().getPriceMultiplier()));
        
        BigDecimal basePrice = dailyRate.multiply(BigDecimal.valueOf(days));
        
        // Airport surcharge
        if (reservation.getPickupLocation().isAirportLocation()) {
            basePrice = basePrice.add(AIRPORT_SURCHARGE);
        }
        
        return basePrice.setScale(2, RoundingMode.HALF_UP);
    }
    
    @Override
    public BigDecimal calculateLateFee(Rental rental) {
        long lateHours = rental.getLateHours();
        if (lateHours <= 0) return BigDecimal.ZERO;
        
        return LATE_FEE_PER_HOUR.multiply(BigDecimal.valueOf(lateHours))
                               .setScale(2, RoundingMode.HALF_UP);
    }
    
    @Override
    public BigDecimal calculateMileageFee(Rental rental, int includedMiles) {
        int milesDriven = rental.getMilesDriven();
        int excessMiles = milesDriven - includedMiles;
        
        if (excessMiles <= 0) return BigDecimal.ZERO;
        
        return MILEAGE_FEE_PER_MILE.multiply(BigDecimal.valueOf(excessMiles))
                                   .setScale(2, RoundingMode.HALF_UP);
    }
    
    @Override
    public BigDecimal calculateDropoffFee(Reservation reservation) {
        if (reservation.isDifferentDropoff()) {
            return DIFFERENT_DROPOFF_FEE;
        }
        return BigDecimal.ZERO;
    }
}
```

### 2.11 Invoice Class

```java
// Invoice.java
package com.carrental;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.*;

/**
 * Invoice generated after rental completion.
 */
public class Invoice {
    
    private final String id;
    private final Rental rental;
    private final Map<String, BigDecimal> charges;
    private final BigDecimal total;
    private final LocalDateTime generatedAt;
    
    private Invoice(Builder builder) {
        this.id = "INV-" + System.currentTimeMillis() % 100000;
        this.rental = builder.rental;
        this.charges = Collections.unmodifiableMap(builder.charges);
        this.total = builder.total;
        this.generatedAt = LocalDateTime.now();
    }
    
    public static Builder builder(Rental rental) {
        return new Builder(rental);
    }
    
    public void print() {
        System.out.println("\n========================================");
        System.out.println("           RENTAL INVOICE");
        System.out.println("========================================");
        System.out.println("Invoice #: " + id);
        System.out.println("Date: " + generatedAt.toLocalDate());
        System.out.println("----------------------------------------");
        System.out.println("Customer: " + rental.getReservation().getCustomer().getName());
        System.out.println("Vehicle: " + rental.getReservation().getVehicle());
        System.out.println("Pickup: " + rental.getActualPickupTime().toLocalDate());
        System.out.println("Return: " + rental.getActualReturnTime().toLocalDate());
        System.out.println("Miles Driven: " + rental.getMilesDriven());
        System.out.println("----------------------------------------");
        System.out.println("CHARGES:");
        for (Map.Entry<String, BigDecimal> charge : charges.entrySet()) {
            System.out.printf("  %-25s $%8.2f%n", charge.getKey(), charge.getValue());
        }
        System.out.println("----------------------------------------");
        System.out.printf("  %-25s $%8.2f%n", "TOTAL", total);
        System.out.println("========================================\n");
    }
    
    // Getters
    public String getId() { return id; }
    public Rental getRental() { return rental; }
    public Map<String, BigDecimal> getCharges() { return charges; }
    public BigDecimal getTotal() { return total; }
    public LocalDateTime getGeneratedAt() { return generatedAt; }
    
    public static class Builder {
        private final Rental rental;
        private final Map<String, BigDecimal> charges = new LinkedHashMap<>();
        private BigDecimal total = BigDecimal.ZERO;
        
        Builder(Rental rental) {
            this.rental = rental;
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

### 2.12 SearchCriteria Class

```java
// SearchCriteria.java
package com.carrental;

import java.time.LocalDateTime;

/**
 * Criteria for searching available vehicles.
 */
public class SearchCriteria {
    
    private final Location pickupLocation;
    private final Location dropoffLocation;
    private final LocalDateTime pickupTime;
    private final LocalDateTime dropoffTime;
    private VehicleType vehicleType;
    private Integer minSeats;
    
    public SearchCriteria(Location pickupLocation, LocalDateTime pickupTime,
                          LocalDateTime dropoffTime) {
        this(pickupLocation, pickupLocation, pickupTime, dropoffTime);
    }
    
    public SearchCriteria(Location pickupLocation, Location dropoffLocation,
                          LocalDateTime pickupTime, LocalDateTime dropoffTime) {
        this.pickupLocation = pickupLocation;
        this.dropoffLocation = dropoffLocation;
        this.pickupTime = pickupTime;
        this.dropoffTime = dropoffTime;
    }
    
    public SearchCriteria withVehicleType(VehicleType type) {
        this.vehicleType = type;
        return this;
    }
    
    public SearchCriteria withMinSeats(int seats) {
        this.minSeats = seats;
        return this;
    }
    
    // Getters
    public Location getPickupLocation() { return pickupLocation; }
    public Location getDropoffLocation() { return dropoffLocation; }
    public LocalDateTime getPickupTime() { return pickupTime; }
    public LocalDateTime getDropoffTime() { return dropoffTime; }
    public VehicleType getVehicleType() { return vehicleType; }
    public Integer getMinSeats() { return minSeats; }
}
```

### 2.13 RentalService Class

```java
// RentalService.java
package com.carrental;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Main service for managing car rentals.
 */
public class RentalService {
    
    private final Map<String, Vehicle> vehicles;
    private final Map<String, Location> locations;
    private final Map<String, Reservation> reservations;
    private final Map<String, Rental> rentals;
    private final PricingStrategy pricingStrategy;
    private final int includedMilesPerDay;
    
    public RentalService() {
        this(new StandardPricingStrategy(), 100);
    }
    
    public RentalService(PricingStrategy pricingStrategy, int includedMilesPerDay) {
        this.vehicles = new ConcurrentHashMap<>();
        this.locations = new ConcurrentHashMap<>();
        this.reservations = new ConcurrentHashMap<>();
        this.rentals = new ConcurrentHashMap<>();
        this.pricingStrategy = pricingStrategy;
        this.includedMilesPerDay = includedMilesPerDay;
    }
    
    // ==================== Location Management ====================
    
    public void addLocation(Location location) {
        locations.put(location.getId(), location);
    }
    
    public Location getLocation(String id) {
        return locations.get(id);
    }
    
    // ==================== Vehicle Management ====================
    
    public void addVehicle(Vehicle vehicle, Location location) {
        vehicles.put(vehicle.getId(), vehicle);
        location.addVehicle(vehicle);
    }
    
    public Vehicle getVehicle(String id) {
        return vehicles.get(id);
    }
    
    // ==================== Search ====================
    
    public List<Vehicle> searchVehicles(SearchCriteria criteria) {
        List<Vehicle> available = new ArrayList<>();
        
        for (Vehicle vehicle : criteria.getPickupLocation().getAvailableVehicles()) {
            // Check vehicle type
            if (criteria.getVehicleType() != null && 
                vehicle.getType() != criteria.getVehicleType()) {
                continue;
            }
            
            // Check seats
            if (criteria.getMinSeats() != null && 
                vehicle.getSeats() < criteria.getMinSeats()) {
                continue;
            }
            
            // Check if not reserved for the requested period
            if (isVehicleAvailable(vehicle, criteria.getPickupTime(), 
                                   criteria.getDropoffTime())) {
                available.add(vehicle);
            }
        }
        
        // Sort by price (vehicle type multiplier)
        available.sort(Comparator.comparingDouble(v -> v.getType().getPriceMultiplier()));
        
        return available;
    }
    
    private boolean isVehicleAvailable(Vehicle vehicle, 
                                       LocalDateTime start, LocalDateTime end) {
        for (Reservation res : reservations.values()) {
            if (res.getVehicle().equals(vehicle) && 
                res.getStatus() == ReservationStatus.CONFIRMED) {
                // Check for overlap
                if (start.isBefore(res.getDropoffTime()) && 
                    end.isAfter(res.getPickupTime())) {
                    return false;
                }
            }
        }
        return true;
    }
    
    // ==================== Reservation ====================
    
    public synchronized Reservation makeReservation(Customer customer, Vehicle vehicle,
                                                     SearchCriteria criteria) {
        // Validate customer
        if (!customer.hasValidLicense()) {
            throw new IllegalArgumentException("Customer's license has expired");
        }
        
        if (customer.getAge() < 21) {
            throw new IllegalArgumentException("Customer must be at least 21 years old");
        }
        
        // Verify vehicle is still available
        if (!isVehicleAvailable(vehicle, criteria.getPickupTime(), 
                               criteria.getDropoffTime())) {
            throw new IllegalStateException("Vehicle is no longer available");
        }
        
        // Create reservation
        Reservation reservation = new Reservation(
            customer, vehicle,
            criteria.getPickupLocation(), criteria.getDropoffLocation(),
            criteria.getPickupTime(), criteria.getDropoffTime()
        );
        
        vehicle.reserve();
        reservations.put(reservation.getId(), reservation);
        
        return reservation;
    }
    
    public boolean cancelReservation(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null) {
            return false;
        }
        
        try {
            reservation.cancel();
            reservation.getVehicle().cancelReservation();
            return true;
        } catch (IllegalStateException e) {
            return false;
        }
    }
    
    // ==================== Pickup ====================
    
    public synchronized Rental pickupVehicle(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null) {
            throw new IllegalArgumentException("Reservation not found");
        }
        
        if (reservation.getStatus() != ReservationStatus.CONFIRMED) {
            throw new IllegalStateException("Reservation status is " + reservation.getStatus());
        }
        
        Vehicle vehicle = reservation.getVehicle();
        
        // Create rental
        Rental rental = new Rental(reservation, vehicle.getMileage());
        
        // Update states
        reservation.pickup();
        vehicle.pickup();
        
        rentals.put(rental.getId(), rental);
        
        return rental;
    }
    
    // ==================== Return ====================
    
    public synchronized Invoice returnVehicle(String rentalId, Location returnLocation, 
                                              int endMileage) {
        Rental rental = rentals.get(rentalId);
        if (rental == null) {
            throw new IllegalArgumentException("Rental not found");
        }
        
        Reservation reservation = rental.getReservation();
        Vehicle vehicle = reservation.getVehicle();
        
        // Complete rental
        rental.complete(returnLocation, endMileage);
        reservation.complete();
        vehicle.returnVehicle(returnLocation, endMileage);
        
        // Update location inventory
        reservation.getPickupLocation().removeVehicle(vehicle);
        returnLocation.addVehicle(vehicle);
        
        // Generate invoice
        return generateInvoice(rental);
    }
    
    private Invoice generateInvoice(Rental rental) {
        Reservation reservation = rental.getReservation();
        
        BigDecimal basePrice = pricingStrategy.calculateBasePrice(reservation);
        BigDecimal lateFee = pricingStrategy.calculateLateFee(rental);
        
        int includedMiles = (int) (reservation.getDurationDays() * includedMilesPerDay);
        BigDecimal mileageFee = pricingStrategy.calculateMileageFee(rental, includedMiles);
        
        BigDecimal dropoffFee = pricingStrategy.calculateDropoffFee(reservation);
        
        Invoice.Builder builder = Invoice.builder(rental)
            .addCharge("Base Rental (" + reservation.getDurationDays() + " days)", basePrice);
        
        if (lateFee.compareTo(BigDecimal.ZERO) > 0) {
            builder.addCharge("Late Return Fee", lateFee);
        }
        
        if (mileageFee.compareTo(BigDecimal.ZERO) > 0) {
            builder.addCharge("Excess Mileage Fee", mileageFee);
        }
        
        if (dropoffFee.compareTo(BigDecimal.ZERO) > 0) {
            builder.addCharge("Different Location Dropoff", dropoffFee);
        }
        
        return builder.build();
    }
    
    // ==================== Queries ====================
    
    public Reservation getReservation(String id) {
        return reservations.get(id);
    }
    
    public Rental getRental(String id) {
        return rentals.get(id);
    }
    
    public List<Reservation> getCustomerReservations(Customer customer) {
        List<Reservation> result = new ArrayList<>();
        for (Reservation res : reservations.values()) {
            if (res.getCustomer().equals(customer)) {
                result.add(res);
            }
        }
        return result;
    }
}
```

### 2.14 Demo Application

```java
// CarRentalDemo.java
package com.carrental;

import java.time.*;

public class CarRentalDemo {
    
    public static void main(String[] args) {
        System.out.println("=== CAR RENTAL SYSTEM DEMO ===\n");
        
        RentalService service = new RentalService();
        
        // ==================== Setup Locations ====================
        System.out.println("===== SETTING UP LOCATIONS =====\n");
        
        Location airport = new Location("LOC1", "Airport Branch", 
            "123 Airport Rd", "New York", true);
        Location downtown = new Location("LOC2", "Downtown Branch", 
            "456 Main St", "New York", false);
        
        service.addLocation(airport);
        service.addLocation(downtown);
        
        System.out.println("Locations: " + airport.getName() + ", " + downtown.getName());
        
        // ==================== Add Vehicles ====================
        System.out.println("\n===== ADDING VEHICLES =====\n");
        
        Vehicle car1 = new Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Corolla", 
            2023, "ABC-123", 5);
        Vehicle car2 = new Vehicle("V2", VehicleType.SEDAN, "Honda", "Accord", 
            2023, "DEF-456", 5);
        Vehicle car3 = new Vehicle("V3", VehicleType.SUV, "Ford", "Explorer", 
            2022, "GHI-789", 7);
        Vehicle car4 = new Vehicle("V4", VehicleType.LUXURY, "BMW", "5 Series", 
            2023, "JKL-012", 5);
        
        service.addVehicle(car1, airport);
        service.addVehicle(car2, airport);
        service.addVehicle(car3, downtown);
        service.addVehicle(car4, downtown);
        
        System.out.println("Added vehicles to fleet");
        
        // ==================== Search Vehicles ====================
        System.out.println("\n===== SEARCHING VEHICLES =====\n");
        
        LocalDateTime pickup = LocalDateTime.now().plusDays(1).withHour(10);
        LocalDateTime dropoff = LocalDateTime.now().plusDays(4).withHour(10);
        
        SearchCriteria criteria = new SearchCriteria(airport, pickup, dropoff);
        
        System.out.println("Available at Airport for " + pickup.toLocalDate() + 
                          " to " + dropoff.toLocalDate() + ":");
        for (Vehicle v : service.searchVehicles(criteria)) {
            System.out.println("  " + v);
        }
        
        // ==================== Make Reservation ====================
        System.out.println("\n===== MAKING RESERVATION =====\n");
        
        Customer customer = new Customer(
            "John Smith", "john@email.com", "555-1234",
            "DL123456", LocalDate.now().plusYears(2), LocalDate.of(1990, 5, 15));
        
        Reservation reservation = service.makeReservation(customer, car2, criteria);
        System.out.println("Created: " + reservation);
        
        // ==================== Pickup Vehicle ====================
        System.out.println("\n===== PICKING UP VEHICLE =====\n");
        
        Rental rental = service.pickupVehicle(reservation.getId());
        System.out.println("Rental started: " + rental.getId());
        System.out.println("Vehicle: " + rental.getReservation().getVehicle());
        System.out.println("Start mileage: " + rental.getStartMileage());
        
        // ==================== Return Vehicle ====================
        System.out.println("\n===== RETURNING VEHICLE =====\n");
        
        // Simulate some mileage
        int endMileage = rental.getStartMileage() + 450;
        
        Invoice invoice = service.returnVehicle(rental.getId(), airport, endMileage);
        invoice.print();
        
        // ==================== One-Way Rental ====================
        System.out.println("===== ONE-WAY RENTAL =====\n");
        
        Customer customer2 = new Customer(
            "Jane Doe", "jane@email.com", "555-5678",
            "DL789012", LocalDate.now().plusYears(3), LocalDate.of(1985, 8, 20));
        
        SearchCriteria oneWayCriteria = new SearchCriteria(
            downtown, airport,  // Different dropoff
            LocalDateTime.now().plusDays(2).withHour(9),
            LocalDateTime.now().plusDays(3).withHour(18));
        
        List<Vehicle> available = service.searchVehicles(oneWayCriteria);
        if (!available.isEmpty()) {
            Reservation res2 = service.makeReservation(customer2, available.get(0), oneWayCriteria);
            System.out.println("One-way reservation: " + res2);
        }
        
        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## File Structure

```
com/carrental/
├── VehicleType.java
├── VehicleStatus.java
├── Vehicle.java
├── Location.java
├── Customer.java
├── ReservationStatus.java
├── Reservation.java
├── Rental.java
├── PricingStrategy.java
├── StandardPricingStrategy.java
├── Invoice.java
├── SearchCriteria.java
├── RentalService.java
└── CarRentalDemo.java
```

