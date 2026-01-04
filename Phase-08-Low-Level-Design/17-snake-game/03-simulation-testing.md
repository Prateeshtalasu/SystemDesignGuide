# 🐍 Snake Game - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Normal Gameplay - Eating Food

```
Initial State:
┌───────────────────────────┐
│ Board: 10x10              │
│ Snake: [(5,5), (5,4), (5,3)] (length 3, heading RIGHT) │
│ Food: (5,7)               │
│ Score: 0                  │
└───────────────────────────┘

Step 1: move(RIGHT)
┌───────────────────────────────────────────────────────────────┐
│ Action: Snake moves RIGHT                                      │
│ New head: (5,6)                                                │
│ Tail removed: (5,3)                                            │
│ Snake: [(5,6), (5,5), (5,4)]                                  │
│ Food eaten: NO                                                 │
│ Score: 0                                                       │
│ Result: MoveResult.OK                                          │
└───────────────────────────────────────────────────────────────┘

Step 2: move(RIGHT)
┌───────────────────────────────────────────────────────────────┐
│ Action: Snake moves RIGHT                                      │
│ New head: (5,7) = Food position!                              │
│ Tail NOT removed (snake grows)                                │
│ Snake: [(5,7), (5,6), (5,5), (5,4)] (length 4)               │
│ Food eaten: YES                                                │
│ Score: 0 + 10 = 10                                            │
│ New food spawned at random empty position                     │
│ Result: MoveResult.ATE_FOOD                                   │
└───────────────────────────────────────────────────────────────┘

Step 3: move(DOWN)
┌───────────────────────────────────────────────────────────────┐
│ Action: Snake moves DOWN                                       │
│ New head: (6,7)                                                │
│ Tail removed: (5,4)                                            │
│ Snake: [(6,7), (5,7), (5,6), (5,5)]                           │
│ Score: 10                                                      │
│ Result: MoveResult.OK                                          │
└───────────────────────────────────────────────────────────────┘
```

---

### Scenario 2: Wall Collision - Game Over

```
Initial State:
┌───────────────────────────┐
│ Board: 10x10 (rows 0-9)   │
│ Snake: [(8,5)] heading DOWN│
│ Score: 50                 │
└───────────────────────────┘

Step 1: move(DOWN)
┌───────────────────────────────────────────────────────────────┐
│ Action: Snake moves DOWN                                       │
│ New head calculated: (9,5)                                    │
│ Bounds check: isInBounds(9,5) = TRUE (row 9 is valid)        │
│ Snake: [(9,5), ...]                                           │
│ Result: MoveResult.OK                                          │
└───────────────────────────────────────────────────────────────┘

Step 2: move(DOWN)
┌───────────────────────────────────────────────────────────────┐
│ Action: Snake moves DOWN                                       │
│ New head calculated: (10,5)                                   │
│ Bounds check: isInBounds(10,5) = FALSE (row 10 out of bounds)│
│ Status: PLAYING → GAME_OVER                                   │
│ Final Score: 50                                               │
│ Result: MoveResult.COLLISION_WALL                             │
└───────────────────────────────────────────────────────────────┘
```

---

### Scenario 3: Self-Collision - Game Over (Failure Scenario)

```
Initial State:
┌───────────────────────────┐
│ Board: 10x10              │
│ Snake: [(5,5), (5,4), (5,3), (5,2), (4,2), (4,3), (4,4)] │
│        (length 7, heading RIGHT)                          │
│ Score: 60                 │
└───────────────────────────┘

Visual:
    2   3   4   5
  ┌───┬───┬───┬───┐
4 │ ○ │ ○ │ ○ │   │   ○ = body
  ├───┼───┼───┼───┤
5 │ ○ │ ○ │ ○ │ ● │   ● = head (moving RIGHT)
  └───┴───┴───┴───┘

Step 1: move(UP) - Snake tries to turn up
┌───────────────────────────────────────────────────────────────┐
│ Action: Snake changes direction to UP and moves               │
│ New head calculated: (4,5)                                    │
│ Bounds check: PASS                                            │
│ Move snake: add head (4,5), remove tail (4,4)                │
│ Snake: [(4,5), (5,5), (5,4), (5,3), (5,2), (4,2), (4,3)]    │
│ Self collision check: PASS (head not on body)                │
│ Result: MoveResult.OK                                          │
└───────────────────────────────────────────────────────────────┘

Step 2: move(LEFT)
┌───────────────────────────────────────────────────────────────┐
│ Action: Snake changes direction to LEFT and moves             │
│ New head calculated: (4,4)                                    │
│ Move snake: add head (4,4)                                    │
│ Self collision check: head (4,4) equals body segment!        │
│ Snake body contains (4,4) at index 6                          │
│ Status: PLAYING → GAME_OVER                                   │
│ Final Score: 60                                               │
│ Result: MoveResult.COLLISION_SELF                             │
└───────────────────────────────────────────────────────────────┘
```

---

### Scenario 4: Direction Reversal Prevention (Edge Case)

```
Initial State:
┌───────────────────────────┐
│ Snake: [(5,5), (5,4)] heading RIGHT│
│ Length: 2                 │
└───────────────────────────┘

Step 1: Try to move LEFT (opposite direction)
┌───────────────────────────────────────────────────────────────┐
│ Action: setDirection(LEFT)                                    │
│ Check: LEFT is opposite of RIGHT                              │
│ Decision: REJECT direction change (would cause instant death) │
│ Direction remains: RIGHT                                      │
│ Snake continues moving RIGHT to (5,6)                        │
│ Result: MoveResult.OK (direction ignored)                     │
└───────────────────────────────────────────────────────────────┘
```

---

### Scenario 5: Concurrency/Race Condition - Rapid Direction Changes

**Initial State:**

```
┌───────────────────────────┐
│ Board: 10x10              │
│ Snake: [(5,5), (5,4), (5,3)] heading RIGHT │
│ Status: RUNNING           │
│ Thread A: Rapid direction changes │
│ Thread B: Game tick processing │
└───────────────────────────┘
```

**Step-by-step (simulating concurrent operations):**

**Thread A:** Rapidly calls `game.changeDirection()` at time T0
**Thread B:** Calls `game.move()` at time T0 (concurrent)

1. **Thread A:** `game.changeDirection(Direction.UP)` at T0

   - Acquires lock (if thread-safe) or processes immediately
   - Updates snake direction: RIGHT → UP
   - Direction queue: [UP]

2. **Thread B:** `game.move()` at T0 (concurrent)

   - Reads current direction: RIGHT (before Thread A's change)
   - Calculates next head: (5,6) based on RIGHT
   - Moves snake with RIGHT direction
   - Result: Snake moves RIGHT, not UP

3. **Thread A:** `game.changeDirection(Direction.LEFT)` at T0+1ms

   - Updates direction: UP → LEFT
   - Direction queue: [UP, LEFT]

4. **Thread B:** `game.move()` at T0+1ms
   - Processes queued direction: UP
   - Moves snake UP to (4,6)
   - Result: Direction change applied

**Final State:**

```
Snake moved RIGHT then UP
Direction changes queued and processed sequentially
No race condition due to single-threaded design or proper synchronization
Game state remains consistent
```

**Note:** In the current single-threaded design, direction changes are queued and processed once per game tick, preventing race conditions. If multi-threaded, proper synchronization (locks, atomic operations) would be needed.

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions

- **Wall Collision**: Game over
- **Self Collision**: Game over
- **Food on Snake**: Regenerate food
- **180° Turn**: Prevent (instant death)

---

## Testing Approach

### Unit Tests

```java
// SnakeTest.java
public class SnakeTest {

    @Test
    void testMove() {
        Snake snake = new Snake(new Position(5, 5));
        snake.setDirection(Direction.RIGHT);

        Position newHead = snake.move(false);

        assertEquals(new Position(5, 6), newHead);
        assertEquals(1, snake.getLength());
    }

    @Test
    void testGrow() {
        Snake snake = new Snake(new Position(5, 5));
        snake.setDirection(Direction.RIGHT);

        snake.move(true);  // Grow

        assertEquals(2, snake.getLength());
        assertEquals(new Position(5, 6), snake.getHead());
        assertEquals(new Position(5, 5), snake.getTail());
    }

    @Test
    void testSelfCollision() {
        Snake snake = new Snake(new Position(5, 5));

        // Create a snake that loops back
        snake.move(true);  // RIGHT
        snake.setDirection(Direction.DOWN);
        snake.move(true);
        snake.setDirection(Direction.LEFT);
        snake.move(true);
        snake.setDirection(Direction.UP);
        snake.move(true);  // Back to start

        assertTrue(snake.collidesWithSelf());
    }

    @Test
    void testCannotReverse() {
        Snake snake = new Snake(new Position(5, 5));
        snake.setDirection(Direction.RIGHT);
        snake.move(true);  // Need length > 1

        snake.setDirection(Direction.LEFT);  // Try to reverse

        assertEquals(Direction.RIGHT, snake.getDirection());
    }
}
```

```java
// SnakeGameTest.java
public class SnakeGameTest {

    @Test
    void testWallCollision() {
        SnakeGame game = new SnakeGame(10, 10);

        // Move to wall
        for (int i = 0; i < 20; i++) {
            MoveResult result = game.move(Direction.RIGHT);
            if (result == MoveResult.COLLISION_WALL) {
                assertTrue(game.isGameOver());
                return;
            }
        }
        fail("Should have hit wall");
    }

    @Test
    void testEatFood() {
        SnakeGame game = new SnakeGame(10, 10);
        Position food = game.getFood();

        // Move towards food
        while (!game.isGameOver()) {
            Direction dir = getDirectionTowards(
                game.getSnake().getHead(), food);
            MoveResult result = game.move(dir);

            if (result == MoveResult.ATE_FOOD) {
                assertEquals(10, game.getScore());
                assertEquals(2, game.getSnake().getLength());
                return;
            }
        }
    }
}
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.

```java
public class WrapAroundBoard extends Board {
    @Override
    public Position wrap(Position pos) {
        int row = pos.getRow();
        int col = pos.getCol();

        if (row < 0) row = getHeight() - 1;
        if (row >= getHeight()) row = 0;
        if (col < 0) col = getWidth() - 1;
        if (col >= getWidth()) col = 0;

        return new Position(row, col);
    }
}

public class SnakeGame {
    public MoveResult move(Direction direction) {
        Position nextHead = snake.getHead().move(direction);
        nextHead = board.wrap(nextHead);  // Wrap around
        // ... rest of logic (no wall collision check)
    }
}
```

### Q2: How would you save high scores?

```java
public class HighScoreManager {
    private final List<ScoreEntry> scores;
    private final String filename;

    public void addScore(String playerName, int score) {
        scores.add(new ScoreEntry(playerName, score, LocalDateTime.now()));
        scores.sort((a, b) -> b.getScore() - a.getScore());
        if (scores.size() > 10) {
            scores.remove(10);
        }
        save();
    }

    public List<ScoreEntry> getTopScores(int count) {
        return scores.subList(0, Math.min(count, scores.size()));
    }
}
```

### Q3: How would you add levels?

```java
public class Level {
    private final int number;
    private final int width;
    private final int height;
    private final Set<Position> obstacles;
    private final int foodToAdvance;
    private final int gameSpeed;
}

public class SnakeGame {
    private Level currentLevel;
    private int foodEatenInLevel;

    private void checkLevelComplete() {
        if (foodEatenInLevel >= currentLevel.getFoodToAdvance()) {
            advanceLevel();
        }
    }

    private void advanceLevel() {
        currentLevel = levelManager.getNextLevel();
        foodEatenInLevel = 0;
        // Reset snake position
    }
}
```

### Q4: What would you do differently with more time?

1. **Add graphics** - Use JavaFX/Swing for visuals
2. **Add animations** - Smooth movement, eating effects
3. **Add sound** - Background music, sound effects
4. **Add skins** - Different snake appearances
5. **Add achievements** - Unlock rewards
6. **Add online leaderboard** - Compete globally
