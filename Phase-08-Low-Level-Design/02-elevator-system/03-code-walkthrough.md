# 🛗 Elevator System - Code Walkthrough

## Building From Scratch: Step-by-Step

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

```java
    public synchronized void addDestination(int floor) {
        // Validation
        if (floor < minFloor || floor > maxFloor) {
            System.out.println("Elevator " + id + ": Invalid floor " + floor);
            return;
        }
        
        if (floor == currentFloor) {
            System.out.println("Elevator " + id + ": Already at floor " + floor);
            return;
        }
        
        // Add to appropriate set based on direction
        if (floor > currentFloor) {
            upStops.add(floor);
        } else {
            downStops.add(floor);
        }
        
        // Wake up if idle
        if (state == ElevatorState.IDLE) {
            determineDirection();
        }
    }
```

**State after addDestination(7) when at floor 3:**
```
Before: upStops = {}, downStops = {}, state = IDLE
After:  upStops = {7}, downStops = {}, state = MOVING_UP
```

```java
    public synchronized boolean move() {
        // Can't move in maintenance or emergency
        if (state == ElevatorState.MAINTENANCE || 
            state == ElevatorState.EMERGENCY) {
            return false;
        }
        
        // Can't move with door open
        if (!door.isClosed()) {
            System.out.println("Elevator " + id + ": Cannot move, door is open!");
            return false;
        }
        
        // Move in current direction
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
        
        return false;  // Can't move (at boundary or idle)
    }
```

**Safety checks explained:**
1. **Maintenance/Emergency**: Elevator must not move
2. **Door open**: Safety hazard, must close first
3. **Boundary check**: Can't go above maxFloor or below minFloor

```java
    public boolean shouldStop() {
        if (direction == Direction.UP) {
            return upStops.contains(currentFloor);
        } else if (direction == Direction.DOWN) {
            return downStops.contains(currentFloor);
        }
        return false;
    }
```

**Example:**
```
Elevator at floor 5, direction = UP, upStops = {5, 8}
shouldStop() → upStops.contains(5) → true
```

```java
    public synchronized void stopAtFloor() {
        state = ElevatorState.STOPPED;
        System.out.println("Elevator " + id + ": Stopping at floor " + currentFloor);
        
        // Remove this floor from stops
        upStops.remove(currentFloor);
        downStops.remove(currentFloor);
        
        // Door operations
        door.open();
        System.out.println("Elevator " + id + ": Waiting for passengers...");
        simulateDelay(DOOR_OPEN_DURATION_MS);
        door.close();
        
        // Determine next direction
        determineDirection();
    }
```

**Sequence at stop:**
1. Change state to STOPPED
2. Remove floor from pending stops
3. Open door
4. Wait for passengers (simulated)
5. Close door
6. Decide next direction

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

```java
    private int calculateScore(Elevator elevator, int requestFloor, Direction requestDir) {
        int currentFloor = elevator.getCurrentFloor();
        Direction elevatorDir = elevator.getDirection();
        
        int distance = Math.abs(currentFloor - requestFloor);
        
        // Idle elevator: just distance
        if (elevator.getState() == ElevatorState.IDLE) {
            return distance;
        }
        
        // Check if moving toward request
        boolean movingToward = 
            (elevatorDir == Direction.UP && requestFloor > currentFloor) ||
            (elevatorDir == Direction.DOWN && requestFloor < currentFloor);
        
        if (movingToward && elevatorDir == requestDir) {
            return distance;  // Best: on the way, same direction
        }
        
        if (movingToward && elevatorDir != requestDir) {
            return distance + 10;  // OK: on the way, different direction
        }
        
        return distance + 20;  // Worst: moving away
    }
}
```

**Score examples:**

| Elevator | Floor | Dir | Request | Score | Reason |
|----------|-------|-----|---------|-------|--------|
| E1 | 3 | UP | F5 UP | 2 | On the way, same direction |
| E2 | 3 | UP | F5 DOWN | 12 | On the way, different direction |
| E3 | 7 | DOWN | F5 UP | 22 | Moving away |
| E4 | 5 | IDLE | F5 UP | 0 | Already there! |

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

```java
    public ElevatorController(int numElevators, int minFloor, int maxFloor, int maxWeightKg) {
        this.elevators = new ArrayList<>();
        this.pendingRequests = new ConcurrentLinkedQueue<>();
        this.scheduler = new LOOKScheduler();  // Default
        this.elevatorThreads = Executors.newFixedThreadPool(numElevators);
        this.running = false;
        
        // Create elevators
        for (int i = 0; i < numElevators; i++) {
            elevators.add(new Elevator(i + 1, minFloor, maxFloor, maxWeightKg));
        }
    }
```

**Why ConcurrentLinkedQueue?**
- Thread-safe without explicit locking
- Non-blocking operations
- Good for producer-consumer pattern

```java
    public void start() {
        running = true;
        
        // Start each elevator in its own thread
        for (Elevator elevator : elevators) {
            elevatorThreads.submit(() -> runElevator(elevator));
        }
        
        // Start request dispatcher in separate thread
        new Thread(this::dispatchRequests, "RequestDispatcher").start();
    }
```

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

```java
    private void runElevator(Elevator elevator) {
        while (running) {
            if (elevator.hasStops()) {
                if (elevator.shouldStop()) {
                    elevator.stopAtFloor();
                } else {
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

## Testing Approach

### Unit Tests

```java
// ElevatorTest.java
public class ElevatorTest {
    
    private Elevator elevator;
    
    @BeforeEach
    void setUp() {
        elevator = new Elevator(1, 0, 10, 1000);
    }
    
    @Test
    void testInitialState() {
        assertEquals(0, elevator.getCurrentFloor());
        assertEquals(Direction.IDLE, elevator.getDirection());
        assertEquals(ElevatorState.IDLE, elevator.getState());
    }
    
    @Test
    void testAddDestinationAbove() {
        elevator.addDestination(5);
        
        assertTrue(elevator.getUpStops().contains(5));
        assertEquals(Direction.UP, elevator.getDirection());
    }
    
    @Test
    void testAddDestinationBelow() {
        // First move elevator up
        elevator.addDestination(5);
        while (elevator.getCurrentFloor() < 5) {
            elevator.move();
        }
        elevator.stopAtFloor();
        
        // Now add destination below
        elevator.addDestination(2);
        
        assertTrue(elevator.getDownStops().contains(2));
    }
    
    @Test
    void testCannotMoveWithDoorOpen() {
        elevator.addDestination(5);
        elevator.getDoor().open();
        
        assertFalse(elevator.move());
    }
    
    @Test
    void testShouldStopAtDestination() {
        elevator.addDestination(3);
        
        // Move to floor 3
        elevator.move();  // 0 → 1
        assertFalse(elevator.shouldStop());
        
        elevator.move();  // 1 → 2
        assertFalse(elevator.shouldStop());
        
        elevator.move();  // 2 → 3
        assertTrue(elevator.shouldStop());
    }
    
    @Test
    void testWeightLimit() {
        assertTrue(elevator.updateWeight(500));   // Add 500kg
        assertTrue(elevator.updateWeight(400));   // Add 400kg (total 900)
        assertFalse(elevator.updateWeight(200));  // Exceeds 1000kg limit
    }
}
```

```java
// SchedulerTest.java
public class SchedulerTest {
    
    private List<Elevator> elevators;
    
    @BeforeEach
    void setUp() {
        elevators = Arrays.asList(
            new Elevator(1, 0, 10, 1000),
            new Elevator(2, 0, 10, 1000),
            new Elevator(3, 0, 10, 1000)
        );
    }
    
    @Test
    void testFCFSSelectsNearest() {
        // Move elevator 2 to floor 4
        Elevator e2 = elevators.get(1);
        e2.addDestination(4);
        while (e2.getCurrentFloor() < 4) e2.move();
        e2.stopAtFloor();
        
        FCFSScheduler scheduler = new FCFSScheduler();
        ExternalRequest request = new ExternalRequest(5, Direction.UP);
        
        Elevator selected = scheduler.selectElevator(elevators, request);
        
        assertEquals(2, selected.getId());  // E2 is closest to floor 5
    }
    
    @Test
    void testSCANPrefersSameDirection() {
        // E1 at floor 3, moving UP
        Elevator e1 = elevators.get(0);
        e1.addDestination(5);
        while (e1.getCurrentFloor() < 3) e1.move();
        
        // E2 at floor 4, IDLE
        Elevator e2 = elevators.get(1);
        e2.addDestination(4);
        while (e2.getCurrentFloor() < 4) e2.move();
        e2.stopAtFloor();
        
        SCANScheduler scheduler = new SCANScheduler();
        ExternalRequest request = new ExternalRequest(6, Direction.UP);
        
        Elevator selected = scheduler.selectElevator(elevators, request);
        
        // E1 should be selected (moving UP toward floor 6)
        assertEquals(1, selected.getId());
    }
}
```

### Integration Tests

```java
// ElevatorSystemIntegrationTest.java
public class ElevatorSystemIntegrationTest {
    
    @Test
    void testFullScenario() throws InterruptedException {
        Building.resetInstance();
        Building building = Building.getInstance("Test", 10, 2, 0, 1000);
        building.start();
        
        ElevatorController controller = building.getController();
        
        // Request from floor 5
        controller.requestElevator(new ExternalRequest(5, Direction.UP));
        
        // Wait for elevator to arrive
        Thread.sleep(10000);
        
        // Verify an elevator is at or near floor 5
        boolean elevatorAtFloor5 = controller.getElevators().stream()
            .anyMatch(e -> e.getCurrentFloor() == 5);
        
        assertTrue(elevatorAtFloor5);
        
        building.stop();
    }
}
```

---

## Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `addDestination()` | O(log n) | ConcurrentSkipListSet.add() |
| `shouldStop()` | O(log n) | Set.contains() |
| `move()` | O(1) | Simple state update |
| `stopAtFloor()` | O(log n) | Set.remove() |
| `selectElevator()` | O(E) | E = number of elevators |
| `calculateScore()` | O(S) | S = stops in elevator |

### Space Complexity

| Data Structure | Space | Purpose |
|----------------|-------|---------|
| `elevators` | O(E) | Store elevator objects |
| `upStops` per elevator | O(F) | F = max floors |
| `downStops` per elevator | O(F) | F = max floors |
| `pendingRequests` | O(R) | R = pending requests |

### Bottlenecks at Scale

**100 elevators, 100 floors:**
- `selectElevator()` becomes O(E × S) = O(100 × 100) = O(10,000)
- Solution: Partition elevators by zone

**1000 requests/second:**
- `pendingRequests` queue grows unbounded
- Solution: Rate limiting, request batching

**Optimization: Zone-based Scheduling**

```java
public class ZonedElevatorController {
    private final Map<Integer, List<Elevator>> elevatorsByZone;
    
    // Zone 1: Floors 0-10
    // Zone 2: Floors 11-20
    // Zone 3: Floors 21-30
    
    public Elevator selectElevator(ExternalRequest request) {
        int zone = request.getFloor() / 10;
        List<Elevator> zoneElevators = elevatorsByZone.get(zone);
        
        // Only search within zone
        return scheduler.selectElevator(zoneElevators, request);
    }
}
```

---

## Interview Follow-ups with Answers

### Q1: How would you handle a stuck elevator?

**Answer:**
```java
public class Elevator {
    private long lastMovementTime;
    private static final long STUCK_THRESHOLD_MS = 30000;  // 30 seconds
    
    public boolean isStuck() {
        if (state == ElevatorState.MOVING_UP || state == ElevatorState.MOVING_DOWN) {
            return System.currentTimeMillis() - lastMovementTime > STUCK_THRESHOLD_MS;
        }
        return false;
    }
    
    public boolean move() {
        // ... existing logic ...
        lastMovementTime = System.currentTimeMillis();
        return true;
    }
}

public class ElevatorController {
    private ScheduledExecutorService healthChecker;
    
    public void startHealthCheck() {
        healthChecker.scheduleAtFixedRate(() -> {
            for (Elevator elevator : elevators) {
                if (elevator.isStuck()) {
                    handleStuckElevator(elevator);
                }
            }
        }, 10, 10, TimeUnit.SECONDS);
    }
    
    private void handleStuckElevator(Elevator elevator) {
        elevator.setMaintenance(true);
        redistributeRequests(elevator);
        notifyMaintenance(elevator);
    }
}
```

### Q2: How would you add VIP/priority floors?

**Answer:**
```java
public class PriorityRequest extends ExternalRequest {
    private final int priority;  // Higher = more important
    
    public PriorityRequest(int floor, Direction direction, int priority) {
        super(floor, direction);
        this.priority = priority;
    }
}

public class PriorityScheduler implements SchedulingStrategy {
    private final Set<Integer> vipFloors = Set.of(1, 10, 20);  // Lobby, exec floors
    
    @Override
    public Elevator selectElevator(List<Elevator> elevators, ExternalRequest request) {
        if (isVIPRequest(request)) {
            // Dedicate an idle elevator if available
            return findIdleElevator(elevators)
                .orElse(findNearestElevator(elevators, request));
        }
        return standardSelection(elevators, request);
    }
    
    private boolean isVIPRequest(ExternalRequest request) {
        return vipFloors.contains(request.getFloor()) ||
               (request instanceof PriorityRequest && 
                ((PriorityRequest) request).getPriority() > 5);
    }
}
```

### Q3: How would you implement destination dispatch?

**Answer:**
Destination dispatch: Users enter destination BEFORE entering elevator.

```java
public class DestinationDispatchController extends ElevatorController {
    
    // User enters destination at lobby kiosk
    public int requestElevatorWithDestination(int sourceFloor, int destFloor) {
        // Find best elevator for this trip
        Elevator best = findBestElevatorForTrip(sourceFloor, destFloor);
        
        if (best != null) {
            // Add both pickup and destination
            best.addDestination(sourceFloor);
            best.addDestination(destFloor);
            
            // Return elevator ID so user knows which one to take
            return best.getId();
        }
        
        return -1;  // No elevator available
    }
    
    private Elevator findBestElevatorForTrip(int source, int dest) {
        Direction tripDirection = source < dest ? Direction.UP : Direction.DOWN;
        
        return elevators.stream()
            .filter(e -> e.canAcceptRequest(new ExternalRequest(source, tripDirection)))
            .filter(e -> !willCauseBacktrack(e, source, dest))
            .min(Comparator.comparingLong(e -> e.estimatedTimeToFloor(source)))
            .orElse(null);
    }
}
```

### Q4: How would you handle peak hours (morning rush)?

**Answer:**
```java
public class RushHourOptimizer {
    
    // During morning rush, most people go UP from lobby
    public void optimizeForMorningRush(ElevatorController controller) {
        // Park some elevators at lobby
        for (int i = 0; i < controller.getElevators().size() / 2; i++) {
            Elevator elevator = controller.getElevator(i + 1);
            if (elevator.getState() == ElevatorState.IDLE) {
                elevator.addDestination(0);  // Send to lobby
            }
        }
        
        // Switch to batch loading
        controller.setScheduler(new BatchLoadingScheduler());
    }
}

public class BatchLoadingScheduler implements SchedulingStrategy {
    
    @Override
    public Elevator selectElevator(List<Elevator> elevators, ExternalRequest request) {
        // During rush, fill elevators before sending
        // Don't dispatch half-empty elevators
        
        Optional<Elevator> loadingElevator = elevators.stream()
            .filter(e -> e.getCurrentFloor() == request.getFloor())
            .filter(e -> e.getState() == ElevatorState.STOPPED)
            .filter(e -> e.getCurrentWeightKg() < e.getMaxWeightKg() * 0.8)
            .findFirst();
        
        if (loadingElevator.isPresent()) {
            return loadingElevator.get();  // Keep loading current elevator
        }
        
        // Otherwise, send a new one
        return findNearestIdle(elevators, request);
    }
}
```

### Q5: What would you do differently with more time?

**Answer:**
1. **Add persistence**: Store elevator state in database for recovery
2. **Add metrics**: Track wait times, trip times, utilization
3. **Add simulation mode**: Test algorithms without real delays
4. **Add REST API**: Control system via HTTP
5. **Add WebSocket**: Real-time position updates to displays
6. **Add ML-based prediction**: Predict demand based on time/day
7. **Add energy optimization**: Regenerative braking, sleep mode

