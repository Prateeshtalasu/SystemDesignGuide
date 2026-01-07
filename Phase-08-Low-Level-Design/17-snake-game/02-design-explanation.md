# 🐍 Snake Game - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Snake Game.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class       | Responsibility                 | Reason for Change         |
| ----------- | ------------------------------ | ------------------------- |
| `Position`  | Represent a coordinate         | Coordinate system changes |
| `Direction` | Define movement directions     | Direction logic changes   |
| `Snake`     | Manage snake body and movement | Snake behavior changes    |
| `Board`     | Define game boundaries         | Board rules change        |
| `SnakeGame` | Coordinate game logic          | Game rules change         |

**SRP in Action:**

```java
// Position ONLY represents coordinates
public class Position {
    private final int row, col;
    public Position move(Direction direction) { }
}

// Snake ONLY manages snake state
public class Snake {
    private final LinkedList<Position> body;
    public Position move(boolean grow) { }
    public boolean collidesWithSelf() { }
}

// SnakeGame coordinates everything
public class SnakeGame {
    public MoveResult move(Direction direction) { }
    private void spawnFood() { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Game Modes:**

```java
public interface GameMode {
    void onFoodEaten(SnakeGame game);
    void onMove(SnakeGame game);
    boolean isGameOver(SnakeGame game);
}

public class ClassicMode implements GameMode { }
public class TimeAttackMode implements GameMode { }
public class ObstacleMode implements GameMode { }
```

**Adding New Food Types:**

```java
public interface Food {
    Position getPosition();
    int getPoints();
    void onEaten(Snake snake);
}

public class NormalFood implements Food { }
public class BonusFood implements Food { }
public class SpeedFood implements Food { }
```

---

### 3. Liskov Substitution Principle (LSP)

**All food types work interchangeably:**

```java
public class SnakeGame {
    private Food currentFood;

    public MoveResult move(Direction direction) {
        if (nextHead.equals(currentFood.getPosition())) {
            score += currentFood.getPoints();
            currentFood.onEaten(snake);
            spawnFood();
        }
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Could be improved:**

```java
public interface Movable {
    Position move(Direction direction);
}

public interface Collidable {
    boolean collidesWith(Position position);
}

public class Snake implements Movable, Collidable { }
```

---

### 5. Dependency Inversion Principle (DIP)

**Better with DIP:**

```java
public interface FoodSpawner {
    Food spawn(Board board, Snake snake);
}

public interface CollisionDetector {
    boolean checkWallCollision(Position pos, Board board);
    boolean checkSelfCollision(Snake snake);
}

public class SnakeGame {
    private final FoodSpawner foodSpawner;
    private final CollisionDetector collisionDetector;
}
```

**Why We Use Concrete Classes in This LLD Implementation:**

For low-level design interviews, we intentionally use concrete classes instead of interfaces for the following reasons:

1. **Simple Game Model**: The system has a simple, well-defined game model (snake game) with single implementations for food spawning and collision detection. Additional interfaces don't add value for demonstrating core LLD skills.

2. **Core Focus**: LLD interviews focus on game loop, snake movement algorithms, and collision detection logic. Adding interface abstractions shifts focus away from these core concepts.

3. **Single Implementation**: There's no requirement for multiple food spawn or collision detection strategies in the interview context. The abstraction doesn't provide value for demonstrating LLD skills.

4. **Production vs Interview**: In production systems, we would absolutely extract `FoodSpawner` and `CollisionDetector` interfaces for:
   - Testability (mock spawners and detectors in unit tests)
   - Flexibility (swap strategies for different game modes)
   - Dependency injection (easier configuration)

**The Trade-off:**
- **Interview Scope**: Concrete classes focus on game loop and movement algorithms
- **Production Scope**: Interfaces provide testability and strategy flexibility

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Board manages grid, Snake manages snake state, Food represents food, SnakeGame coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new board types, food spawners, collision detectors) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All board implementations and collision detectors properly implement their contracts. They are substitutable. | N/A | - |
| **ISP** | PASS | Interfaces are minimal and focused. Movable and Collidable are well-segregated. No unused methods. | N/A | - |
| **DIP** | ACCEPTABLE (LLD Scope) | SnakeGame depends on concrete classes. For LLD interview scope, this is acceptable as it focuses on core game loop and collision detection algorithms. In production, we would depend on FoodSpawner and CollisionDetector interfaces. | See "Why We Use Concrete Classes" section above for detailed justification. This is an intentional design decision for interview context, not an oversight. | Interview: Simpler, focuses on core LLD skills. Production: More abstraction layers, but improves testability and flexibility |

---

## Design Patterns Used

### 1. State Pattern (Potential)

**Where:** Game status management

```java
public interface GameState {
    MoveResult handleMove(SnakeGame game, Direction direction);
    void handlePause(SnakeGame game);
}

public class RunningState implements GameState {
    @Override
    public MoveResult handleMove(SnakeGame game, Direction direction) {
        // Process move
    }
}

public class PausedState implements GameState {
    @Override
    public MoveResult handleMove(SnakeGame game, Direction direction) {
        return null;  // Can't move while paused
    }
}
```

---

### 2. Observer Pattern (Potential)

**Where:** Game events

```java
public interface GameObserver {
    void onFoodEaten(int score);
    void onGameOver(int finalScore);
    void onSnakeGrow(int newLength);
}

public class SoundManager implements GameObserver {
    @Override
    public void onFoodEaten(int score) {
        playEatSound();
    }
}
```

---

### 3. Strategy Pattern (Potential)

**Where:** AI snake movement

```java
public interface MovementStrategy {
    Direction getNextDirection(Snake snake, Position food, Board board);
}

public class GreedyStrategy implements MovementStrategy {
    @Override
    public Direction getNextDirection(Snake snake, Position food, Board board) {
        // Move towards food
    }
}

public class PathfindingStrategy implements MovementStrategy {
    @Override
    public Direction getNextDirection(Snake snake, Position food, Board board) {
        // Use A* to find safe path
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Using Array-Based Snake Body Instead of LinkedList

**What it is:** Store snake body as an array or ArrayList, shifting elements on each move instead of using LinkedList with addFirst/removeLast.

**Why rejected:** Array-based approach requires O(N) operations to shift elements on every move, while LinkedList provides O(1) addFirst and removeLast operations. For a game that moves frequently, LinkedList is more efficient. The array approach also requires manual index management and is more error-prone.

**What breaks:**
- Performance degrades with snake length (O(N) shift operations)
- More complex code for managing head/tail indices
- Higher memory overhead from array resizing
- Slower collision detection (linear search through array)

### Alternative 2: Using a 2D Array for the Entire Board State

**What it is:** Represent the entire board as a 2D array storing cell states (EMPTY, SNAKE, FOOD) instead of tracking snake body positions separately.

**Why rejected:** This approach wastes memory (O(W×H) space) and makes operations like "find empty cell for food" more expensive. The current design only stores snake body positions (O(N) where N << W×H) and calculates board state on demand. For large boards with small snakes, this is much more memory-efficient.

**What breaks:**
- Memory usage scales with board size, not snake size
- Food spawning requires scanning entire board (O(W×H))
- Board rendering requires full array traversal
- Less flexible for dynamic board sizes

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you implement wraparound walls?

**Answer:**

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

---

### Q2: How would you save high scores?

**Answer:**

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

---

### Q3: How would you add levels?

**Answer:**

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

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add graphics** - Use JavaFX/Swing for visuals
2. **Add animations** - Smooth movement, eating effects
3. **Add sound** - Background music, sound effects
4. **Add skins** - Different snake appearances
5. **Add achievements** - Unlock rewards
6. **Add online leaderboard** - Compete globally
7. **Add multiplayer** - Multiple snakes on same board
8. **Add power-ups** - Speed boost, invincibility

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation          | Complexity | Explanation                        |
| ------------------ | ---------- | ---------------------------------- |
| `move`             | O(N)       | N = snake length (collision check) |
| `collidesWithSelf` | O(N)       | Check each body segment            |
| `spawnFood`        | O(W×H)     | Find empty positions               |
| `containsPosition` | O(N)       | LinkedList search                  |

### Space Complexity

| Component  | Space  |
| ---------- | ------ | -------------------- |
| Snake body | O(N)   |
| Board      | O(1)   | Just dimensions      |
| Food spawn | O(W×H) | Empty positions list |

### Bottlenecks at Scale

**10x Usage (10×10 → 30×30 board):**
- Problem: Collision detection becomes more expensive (O(n) for snake length), game state updates slower, rendering overhead increases
- Solution: Optimize collision detection (spatial partitioning), use efficient rendering (dirty rectangle updates), cache board state
- Tradeoff: Additional code complexity, cache invalidation logic

**100x Usage (10×10 → 100×100 board):**
- Problem: Snake movement and collision checks become bottleneck, game state updates too slow for real-time gameplay, memory usage grows
- Solution: Use spatial indexing (grid-based collision detection), implement game state compression, use efficient data structures
- Tradeoff: Algorithm complexity increases, compression/decompression overhead

---

## Interview Follow-ups

### Q1: How would you add obstacles?

```java
public class SnakeGame {
    private final Set<Position> obstacles;

    public void addObstacle(Position pos) {
        obstacles.add(pos);
    }

    public MoveResult move(Direction direction) {
        Position nextHead = snake.getHead().move(direction);

        if (obstacles.contains(nextHead)) {
            status = GameStatus.GAME_OVER;
            return MoveResult.COLLISION_OBSTACLE;
        }
        // ... rest of logic
    }
}
```

### Q2: How would you implement multiplayer?

```java
public class MultiplayerSnakeGame {
    private final List<Snake> snakes;
    private final List<Integer> scores;

    public MoveResult move(int playerIndex, Direction direction) {
        Snake snake = snakes.get(playerIndex);
        // Check collision with other snakes
        for (int i = 0; i < snakes.size(); i++) {
            if (i != playerIndex && snakes.get(i).containsPosition(nextHead)) {
                return MoveResult.COLLISION_OTHER_SNAKE;
            }
        }
    }
}
```

### Q3: How would you add power-ups?

```java
public interface PowerUp {
    void apply(Snake snake, SnakeGame game);
    int getDuration();
}

public class SpeedBoost implements PowerUp {
    @Override
    public void apply(Snake snake, SnakeGame game) {
        game.setSpeed(game.getSpeed() * 2);
    }
}

public class Invincibility implements PowerUp {
    @Override
    public void apply(Snake snake, SnakeGame game) {
        snake.setInvincible(true);
    }
}
```

### Q4: How would you optimize collision detection?

```java
public class Snake {
    private final Set<Position> bodySet;  // O(1) lookup
    private final LinkedList<Position> bodyList;  // Ordered

    public boolean containsPosition(Position pos) {
        return bodySet.contains(pos);  // O(1) instead of O(N)
    }

    public Position move(boolean grow) {
        Position newHead = getHead().move(direction);
        bodyList.addFirst(newHead);
        bodySet.add(newHead);

        if (!grow) {
            Position tail = bodyList.removeLast();
            bodySet.remove(tail);
        }
        return newHead;
    }
}
```
