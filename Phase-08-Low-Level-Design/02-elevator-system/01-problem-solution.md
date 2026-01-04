# 🛗 Elevator System - Problem Solution

## STEP 0: REQUIREMENTS QUICKPASS

### Core Functional Requirements

- Support multiple elevators in a building
- Handle requests from multiple floors (external requests from floor buttons)
- Handle internal requests (destination buttons inside elevator)
- Implement scheduling algorithms (SCAN, LOOK, FCFS)
- Manage up/down states and direction changes
- Enforce weight limits per elevator
- Handle emergency situations (emergency stop)
- Support maintenance mode for individual elevators
- Provide real-time status display of all elevators

### Explicit Out-of-Scope Items

- Destination dispatch (entering destination before boarding)
- VIP/priority floors and passengers
- Elevator grouping by zones (low-rise, high-rise)
- Energy optimization and regenerative braking
- Integration with building security systems
- Fire service mode
- Accessibility features (audio announcements, braille)
- Real-time analytics and reporting dashboard
- Physical hardware integration (sensors, motors)

### Assumptions and Constraints

- **Single Building**: The system manages elevators within one building
- **Fixed Floor Configuration**: Building floors are predetermined and don't change
- **Uniform Elevator Capacity**: All elevators have the same weight limit
- **Simple Request Model**: Users press UP/DOWN on floors, destination inside elevator
- **No Passenger Tracking**: System doesn't track individual passengers
- **Simulated Time**: Door operations and floor travel have simulated delays
- **No External Database**: All state is managed in-memory
- **No UI/Frontend**: System interaction is via method calls and console output

### Scale Assumptions

- **Single Process**: The entire system runs within a single JVM
- **Multi-threaded**: Each elevator runs in its own thread
- **Moderate Scale**: Up to ~10 elevators, ~50 floors, ~100 concurrent requests

### Concurrency Model Expectations

- **Thread-per-Elevator**: Each elevator has a dedicated thread for movement
- **Central Dispatcher**: Single dispatcher thread assigns requests to elevators
- **Thread-Safe Collections**: `ConcurrentLinkedQueue` for requests, `ConcurrentSkipListSet` for stops
- **Synchronized Methods**: Critical state changes are synchronized
- **Volatile Flags**: `running` flag is volatile for visibility across threads

### Public APIs (Main methods exposed for interaction)

- `Building.getInstance(name, floors, elevators, minFloor, maxWeight)`: Initialize building
- `Building.start()`: Start the elevator system
- `Building.stop()`: Stop the elevator system
- `ElevatorController.requestElevator(ExternalRequest)`: Handle floor button press
- `ElevatorController.requestFloor(elevatorId, floor)`: Handle destination button press
- `ElevatorController.setScheduler(SchedulingStrategy)`: Change scheduling algorithm
- `ElevatorController.emergencyStopAll()`: Emergency stop all elevators
- `ElevatorController.displayStatus()`: Show current state of all elevators
- `Elevator.setMaintenance(boolean)`: Put elevator in/out of maintenance

### Public API Usage Examples

```java
// Example 1: Basic usage
Building building = Building.getInstance("Tech Tower", 10, 3, 0, 1000);
building.start();
ElevatorController controller = building.getController();

// Request elevator from floor 5 going DOWN
Floor floor5 = building.getFloor(5);
ExternalRequest request = floor5.pressDownButton();
controller.requestElevator(request);

// Example 2: Typical workflow
// Person on floor 2 wants to go UP
controller.requestElevator(building.getFloor(2).pressUpButton());

// Person inside Elevator 1 presses floor 7
controller.requestFloor(1, 7);

// Check status
controller.displayStatus();

// Example 3: Edge case - Invalid floor request
controller.requestFloor(1, 15);  // Floor 15 doesn't exist (max is 9)
// System handles gracefully: "Elevator 1: Invalid floor 15"

// Example 4: Change scheduling algorithm
controller.setScheduler(new SCANScheduler());

// Example 5: Emergency stop
controller.emergencyStopAll();

// Example 6: Maintenance mode
Elevator elevator = controller.getElevator(1);
elevator.setMaintenance(true);  // Take elevator out of service
elevator.setMaintenance(false);  // Return to service
```

### Invariants the System Must Always Maintain

- **Door Safety**: Elevator cannot move while doors are open
- **Direction Consistency**: Elevator serves all requests in current direction before reversing
- **Weight Limit**: Elevator rejects passengers when weight limit exceeded
- **Floor Bounds**: Elevator cannot move beyond min/max floor
- **State Validity**: Elevator state transitions follow valid paths (no MOVING → EMERGENCY without stop)
- **Request Assignment**: Each external request is assigned to exactly one elevator
- **No Duplicate Stops**: Same floor cannot appear twice in an elevator's stop list

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              ELEVATOR SYSTEM                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐ │
│  │     Building     │────────►│ ElevatorController│────────►│     Elevator     │ │
│  │   (Singleton)    │         │    (Scheduler)    │         │                  │ │
│  └──────────────────┘         └────────┬─────────┘         └────────┬─────────┘ │
│          │                             │                            │           │
│          │                    ┌────────┴────────┐                   │           │
│          │                    │                 │                   │           │
│          ▼                    ▼                 ▼                   ▼           │
│  ┌──────────────┐    ┌──────────────┐  ┌──────────────┐    ┌──────────────┐    │
│  │    Floor     │    │ SCANScheduler│  │ LOOKScheduler│    │   ElevatorCar│    │
│  │              │    │              │  │              │    │              │    │
│  └──────────────┘    └──────────────┘  └──────────────┘    └──────────────┘    │
│          │                                                         │           │
│          │                                                         │           │
│          ▼                                                         ▼           │
│  ┌──────────────┐                                          ┌──────────────┐    │
│  │ FloorButton  │                                          │ ElevatorPanel│    │
│  │ (Up/Down)    │                                          │ (Inside Car) │    │
│  └──────────────┘                                          └──────────────┘    │
│                                                                    │           │
│  ┌──────────────┐         ┌──────────────────┐                    │           │
│  │   Request    │◄────────│      Door        │◄───────────────────┘           │
│  │              │         │                  │                                 │
│  └──────────────┘         └──────────────────┘                                 │
│         │                                                                       │
│    ┌────┴────┐                                                                  │
│    │         │                                                                  │
│    ▼         ▼                                                                  │
│ ┌────────┐ ┌────────┐                                                          │
│ │External│ │Internal│                                                          │
│ │Request │ │Request │                                                          │
│ └────────┘ └────────┘                                                          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Relationships Summary

| Relationship                            | Type        | Description                  |
| --------------------------------------- | ----------- | ---------------------------- |
| Building → Floor                        | Composition | Building owns floors         |
| Building → ElevatorController           | Composition | Building owns controller     |
| ElevatorController → Elevator           | Aggregation | Controller manages elevators |
| ElevatorController → SchedulingStrategy | Association | Uses strategy for scheduling |
| Elevator → ElevatorCar                  | Composition | Elevator owns its car        |
| Elevator → Door                         | Composition | Elevator owns its door       |
| Floor → FloorButton                     | Composition | Each floor has buttons       |
| Request → Direction                     | Association | Request has a direction      |

---

### Responsibilities Table

| Class                            | Owns                                                    | Why                                                                                                           |
| -------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `Building`                       | Building structure and elevator system initialization   | Central singleton entry point - owns all floors and controller, ensures single system instance                |
| `ElevatorController`             | Request routing and elevator coordination               | Central dispatcher coordinates all elevators - separates scheduling logic from elevator mechanics             |
| `Elevator`                       | Single elevator's state, movement, and stop management  | Encapsulates individual elevator behavior - manages its own position, direction, stops, and state transitions |
| `Door`                           | Door open/close operations and state                    | Separates door mechanics from elevator logic - enables door safety rules and state management                 |
| `Request` (abstract)             | Request representation and comparison                   | Base class for all request types - enables polymorphism and consistent request handling                       |
| `ExternalRequest`                | Floor button requests (UP/DOWN)                         | Represents external requests from floor buttons - separate from internal destination requests                 |
| `InternalRequest`                | Destination button requests from inside elevator        | Represents internal requests from elevator panels - knows source elevator for routing                         |
| `Floor`                          | Floor representation and button states                  | Encapsulates floor-level behavior - manages up/down button states and floor identity                          |
| `SchedulingStrategy` (interface) | Algorithm for selecting which elevator serves a request | Strategy pattern enables pluggable algorithms - SCAN, LOOK, FCFS can be swapped without code changes          |
| `FCFSScheduler`                  | First-come-first-served scheduling algorithm            | Simplest scheduling algorithm - serves as baseline implementation                                             |
| `SCANScheduler`                  | SCAN (elevator) scheduling algorithm                    | Handles requests in one direction then reverses - efficient for sequential requests                           |
| `LOOKScheduler`                  | LOOK scheduling algorithm variant                       | Similar to SCAN but doesn't go to extremes if no requests - more efficient than SCAN                          |

---

## STEP 2: Complete Final Implementation

> **Verified:** This code compiles successfully with Java 11+.

### 2.1 Enums

```java
// Direction.java
package com.elevator.enums;

/**
 * Represents the direction of elevator movement or request.
 */
public enum Direction {
    UP,
    DOWN,
    IDLE  // Elevator is stationary with no pending requests
}
```

```java
// ElevatorState.java
package com.elevator.enums;

/**
 * Represents the current operational state of an elevator.
 */
public enum ElevatorState {
    MOVING_UP,      // Elevator is moving upward
    MOVING_DOWN,    // Elevator is moving downward
    IDLE,           // Elevator is stationary, no requests
    STOPPED,        // Elevator stopped at a floor (doors may be open)
    MAINTENANCE,    // Elevator is out of service
    EMERGENCY       // Emergency stop activated
}
```

```java
// DoorState.java
package com.elevator.enums;

/**
 * Represents the state of elevator doors.
 */
public enum DoorState {
    OPEN,
    CLOSED,
    OPENING,
    CLOSING
}
```

### 2.2 Request Classes

```java
// Request.java
package com.elevator.models;

import com.elevator.enums.Direction;
import java.time.LocalDateTime;

/**
 * Base class for elevator requests.
 *
 * WHY: Requests can come from outside (floor buttons) or
 * inside (elevator panel). Both need to be tracked.
 */
public abstract class Request implements Comparable<Request> {

    protected final int floor;
    protected final LocalDateTime timestamp;
    protected boolean served;

    protected Request(int floor) {
        this.floor = floor;
        this.timestamp = LocalDateTime.now();
        this.served = false;
    }

    public int getFloor() { return floor; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public boolean isServed() { return served; }
    public void markServed() { this.served = true; }

    /**
     * Requests are compared by floor for scheduling.
     */
    @Override
    public int compareTo(Request other) {
        return Integer.compare(this.floor, other.floor);
    }

    public abstract Direction getDirection();
}
```

```java
// ExternalRequest.java
package com.elevator.models;

import com.elevator.enums.Direction;

/**
 * Request from a floor button (outside the elevator).
 *
 * User presses UP or DOWN button on a floor.
 * The elevator needs to come to this floor, then
 * the user will press their destination inside.
 */
public class ExternalRequest extends Request {

    private final Direction direction;

    public ExternalRequest(int floor, Direction direction) {
        super(floor);
        if (direction == Direction.IDLE) {
            throw new IllegalArgumentException(
                "External request must have UP or DOWN direction");
        }
        this.direction = direction;
    }

    @Override
    public Direction getDirection() {
        return direction;
    }

    @Override
    public String toString() {
        return String.format("ExternalRequest[floor=%d, direction=%s]",
                            floor, direction);
    }
}
```

```java
// InternalRequest.java
package com.elevator.models;

import com.elevator.enums.Direction;

/**
 * Request from inside the elevator (destination floor).
 *
 * User inside the elevator presses a floor button.
 * Direction is determined by comparing current floor
 * to requested floor.
 */
public class InternalRequest extends Request {

    private final int sourceFloor;  // Where the request was made

    public InternalRequest(int destinationFloor, int sourceFloor) {
        super(destinationFloor);
        this.sourceFloor = sourceFloor;
    }

    @Override
    public Direction getDirection() {
        if (floor > sourceFloor) {
            return Direction.UP;
        } else if (floor < sourceFloor) {
            return Direction.DOWN;
        }
        return Direction.IDLE;
    }

    public int getSourceFloor() {
        return sourceFloor;
    }

    @Override
    public String toString() {
        return String.format("InternalRequest[from=%d, to=%d]",
                            sourceFloor, floor);
    }
}
```

### 2.3 Door Class

```java
// Door.java
package com.elevator.models;

import com.elevator.enums.DoorState;

/**
 * Represents the elevator door.
 *
 * RESPONSIBILITY: Manage door state and transitions.
 * Door operations take time (simulated with delays).
 */
public class Door {

    private DoorState state;
    private static final long DOOR_OPERATION_TIME_MS = 2000;  // 2 seconds

    public Door() {
        this.state = DoorState.CLOSED;
    }

    /**
     * Opens the door.
     * In real system, this would be async with sensors.
     */
    public synchronized void open() {
        if (state == DoorState.OPEN || state == DoorState.OPENING) {
            return;  // Already open or opening
        }

        state = DoorState.OPENING;
        System.out.println("  Door: Opening...");

        // Simulate door opening time
        simulateDelay(DOOR_OPERATION_TIME_MS);

        state = DoorState.OPEN;
        System.out.println("  Door: Open");
    }

    /**
     * Closes the door.
     */
    public synchronized void close() {
        if (state == DoorState.CLOSED || state == DoorState.CLOSING) {
            return;  // Already closed or closing
        }

        state = DoorState.CLOSING;
        System.out.println("  Door: Closing...");

        // Simulate door closing time
        simulateDelay(DOOR_OPERATION_TIME_MS);

        state = DoorState.CLOSED;
        System.out.println("  Door: Closed");
    }

    /**
     * Emergency open (immediate).
     */
    public synchronized void emergencyOpen() {
        state = DoorState.OPEN;
        System.out.println("  Door: EMERGENCY OPEN");
    }

    public DoorState getState() {
        return state;
    }

    public boolean isOpen() {
        return state == DoorState.OPEN;
    }

    public boolean isClosed() {
        return state == DoorState.CLOSED;
    }

    private void simulateDelay(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 2.4 Elevator Class

```java
// Elevator.java
package com.elevator.models;

import com.elevator.enums.Direction;
import com.elevator.enums.ElevatorState;
import java.util.*;
import java.util.concurrent.ConcurrentSkipListSet;

/**
 * Represents a single elevator in the building.
 *
 * RESPONSIBILITY:
 * - Track current floor and direction
 * - Manage internal requests (destination buttons)
 * - Control door operations
 * - Enforce weight limits
 *
 * THREAD SAFETY: Uses concurrent collections and
 * synchronized methods for safe multi-threaded access.
 */
public class Elevator {

    private final int id;
    private final int minFloor;
    private final int maxFloor;
    private final int maxWeightKg;
    private final Door door;

    private int currentFloor;
    private Direction direction;
    private ElevatorState state;
    private int currentWeightKg;

    // Destinations requested from inside the elevator
    // TreeSet keeps them sorted for efficient SCAN
    private final NavigableSet<Integer> upStops;
    private final NavigableSet<Integer> downStops;

    // Time to move one floor (milliseconds)
    private static final long FLOOR_TRAVEL_TIME_MS = 1000;

    // Time doors stay open at a stop
    private static final long DOOR_OPEN_DURATION_MS = 3000;

    public Elevator(int id, int minFloor, int maxFloor, int maxWeightKg) {
        this.id = id;
        this.minFloor = minFloor;
        this.maxFloor = maxFloor;
        this.maxWeightKg = maxWeightKg;
        this.door = new Door();

        this.currentFloor = minFloor;  // Start at ground floor
        this.direction = Direction.IDLE;
        this.state = ElevatorState.IDLE;
        this.currentWeightKg = 0;

        // Concurrent sorted sets for thread-safe sorted access
        this.upStops = new ConcurrentSkipListSet<>();
        this.downStops = new ConcurrentSkipListSet<>(Collections.reverseOrder());
    }

    /**
     * Adds a destination floor request.
     * Called when someone inside presses a floor button.
     */
    public synchronized void addDestination(int floor) {
        if (floor < minFloor || floor > maxFloor) {
            System.out.println("Elevator " + id + ": Invalid floor " + floor);
            return;
        }

        if (floor == currentFloor) {
            System.out.println("Elevator " + id + ": Already at floor " + floor);
            return;
        }

        if (floor > currentFloor) {
            upStops.add(floor);
            System.out.println("Elevator " + id + ": Added UP stop at floor " + floor);
        } else {
            downStops.add(floor);
            System.out.println("Elevator " + id + ": Added DOWN stop at floor " + floor);
        }

        // If idle, start moving
        if (state == ElevatorState.IDLE) {
            determineDirection();
        }
    }

    /**
     * Determines the direction based on pending stops.
     */
    private void determineDirection() {
        if (!upStops.isEmpty() && (downStops.isEmpty() ||
            direction == Direction.UP || direction == Direction.IDLE)) {
            direction = Direction.UP;
            state = ElevatorState.MOVING_UP;
        } else if (!downStops.isEmpty()) {
            direction = Direction.DOWN;
            state = ElevatorState.MOVING_DOWN;
        } else {
            direction = Direction.IDLE;
            state = ElevatorState.IDLE;
        }
    }

    /**
     * Moves the elevator one floor in current direction.
     * Returns true if moved, false if no movement needed.
     */
    public synchronized boolean move() {
        if (state == ElevatorState.MAINTENANCE ||
            state == ElevatorState.EMERGENCY) {
            return false;
        }

        if (!door.isClosed()) {
            System.out.println("Elevator " + id + ": Cannot move, door is open!");
            return false;
        }

        if (direction == Direction.UP && currentFloor < maxFloor) {
            currentFloor++;
            System.out.println("Elevator " + id + ": Moving UP to floor " + currentFloor);
            simulateDelay(FLOOR_TRAVEL_TIME_MS);
            return true;
        } else if (direction == Direction.DOWN && currentFloor > minFloor) {
            currentFloor--;
            System.out.println("Elevator " + id + ": Moving DOWN to floor " + currentFloor);
            simulateDelay(FLOOR_TRAVEL_TIME_MS);
            return true;
        }

        return false;
    }

    /**
     * Checks if elevator should stop at current floor.
     */
    public boolean shouldStop() {
        if (direction == Direction.UP) {
            return upStops.contains(currentFloor);
        } else if (direction == Direction.DOWN) {
            return downStops.contains(currentFloor);
        }
        return false;
    }

    /**
     * Stops at current floor, opens doors, waits, closes doors.
     */
    public synchronized void stopAtFloor() {
        state = ElevatorState.STOPPED;
        System.out.println("Elevator " + id + ": Stopping at floor " + currentFloor);

        // Remove this floor from stops
        upStops.remove(currentFloor);
        downStops.remove(currentFloor);

        // Open doors
        door.open();

        // Wait for passengers
        System.out.println("Elevator " + id + ": Waiting for passengers...");
        simulateDelay(DOOR_OPEN_DURATION_MS);

        // Close doors
        door.close();

        // Determine next direction
        determineDirection();
    }

    /**
     * Checks if this elevator can accept the given external request.
     * Used by scheduler to find best elevator.
     */
    public boolean canAcceptRequest(ExternalRequest request) {
        if (state == ElevatorState.MAINTENANCE ||
            state == ElevatorState.EMERGENCY) {
            return false;
        }

        int requestFloor = request.getFloor();
        Direction requestDir = request.getDirection();

        // If idle, can accept any request
        if (state == ElevatorState.IDLE) {
            return true;
        }

        // If moving in same direction and request is on the way
        if (direction == Direction.UP && requestDir == Direction.UP) {
            return requestFloor >= currentFloor;
        }
        if (direction == Direction.DOWN && requestDir == Direction.DOWN) {
            return requestFloor <= currentFloor;
        }

        return false;
    }

    /**
     * Calculates distance to a floor (for scheduling).
     */
    public int distanceTo(int floor) {
        return Math.abs(currentFloor - floor);
    }

    /**
     * Calculates estimated time to reach a floor.
     */
    public long estimatedTimeToFloor(int floor) {
        int distance = distanceTo(floor);
        int stopsOnWay = countStopsOnWay(floor);

        // Time = travel time + stop time for each intermediate stop
        return distance * FLOOR_TRAVEL_TIME_MS +
               stopsOnWay * (DOOR_OPEN_DURATION_MS + 2 * 2000);  // door open/close
    }

    private int countStopsOnWay(int targetFloor) {
        int count = 0;
        NavigableSet<Integer> stops = (targetFloor > currentFloor) ? upStops : downStops;

        for (int stop : stops) {
            if ((targetFloor > currentFloor && stop < targetFloor) ||
                (targetFloor < currentFloor && stop > targetFloor)) {
                count++;
            }
        }
        return count;
    }

    /**
     * Emergency stop - immediately stops and opens doors.
     */
    public synchronized void emergencyStop() {
        state = ElevatorState.EMERGENCY;
        direction = Direction.IDLE;
        door.emergencyOpen();
        System.out.println("Elevator " + id + ": EMERGENCY STOP at floor " + currentFloor);
    }

    /**
     * Sets elevator to maintenance mode.
     */
    public synchronized void setMaintenance(boolean maintenance) {
        if (maintenance) {
            state = ElevatorState.MAINTENANCE;
            direction = Direction.IDLE;
            upStops.clear();
            downStops.clear();
            System.out.println("Elevator " + id + ": Entering MAINTENANCE mode");
        } else {
            state = ElevatorState.IDLE;
            System.out.println("Elevator " + id + ": Exiting MAINTENANCE mode");
        }
    }

    /**
     * Updates current weight (called when passengers enter/exit).
     */
    public synchronized boolean updateWeight(int deltaKg) {
        int newWeight = currentWeightKg + deltaKg;

        if (newWeight > maxWeightKg) {
            System.out.println("Elevator " + id + ": OVERWEIGHT! Max: " +
                              maxWeightKg + "kg, Current: " + newWeight + "kg");
            return false;
        }

        if (newWeight < 0) {
            newWeight = 0;
        }

        currentWeightKg = newWeight;
        return true;
    }

    public boolean hasStops() {
        return !upStops.isEmpty() || !downStops.isEmpty();
    }

    // Getters
    public int getId() { return id; }
    public int getCurrentFloor() { return currentFloor; }
    public Direction getDirection() { return direction; }
    public ElevatorState getState() { return state; }
    public int getCurrentWeightKg() { return currentWeightKg; }
    public int getMaxWeightKg() { return maxWeightKg; }
    public Door getDoor() { return door; }
    public Set<Integer> getUpStops() { return Collections.unmodifiableSet(upStops); }
    public Set<Integer> getDownStops() { return Collections.unmodifiableSet(downStops); }

    private void simulateDelay(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    @Override
    public String toString() {
        return String.format(
            "Elevator[id=%d, floor=%d, direction=%s, state=%s, upStops=%s, downStops=%s]",
            id, currentFloor, direction, state, upStops, downStops);
    }
}
```

### 2.5 Scheduling Strategy (Strategy Pattern)

```java
// SchedulingStrategy.java
package com.elevator.scheduling;

import com.elevator.models.Elevator;
import com.elevator.models.ExternalRequest;
import java.util.List;

/**
 * Strategy interface for elevator scheduling algorithms.
 *
 * Different algorithms optimize for different goals:
 * - FCFS: Simple, fair, but inefficient
 * - SCAN: Efficient, like disk scheduling
 * - LOOK: Optimized SCAN that doesn't go to extremes
 */
public interface SchedulingStrategy {

    /**
     * Selects the best elevator to handle an external request.
     *
     * @param elevators List of available elevators
     * @param request The external request to handle
     * @return The selected elevator, or null if none available
     */
    Elevator selectElevator(List<Elevator> elevators, ExternalRequest request);

    /**
     * Returns the name of this scheduling algorithm.
     */
    String getName();
}
```

```java
// FCFSScheduler.java
package com.elevator.scheduling;

import com.elevator.models.Elevator;
import com.elevator.models.ExternalRequest;
import java.util.List;

/**
 * First-Come-First-Served scheduler.
 *
 * ALGORITHM: Assign request to the nearest available elevator.
 *
 * PROS:
 * - Simple to implement
 * - Fair (no request waits forever)
 *
 * CONS:
 * - Inefficient (doesn't consider direction)
 * - Can cause "ping-pong" effect
 */
public class FCFSScheduler implements SchedulingStrategy {

    @Override
    public Elevator selectElevator(List<Elevator> elevators, ExternalRequest request) {
        Elevator nearest = null;
        int minDistance = Integer.MAX_VALUE;

        for (Elevator elevator : elevators) {
            if (elevator.canAcceptRequest(request)) {
                int distance = elevator.distanceTo(request.getFloor());
                if (distance < minDistance) {
                    minDistance = distance;
                    nearest = elevator;
                }
            }
        }

        return nearest;
    }

    @Override
    public String getName() {
        return "FCFS (First-Come-First-Served)";
    }
}
```

```java
// SCANScheduler.java
package com.elevator.scheduling;

import com.elevator.enums.Direction;
import com.elevator.enums.ElevatorState;
import com.elevator.models.Elevator;
import com.elevator.models.ExternalRequest;
import java.util.List;

/**
 * SCAN (Elevator) scheduler.
 *
 * ALGORITHM: Like a disk head, elevator moves in one direction
 * serving all requests, then reverses at the end.
 *
 * Named "elevator algorithm" because it's how real elevators work!
 *
 * PROS:
 * - Efficient (minimizes direction changes)
 * - Good throughput
 * - Bounded wait time
 *
 * CONS:
 * - Requests at extremes may wait longer
 * - Not optimal for light loads
 */
public class SCANScheduler implements SchedulingStrategy {

    @Override
    public Elevator selectElevator(List<Elevator> elevators, ExternalRequest request) {
        Elevator best = null;
        int bestScore = Integer.MAX_VALUE;

        int requestFloor = request.getFloor();
        Direction requestDir = request.getDirection();

        for (Elevator elevator : elevators) {
            if (!elevator.canAcceptRequest(request)) {
                continue;
            }

            int score = calculateScore(elevator, requestFloor, requestDir);

            if (score < bestScore) {
                bestScore = score;
                best = elevator;
            }
        }

        return best;
    }

    /**
     * Calculates a score for elevator selection.
     * Lower score = better choice.
     *
     * Scoring factors:
     * 1. Distance to request floor
     * 2. Whether elevator is moving toward request
     * 3. Whether directions match
     */
    private int calculateScore(Elevator elevator, int requestFloor, Direction requestDir) {
        int currentFloor = elevator.getCurrentFloor();
        Direction elevatorDir = elevator.getDirection();

        int distance = Math.abs(currentFloor - requestFloor);

        // If elevator is idle, just use distance
        if (elevator.getState() == ElevatorState.IDLE) {
            return distance;
        }

        // If moving toward request and same direction, best case
        boolean movingToward = (elevatorDir == Direction.UP && requestFloor > currentFloor) ||
                               (elevatorDir == Direction.DOWN && requestFloor < currentFloor);

        if (movingToward && elevatorDir == requestDir) {
            return distance;  // Best: on the way
        }

        if (movingToward && elevatorDir != requestDir) {
            return distance + 10;  // Will pick up, but different direction
        }

        // Moving away from request
        return distance + 20;  // Worst: needs to reverse
    }

    @Override
    public String getName() {
        return "SCAN (Elevator Algorithm)";
    }
}
```

```java
// LOOKScheduler.java
package com.elevator.scheduling;

import com.elevator.enums.Direction;
import com.elevator.enums.ElevatorState;
import com.elevator.models.Elevator;
import com.elevator.models.ExternalRequest;
import java.util.List;

/**
 * LOOK scheduler - optimized SCAN.
 *
 * ALGORITHM: Like SCAN, but reverses direction when there are
 * no more requests in current direction (doesn't go to extremes).
 *
 * PROS:
 * - More efficient than SCAN
 * - Doesn't waste time going to unused floors
 *
 * CONS:
 * - Slightly more complex
 * - May cause starvation in edge cases
 */
public class LOOKScheduler implements SchedulingStrategy {

    @Override
    public Elevator selectElevator(List<Elevator> elevators, ExternalRequest request) {
        Elevator best = null;
        long bestTime = Long.MAX_VALUE;

        for (Elevator elevator : elevators) {
            if (!elevator.canAcceptRequest(request)) {
                continue;
            }

            long estimatedTime = elevator.estimatedTimeToFloor(request.getFloor());

            // Adjust time based on direction match
            if (elevator.getState() != ElevatorState.IDLE) {
                boolean sameDirection = elevator.getDirection() == request.getDirection();
                if (!sameDirection) {
                    estimatedTime *= 1.5;  // Penalty for direction mismatch
                }
            }

            if (estimatedTime < bestTime) {
                bestTime = estimatedTime;
                best = elevator;
            }
        }

        return best;
    }

    @Override
    public String getName() {
        return "LOOK (Optimized SCAN)";
    }
}
```

### 2.6 Elevator Controller

```java
// ElevatorController.java
package com.elevator.controller;

import com.elevator.models.Elevator;
import com.elevator.models.ExternalRequest;
import com.elevator.models.InternalRequest;
import com.elevator.scheduling.SchedulingStrategy;
import com.elevator.scheduling.LOOKScheduler;
import java.util.*;
import java.util.concurrent.*;

/**
 * Central controller for all elevators in the building.
 *
 * RESPONSIBILITY:
 * - Receive external requests (from floor buttons)
 * - Dispatch requests to appropriate elevators
 * - Coordinate elevator movements
 * - Handle system-wide operations (emergency, maintenance)
 */
public class ElevatorController {

    private final List<Elevator> elevators;
    private final Queue<ExternalRequest> pendingRequests;
    private SchedulingStrategy scheduler;
    private final ExecutorService elevatorThreads;
    private volatile boolean running;

    public ElevatorController(int numElevators, int minFloor, int maxFloor, int maxWeightKg) {
        this.elevators = new ArrayList<>();
        this.pendingRequests = new ConcurrentLinkedQueue<>();
        this.scheduler = new LOOKScheduler();  // Default scheduler
        this.elevatorThreads = Executors.newFixedThreadPool(numElevators);
        this.running = false;

        // Create elevators
        for (int i = 0; i < numElevators; i++) {
            elevators.add(new Elevator(i + 1, minFloor, maxFloor, maxWeightKg));
        }

        System.out.println("ElevatorController initialized with " + numElevators +
                          " elevators using " + scheduler.getName());
    }

    /**
     * Sets the scheduling strategy.
     */
    public void setScheduler(SchedulingStrategy scheduler) {
        this.scheduler = scheduler;
        System.out.println("Scheduler changed to: " + scheduler.getName());
    }

    /**
     * Starts the elevator controller.
     * Each elevator runs in its own thread.
     */
    public void start() {
        running = true;

        for (Elevator elevator : elevators) {
            elevatorThreads.submit(() -> runElevator(elevator));
        }

        // Start request dispatcher
        new Thread(this::dispatchRequests, "RequestDispatcher").start();

        System.out.println("ElevatorController started");
    }

    /**
     * Stops the elevator controller.
     */
    public void stop() {
        running = false;
        elevatorThreads.shutdown();
        try {
            elevatorThreads.awaitTermination(10, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("ElevatorController stopped");
    }

    /**
     * Handles an external request (from floor button).
     */
    public void requestElevator(ExternalRequest request) {
        System.out.println("\n>>> New request: " + request);
        pendingRequests.offer(request);
    }

    /**
     * Handles an internal request (from inside elevator).
     */
    public void requestFloor(int elevatorId, int floor) {
        Elevator elevator = getElevator(elevatorId);
        if (elevator != null) {
            InternalRequest request = new InternalRequest(floor, elevator.getCurrentFloor());
            System.out.println("\n>>> Internal request: " + request);
            elevator.addDestination(floor);
        }
    }

    /**
     * Dispatches pending requests to elevators.
     */
    private void dispatchRequests() {
        while (running) {
            ExternalRequest request = pendingRequests.poll();

            if (request != null) {
                Elevator selected = scheduler.selectElevator(elevators, request);

                if (selected != null) {
                    System.out.println("Dispatching request to Elevator " + selected.getId());
                    selected.addDestination(request.getFloor());
                    request.markServed();
                } else {
                    // No elevator available, re-queue
                    pendingRequests.offer(request);
                }
            }

            // Small delay to prevent busy-waiting
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    /**
     * Main loop for a single elevator.
     */
    private void runElevator(Elevator elevator) {
        while (running) {
            if (elevator.hasStops()) {
                // Check if we should stop at current floor
                if (elevator.shouldStop()) {
                    elevator.stopAtFloor();
                } else {
                    // Move to next floor
                    elevator.move();
                }
            } else {
                // No stops, wait for requests
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }

    /**
     * Emergency stop all elevators.
     */
    public void emergencyStopAll() {
        System.out.println("\n!!! EMERGENCY STOP ALL ELEVATORS !!!");
        for (Elevator elevator : elevators) {
            elevator.emergencyStop();
        }
    }

    /**
     * Gets an elevator by ID.
     */
    public Elevator getElevator(int id) {
        return elevators.stream()
                .filter(e -> e.getId() == id)
                .findFirst()
                .orElse(null);
    }

    /**
     * Gets status of all elevators.
     */
    public void displayStatus() {
        System.out.println("\n========== ELEVATOR STATUS ==========");
        for (Elevator elevator : elevators) {
            System.out.println(elevator);
        }
        System.out.println("Pending requests: " + pendingRequests.size());
        System.out.println("=====================================\n");
    }

    public List<Elevator> getElevators() {
        return Collections.unmodifiableList(elevators);
    }
}
```

### 2.7 Floor and Building Classes

```java
// Floor.java
package com.elevator.models;

import com.elevator.enums.Direction;

/**
 * Represents a floor in the building.
 *
 * Each floor has:
 * - UP button (except top floor)
 * - DOWN button (except ground floor)
 * - Display showing elevator positions
 */
public class Floor {

    private final int floorNumber;
    private final String name;
    private final boolean hasUpButton;
    private final boolean hasDownButton;

    public Floor(int floorNumber, String name, int minFloor, int maxFloor) {
        this.floorNumber = floorNumber;
        this.name = name;
        this.hasUpButton = floorNumber < maxFloor;
        this.hasDownButton = floorNumber > minFloor;
    }

    /**
     * Simulates pressing the UP button.
     */
    public ExternalRequest pressUpButton() {
        if (!hasUpButton) {
            System.out.println("Floor " + floorNumber + ": No UP button (top floor)");
            return null;
        }
        System.out.println("Floor " + floorNumber + ": UP button pressed");
        return new ExternalRequest(floorNumber, Direction.UP);
    }

    /**
     * Simulates pressing the DOWN button.
     */
    public ExternalRequest pressDownButton() {
        if (!hasDownButton) {
            System.out.println("Floor " + floorNumber + ": No DOWN button (ground floor)");
            return null;
        }
        System.out.println("Floor " + floorNumber + ": DOWN button pressed");
        return new ExternalRequest(floorNumber, Direction.DOWN);
    }

    public int getFloorNumber() { return floorNumber; }
    public String getName() { return name; }

    @Override
    public String toString() {
        return String.format("Floor[%d: %s]", floorNumber, name);
    }
}
```

```java
// Building.java
package com.elevator.models;

import com.elevator.controller.ElevatorController;
import java.util.*;

/**
 * Represents the building containing elevators.
 *
 * PATTERN: Singleton - One building per system.
 *
 * RESPONSIBILITY:
 * - Configure building parameters
 * - Create and manage floors
 * - Provide access to elevator controller
 */
public class Building {

    private static Building instance;

    private final String name;
    private final int numFloors;
    private final int minFloor;
    private final int maxFloor;
    private final List<Floor> floors;
    private final ElevatorController controller;

    private Building(String name, int numFloors, int numElevators,
                     int minFloor, int maxWeightKg) {
        this.name = name;
        this.numFloors = numFloors;
        this.minFloor = minFloor;
        this.maxFloor = minFloor + numFloors - 1;
        this.floors = new ArrayList<>();

        // Create floors
        for (int i = minFloor; i <= maxFloor; i++) {
            String floorName = (i == 0) ? "Ground Floor" :
                              (i < 0) ? "Basement " + Math.abs(i) :
                              "Floor " + i;
            floors.add(new Floor(i, floorName, minFloor, maxFloor));
        }

        // Create elevator controller
        this.controller = new ElevatorController(numElevators, minFloor, maxFloor, maxWeightKg);
    }

    public static synchronized Building getInstance(String name, int numFloors,
                                                    int numElevators, int minFloor,
                                                    int maxWeightKg) {
        if (instance == null) {
            instance = new Building(name, numFloors, numElevators, minFloor, maxWeightKg);
        }
        return instance;
    }

    public static Building getInstance() {
        if (instance == null) {
            throw new IllegalStateException("Building not initialized");
        }
        return instance;
    }

    public static void resetInstance() {
        if (instance != null) {
            instance.controller.stop();
        }
        instance = null;
    }

    public void start() {
        controller.start();
        System.out.println("Building '" + name + "' elevator system started");
    }

    public void stop() {
        controller.stop();
        System.out.println("Building '" + name + "' elevator system stopped");
    }

    public Floor getFloor(int floorNumber) {
        int index = floorNumber - minFloor;
        if (index >= 0 && index < floors.size()) {
            return floors.get(index);
        }
        return null;
    }

    public ElevatorController getController() {
        return controller;
    }

    public String getName() { return name; }
    public int getNumFloors() { return numFloors; }
    public int getMinFloor() { return minFloor; }
    public int getMaxFloor() { return maxFloor; }
    public List<Floor> getFloors() { return Collections.unmodifiableList(floors); }
}
```

### 2.8 Main Application (Demo)

```java
// ElevatorSystemDemo.java
package com.elevator;

import com.elevator.controller.ElevatorController;
import com.elevator.models.*;
import com.elevator.enums.Direction;
import com.elevator.scheduling.*;

/**
 * Demonstration of the Elevator System.
 */
public class ElevatorSystemDemo {

    public static void main(String[] args) throws InterruptedException {
        // Reset any existing instance
        Building.resetInstance();

        System.out.println("=== ELEVATOR SYSTEM DEMO ===\n");

        // Create building: 10 floors (0-9), 3 elevators, 1000kg max weight
        Building building = Building.getInstance(
            "Tech Tower",
            10,      // floors
            3,       // elevators
            0,       // min floor (ground)
            1000     // max weight kg
        );

        // Start the elevator system
        building.start();

        // Wait for system to initialize
        Thread.sleep(1000);

        ElevatorController controller = building.getController();

        // Display initial status
        controller.displayStatus();

        // ==================== Scenario 1: Simple Request ====================
        System.out.println("\n===== SCENARIO 1: Simple Request =====");
        System.out.println("Person on floor 5 wants to go DOWN\n");

        Floor floor5 = building.getFloor(5);
        ExternalRequest request1 = floor5.pressDownButton();
        controller.requestElevator(request1);

        Thread.sleep(3000);
        controller.displayStatus();

        // ==================== Scenario 2: Multiple Requests ====================
        System.out.println("\n===== SCENARIO 2: Multiple Requests =====");
        System.out.println("Multiple people requesting elevators\n");

        // Person on floor 2 wants UP
        Floor floor2 = building.getFloor(2);
        controller.requestElevator(floor2.pressUpButton());

        // Person on floor 8 wants DOWN
        Floor floor8 = building.getFloor(8);
        controller.requestElevator(floor8.pressDownButton());

        // Person on floor 0 wants UP
        Floor floor0 = building.getFloor(0);
        controller.requestElevator(floor0.pressUpButton());

        Thread.sleep(5000);
        controller.displayStatus();

        // ==================== Scenario 3: Internal Request ====================
        System.out.println("\n===== SCENARIO 3: Internal Request =====");
        System.out.println("Person inside Elevator 1 presses floor 7\n");

        controller.requestFloor(1, 7);

        Thread.sleep(3000);
        controller.displayStatus();

        // ==================== Scenario 4: Change Scheduler ====================
        System.out.println("\n===== SCENARIO 4: Change Scheduler =====");

        controller.setScheduler(new SCANScheduler());

        // More requests with new scheduler
        controller.requestElevator(building.getFloor(3).pressUpButton());
        controller.requestElevator(building.getFloor(6).pressDownButton());

        Thread.sleep(5000);
        controller.displayStatus();

        // ==================== Scenario 5: Emergency Stop ====================
        System.out.println("\n===== SCENARIO 5: Emergency Stop =====");

        controller.emergencyStopAll();
        controller.displayStatus();

        // Cleanup
        Thread.sleep(2000);
        building.stop();

        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## STEP 4: Code Walkthrough - Building From Scratch

### Phase 1: Start with Enums (Define the Vocabulary)

Before writing any logic, define what states and directions exist.

```java
// Step 1: Direction enum
// File: com/elevator/enums/Direction.java

package com.elevator.enums;

public enum Direction {
    UP,      // Moving toward higher floors
    DOWN,    // Moving toward lower floors
    IDLE     // Not moving
}
```

**Why Direction first?**

- Elevators and requests both have direction
- Core concept used everywhere
- Simple, no dependencies

```java
// Step 2: ElevatorState enum
// File: com/elevator/enums/ElevatorState.java

public enum ElevatorState {
    MOVING_UP,     // Actively moving up
    MOVING_DOWN,   // Actively moving down
    IDLE,          // Waiting for requests
    STOPPED,       // At a floor, doors may be open
    MAINTENANCE,   // Out of service
    EMERGENCY      // Emergency stop
}
```

**Why separate Direction and State?**

- Direction: Where elevator is headed
- State: What elevator is doing
- An IDLE elevator has no direction
- A STOPPED elevator remembers its direction

---

### Phase 2: Build the Request System

```java
// Step 3: Abstract Request class
// File: com/elevator/models/Request.java

package com.elevator.models;

import com.elevator.enums.Direction;
import java.time.LocalDateTime;

public abstract class Request implements Comparable<Request> {

    protected final int floor;
    protected final LocalDateTime timestamp;
    protected boolean served;
```

**Line-by-line explanation:**

- `abstract class Request` - Can't create generic Request, must be External or Internal
- `protected final int floor` - Target floor, immutable, accessible to subclasses
- `protected final LocalDateTime timestamp` - When request was made (for FCFS ordering)
- `protected boolean served` - Track if request has been handled

```java
    protected Request(int floor) {
        this.floor = floor;
        this.timestamp = LocalDateTime.now();
        this.served = false;
    }
```

**Constructor logic:**

- Takes only floor (timestamp auto-generated)
- All requests start as unserved
- Protected: only subclasses can call

```java
    @Override
    public int compareTo(Request other) {
        return Integer.compare(this.floor, other.floor);
    }

    public abstract Direction getDirection();
}
```

**Why Comparable?**

- Requests can be sorted by floor
- Useful for SCAN algorithm (process in floor order)
- TreeSet/PriorityQueue can use natural ordering

---

### Phase 3: Build the Door (Simple Component First)

```java
// Step 4: Door class
// File: com/elevator/models/Door.java

package com.elevator.models;

import com.elevator.enums.DoorState;

public class Door {

    private DoorState state;
    private static final long DOOR_OPERATION_TIME_MS = 2000;

    public Door() {
        this.state = DoorState.CLOSED;  // Doors start closed
    }
```

**Design decisions:**

- Door is a separate class (SRP)
- Initial state is CLOSED (safe default)
- Operation time is configurable constant

```java
    public synchronized void open() {
        if (state == DoorState.OPEN || state == DoorState.OPENING) {
            return;  // Idempotent - calling open twice is safe
        }

        state = DoorState.OPENING;
        System.out.println("  Door: Opening...");

        simulateDelay(DOOR_OPERATION_TIME_MS);

        state = DoorState.OPEN;
        System.out.println("  Door: Open");
    }
```

**Why synchronized?**

- Multiple threads might try to open/close
- State transitions must be atomic
- Prevents door being in invalid state

**Why check current state first?**

- Idempotency: calling open() on open door does nothing
- Prevents redundant operations
- Real elevators work this way

---

### Phase 4: Build the Elevator (Core Component)

```java
// Step 5: Elevator class
// File: com/elevator/models/Elevator.java

package com.elevator.models;

import java.util.*;
import java.util.concurrent.ConcurrentSkipListSet;

public class Elevator {

    // Identity (immutable)
    private final int id;
    private final int minFloor;
    private final int maxFloor;
    private final int maxWeightKg;
    private final Door door;

    // State (mutable)
    private int currentFloor;
    private Direction direction;
    private ElevatorState state;
    private int currentWeightKg;

    // Destinations (concurrent sorted sets)
    private final NavigableSet<Integer> upStops;
    private final NavigableSet<Integer> downStops;
```

**Why two stop sets?**

- `upStops`: Floors to visit when going UP (sorted ascending)
- `downStops`: Floors to visit when going DOWN (sorted descending)
- Enables efficient SCAN: always get next stop in O(1)

**Why ConcurrentSkipListSet?**

- Thread-safe (multiple threads add destinations)
- Sorted (for SCAN algorithm)
- O(log n) operations

```java
    public Elevator(int id, int minFloor, int maxFloor, int maxWeightKg) {
        this.id = id;
        this.minFloor = minFloor;
        this.maxFloor = maxFloor;
        this.maxWeightKg = maxWeightKg;
        this.door = new Door();  // Composition: Elevator owns Door

        this.currentFloor = minFloor;  // Start at ground
        this.direction = Direction.IDLE;
        this.state = ElevatorState.IDLE;
        this.currentWeightKg = 0;

        // Ascending order for up stops
        this.upStops = new ConcurrentSkipListSet<>();
        // Descending order for down stops
        this.downStops = new ConcurrentSkipListSet<>(Collections.reverseOrder());
    }
```

**Why reverseOrder for downStops?**

- When going DOWN, we want highest floor first
- Example: At floor 8, downStops = {5, 3, 1}
- First element (5) is the next stop

---

### Phase 5: Build the Scheduling Strategies

```java
// Step 6: Strategy interface
// File: com/elevator/scheduling/SchedulingStrategy.java

public interface SchedulingStrategy {
    Elevator selectElevator(List<Elevator> elevators, ExternalRequest request);
    String getName();
}
```

**Interface design:**

- `selectElevator`: Core method, returns best elevator
- `getName`: For logging/debugging
- No state in interface (strategies are stateless)

```java
// Step 7: SCAN Scheduler
// File: com/elevator/scheduling/SCANScheduler.java

public class SCANScheduler implements SchedulingStrategy {

    @Override
    public Elevator selectElevator(List<Elevator> elevators, ExternalRequest request) {
        Elevator best = null;
        int bestScore = Integer.MAX_VALUE;

        for (Elevator elevator : elevators) {
            if (!elevator.canAcceptRequest(request)) {
                continue;  // Skip unavailable elevators
            }

            int score = calculateScore(elevator, request.getFloor(), request.getDirection());

            if (score < bestScore) {
                bestScore = score;
                best = elevator;
            }
        }

        return best;
    }
```

**Selection algorithm:**

1. Filter to available elevators
2. Calculate score for each
3. Return lowest score (best fit)

**Score examples:**

| Elevator | Floor | Dir  | Request | Score | Reason                          |
| -------- | ----- | ---- | ------- | ----- | ------------------------------- |
| E1       | 3     | UP   | F5 UP   | 2     | On the way, same direction      |
| E2       | 3     | UP   | F5 DOWN | 12    | On the way, different direction |
| E3       | 7     | DOWN | F5 UP   | 22    | Moving away                     |
| E4       | 5     | IDLE | F5 UP   | 0     | Already there!                  |

---

### Phase 6: Build the Controller

```java
// Step 8: ElevatorController
// File: com/elevator/controller/ElevatorController.java

public class ElevatorController {

    private final List<Elevator> elevators;
    private final Queue<ExternalRequest> pendingRequests;
    private SchedulingStrategy scheduler;
    private final ExecutorService elevatorThreads;
    private volatile boolean running;
```

**Components:**

- `elevators`: All elevators under control
- `pendingRequests`: Queue of unassigned requests
- `scheduler`: Current scheduling algorithm
- `elevatorThreads`: Thread pool for elevator operations
- `running`: Flag to stop the system

**Why ConcurrentLinkedQueue?**

- Thread-safe without explicit locking
- Non-blocking operations
- Good for producer-consumer pattern

**Threading model:**

```
Main Thread
    │
    ├── RequestDispatcher Thread
    │       └── Polls pendingRequests, assigns to elevators
    │
    ├── Elevator 1 Thread
    │       └── Runs runElevator(elevator1)
    │
    ├── Elevator 2 Thread
    │       └── Runs runElevator(elevator2)
    │
    └── Elevator 3 Thread
            └── Runs runElevator(elevator3)
```

**Elevator loop logic:**

```
while running:
    if has stops:
        if at stop floor:
            stop, open door, close door
        else:
            move one floor
    else:
        sleep (wait for requests)
```

---

## File Structure

```
com/elevator/
├── enums/
│   ├── Direction.java
│   ├── ElevatorState.java
│   └── DoorState.java
├── models/
│   ├── Request.java
│   ├── ExternalRequest.java
│   ├── InternalRequest.java
│   ├── Door.java
│   ├── Elevator.java
│   ├── Floor.java
│   └── Building.java
├── scheduling/
│   ├── SchedulingStrategy.java
│   ├── FCFSScheduler.java
│   ├── SCANScheduler.java
│   └── LOOKScheduler.java
├── controller/
│   └── ElevatorController.java
└── ElevatorSystemDemo.java
```
