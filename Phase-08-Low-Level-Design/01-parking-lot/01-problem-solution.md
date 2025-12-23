# 🚗 Parking Lot System - Complete Solution

## Problem Statement

Design a parking lot system that can:
- Support multiple vehicle types (motorcycle, car, truck)
- Have different parking spot types (compact, large, handicapped)
- Handle entry and exit of vehicles
- Track availability in real-time
- Support multiple floors
- Calculate parking fees based on duration

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              PARKING LOT SYSTEM                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │   Vehicle    │◄────────│   ParkingTicket  │────────►│   ParkingSpot    │    │
│  │  (abstract)  │         │                  │         │   (abstract)     │    │
│  └──────┬───────┘         └────────┬─────────┘         └────────┬─────────┘    │
│         │                          │                            │              │
│    ┌────┴────┬────┐               │                   ┌────────┼────────┐     │
│    │         │    │               │                   │        │        │     │
│    ▼         ▼    ▼               │                   ▼        ▼        ▼     │
│ ┌──────┐ ┌─────┐ ┌─────┐         │            ┌────────┐ ┌────────┐ ┌──────┐ │
│ │Motor │ │ Car │ │Truck│         │            │Compact │ │ Large  │ │Handi │ │
│ │cycle │ │     │ │     │         │            │  Spot  │ │  Spot  │ │capped│ │
│ └──────┘ └─────┘ └─────┘         │            └────────┘ └────────┘ └──────┘ │
│                                   │                                           │
│  ┌──────────────┐         ┌──────┴───────────┐         ┌──────────────┐      │
│  │ ParkingFloor │◄────────│   ParkingLot     │────────►│ EntryPanel   │      │
│  │              │         │   (Singleton)    │         │              │      │
│  └──────────────┘         └──────────────────┘         └──────────────┘      │
│         │                         │                            │              │
│         │                         │                            │              │
│         ▼                         ▼                            ▼              │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────┐      │
│  │ DisplayBoard │         │  PaymentService  │         │  ExitPanel   │      │
│  │              │         │                  │         │              │      │
│  └──────────────┘         └──────────────────┘         └──────────────┘      │
│                                   │                                           │
│                           ┌───────┴───────┐                                   │
│                           │               │                                   │
│                           ▼               ▼                                   │
│                    ┌────────────┐  ┌────────────┐                             │
│                    │   Cash     │  │   Card     │                             │
│                    │  Payment   │  │  Payment   │                             │
│                    └────────────┘  └────────────┘                             │
│                                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Relationships Summary

| Relationship | Type | Description |
|-------------|------|-------------|
| ParkingLot → ParkingFloor | Composition | Lot owns floors, floors cannot exist without lot |
| ParkingFloor → ParkingSpot | Composition | Floor owns spots |
| ParkingFloor → DisplayBoard | Composition | Each floor has a display board |
| ParkingLot → EntryPanel | Composition | Lot has entry panels |
| ParkingLot → ExitPanel | Composition | Lot has exit panels |
| ParkingTicket → Vehicle | Association | Ticket references vehicle |
| ParkingTicket → ParkingSpot | Association | Ticket references assigned spot |
| PaymentService → Payment | Aggregation | Service uses payment strategies |

---

## STEP 2: Complete Java Implementation

### 2.1 Enums (Define constants first)

```java
// VehicleType.java
package com.parkinglot.enums;

/**
 * Defines all supported vehicle types in the parking lot.
 * Each vehicle type determines which parking spots it can use.
 */
public enum VehicleType {
    MOTORCYCLE,  // Smallest, can fit in any spot
    CAR,         // Medium, needs compact or larger
    TRUCK        // Largest, needs large spots only
}
```

```java
// ParkingSpotType.java
package com.parkinglot.enums;

/**
 * Defines types of parking spots available.
 * Each type has different size and accessibility characteristics.
 */
public enum ParkingSpotType {
    COMPACT,      // For motorcycles and cars
    LARGE,        // For any vehicle including trucks
    HANDICAPPED   // Reserved spots with special access
}
```

```java
// TicketStatus.java
package com.parkinglot.enums;

/**
 * Tracks the lifecycle of a parking ticket.
 */
public enum TicketStatus {
    ACTIVE,     // Vehicle is currently parked
    PAID,       // Payment completed, ready to exit
    LOST        // Ticket lost, special handling required
}
```

```java
// PaymentStatus.java
package com.parkinglot.enums;

/**
 * Tracks payment transaction status.
 */
public enum PaymentStatus {
    PENDING,    // Payment not yet processed
    COMPLETED,  // Payment successful
    FAILED,     // Payment failed
    REFUNDED    // Payment was refunded
}
```

### 2.2 Vehicle Classes (What gets parked)

```java
// Vehicle.java
package com.parkinglot.models;

import com.parkinglot.enums.VehicleType;

/**
 * Abstract base class for all vehicles.
 * 
 * WHY ABSTRACT: We never create a generic "Vehicle", only specific types.
 * This enforces that every vehicle has a concrete type.
 * 
 * RESPONSIBILITY: Hold vehicle identification and type information.
 */
public abstract class Vehicle {
    
    private final String licensePlate;  // Unique identifier, immutable
    private final VehicleType type;     // Determined by subclass
    
    /**
     * Protected constructor - only subclasses can call this.
     * 
     * @param licensePlate Unique vehicle identifier (e.g., "ABC-1234")
     * @param type The type of vehicle (set by subclass)
     */
    protected Vehicle(String licensePlate, VehicleType type) {
        if (licensePlate == null || licensePlate.trim().isEmpty()) {
            throw new IllegalArgumentException("License plate cannot be null or empty");
        }
        this.licensePlate = licensePlate.toUpperCase().trim();
        this.type = type;
    }
    
    public String getLicensePlate() {
        return licensePlate;
    }
    
    public VehicleType getType() {
        return type;
    }
    
    /**
     * Determines if this vehicle can fit in a given spot type.
     * Each subclass defines its own fitting rules.
     * 
     * @param spotType The type of parking spot
     * @return true if vehicle can park in this spot type
     */
    public abstract boolean canFitInSpot(ParkingSpotType spotType);
    
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
    
    @Override
    public String toString() {
        return String.format("%s[licensePlate=%s]", getClass().getSimpleName(), licensePlate);
    }
}
```

```java
// Motorcycle.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;
import com.parkinglot.enums.VehicleType;

/**
 * Represents a motorcycle.
 * 
 * FITTING RULES: Motorcycles are small and can fit in ANY spot type.
 * In real parking lots, motorcycles often have dedicated areas,
 * but for flexibility, we allow them in any available spot.
 */
public class Motorcycle extends Vehicle {
    
    public Motorcycle(String licensePlate) {
        super(licensePlate, VehicleType.MOTORCYCLE);
    }
    
    @Override
    public boolean canFitInSpot(ParkingSpotType spotType) {
        // Motorcycles can fit anywhere
        return true;
    }
}
```

```java
// Car.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;
import com.parkinglot.enums.VehicleType;

/**
 * Represents a car (sedan, SUV, etc.).
 * 
 * FITTING RULES: Cars need at least a compact spot.
 * They can also use large spots if compact is unavailable.
 * Handicapped spots require special permit (handled separately).
 */
public class Car extends Vehicle {
    
    private final boolean hasHandicappedPermit;
    
    public Car(String licensePlate) {
        this(licensePlate, false);
    }
    
    public Car(String licensePlate, boolean hasHandicappedPermit) {
        super(licensePlate, VehicleType.CAR);
        this.hasHandicappedPermit = hasHandicappedPermit;
    }
    
    public boolean hasHandicappedPermit() {
        return hasHandicappedPermit;
    }
    
    @Override
    public boolean canFitInSpot(ParkingSpotType spotType) {
        switch (spotType) {
            case COMPACT:
            case LARGE:
                return true;
            case HANDICAPPED:
                return hasHandicappedPermit;
            default:
                return false;
        }
    }
}
```

```java
// Truck.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;
import com.parkinglot.enums.VehicleType;

/**
 * Represents a truck or large vehicle.
 * 
 * FITTING RULES: Trucks are large and need LARGE spots only.
 * They cannot fit in compact spots due to size constraints.
 */
public class Truck extends Vehicle {
    
    public Truck(String licensePlate) {
        super(licensePlate, VehicleType.TRUCK);
    }
    
    @Override
    public boolean canFitInSpot(ParkingSpotType spotType) {
        // Trucks only fit in large spots
        return spotType == ParkingSpotType.LARGE;
    }
}
```

### 2.3 Parking Spot Classes (Where vehicles park)

```java
// ParkingSpot.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;

/**
 * Abstract base class for all parking spots.
 * 
 * RESPONSIBILITY: Manage a single parking space's state.
 * Each spot knows its location, type, and whether it's occupied.
 * 
 * WHY ABSTRACT: Different spot types have different assignment rules.
 * The canFitVehicle() method is delegated to subclasses.
 */
public abstract class ParkingSpot {
    
    private final String spotId;           // Unique identifier (e.g., "F1-A001")
    private final ParkingSpotType type;    // Type of this spot
    private final int floorNumber;         // Which floor this spot is on
    
    private Vehicle parkedVehicle;         // Currently parked vehicle (null if empty)
    private boolean isAvailable;           // Quick availability check
    
    /**
     * Creates a new parking spot.
     * 
     * @param spotId Unique identifier for this spot
     * @param type The type of parking spot
     * @param floorNumber The floor this spot is located on
     */
    protected ParkingSpot(String spotId, ParkingSpotType type, int floorNumber) {
        this.spotId = spotId;
        this.type = type;
        this.floorNumber = floorNumber;
        this.isAvailable = true;
        this.parkedVehicle = null;
    }
    
    /**
     * Determines if a vehicle can be assigned to this spot.
     * Considers both spot availability and vehicle compatibility.
     * 
     * @param vehicle The vehicle to check
     * @return true if vehicle can park here
     */
    public boolean canFitVehicle(Vehicle vehicle) {
        if (!isAvailable) {
            return false;
        }
        return vehicle.canFitInSpot(this.type);
    }
    
    /**
     * Parks a vehicle in this spot.
     * 
     * @param vehicle The vehicle to park
     * @return true if parking successful, false if spot occupied or incompatible
     */
    public synchronized boolean parkVehicle(Vehicle vehicle) {
        if (!canFitVehicle(vehicle)) {
            return false;
        }
        this.parkedVehicle = vehicle;
        this.isAvailable = false;
        return true;
    }
    
    /**
     * Removes the vehicle from this spot.
     * 
     * @return The vehicle that was parked, or null if spot was empty
     */
    public synchronized Vehicle removeVehicle() {
        Vehicle vehicle = this.parkedVehicle;
        this.parkedVehicle = null;
        this.isAvailable = true;
        return vehicle;
    }
    
    // Getters
    public String getSpotId() { return spotId; }
    public ParkingSpotType getType() { return type; }
    public int getFloorNumber() { return floorNumber; }
    public Vehicle getParkedVehicle() { return parkedVehicle; }
    public boolean isAvailable() { return isAvailable; }
    
    @Override
    public String toString() {
        return String.format("ParkingSpot[id=%s, type=%s, floor=%d, available=%s]",
                spotId, type, floorNumber, isAvailable);
    }
}
```

```java
// CompactSpot.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;

/**
 * A compact parking spot for motorcycles and cars.
 * 
 * REAL-WORLD: Compact spots are typically 8ft wide.
 * They're cheaper to build (more spots per area) but
 * can't accommodate larger vehicles.
 */
public class CompactSpot extends ParkingSpot {
    
    public CompactSpot(String spotId, int floorNumber) {
        super(spotId, ParkingSpotType.COMPACT, floorNumber);
    }
}
```

```java
// LargeSpot.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;

/**
 * A large parking spot that can accommodate any vehicle.
 * 
 * REAL-WORLD: Large spots are typically 10-12ft wide.
 * They're more expensive (fewer spots per area) but
 * provide flexibility for all vehicle types.
 */
public class LargeSpot extends ParkingSpot {
    
    public LargeSpot(String spotId, int floorNumber) {
        super(spotId, ParkingSpotType.LARGE, floorNumber);
    }
}
```

```java
// HandicappedSpot.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;

/**
 * A handicapped-accessible parking spot.
 * 
 * REAL-WORLD: These spots are wider (for wheelchair access),
 * located near entrances, and legally reserved for permit holders.
 * 
 * DESIGN NOTE: Access control is handled at the vehicle level
 * (Car.hasHandicappedPermit), not at the spot level.
 */
public class HandicappedSpot extends ParkingSpot {
    
    public HandicappedSpot(String spotId, int floorNumber) {
        super(spotId, ParkingSpotType.HANDICAPPED, floorNumber);
    }
}
```

### 2.4 Parking Ticket (Tracks parking session)

```java
// ParkingTicket.java
package com.parkinglot.models;

import com.parkinglot.enums.TicketStatus;
import java.time.LocalDateTime;
import java.time.Duration;
import java.util.UUID;

/**
 * Represents a parking session from entry to exit.
 * 
 * RESPONSIBILITY: Track parking duration and calculate fees.
 * This is the "receipt" that connects vehicle, spot, and time.
 * 
 * IMMUTABLE FIELDS: ticketId, entryTime, vehicle, spot
 * MUTABLE FIELDS: exitTime, status, paymentAmount
 * 
 * WHY: Entry data is fixed once created, but exit data
 * and payment status change during the parking session.
 */
public class ParkingTicket {
    
    private final String ticketId;
    private final LocalDateTime entryTime;
    private final Vehicle vehicle;
    private final ParkingSpot parkingSpot;
    
    private LocalDateTime exitTime;
    private TicketStatus status;
    private double paymentAmount;
    
    // Hourly rates by spot type (could be externalized to config)
    private static final double COMPACT_HOURLY_RATE = 2.0;
    private static final double LARGE_HOURLY_RATE = 3.0;
    private static final double HANDICAPPED_HOURLY_RATE = 1.5;
    
    /**
     * Creates a new parking ticket when vehicle enters.
     * 
     * @param vehicle The vehicle being parked
     * @param parkingSpot The assigned parking spot
     */
    public ParkingTicket(Vehicle vehicle, ParkingSpot parkingSpot) {
        this.ticketId = generateTicketId();
        this.entryTime = LocalDateTime.now();
        this.vehicle = vehicle;
        this.parkingSpot = parkingSpot;
        this.status = TicketStatus.ACTIVE;
        this.paymentAmount = 0.0;
    }
    
    /**
     * Generates a unique ticket ID.
     * Format: TKT-{timestamp}-{random}
     * 
     * In production, this might use a distributed ID generator
     * like Snowflake for guaranteed uniqueness across servers.
     */
    private String generateTicketId() {
        return "TKT-" + System.currentTimeMillis() + "-" + 
               UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
    
    /**
     * Calculates the parking fee based on duration and spot type.
     * 
     * PRICING LOGIC:
     * - First hour: Full rate
     * - Subsequent hours: Prorated (rounded up)
     * - Different rates for different spot types
     * 
     * @return The calculated fee in dollars
     */
    public double calculateFee() {
        LocalDateTime endTime = (exitTime != null) ? exitTime : LocalDateTime.now();
        Duration duration = Duration.between(entryTime, endTime);
        
        // Calculate hours (minimum 1 hour, round up partial hours)
        long minutes = duration.toMinutes();
        long hours = (minutes + 59) / 60;  // Round up
        if (hours == 0) hours = 1;  // Minimum 1 hour charge
        
        // Get hourly rate based on spot type
        double hourlyRate = getHourlyRate();
        
        return hours * hourlyRate;
    }
    
    /**
     * Gets the hourly rate based on parking spot type.
     */
    private double getHourlyRate() {
        switch (parkingSpot.getType()) {
            case COMPACT:
                return COMPACT_HOURLY_RATE;
            case LARGE:
                return LARGE_HOURLY_RATE;
            case HANDICAPPED:
                return HANDICAPPED_HOURLY_RATE;
            default:
                return COMPACT_HOURLY_RATE;
        }
    }
    
    /**
     * Marks the ticket as paid and records exit time.
     * 
     * @param amount The amount paid
     */
    public void markAsPaid(double amount) {
        this.exitTime = LocalDateTime.now();
        this.paymentAmount = amount;
        this.status = TicketStatus.PAID;
    }
    
    /**
     * Marks ticket as lost (for special handling).
     * Lost tickets typically incur maximum daily charge.
     */
    public void markAsLost() {
        this.status = TicketStatus.LOST;
    }
    
    /**
     * Gets the duration of parking in a human-readable format.
     */
    public String getParkingDuration() {
        LocalDateTime endTime = (exitTime != null) ? exitTime : LocalDateTime.now();
        Duration duration = Duration.between(entryTime, endTime);
        
        long hours = duration.toHours();
        long minutes = duration.toMinutesPart();
        
        return String.format("%d hours, %d minutes", hours, minutes);
    }
    
    // Getters
    public String getTicketId() { return ticketId; }
    public LocalDateTime getEntryTime() { return entryTime; }
    public LocalDateTime getExitTime() { return exitTime; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSpot getParkingSpot() { return parkingSpot; }
    public TicketStatus getStatus() { return status; }
    public double getPaymentAmount() { return paymentAmount; }
    
    @Override
    public String toString() {
        return String.format(
            "ParkingTicket[id=%s, vehicle=%s, spot=%s, status=%s, duration=%s]",
            ticketId, vehicle.getLicensePlate(), parkingSpot.getSpotId(), 
            status, getParkingDuration()
        );
    }
}
```

### 2.5 Parking Floor (Manages spots on one floor)

```java
// ParkingFloor.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Represents a single floor in the parking lot.
 * 
 * RESPONSIBILITY: Manage all parking spots on this floor.
 * Provides spot allocation, availability tracking, and display updates.
 * 
 * THREAD SAFETY: Uses ConcurrentHashMap for spot storage
 * and synchronized methods for allocation to handle
 * concurrent entry requests.
 */
public class ParkingFloor {
    
    private final int floorNumber;
    private final String floorName;
    
    // Spots organized by type for efficient lookup
    private final Map<ParkingSpotType, List<ParkingSpot>> spotsByType;
    
    // Quick lookup by spot ID
    private final Map<String, ParkingSpot> spotsById;
    
    // Display board for this floor
    private final DisplayBoard displayBoard;
    
    /**
     * Creates a new parking floor.
     * 
     * @param floorNumber The floor number (0 = ground, 1 = first, etc.)
     * @param floorName Human-readable name (e.g., "Ground Floor", "Level 1")
     */
    public ParkingFloor(int floorNumber, String floorName) {
        this.floorNumber = floorNumber;
        this.floorName = floorName;
        this.spotsByType = new ConcurrentHashMap<>();
        this.spotsById = new ConcurrentHashMap<>();
        this.displayBoard = new DisplayBoard(floorNumber);
        
        // Initialize spot type lists
        for (ParkingSpotType type : ParkingSpotType.values()) {
            spotsByType.put(type, Collections.synchronizedList(new ArrayList<>()));
        }
    }
    
    /**
     * Adds a parking spot to this floor.
     * 
     * @param spot The spot to add
     */
    public void addParkingSpot(ParkingSpot spot) {
        spotsByType.get(spot.getType()).add(spot);
        spotsById.put(spot.getSpotId(), spot);
        updateDisplayBoard();
    }
    
    /**
     * Finds an available spot for the given vehicle.
     * 
     * ALLOCATION STRATEGY:
     * 1. Try to find a spot of the "ideal" type for the vehicle
     * 2. If not available, try larger spots
     * 3. Return null if no suitable spot found
     * 
     * This prevents small vehicles from taking large spots
     * when appropriate spots are available.
     * 
     * @param vehicle The vehicle to park
     * @return An available spot, or null if none found
     */
    public synchronized ParkingSpot findAvailableSpot(Vehicle vehicle) {
        // Define search order based on vehicle type
        List<ParkingSpotType> searchOrder = getSearchOrder(vehicle);
        
        for (ParkingSpotType spotType : searchOrder) {
            List<ParkingSpot> spots = spotsByType.get(spotType);
            for (ParkingSpot spot : spots) {
                if (spot.canFitVehicle(vehicle)) {
                    return spot;
                }
            }
        }
        
        return null;  // No suitable spot found
    }
    
    /**
     * Determines the order to search for spots based on vehicle type.
     * Smaller vehicles should prefer smaller spots first.
     */
    private List<ParkingSpotType> getSearchOrder(Vehicle vehicle) {
        List<ParkingSpotType> order = new ArrayList<>();
        
        switch (vehicle.getType()) {
            case MOTORCYCLE:
                // Motorcycles: try compact first, then large
                order.add(ParkingSpotType.COMPACT);
                order.add(ParkingSpotType.LARGE);
                break;
            case CAR:
                // Cars: try compact first, then large, then handicapped if permitted
                order.add(ParkingSpotType.COMPACT);
                order.add(ParkingSpotType.LARGE);
                if (vehicle instanceof Car && ((Car) vehicle).hasHandicappedPermit()) {
                    order.add(ParkingSpotType.HANDICAPPED);
                }
                break;
            case TRUCK:
                // Trucks: only large spots
                order.add(ParkingSpotType.LARGE);
                break;
        }
        
        return order;
    }
    
    /**
     * Gets the count of available spots by type.
     */
    public Map<ParkingSpotType, Integer> getAvailableSpotCounts() {
        Map<ParkingSpotType, Integer> counts = new EnumMap<>(ParkingSpotType.class);
        
        for (ParkingSpotType type : ParkingSpotType.values()) {
            int available = (int) spotsByType.get(type).stream()
                    .filter(ParkingSpot::isAvailable)
                    .count();
            counts.put(type, available);
        }
        
        return counts;
    }
    
    /**
     * Gets total spot count by type.
     */
    public Map<ParkingSpotType, Integer> getTotalSpotCounts() {
        Map<ParkingSpotType, Integer> counts = new EnumMap<>(ParkingSpotType.class);
        
        for (ParkingSpotType type : ParkingSpotType.values()) {
            counts.put(type, spotsByType.get(type).size());
        }
        
        return counts;
    }
    
    /**
     * Updates the display board with current availability.
     */
    public void updateDisplayBoard() {
        displayBoard.update(getAvailableSpotCounts());
    }
    
    /**
     * Checks if this floor has any available spots for the vehicle.
     */
    public boolean hasAvailableSpot(Vehicle vehicle) {
        return findAvailableSpot(vehicle) != null;
    }
    
    /**
     * Gets a spot by its ID.
     */
    public ParkingSpot getSpotById(String spotId) {
        return spotsById.get(spotId);
    }
    
    // Getters
    public int getFloorNumber() { return floorNumber; }
    public String getFloorName() { return floorName; }
    public DisplayBoard getDisplayBoard() { return displayBoard; }
    
    /**
     * Gets all spots on this floor.
     */
    public List<ParkingSpot> getAllSpots() {
        List<ParkingSpot> allSpots = new ArrayList<>();
        for (List<ParkingSpot> spots : spotsByType.values()) {
            allSpots.addAll(spots);
        }
        return Collections.unmodifiableList(allSpots);
    }
}
```

### 2.6 Display Board (Shows availability)

```java
// DisplayBoard.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;
import java.util.EnumMap;
import java.util.Map;

/**
 * Display board showing parking availability on a floor.
 * 
 * REAL-WORLD: These are the electronic signs you see at
 * parking garage entrances showing "COMPACT: 45, LARGE: 12"
 * 
 * RESPONSIBILITY: Display current availability counts.
 * This is a "dumb" display that just shows what it's told.
 */
public class DisplayBoard {
    
    private final int floorNumber;
    private final Map<ParkingSpotType, Integer> availableCounts;
    
    public DisplayBoard(int floorNumber) {
        this.floorNumber = floorNumber;
        this.availableCounts = new EnumMap<>(ParkingSpotType.class);
        
        // Initialize all counts to 0
        for (ParkingSpotType type : ParkingSpotType.values()) {
            availableCounts.put(type, 0);
        }
    }
    
    /**
     * Updates the display with new availability counts.
     * 
     * @param counts Map of spot type to available count
     */
    public void update(Map<ParkingSpotType, Integer> counts) {
        availableCounts.clear();
        availableCounts.putAll(counts);
    }
    
    /**
     * Renders the display board as a string (for console output).
     * In a real system, this would update an actual display.
     */
    public String render() {
        StringBuilder sb = new StringBuilder();
        sb.append("╔════════════════════════════════════╗\n");
        sb.append(String.format("║        FLOOR %d - AVAILABILITY       ║\n", floorNumber));
        sb.append("╠════════════════════════════════════╣\n");
        
        for (ParkingSpotType type : ParkingSpotType.values()) {
            int count = availableCounts.getOrDefault(type, 0);
            String status = count > 0 ? "AVAILABLE" : "FULL";
            sb.append(String.format("║  %-12s: %3d  [%s]%s║\n", 
                    type, count, status, status.equals("FULL") ? "      " : "  "));
        }
        
        sb.append("╚════════════════════════════════════╝");
        return sb.toString();
    }
    
    public Map<ParkingSpotType, Integer> getAvailableCounts() {
        return new EnumMap<>(availableCounts);
    }
    
    @Override
    public String toString() {
        return render();
    }
}
```

### 2.7 Entry and Exit Panels

```java
// EntryPanel.java
package com.parkinglot.models;

/**
 * Entry panel at parking lot entrance.
 * 
 * RESPONSIBILITY: Issue parking tickets to entering vehicles.
 * 
 * REAL-WORLD: This is the machine that:
 * 1. Scans your license plate (or you press a button)
 * 2. Prints a ticket
 * 3. Opens the gate
 */
public class EntryPanel {
    
    private final String panelId;
    private final ParkingLot parkingLot;
    
    public EntryPanel(String panelId, ParkingLot parkingLot) {
        this.panelId = panelId;
        this.parkingLot = parkingLot;
    }
    
    /**
     * Processes a vehicle entry.
     * 
     * @param vehicle The vehicle entering
     * @return A parking ticket if spot available, null otherwise
     */
    public ParkingTicket processEntry(Vehicle vehicle) {
        System.out.println("Entry Panel " + panelId + ": Processing entry for " + 
                          vehicle.getLicensePlate());
        
        ParkingTicket ticket = parkingLot.parkVehicle(vehicle);
        
        if (ticket != null) {
            System.out.println("Entry Panel " + panelId + ": Ticket issued - " + 
                              ticket.getTicketId());
            System.out.println("Entry Panel " + panelId + ": Assigned spot - " + 
                              ticket.getParkingSpot().getSpotId());
        } else {
            System.out.println("Entry Panel " + panelId + ": Parking lot is full!");
        }
        
        return ticket;
    }
    
    public String getPanelId() {
        return panelId;
    }
}
```

```java
// ExitPanel.java
package com.parkinglot.models;

import com.parkinglot.services.PaymentService;

/**
 * Exit panel at parking lot exit.
 * 
 * RESPONSIBILITY: Process payment and release vehicles.
 * 
 * REAL-WORLD: This is the machine that:
 * 1. Scans your ticket
 * 2. Calculates the fee
 * 3. Accepts payment
 * 4. Opens the exit gate
 */
public class ExitPanel {
    
    private final String panelId;
    private final ParkingLot parkingLot;
    private final PaymentService paymentService;
    
    public ExitPanel(String panelId, ParkingLot parkingLot, PaymentService paymentService) {
        this.panelId = panelId;
        this.parkingLot = parkingLot;
        this.paymentService = paymentService;
    }
    
    /**
     * Processes a vehicle exit.
     * 
     * @param ticket The parking ticket
     * @return true if exit successful, false otherwise
     */
    public boolean processExit(ParkingTicket ticket) {
        System.out.println("Exit Panel " + panelId + ": Processing exit for ticket " + 
                          ticket.getTicketId());
        
        // Calculate fee
        double fee = ticket.calculateFee();
        System.out.println("Exit Panel " + panelId + ": Parking fee is $" + 
                          String.format("%.2f", fee));
        System.out.println("Exit Panel " + panelId + ": Duration - " + 
                          ticket.getParkingDuration());
        
        // Process payment
        boolean paymentSuccess = paymentService.processPayment(ticket, fee);
        
        if (paymentSuccess) {
            // Release the parking spot
            parkingLot.releaseSpot(ticket);
            System.out.println("Exit Panel " + panelId + ": Exit successful. Have a nice day!");
            return true;
        } else {
            System.out.println("Exit Panel " + panelId + ": Payment failed. Please try again.");
            return false;
        }
    }
    
    /**
     * Handles lost ticket scenario.
     * Charges maximum daily rate.
     */
    public boolean processLostTicket(Vehicle vehicle) {
        System.out.println("Exit Panel " + panelId + ": Processing lost ticket for " + 
                          vehicle.getLicensePlate());
        
        ParkingTicket ticket = parkingLot.findTicketByVehicle(vehicle);
        
        if (ticket == null) {
            System.out.println("Exit Panel " + panelId + ": Vehicle not found in system!");
            return false;
        }
        
        ticket.markAsLost();
        
        // Lost ticket fee: maximum daily rate
        double lostTicketFee = 50.0;
        System.out.println("Exit Panel " + panelId + ": Lost ticket fee is $" + 
                          String.format("%.2f", lostTicketFee));
        
        boolean paymentSuccess = paymentService.processPayment(ticket, lostTicketFee);
        
        if (paymentSuccess) {
            parkingLot.releaseSpot(ticket);
            return true;
        }
        
        return false;
    }
    
    public String getPanelId() {
        return panelId;
    }
}
```

### 2.8 Payment System

```java
// Payment.java
package com.parkinglot.payment;

import com.parkinglot.enums.PaymentStatus;
import java.time.LocalDateTime;

/**
 * Abstract base class for payment methods.
 * 
 * WHY ABSTRACT: Different payment methods (cash, card, mobile)
 * have different processing logic. This allows adding new
 * payment methods without changing existing code.
 * 
 * PATTERN: Strategy Pattern - payment processing varies by type.
 */
public abstract class Payment {
    
    protected final double amount;
    protected final LocalDateTime timestamp;
    protected PaymentStatus status;
    
    protected Payment(double amount) {
        this.amount = amount;
        this.timestamp = LocalDateTime.now();
        this.status = PaymentStatus.PENDING;
    }
    
    /**
     * Processes the payment.
     * Each payment type implements its own processing logic.
     * 
     * @return true if payment successful
     */
    public abstract boolean processPayment();
    
    public double getAmount() { return amount; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public PaymentStatus getStatus() { return status; }
}
```

```java
// CashPayment.java
package com.parkinglot.payment;

import com.parkinglot.enums.PaymentStatus;

/**
 * Cash payment processing.
 * 
 * REAL-WORLD: Handles cash insertion, validation, and change dispensing.
 * For simplicity, we assume exact payment or overpayment with change.
 */
public class CashPayment extends Payment {
    
    private final double cashTendered;
    private double changeReturned;
    
    public CashPayment(double amount, double cashTendered) {
        super(amount);
        this.cashTendered = cashTendered;
        this.changeReturned = 0.0;
    }
    
    @Override
    public boolean processPayment() {
        if (cashTendered < amount) {
            System.out.println("Insufficient cash. Required: $" + amount + 
                              ", Tendered: $" + cashTendered);
            this.status = PaymentStatus.FAILED;
            return false;
        }
        
        this.changeReturned = cashTendered - amount;
        this.status = PaymentStatus.COMPLETED;
        
        if (changeReturned > 0) {
            System.out.println("Change returned: $" + String.format("%.2f", changeReturned));
        }
        
        return true;
    }
    
    public double getChangeReturned() { return changeReturned; }
}
```

```java
// CardPayment.java
package com.parkinglot.payment;

import com.parkinglot.enums.PaymentStatus;

/**
 * Credit/Debit card payment processing.
 * 
 * REAL-WORLD: Would integrate with payment gateway (Stripe, Square, etc.)
 * For this design, we simulate successful payment.
 */
public class CardPayment extends Payment {
    
    private final String cardNumber;  // Last 4 digits only for security
    private String transactionId;
    
    public CardPayment(double amount, String cardNumber) {
        super(amount);
        // Store only last 4 digits
        this.cardNumber = cardNumber.length() >= 4 ? 
                         "****" + cardNumber.substring(cardNumber.length() - 4) : 
                         "****";
    }
    
    @Override
    public boolean processPayment() {
        // Simulate payment gateway call
        System.out.println("Processing card payment for card: " + cardNumber);
        
        // In real implementation, this would:
        // 1. Connect to payment gateway
        // 2. Validate card
        // 3. Authorize transaction
        // 4. Capture payment
        
        // Simulate 95% success rate
        boolean success = Math.random() < 0.95;
        
        if (success) {
            this.transactionId = "TXN-" + System.currentTimeMillis();
            this.status = PaymentStatus.COMPLETED;
            System.out.println("Payment successful. Transaction ID: " + transactionId);
            return true;
        } else {
            this.status = PaymentStatus.FAILED;
            System.out.println("Card payment declined.");
            return false;
        }
    }
    
    public String getTransactionId() { return transactionId; }
}
```

```java
// PaymentService.java
package com.parkinglot.services;

import com.parkinglot.models.ParkingTicket;
import com.parkinglot.payment.CardPayment;
import com.parkinglot.payment.CashPayment;
import com.parkinglot.payment.Payment;

/**
 * Service for processing parking payments.
 * 
 * RESPONSIBILITY: Coordinate payment processing.
 * Acts as a facade over different payment methods.
 */
public class PaymentService {
    
    /**
     * Processes payment for a parking ticket.
     * Default to card payment for simulation.
     */
    public boolean processPayment(ParkingTicket ticket, double amount) {
        // For simulation, use card payment
        Payment payment = new CardPayment(amount, "1234567890123456");
        boolean success = payment.processPayment();
        
        if (success) {
            ticket.markAsPaid(amount);
        }
        
        return success;
    }
    
    /**
     * Processes cash payment.
     */
    public boolean processCashPayment(ParkingTicket ticket, double amount, double cashTendered) {
        Payment payment = new CashPayment(amount, cashTendered);
        boolean success = payment.processPayment();
        
        if (success) {
            ticket.markAsPaid(amount);
        }
        
        return success;
    }
    
    /**
     * Processes card payment.
     */
    public boolean processCardPayment(ParkingTicket ticket, double amount, String cardNumber) {
        Payment payment = new CardPayment(amount, cardNumber);
        boolean success = payment.processPayment();
        
        if (success) {
            ticket.markAsPaid(amount);
        }
        
        return success;
    }
}
```

### 2.9 Parking Lot (Main orchestrator)

```java
// ParkingLot.java
package com.parkinglot.models;

import com.parkinglot.enums.ParkingSpotType;
import com.parkinglot.services.PaymentService;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Main parking lot class that orchestrates all operations.
 * 
 * PATTERN: Singleton - Only one parking lot instance exists.
 * This ensures consistent state across all entry/exit panels.
 * 
 * RESPONSIBILITY: 
 * - Manage floors and their spots
 * - Track active tickets
 * - Coordinate entry and exit operations
 */
public class ParkingLot {
    
    // Singleton instance
    private static ParkingLot instance;
    
    private final String name;
    private final String address;
    
    // Floors in the parking lot
    private final List<ParkingFloor> floors;
    
    // Active tickets (ticketId -> ticket)
    private final Map<String, ParkingTicket> activeTickets;
    
    // Vehicle to ticket mapping for quick lookup
    private final Map<String, ParkingTicket> vehicleToTicket;
    
    // Entry and exit panels
    private final List<EntryPanel> entryPanels;
    private final List<ExitPanel> exitPanels;
    
    // Payment service
    private final PaymentService paymentService;
    
    /**
     * Private constructor for Singleton pattern.
     */
    private ParkingLot(String name, String address) {
        this.name = name;
        this.address = address;
        this.floors = Collections.synchronizedList(new ArrayList<>());
        this.activeTickets = new ConcurrentHashMap<>();
        this.vehicleToTicket = new ConcurrentHashMap<>();
        this.entryPanels = new ArrayList<>();
        this.exitPanels = new ArrayList<>();
        this.paymentService = new PaymentService();
    }
    
    /**
     * Gets the singleton instance, creating it if necessary.
     * 
     * THREAD SAFETY: Double-checked locking pattern.
     */
    public static synchronized ParkingLot getInstance(String name, String address) {
        if (instance == null) {
            instance = new ParkingLot(name, address);
        }
        return instance;
    }
    
    /**
     * Gets existing instance (must be initialized first).
     */
    public static ParkingLot getInstance() {
        if (instance == null) {
            throw new IllegalStateException("ParkingLot not initialized. Call getInstance(name, address) first.");
        }
        return instance;
    }
    
    /**
     * Resets the singleton (for testing purposes).
     */
    public static void resetInstance() {
        instance = null;
    }
    
    // ==================== Configuration Methods ====================
    
    /**
     * Adds a floor to the parking lot.
     */
    public void addFloor(ParkingFloor floor) {
        floors.add(floor);
        System.out.println("Added floor: " + floor.getFloorName());
    }
    
    /**
     * Adds an entry panel.
     */
    public EntryPanel addEntryPanel(String panelId) {
        EntryPanel panel = new EntryPanel(panelId, this);
        entryPanels.add(panel);
        return panel;
    }
    
    /**
     * Adds an exit panel.
     */
    public ExitPanel addExitPanel(String panelId) {
        ExitPanel panel = new ExitPanel(panelId, this, paymentService);
        exitPanels.add(panel);
        return panel;
    }
    
    // ==================== Core Operations ====================
    
    /**
     * Parks a vehicle and issues a ticket.
     * 
     * ALGORITHM:
     * 1. Check if vehicle is already parked
     * 2. Find available spot across all floors
     * 3. Park vehicle in spot
     * 4. Create and return ticket
     * 
     * @param vehicle The vehicle to park
     * @return Parking ticket, or null if no spot available
     */
    public synchronized ParkingTicket parkVehicle(Vehicle vehicle) {
        // Check if already parked
        if (vehicleToTicket.containsKey(vehicle.getLicensePlate())) {
            System.out.println("Vehicle " + vehicle.getLicensePlate() + " is already parked!");
            return null;
        }
        
        // Find available spot
        ParkingSpot spot = findAvailableSpot(vehicle);
        
        if (spot == null) {
            System.out.println("No available spot for " + vehicle.getType());
            return null;
        }
        
        // Park the vehicle
        spot.parkVehicle(vehicle);
        
        // Create ticket
        ParkingTicket ticket = new ParkingTicket(vehicle, spot);
        activeTickets.put(ticket.getTicketId(), ticket);
        vehicleToTicket.put(vehicle.getLicensePlate(), ticket);
        
        // Update display boards
        updateAllDisplayBoards();
        
        return ticket;
    }
    
    /**
     * Finds an available spot for the vehicle.
     * Searches floors from bottom to top.
     */
    private ParkingSpot findAvailableSpot(Vehicle vehicle) {
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.findAvailableSpot(vehicle);
            if (spot != null) {
                return spot;
            }
        }
        return null;
    }
    
    /**
     * Releases a parking spot when vehicle exits.
     * 
     * @param ticket The parking ticket
     */
    public synchronized void releaseSpot(ParkingTicket ticket) {
        ParkingSpot spot = ticket.getParkingSpot();
        spot.removeVehicle();
        
        activeTickets.remove(ticket.getTicketId());
        vehicleToTicket.remove(ticket.getVehicle().getLicensePlate());
        
        updateAllDisplayBoards();
        
        System.out.println("Spot " + spot.getSpotId() + " is now available.");
    }
    
    /**
     * Finds a ticket by vehicle license plate.
     */
    public ParkingTicket findTicketByVehicle(Vehicle vehicle) {
        return vehicleToTicket.get(vehicle.getLicensePlate());
    }
    
    /**
     * Gets a ticket by ticket ID.
     */
    public ParkingTicket getTicket(String ticketId) {
        return activeTickets.get(ticketId);
    }
    
    // ==================== Status Methods ====================
    
    /**
     * Updates all floor display boards.
     */
    private void updateAllDisplayBoards() {
        for (ParkingFloor floor : floors) {
            floor.updateDisplayBoard();
        }
    }
    
    /**
     * Gets total availability across all floors.
     */
    public Map<ParkingSpotType, Integer> getTotalAvailability() {
        Map<ParkingSpotType, Integer> total = new EnumMap<>(ParkingSpotType.class);
        
        for (ParkingSpotType type : ParkingSpotType.values()) {
            total.put(type, 0);
        }
        
        for (ParkingFloor floor : floors) {
            Map<ParkingSpotType, Integer> floorCounts = floor.getAvailableSpotCounts();
            for (ParkingSpotType type : ParkingSpotType.values()) {
                total.merge(type, floorCounts.get(type), Integer::sum);
            }
        }
        
        return total;
    }
    
    /**
     * Checks if parking lot is full for a vehicle type.
     */
    public boolean isFull(Vehicle vehicle) {
        return findAvailableSpot(vehicle) == null;
    }
    
    /**
     * Displays status of entire parking lot.
     */
    public void displayStatus() {
        System.out.println("\n" + "=".repeat(50));
        System.out.println("PARKING LOT STATUS: " + name);
        System.out.println("Address: " + address);
        System.out.println("=".repeat(50));
        
        System.out.println("\nActive Vehicles: " + activeTickets.size());
        
        System.out.println("\nAvailability by Floor:");
        for (ParkingFloor floor : floors) {
            System.out.println("\n" + floor.getDisplayBoard().render());
        }
        
        System.out.println("\nTotal Availability:");
        Map<ParkingSpotType, Integer> total = getTotalAvailability();
        for (ParkingSpotType type : ParkingSpotType.values()) {
            System.out.println("  " + type + ": " + total.get(type));
        }
        
        System.out.println("=".repeat(50) + "\n");
    }
    
    // Getters
    public String getName() { return name; }
    public String getAddress() { return address; }
    public List<ParkingFloor> getFloors() { return Collections.unmodifiableList(floors); }
    public int getActiveVehicleCount() { return activeTickets.size(); }
    public List<EntryPanel> getEntryPanels() { return Collections.unmodifiableList(entryPanels); }
    public List<ExitPanel> getExitPanels() { return Collections.unmodifiableList(exitPanels); }
}
```

### 2.10 Main Application (Demo)

```java
// ParkingLotDemo.java
package com.parkinglot;

import com.parkinglot.models.*;
import com.parkinglot.enums.ParkingSpotType;

/**
 * Demonstration of the Parking Lot System.
 * 
 * This simulates a real parking lot with:
 * - 2 floors
 * - Multiple spot types
 * - Vehicle entry and exit
 * - Payment processing
 */
public class ParkingLotDemo {
    
    public static void main(String[] args) {
        // Reset any existing instance (for clean demo)
        ParkingLot.resetInstance();
        
        // ==================== Setup ====================
        System.out.println("Setting up parking lot...\n");
        
        // Create parking lot
        ParkingLot parkingLot = ParkingLot.getInstance(
            "Downtown Parking", 
            "123 Main Street"
        );
        
        // Create floors
        ParkingFloor floor0 = new ParkingFloor(0, "Ground Floor");
        ParkingFloor floor1 = new ParkingFloor(1, "Level 1");
        
        // Add spots to ground floor
        // Compact spots: C001-C005
        for (int i = 1; i <= 5; i++) {
            floor0.addParkingSpot(new CompactSpot(
                String.format("F0-C%03d", i), 0));
        }
        // Large spots: L001-L003
        for (int i = 1; i <= 3; i++) {
            floor0.addParkingSpot(new LargeSpot(
                String.format("F0-L%03d", i), 0));
        }
        // Handicapped spots: H001-H002
        for (int i = 1; i <= 2; i++) {
            floor0.addParkingSpot(new HandicappedSpot(
                String.format("F0-H%03d", i), 0));
        }
        
        // Add spots to level 1
        for (int i = 1; i <= 5; i++) {
            floor1.addParkingSpot(new CompactSpot(
                String.format("F1-C%03d", i), 1));
        }
        for (int i = 1; i <= 3; i++) {
            floor1.addParkingSpot(new LargeSpot(
                String.format("F1-L%03d", i), 1));
        }
        
        // Add floors to parking lot
        parkingLot.addFloor(floor0);
        parkingLot.addFloor(floor1);
        
        // Create entry and exit panels
        EntryPanel entry1 = parkingLot.addEntryPanel("ENTRY-1");
        EntryPanel entry2 = parkingLot.addEntryPanel("ENTRY-2");
        ExitPanel exit1 = parkingLot.addExitPanel("EXIT-1");
        
        // Display initial status
        parkingLot.displayStatus();
        
        // ==================== Scenario 1: Cars Entering ====================
        System.out.println("\n===== SCENARIO 1: Cars Entering =====\n");
        
        Car car1 = new Car("ABC-1234");
        Car car2 = new Car("XYZ-5678");
        Car handicappedCar = new Car("HND-0001", true);
        
        ParkingTicket ticket1 = entry1.processEntry(car1);
        System.out.println();
        ParkingTicket ticket2 = entry1.processEntry(car2);
        System.out.println();
        ParkingTicket ticket3 = entry2.processEntry(handicappedCar);
        
        parkingLot.displayStatus();
        
        // ==================== Scenario 2: Truck Entering ====================
        System.out.println("\n===== SCENARIO 2: Truck Entering =====\n");
        
        Truck truck1 = new Truck("TRK-9999");
        ParkingTicket ticket4 = entry1.processEntry(truck1);
        
        parkingLot.displayStatus();
        
        // ==================== Scenario 3: Motorcycle Entering ====================
        System.out.println("\n===== SCENARIO 3: Motorcycle Entering =====\n");
        
        Motorcycle bike1 = new Motorcycle("BIKE-001");
        ParkingTicket ticket5 = entry2.processEntry(bike1);
        
        // ==================== Scenario 4: Vehicle Exiting ====================
        System.out.println("\n===== SCENARIO 4: Vehicle Exiting =====\n");
        
        // Simulate some time passing (for fee calculation demo)
        // In real scenario, hours would pass
        
        if (ticket1 != null) {
            System.out.println("Car 1 exiting...");
            exit1.processExit(ticket1);
        }
        
        parkingLot.displayStatus();
        
        // ==================== Scenario 5: Duplicate Entry Attempt ====================
        System.out.println("\n===== SCENARIO 5: Duplicate Entry Attempt =====\n");
        
        // Try to park the same car again (should fail)
        ParkingTicket duplicateTicket = entry1.processEntry(car2);
        System.out.println("Duplicate entry result: " + 
            (duplicateTicket == null ? "Rejected (as expected)" : "ERROR - should have been rejected"));
        
        // ==================== Scenario 6: Lost Ticket ====================
        System.out.println("\n===== SCENARIO 6: Lost Ticket Handling =====\n");
        
        // Simulate lost ticket for truck
        if (ticket4 != null) {
            System.out.println("Truck driver lost ticket...");
            exit1.processLostTicket(truck1);
        }
        
        // Final status
        System.out.println("\n===== FINAL STATUS =====");
        parkingLot.displayStatus();
    }
}
```

---

## STEP 5: Simulation / Dry Run

### Scenario A: Normal Parking Flow

```
Initial State:
- Floor 0: 5 compact, 3 large, 2 handicapped (all empty)
- Floor 1: 5 compact, 3 large (all empty)

Step 1: Car "ABC-1234" enters via Entry Panel 1
- findAvailableSpot() searches Floor 0 first
- Search order for CAR: COMPACT → LARGE → HANDICAPPED
- Finds F0-C001 (first compact spot on floor 0)
- parkVehicle() called on F0-C001
- Ticket TKT-xxx-yyy created
- vehicleToTicket["ABC-1234"] = ticket
- activeTickets["TKT-xxx-yyy"] = ticket
- Display board updates: Floor 0 COMPACT: 4

Step 2: Truck "TRK-9999" enters via Entry Panel 1
- Search order for TRUCK: LARGE only
- Finds F0-L001 (first large spot on floor 0)
- Ticket created
- Display board updates: Floor 0 LARGE: 2

Step 3: Car "ABC-1234" tries to enter again
- vehicleToTicket.containsKey("ABC-1234") returns true
- Returns null (entry denied)

Step 4: Car "ABC-1234" exits via Exit Panel 1
- calculateFee() called: 1 hour × $2.00 = $2.00
- Payment processed
- ticket.markAsPaid($2.00)
- releaseSpot() called
- F0-C001.removeVehicle()
- Display board updates: Floor 0 COMPACT: 5
- vehicleToTicket.remove("ABC-1234")
- activeTickets.remove(ticketId)
```

### Scenario B: Handicapped Parking

```
Step 1: Car with permit "HND-0001" enters
- hasHandicappedPermit = true
- Search order: COMPACT → LARGE → HANDICAPPED
- If compact available, uses compact (saves handicapped for those who need it)
- If only handicapped available, can use it

Step 2: Car without permit tries handicapped spot
- canFitInSpot(HANDICAPPED) returns false
- Spot skipped in search
```

### Scenario C: Parking Lot Full

```
State: All spots occupied

Step 1: New car enters
- findAvailableSpot() searches all floors
- All spots return isAvailable = false
- Returns null
- Entry panel displays "Parking lot is full!"
```

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

---

## File Structure

```
com/parkinglot/
├── enums/
│   ├── VehicleType.java
│   ├── ParkingSpotType.java
│   ├── TicketStatus.java
│   └── PaymentStatus.java
├── models/
│   ├── Vehicle.java
│   ├── Motorcycle.java
│   ├── Car.java
│   ├── Truck.java
│   ├── ParkingSpot.java
│   ├── CompactSpot.java
│   ├── LargeSpot.java
│   ├── HandicappedSpot.java
│   ├── ParkingTicket.java
│   ├── ParkingFloor.java
│   ├── DisplayBoard.java
│   ├── EntryPanel.java
│   ├── ExitPanel.java
│   └── ParkingLot.java
├── payment/
│   ├── Payment.java
│   ├── CashPayment.java
│   └── CardPayment.java
├── services/
│   └── PaymentService.java
└── ParkingLotDemo.java
```

