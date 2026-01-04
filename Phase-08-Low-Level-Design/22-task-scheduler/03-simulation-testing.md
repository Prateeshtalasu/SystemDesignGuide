# ⏰ Design a Task Scheduler - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Priority-Based Task Execution

**Initial State:**
```
TaskScheduler: Running
Strategy: PriorityStrategy
Queue: []
All Tasks: {}
```

**Step-by-step:**

1. `submit(Task A, priority=LOW)`
   - Queue: [A(LOW)]
   - All Tasks: {A_id → Task A}

2. `submit(Task B, priority=HIGH)`
   - Queue: [A(LOW), B(HIGH)]
   - All Tasks: {A_id → Task A, B_id → Task B}

3. `submit(Task C, priority=CRITICAL)`
   - Queue: [A(LOW), B(HIGH), C(CRITICAL)]
   - All Tasks: {A_id → Task A, B_id → Task B, C_id → Task C}

4. `processQueue()` iteration 1:
   - `strategy.selectNext()` → Returns C(CRITICAL) (highest priority)
   - Queue: [A(LOW), B(HIGH)]
   - Execute Task C: Status PENDING → RUNNING → COMPLETED
   - Results: {C_id → TaskResult(COMPLETED)}

5. `processQueue()` iteration 2:
   - `strategy.selectNext()` → Returns B(HIGH)
   - Queue: [A(LOW)]
   - Execute Task B: Status PENDING → RUNNING → COMPLETED
   - Results: {C_id → TaskResult, B_id → TaskResult}

6. `processQueue()` iteration 3:
   - `strategy.selectNext()` → Returns A(LOW)
   - Queue: []
   - Execute Task A: Status PENDING → RUNNING → COMPLETED
   - Results: {C_id → TaskResult, B_id → TaskResult, A_id → TaskResult}

**Final State:**
```
Queue: [] (empty)
All Tasks: {A_id → Task A(COMPLETED), B_id → Task B(COMPLETED), C_id → Task C(COMPLETED)}
Results: {A_id → TaskResult, B_id → TaskResult, C_id → TaskResult}
Execution Order: C → B → A (by priority)
```

---

### Scenario 2: Failure/Invalid Input - Task Cancellation

**Initial State:**
```
TaskScheduler: Running
Queue: [Task X(PENDING), Task Y(PENDING)]
All Tasks: {X_id → Task X, Y_id → Task Y}
```

**Step-by-step:**

1. `submit(Task Z, priority=HIGH)`
   - Queue: [X(PENDING), Y(PENDING), Z(PENDING)]
   - All Tasks: {X_id → Task X, Y_id → Task Y, Z_id → Task Z}

2. `cancel(Y_id)` while Task Y is PENDING
   - Task Y status: PENDING → CANCELLED
   - Queue: [X(PENDING), Z(PENDING)] (Y removed)
   - Return: true (cancellation successful)

3. Start execution of Task Z (it's running now)
   - Task Z status: PENDING → RUNNING

4. `cancel(Z_id)` while Task Z is RUNNING
   - Task Z status: RUNNING (unchanged, cannot cancel)
   - Return: false (cancellation failed)

**Final State:**
```
Queue: [X(PENDING)]
All Tasks: {X_id → Task X(PENDING), Y_id → Task Y(CANCELLED), Z_id → Task Z(RUNNING)}
Cancellation results: Y (success), Z (failed)
```

---

### Scenario 3: Edge Case - Task Dependencies

**Initial State:**
```
TaskScheduler: Running
Queue: []
All Tasks: {}
```

**Step-by-step:**

1. `submit(Task Parent)`
   - Queue: [Parent(PENDING)]
   - All Tasks: {Parent_id → Task Parent}

2. `submit(Task Child, dependencies=[Parent_id])`
   - Queue: [Parent(PENDING), Child(PENDING)]
   - All Tasks: {Parent_id → Task Parent, Child_id → Task Child(depends on Parent)}

3. `processQueue()` iteration 1:
   - `strategy.selectNext()` → Returns Parent
   - Check dependencies for Parent: None → Execute
   - Status: Parent PENDING → RUNNING → COMPLETED
   - Results: {Parent_id → TaskResult(COMPLETED)}

4. `processQueue()` iteration 2:
   - `strategy.selectNext()` → Returns Child
   - Check dependencies: allDependenciesComplete(Child)?
     - Check Parent status: COMPLETED ✓
   - Dependencies satisfied → Execute Child
   - Status: Child PENDING → RUNNING → COMPLETED
   - Results: {Parent_id → TaskResult, Child_id → TaskResult}

**Final State:**
```
Queue: []
All Tasks: {Parent_id → Task Parent(COMPLETED), Child_id → Task Child(COMPLETED)}
Execution Order: Parent → Child (dependency order maintained)
```

---

### Scenario 4: Concurrency/Race Condition - Concurrent Task Submission and Cancellation

**Initial State:**
```
TaskScheduler: Running
Queue: []
All Tasks: {}
Worker Threads: 4
```

**Step-by-step:**

1. Thread 1: `submit(Task A)`
   - Queue: [A(PENDING)]
   - All Tasks: {A_id → Task A}

2. Thread 2: `submit(Task B)` (concurrent)
   - Queue: [A(PENDING), B(PENDING)]
   - All Tasks: {A_id → Task A, B_id → Task B}

3. Thread 3: `cancel(A_id)` (concurrent with execution)
   - Check: Task A status is PENDING
   - Task A status: PENDING → CANCELLED
   - Queue: [B(PENDING)] (A removed)

4. Thread 4: `processQueue()` (concurrent)
   - `strategy.selectNext()` → Attempts to get Task A
   - Race condition: If A was already selected before cancellation, it may execute
   - If A was cancelled before selection, B is selected instead

5. Thread 1: `getStatus(A_id)` (concurrent)
   - Returns: CANCELLED (if cancellation won) or RUNNING (if execution won)

**Final State:**
```
Queue: [B(PENDING)] or [] (depending on race outcome)
All Tasks: {A_id → Task A(CANCELLED or COMPLETED), B_id → Task B(PENDING or COMPLETED)}
Race Condition: Cancellation vs execution timing determines final state
```

**Key Observation:** ConcurrentLinkedQueue and ConcurrentHashMap ensure thread-safe operations, but cancellation of a task that's already being processed may not prevent execution.

---

### Visual Trace: Task Execution

```
Submit 3 tasks with different priorities:

Task A (LOW), Task B (HIGH), Task C (CRITICAL)

┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Add to Queue                                            │
├─────────────────────────────────────────────────────────────────┤
│ Queue: [A(LOW), B(HIGH), C(CRITICAL)]                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: PriorityStrategy.selectNext()                           │
├─────────────────────────────────────────────────────────────────┤
│ Find highest priority: C(CRITICAL)                              │
│ Queue: [A(LOW), B(HIGH)]                                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Execute Task C                                          │
├─────────────────────────────────────────────────────────────────┤
│ Status: RUNNING → COMPLETED                                     │
│ Store result                                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Next iteration - select B(HIGH)                         │
├─────────────────────────────────────────────────────────────────┤
│ Queue: [A(LOW)]                                                 │
│ Execute B...                                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## STEP 6: Edge Cases & Testing Strategy

### Edge Cases

| Category | Edge Case | Expected Behavior |
|----------|-----------|-------------------|
| Submit | Null task | Throw IllegalArgumentException |
| Submit | Duplicate task ID | Should not happen (UUID) |
| Cancel | Cancel running task | Return false (cannot cancel) |
| Cancel | Cancel completed task | Return false |
| Cancel | Cancel pending task | Return true, status = CANCELLED |
| Dependency | Circular dependency | Deadlock (need detection) |
| Dependency | Dependency failed | Dependent task waits forever |
| Retry | Max retries exceeded | Status = FAILED |
| Retry | Retry delay = 0 | Immediate retry |
| Priority | All same priority | FIFO within priority |
| Recurring | Cancel recurring task | Stop future executions |
| Scheduler | Stop while tasks running | Wait for completion |

### Unit Tests

```java
@Test
void testPriorityScheduling() {
    TaskScheduler scheduler = new TaskScheduler();
    scheduler.setStrategy(new PriorityStrategy());
    
    AtomicInteger order = new AtomicInteger(0);
    List<Integer> executionOrder = new ArrayList<>();
    
    Task low = new Task("Low", () -> {
        executionOrder.add(order.incrementAndGet());
        return null;
    }).withPriority(TaskPriority.LOW);
    
    Task high = new Task("High", () -> {
        executionOrder.add(order.incrementAndGet());
        return null;
    }).withPriority(TaskPriority.HIGH);
    
    scheduler.submit(low);
    scheduler.submit(high);
    scheduler.start();
    
    Thread.sleep(500);
    
    // High should execute before Low
    assertEquals(Arrays.asList(1, 2), executionOrder);
}

@Test
void testRetry() {
    AtomicInteger attempts = new AtomicInteger(0);
    Task task = new Task("Retry", () -> {
        if (attempts.incrementAndGet() < 3) {
            throw new RuntimeException("Fail");
        }
        return "Success";
    }).withRetry(3, 100);
    
    scheduler.submit(task);
    Thread.sleep(1000);
    
    assertEquals(3, attempts.get());
    assertEquals(TaskStatus.COMPLETED, task.getStatus());
}
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.
