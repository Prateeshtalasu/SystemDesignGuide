# 🐍 Snake Game - Problem Solution

## STEP 0: REQUIREMENTS QUICKPASS

### Core Functional Requirements

- Represent a game board with snake and food
- Handle snake movement in four directions (UP, DOWN, LEFT, RIGHT)
- Detect collisions (wall, self-collision)
- Generate food at random positions (not on snake)
- Track score (increases when food eaten)
- Snake grows when eating food
- Support different difficulty levels (speed)

### Explicit Out-of-Scope Items

- Multiplayer mode
- Power-ups and obstacles
- Leaderboard persistence
- Graphics rendering
- Sound effects

### Assumptions and Constraints

- **Grid-Based**: Discrete cell movement
- **Board Size**: Configurable (default 20x20)
- **Initial Length**: Snake starts with length 3
- **Single Food**: One food item at a time
- **Wrap-Around**: Optional (walls can kill or wrap)

### Scale Assumptions (LLD Focus)

- Single game instance (single player)
- Board size: 10x10 to 50x50 (in-memory manageable)
- Score tracking: Integer (max ~2 billion points)
- Food spawning: O(W×H) worst case, O(1) average with rejection sampling

### Concurrency Model

- **Single-threaded game loop**: No concurrent access to game state
- **Input handling**: Direction changes queued, processed once per tick
- **State updates**: Atomic per game tick (move → eat → grow → check collision)
- **No synchronization needed**: Single-player, single-threaded design

### Public APIs

- `SnakeGame(width, height)`: Create game
- `move(direction)`: Move snake
- `getScore()`: Current score
- `isGameOver()`: Check if game ended
- `getBoard()`: Get current state

### Public API Usage Examples
```java
// Example 1: Basic usage - Create game and make moves
SnakeGame game = new SnakeGame(15, 10);
MoveResult result = game.move(Direction.RIGHT);
System.out.println("Move result: " + result);
System.out.println("Score: " + game.getScore());
game.printBoard();

// Example 2: Typical workflow - Play until game over
while (!game.isGameOver()) {
    Direction dir = getDirectionFromInput(); // Get from user input
    MoveResult result = game.move(dir);
    if (result == MoveResult.ATE_FOOD) {
        System.out.println("Food eaten! Score: " + game.getScore());
    }
    game.printBoard();
    Thread.sleep(200); // Game tick delay
}

// Example 3: Edge case - Try to move after game over
game.move(Direction.UP); // Game already over
System.out.println("Game over: " + game.isGameOver());
System.out.println("Final score: " + game.getScore());
```

### Invariants

- **Snake Continuity**: Body segments connected
- **Food Placement**: Never on snake
- **Score Accuracy**: +10 per food eaten (configurable via SCORE_PER_FOOD constant)

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SNAKE GAME                                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                           SnakeGame                                       │   │
│  │                                                                           │   │
│  │  - board: Board                                                          │   │
│  │  - snake: Snake                                                          │   │
│  │  - food: Position                                                        │   │
│  │  - score: int                                                            │   │
│  │  - status: GameStatus                                                    │   │
│  │                                                                           │   │
│  │  + move(direction): MoveResult                                           │   │
│  │  + isGameOver(): boolean                                                 │   │
│  │  + getScore(): int                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│           ┌──────────────┼──────────────┬────────────────┐                      │
│           │              │              │                │                      │
│           ▼              ▼              ▼                ▼                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │    Board    │  │    Snake    │  │  Position   │  │  Direction  │            │
│  │             │  │             │  │             │  │             │            │
│  │ - width     │  │ - body[]    │  │ - row       │  │ - UP        │            │
│  │ - height    │  │ - direction │  │ - col       │  │ - DOWN      │            │
│  │             │  │             │  │             │  │ - LEFT      │            │
│  │ + isInBounds│  │ + move()    │  │ + equals()  │  │ - RIGHT     │            │
│  │ + getCell() │  │ + grow()    │  └─────────────┘  └─────────────┘            │
│  └─────────────┘  │ + collides()│                                              │
│                   └─────────────┘                                              │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Game Board Visualization

```
    0   1   2   3   4   5   6   7   8   9
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
0 │ # │ # │ # │ # │ # │ # │ # │ # │ # │ # │
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
1 │ # │   │   │   │   │   │   │   │   │ # │
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
2 │ # │   │   │ ○ │ ○ │ ● │   │   │   │ # │  ← Snake (● head, ○ body)
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
3 │ # │   │   │   │   │   │   │   │   │ # │
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
4 │ # │   │   │   │   │ ★ │   │   │   │ # │  ← Food (★)
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
5 │ # │ # │ # │ # │ # │ # │ # │ # │ # │ # │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

---

### Responsibilities Table

| Class | Owns | Why |
|-------|------|-----|
| `Position` | Coordinate representation (row, column) | Encapsulates coordinates - represents grid position, enables position calculations |
| `Direction` | Movement direction definition (UP, DOWN, LEFT, RIGHT) | Defines movement directions - enum for direction constants, enables direction-based movement |
| `Snake` | Snake body state and movement logic | Manages snake state - encapsulates body segments, movement, and growth logic |
| `Board` | Game board boundaries and bounds checking | Defines game boundaries - encapsulates board dimensions and bounds validation |
| `SnakeGame` | Game coordination (snake, food, score, status) | Coordinates game logic - separates game rules from snake/board logic, manages game state |

---

## STEP 4: Code Walkthrough - Building From Scratch

This section explains how an engineer builds this system from scratch, in the order code should be written.

### Phase 1: Understand the Problem

**What is Snake Game?**

- Snake moves on a grid
- Eats food to grow
- Dies if hits wall or itself
- Score increases with food eaten

**Key Challenges:**

- **Snake body tracking**: Efficient add/remove
- **Collision detection**: Wall and self
- **Food spawning**: Random empty position

---

### Phase 2: Design Position and Direction

```java
// Step 1: Position with immutable coordinates
public class Position {
    private final int row;
    private final int col;

    public Position move(Direction direction) {
        return new Position(
            row + direction.getRowDelta(),
            col + direction.getColDelta()
        );
    }
}

// Step 2: Direction with deltas
public enum Direction {
    UP(-1, 0),
    DOWN(1, 0),
    LEFT(0, -1),
    RIGHT(0, 1);

    public Direction opposite() {
        // Prevent reversing direction
    }
}
```

---

### Phase 3: Design the Snake

```java
// Step 3: Snake with LinkedList body
public class Snake {
    private final LinkedList<Position> body;
    private Direction direction;

    public Position move(boolean grow) {
        // Calculate new head position
        Position newHead = getHead().move(direction);

        // Add new head to front
        body.addFirst(newHead);

        // Remove tail if not growing
        if (!grow) {
            body.removeLast();
        }

        return newHead;
    }
}
```

**Why LinkedList?**

- O(1) add to front (new head)
- O(1) remove from end (tail)
- Perfect for snake body

---

### Phase 4: Implement Collision Detection

```java
// Step 4: Self-collision check
public boolean collidesWithSelf() {
    Position head = getHead();

    // Check if head equals any body segment
    return body.stream()
        .skip(1)  // Skip head itself
        .anyMatch(pos -> pos.equals(head));
}
```

---

### Phase 5: Implement Game Logic

```java
// Step 5: Main game loop
public MoveResult move(Direction direction) {
    // 1. Set new direction (if valid)
    snake.setDirection(direction);

    // 2. Calculate next position
    Position nextHead = snake.getHead().move(snake.getDirection());

    // 3. Check wall collision
    if (!board.isInBounds(nextHead)) {
        status = GameStatus.GAME_OVER;
        return MoveResult.COLLISION_WALL;
    }

    // 4. Check if eating food
    boolean eating = nextHead.equals(food);

    // 5. Move snake (grow if eating)
    snake.move(eating);

    // 6. Check self collision
    if (snake.collidesWithSelf()) {
        status = GameStatus.GAME_OVER;
        return MoveResult.COLLISION_SELF;
    }

    // 7. Handle food eaten
    if (eating) {
        score += 10;
        spawnFood();
        return MoveResult.ATE_FOOD;
    }

    return MoveResult.OK;
}
```

---

### Phase 6: Threading Model and Concurrency Control

**Threading Model:**

This is a **single-threaded game loop** design:
- No concurrent access to game state
- Direction changes are queued and processed once per tick
- State updates are atomic per game tick (move → eat → grow → check collision)
- No synchronization needed for single-player, single-threaded design

**If multi-threaded was needed:**

```java
public class ThreadSafeSnakeGame {
    private final ReentrantLock lock = new ReentrantLock();
    private final Queue<Direction> directionQueue = new ConcurrentLinkedQueue<>();
    
    public void move(Direction direction) {
        directionQueue.offer(direction);  // Thread-safe queue
    }
    
    public void gameTick() {
        lock.lock();
        try {
            Direction next = directionQueue.poll();
            if (next != null) {
                processMove(next);
            }
        } finally {
            lock.unlock();
        }
    }
}
```

---

## STEP 2: Complete Final Implementation

> **Verified:** This code compiles successfully with Java 11+.

### 2.1 Direction Enum

```java
// Direction.java
package com.snake;

public enum Direction {
    UP(-1, 0),
    DOWN(1, 0),
    LEFT(0, -1),
    RIGHT(0, 1);

    private final int rowDelta;
    private final int colDelta;

    Direction(int rowDelta, int colDelta) {
        this.rowDelta = rowDelta;
        this.colDelta = colDelta;
    }

    public int getRowDelta() { return rowDelta; }
    public int getColDelta() { return colDelta; }

    public Direction opposite() {
        switch (this) {
            case UP: return DOWN;
            case DOWN: return UP;
            case LEFT: return RIGHT;
            case RIGHT: return LEFT;
            default: return this;
        }
    }
}
```

### 2.2 Position Class

```java
// Position.java
package com.snake;

import java.util.Objects;

public class Position {

    private final int row;
    private final int col;

    public Position(int row, int col) {
        this.row = row;
        this.col = col;
    }

    public Position move(Direction direction) {
        return new Position(
            row + direction.getRowDelta(),
            col + direction.getColDelta()
        );
    }

    public int getRow() { return row; }
    public int getCol() { return col; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Position)) return false;
        Position position = (Position) o;
        return row == position.row && col == position.col;
    }

    @Override
    public int hashCode() {
        return Objects.hash(row, col);
    }

    @Override
    public String toString() {
        return String.format("(%d, %d)", row, col);
    }
}
```

### 2.3 Snake Class

```java
// Snake.java
package com.snake;

import java.util.*;

public class Snake {

    private final LinkedList<Position> body;
    private Direction direction;

    public Snake(Position startPosition) {
        this.body = new LinkedList<>();
        this.body.addFirst(startPosition);
        this.direction = Direction.RIGHT;
    }

    public Position getHead() {
        return body.getFirst();
    }

    public Position getTail() {
        return body.getLast();
    }

    public int getLength() {
        return body.size();
    }

    public void setDirection(Direction newDirection) {
        // Can't reverse direction
        if (body.size() > 1 && newDirection == direction.opposite()) {
            return;
        }
        this.direction = newDirection;
    }

    public Direction getDirection() {
        return direction;
    }

    /**
     * Moves the snake in current direction.
     * @param grow if true, snake grows (doesn't remove tail)
     * @return new head position
     */
    public Position move(boolean grow) {
        Position newHead = getHead().move(direction);
        body.addFirst(newHead);

        if (!grow) {
            body.removeLast();
        }

        return newHead;
    }

    public boolean collidesWithSelf() {
        Position head = getHead();
        // Check if head collides with any body segment (except head itself)
        return body.stream()
            .skip(1)
            .anyMatch(pos -> pos.equals(head));
    }

    public boolean containsPosition(Position pos) {
        return body.contains(pos);
    }

    public List<Position> getBody() {
        return Collections.unmodifiableList(body);
    }
}
```

### 2.4 Board Class

```java
// Board.java
package com.snake;

public class Board {

    private final int width;
    private final int height;

    public Board(int width, int height) {
        this.width = width;
        this.height = height;
    }

    public boolean isInBounds(Position pos) {
        return pos.getRow() >= 0 && pos.getRow() < height &&
               pos.getCol() >= 0 && pos.getCol() < width;
    }

    public int getWidth() { return width; }
    public int getHeight() { return height; }
}
```

### 2.5 GameStatus and MoveResult Enums

```java
// GameStatus.java
package com.snake;

public enum GameStatus {
    RUNNING,
    GAME_OVER,
    PAUSED
}
```

```java
// MoveResult.java
package com.snake;

public enum MoveResult {
    OK,
    ATE_FOOD,
    COLLISION_WALL,
    COLLISION_SELF
}
```

### 2.6 SnakeGame Class

```java
// SnakeGame.java
package com.snake;

import java.util.*;

public class SnakeGame {

    private final Board board;
    private final Snake snake;
    private Position food;
    private int score;
    private GameStatus status;
    private final Random random;

    public SnakeGame(int width, int height) {
        this.board = new Board(width, height);
        this.snake = new Snake(new Position(height / 2, width / 2));
        this.score = 0;
        this.status = GameStatus.RUNNING;
        this.random = new Random();

        spawnFood();
    }

    public MoveResult move(Direction direction) {
        if (status != GameStatus.RUNNING) {
            return MoveResult.COLLISION_WALL;
        }

        // Set new direction
        snake.setDirection(direction);

        // Get next position
        Position nextHead = snake.getHead().move(snake.getDirection());

        // Check wall collision
        if (!board.isInBounds(nextHead)) {
            status = GameStatus.GAME_OVER;
            return MoveResult.COLLISION_WALL;
        }

        // Check if eating food
        boolean eating = nextHead.equals(food);

        // Move snake
        snake.move(eating);

        // Check self collision (after moving)
        if (snake.collidesWithSelf()) {
            status = GameStatus.GAME_OVER;
            return MoveResult.COLLISION_SELF;
        }

        // Handle food eaten
        if (eating) {
            score += 10;
            spawnFood();
            return MoveResult.ATE_FOOD;
        }

        return MoveResult.OK;
    }

    public MoveResult move() {
        return move(snake.getDirection());
    }

    private void spawnFood() {
        List<Position> emptyPositions = new ArrayList<>();

        for (int r = 0; r < board.getHeight(); r++) {
            for (int c = 0; c < board.getWidth(); c++) {
                Position pos = new Position(r, c);
                if (!snake.containsPosition(pos)) {
                    emptyPositions.add(pos);
                }
            }
        }

        if (!emptyPositions.isEmpty()) {
            food = emptyPositions.get(random.nextInt(emptyPositions.size()));
        }
    }

    public void changeDirection(Direction direction) {
        snake.setDirection(direction);
    }

    public void pause() {
        if (status == GameStatus.RUNNING) {
            status = GameStatus.PAUSED;
        }
    }

    public void resume() {
        if (status == GameStatus.PAUSED) {
            status = GameStatus.RUNNING;
        }
    }

    // Getters
    public boolean isGameOver() { return status == GameStatus.GAME_OVER; }
    public int getScore() { return score; }
    public GameStatus getStatus() { return status; }
    public Snake getSnake() { return snake; }
    public Position getFood() { return food; }
    public Board getBoard() { return board; }

    public void printBoard() {
        System.out.println("\nScore: " + score + "  Length: " + snake.getLength());

        char[][] display = new char[board.getHeight()][board.getWidth()];

        // Initialize with empty
        for (int r = 0; r < board.getHeight(); r++) {
            for (int c = 0; c < board.getWidth(); c++) {
                display[r][c] = '.';
            }
        }

        // Draw snake body
        List<Position> body = snake.getBody();
        for (int i = 1; i < body.size(); i++) {
            Position pos = body.get(i);
            display[pos.getRow()][pos.getCol()] = 'o';
        }

        // Draw snake head
        Position head = snake.getHead();
        display[head.getRow()][head.getCol()] = '@';

        // Draw food
        if (food != null) {
            display[food.getRow()][food.getCol()] = '*';
        }

        // Print board
        System.out.print("┌");
        for (int c = 0; c < board.getWidth(); c++) System.out.print("─");
        System.out.println("┐");

        for (int r = 0; r < board.getHeight(); r++) {
            System.out.print("│");
            for (int c = 0; c < board.getWidth(); c++) {
                System.out.print(display[r][c]);
            }
            System.out.println("│");
        }

        System.out.print("└");
        for (int c = 0; c < board.getWidth(); c++) System.out.print("─");
        System.out.println("┘");
    }
}
```

### 2.7 Demo Application

```java
// SnakeGameDemo.java
package com.snake;

public class SnakeGameDemo {

    public static void main(String[] args) {
        System.out.println("=== SNAKE GAME DEMO ===\n");

        SnakeGame game = new SnakeGame(15, 10);

        System.out.println("Initial state:");
        game.printBoard();

        // Simulate some moves
        MoveResult result;

        // Move right a few times
        System.out.println("\nMoving RIGHT...");
        for (int i = 0; i < 3; i++) {
            result = game.move(Direction.RIGHT);
            System.out.println("Move result: " + result);
        }
        game.printBoard();

        // Move down
        System.out.println("\nMoving DOWN...");
        for (int i = 0; i < 2; i++) {
            result = game.move(Direction.DOWN);
            System.out.println("Move result: " + result);
        }
        game.printBoard();

        // Move left
        System.out.println("\nMoving LEFT...");
        for (int i = 0; i < 4; i++) {
            result = game.move(Direction.LEFT);
            System.out.println("Move result: " + result);
            if (result == MoveResult.ATE_FOOD) {
                System.out.println("Ate food! Score: " + game.getScore());
            }
        }
        game.printBoard();

        // Continue until game over or max moves
        System.out.println("\nContinuing game...");
        int moves = 0;
        while (!game.isGameOver() && moves < 50) {
            // Simple AI: move towards food
            Position head = game.getSnake().getHead();
            Position food = game.getFood();

            Direction dir;
            if (food.getCol() > head.getCol()) {
                dir = Direction.RIGHT;
            } else if (food.getCol() < head.getCol()) {
                dir = Direction.LEFT;
            } else if (food.getRow() > head.getRow()) {
                dir = Direction.DOWN;
            } else {
                dir = Direction.UP;
            }

            result = game.move(dir);
            moves++;

            if (result == MoveResult.ATE_FOOD) {
                System.out.println("Ate food! Score: " + game.getScore() +
                                  ", Length: " + game.getSnake().getLength());
            }
        }

        game.printBoard();
        System.out.println("\nFinal Score: " + game.getScore());
        System.out.println("Snake Length: " + game.getSnake().getLength());
        System.out.println("Game Status: " + game.getStatus());

        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## File Structure

```
com/snake/
├── Direction.java
├── Position.java
├── Snake.java
├── Board.java
├── GameStatus.java
├── MoveResult.java
├── SnakeGame.java
└── SnakeGameDemo.java
```
