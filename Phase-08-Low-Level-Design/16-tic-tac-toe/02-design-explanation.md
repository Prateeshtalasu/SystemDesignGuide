# ⭕ Tic-Tac-Toe Game - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Tic-Tac-Toe Game.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Board` | Manage game board state | Board rules change |
| `Player` | Represent a player | Player model changes |
| `AIPlayer` | AI move calculation | AI algorithm changes |
| `Move` | Represent a single move | Move model changes |
| `Game` | Coordinate game flow | Game rules change |

**SRP in Action:**

```java
// Board ONLY manages board state
public class Board {
    public boolean placeSymbol(int row, int col, Symbol symbol) { }
    public boolean checkWin(Symbol symbol) { }
    public boolean isFull() { }
}

// Player ONLY represents player identity
public class Player {
    private final String name;
    private final Symbol symbol;
}

// AIPlayer ONLY handles AI logic
public class AIPlayer extends Player {
    public Move findBestMove(Board board) { }
    private int minimax(...) { }
}

// Game ONLY coordinates game flow
public class Game {
    public boolean makeMove(int row, int col) { }
    private void updateGameStatus() { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Player Types:**

```java
// No changes to existing code
public class RandomAIPlayer extends Player {
    public Move makeMove(Board board) {
        // Random valid move
    }
}

public class NetworkPlayer extends Player {
    public Move makeMove(Board board) {
        // Get move from network
    }
}
```

**Extending to NxN Board:**

```java
// Board already supports any size
public Board(int size) {
    this.size = size;
    this.grid = new Symbol[size][size];
}

// Win checking works for any size
private boolean checkLine(Symbol symbol, int startRow, int startCol,
                         int rowDelta, int colDelta) {
    for (int i = 0; i < size; i++) {  // Uses size
        // ...
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**AIPlayer can substitute Player:**

```java
public class Game {
    private final Player[] players;  // Can be Player or AIPlayer
    
    public boolean makeMove(int row, int col) {
        Player currentPlayer = getCurrentPlayer();
        
        if (currentPlayer instanceof AIPlayer) {
            move = ((AIPlayer) currentPlayer).makeMove(board, row, col);
        } else {
            move = currentPlayer.makeMove(board, row, col);
        }
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Could be improved:**

```java
public interface Playable {
    Move makeMove(Board board, int row, int col);
}

public interface AICapable {
    Move findBestMove(Board board);
}

public class Player implements Playable { }
public class AIPlayer implements Playable, AICapable { }
```

---

### 5. Dependency Inversion Principle (DIP)

**Better with DIP:**

```java
public interface MoveStrategy {
    Move calculateMove(Board board, Symbol symbol);
}

public class MinimaxStrategy implements MoveStrategy {
    @Override
    public Move calculateMove(Board board, Symbol symbol) {
        // Minimax algorithm
    }
}

public class RandomStrategy implements MoveStrategy {
    @Override
    public Move calculateMove(Board board, Symbol symbol) {
        // Random move
    }
}

public class AIPlayer extends Player {
    private final MoveStrategy strategy;
    
    public AIPlayer(String name, Symbol symbol, MoveStrategy strategy) {
        super(name, symbol);
        this.strategy = strategy;
    }
}
```

**Why We Use Concrete Classes in This LLD Implementation:**

For low-level design interviews, we intentionally use concrete classes instead of additional interfaces for the following reasons:

1. **Simple Game Model**: The system has a simple, well-defined game model (tic-tac-toe) with two player types (human/AI). Additional interfaces don't add value for demonstrating core LLD skills.

2. **Core Focus**: LLD interviews focus on game state management, win detection algorithms, and board representation. Adding interface abstractions shifts focus away from these core concepts.

3. **Interview Time Constraints**: Implementing full interface hierarchies takes time away from demonstrating more critical LLD concepts like state transitions and game logic.

4. **Production vs Interview**: In production systems, we would absolutely extract `Playable`, `AICapable`, and `MoveStrategy` interfaces for:
   - Testability (mock players and strategies in unit tests)
   - Flexibility (swap AI strategies, add new player types)
   - Dependency injection (easier configuration)

**The Trade-off:**
- **Interview Scope**: Concrete classes focus on game logic and state management
- **Production Scope**: Interfaces provide testability and strategy flexibility

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Board manages game state, Player handles moves, Game coordinates, AIPlayer implements AI. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new player types, strategies) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All Player implementations (HumanPlayer, AIPlayer) properly implement the Player interface contract. They are substitutable. | N/A | - |
| **ISP** | ACCEPTABLE (LLD Scope) | Could benefit from Playable and AICapable interfaces. For LLD interview scope, using concrete classes is acceptable as it focuses on core game logic. In production, we would extract Playable and AICapable interfaces. | See "Why We Use Concrete Classes" section above for detailed justification. This is an intentional design decision for interview context. | Interview: Simpler, focuses on core LLD skills. Production: More interfaces/files, but increases flexibility |
| **DIP** | ACCEPTABLE (LLD Scope) | Game depends on concrete Player implementations. For LLD interview scope, this is acceptable as it focuses on core game algorithms. In production, we would depend on MoveStrategy interface. | See "Why We Use Concrete Classes" section above for detailed justification. This is an intentional design decision for interview context, not an oversight. | Interview: Simpler, focuses on core LLD skills. Production: More abstraction layers, but improves testability and strategy flexibility |

---

## Design Patterns Used

### 1. Strategy Pattern (Potential)

**Where:** AI move calculation

```java
public interface MoveStrategy {
    Move calculateMove(Board board, Symbol symbol);
}

public class MinimaxStrategy implements MoveStrategy { }
public class AlphaBetaStrategy implements MoveStrategy { }
public class RandomStrategy implements MoveStrategy { }
```

---

### 2. State Pattern (Potential)

**Where:** Game status management

```java
public interface GameState {
    boolean makeMove(Game game, int row, int col);
    boolean isGameOver();
}

public class InProgressState implements GameState {
    @Override
    public boolean makeMove(Game game, int row, int col) {
        // Process move, check win, switch player
    }
}

public class GameOverState implements GameState {
    @Override
    public boolean makeMove(Game game, int row, int col) {
        return false;  // Can't make moves
    }
}
```

---

### 3. Command Pattern (Potential)

**Where:** Move history and undo

```java
public interface Command {
    void execute();
    void undo();
}

public class MoveCommand implements Command {
    private final Board board;
    private final int row, col;
    private final Symbol symbol;
    
    @Override
    public void execute() {
        board.placeSymbol(row, col, symbol);
    }
    
    @Override
    public void undo() {
        board.removeSymbol(row, col);
    }
}
```

---

## Minimax Algorithm Explained

### Basic Concept

```
Minimax: Find the optimal move assuming both players play perfectly

AI (Maximizer): Wants to maximize score
Opponent (Minimizer): Wants to minimize score

Score values:
  +10: AI wins
  -10: Opponent wins
  0: Draw
```

### Algorithm Flow

```
minimax(board, depth, isMaximizing):
    if AI wins: return +10 - depth
    if opponent wins: return -10 + depth
    if draw: return 0
    
    if isMaximizing:
        bestScore = -∞
        for each empty cell:
            place AI symbol
            score = minimax(board, depth+1, false)
            remove symbol
            bestScore = max(bestScore, score)
        return bestScore
    else:
        bestScore = +∞
        for each empty cell:
            place opponent symbol
            score = minimax(board, depth+1, true)
            remove symbol
            bestScore = min(bestScore, score)
        return bestScore
```

### Alpha-Beta Pruning

```
Optimization: Skip branches that can't affect the result

alpha: Best score maximizer can guarantee
beta: Best score minimizer can guarantee

if beta <= alpha:
    prune (skip remaining branches)
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single List for Board State

**What it is:**

```java
// Rejected approach
public class Board {
    private List<Symbol> cells;  // Flat list of 9 cells
    
    public void placeSymbol(int row, int col, Symbol symbol) {
        int index = row * 3 + col;
        cells.set(index, symbol);
    }
    
    public boolean checkWin(Symbol symbol) {
        // Check all rows, cols, diagonals by calculating indices
        // Hard to read: checkRow(0) means checking indices 0,1,2
    }
}
```

**Why rejected:**
- Less intuitive - requires index calculation for row/col access
- Harder to visualize and debug
- Win condition checking becomes more complex with index arithmetic
- Not as readable as 2D array representation

**What breaks:**
- Code readability suffers (index calculations everywhere)
- Debugging becomes harder (can't easily visualize board)
- Extension to NxN boards requires changing all index calculations
- Win condition logic becomes error-prone

---

### Alternative 2: Set-Based Win Checking

**What it is:**

```java
// Rejected approach
public class Board {
    private Set<Position> xPositions;
    private Set<Position> oPositions;
    
    public boolean checkWin(Symbol symbol) {
        Set<Position> positions = (symbol == Symbol.X) ? xPositions : oPositions;
        
        // Check if any win pattern is subset of positions
        for (Set<Position> winPattern : WIN_PATTERNS) {
            if (positions.containsAll(winPattern)) {
                return true;
            }
        }
        return false;
    }
}
```

**Why rejected:**
- More memory overhead (storing positions in sets)
- Set operations for each win check (containsAll)
- Harder to get board state for display
- Over-engineered for simple 3x3 board

**What breaks:**
- Performance: Set operations slower than direct array checks
- Memory: Additional Set storage per symbol
- Complexity: More complex than needed for small board
- Display: Harder to iterate board for rendering

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you optimize for larger boards?

**Answer:**

```java
// Use transposition table (memoization)
public class AIPlayer {
    private final Map<String, Integer> transpositionTable;
    
    private int minimax(Board board, ...) {
        String key = board.getStateKey();
        if (transpositionTable.containsKey(key)) {
            return transpositionTable.get(key);
        }
        
        int score = // ... calculate
        transpositionTable.put(key, score);
        return score;
    }
}
```

---

### Q2: How would you add game replay?

**Answer:**

```java
public class GameReplay {
    private final List<Move> moves;
    private int currentIndex;
    
    public void stepForward() {
        if (currentIndex < moves.size()) {
            Move move = moves.get(currentIndex++);
            board.placeSymbol(move.getRow(), move.getCol(), move.getSymbol());
        }
    }
    
    public void stepBackward() {
        if (currentIndex > 0) {
            Move move = moves.get(--currentIndex);
            board.removeSymbol(move.getRow(), move.getCol());
        }
    }
}
```

---

### Q3: How would you save/load games?

**Answer:**

```java
public class GameSaver {
    public void save(Game game, String filename) {
        try (ObjectOutputStream out = new ObjectOutputStream(
                new FileOutputStream(filename))) {
            out.writeObject(game.getMoveHistory());
        }
    }
    
    public Game load(String filename) {
        List<Move> moves = // load from file
        Game game = new Game(player1, player2);
        for (Move move : moves) {
            game.makeMove(move.getRow(), move.getCol());
        }
        return game;
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add GUI** - Swing/JavaFX interface
2. **Add sound effects** - Move sounds, win celebration
3. **Add statistics** - Win/loss tracking
4. **Add tournaments** - Multiple game series
5. **Add custom symbols** - Emoji or images
6. **Add hints** - Show best move for human
7. **Add online multiplayer** - Network play
8. **Add difficulty levels** - Adjust AI strength

---

### Q5: How would you implement thread-safe game for concurrent access?

**Answer:**

```java
public class ThreadSafeGame {
    private final ReentrantLock lock = new ReentrantLock();
    private Board board;
    private Player currentPlayer;
    
    public boolean makeMove(int row, int col) {
        lock.lock();
        try {
            if (isGameOver()) {
                throw new IllegalStateException("Game is over");
            }
            if (!board.isEmpty(row, col)) {
                throw new IllegalArgumentException("Cell occupied");
            }
            board.placeSymbol(row, col, currentPlayer.getSymbol());
            switchPlayer();
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    public Player getCurrentPlayer() {
        lock.lock();
        try {
            return currentPlayer;
        } finally {
            lock.unlock();
        }
    }
}
```

**Considerations:**
- Lock protects game state during moves
- Prevents race conditions on concurrent move attempts
- Read operations also protected for consistency
- Lock granularity balances safety and performance

---

### Q6: How would you implement move validation with better error messages?

**Answer:**

```java
public class ValidatedGame {
    public enum ValidationResult {
        VALID,
        CELL_OCCUPIED,
        OUT_OF_BOUNDS,
        GAME_OVER,
        WRONG_PLAYER_TURN
    }
    
    public ValidationResult validateMove(int row, int col, Player player) {
        if (isGameOver()) {
            return ValidationResult.GAME_OVER;
        }
        if (player != getCurrentPlayer()) {
            return ValidationResult.WRONG_PLAYER_TURN;
        }
        if (!board.isValidPosition(row, col)) {
            return ValidationResult.OUT_OF_BOUNDS;
        }
        if (!board.isEmpty(row, col)) {
            return ValidationResult.CELL_OCCUPIED;
        }
        return ValidationResult.VALID;
    }
    
    public boolean makeMove(int row, int col, Player player) {
        ValidationResult result = validateMove(row, col, player);
        if (result != ValidationResult.VALID) {
            throw new IllegalMoveException(result, getErrorMessage(result));
        }
        return executeMove(row, col, player);
    }
}
```

**Benefits:**
- Clear error messages for each validation failure
- Separates validation from execution
- Reusable validation logic
- Better user experience

---

### Q7: How would you implement game statistics tracking?

**Answer:**

```java
public class GameStatistics {
    private final Map<Player, PlayerStats> playerStats;
    
    private static class PlayerStats {
        int wins;
        int losses;
        int draws;
        int totalGames;
        double winRate;
    }
    
    public void recordGame(Game game) {
        Player winner = game.getWinner();
        Player[] players = game.getPlayers();
        
        for (Player player : players) {
            PlayerStats stats = playerStats.get(player);
            stats.totalGames++;
            
            if (winner == null) {
                stats.draws++;
            } else if (winner == player) {
                stats.wins++;
            } else {
                stats.losses++;
            }
            
            stats.winRate = (double) stats.wins / stats.totalGames;
        }
    }
    
    public PlayerStats getStats(Player player) {
        return playerStats.get(player);
    }
}
```

**Features:**
- Win/loss/draw tracking per player
- Win rate calculation
- Historical game data
- Can extend to move-level analytics

---

### Q8: How would you optimize minimax with iterative deepening?

**Answer:**

```java
public class OptimizedAIPlayer {
    public Move findBestMove(Board board, int maxTimeMs) {
        long startTime = System.currentTimeMillis();
        Move bestMove = null;
        
        // Start with depth 1, increase until time runs out
        for (int depth = 1; depth <= MAX_DEPTH; depth++) {
            long elapsed = System.currentTimeMillis() - startTime;
            if (elapsed >= maxTimeMs) {
                break;  // Return best move found so far
            }
            
            Move move = minimax(board, depth, true);
            if (move != null) {
                bestMove = move;
            }
        }
        
        return bestMove;
    }
    
    private Move minimax(Board board, int depth, boolean isMaximizing) {
        // Same minimax but with depth limit
        if (depth == 0 || board.isGameOver()) {
            return new Move(evaluate(board), null);
        }
        // ... rest of minimax
    }
}
```

**Benefits:**
- Guarantees move within time limit
- Progressively better moves with more time
- Graceful degradation under time pressure
- Better than fixed-depth search

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `placeSymbol` | O(1) | Array access |
| `checkWin` | O(N) | N = board size |
| `minimax` | O(b^d) | b = branching, d = depth |
| With pruning | O(b^(d/2)) | Significant improvement |

For 3x3 board:
- Without pruning: O(9!) = 362,880
- With pruning: ~O(9^4.5) ≈ 2,000

### Space Complexity

| Component | Space |
|-----------|-------|
| Board | O(N²) |
| Minimax recursion | O(N²) | depth |
| Move history | O(N²) |

### Bottlenecks at Scale

**10x Usage (3×3 → 5×5 board):**
- Problem: Minimax search space grows exponentially (O(b^d)), AI move calculation becomes slow, memory for game tree increases
- Solution: Implement alpha-beta pruning, use iterative deepening, cache evaluated positions (transposition table)
- Tradeoff: Algorithm complexity increases, cache memory overhead

**100x Usage (3×3 → 10×10 board):**
- Problem: Minimax becomes computationally infeasible, game tree too large for memory, real-time AI moves impossible
- Solution: Use heuristic evaluation instead of full minimax, implement machine learning-based move selection, limit search depth
- Tradeoff: AI quality may decrease, need ML model training infrastructure


### Q1: How would you extend to Connect-4?

```java
public class Connect4Board extends Board {
    @Override
    public boolean placeSymbol(int col, Symbol symbol) {
        // Find lowest empty row in column
        for (int row = getSize() - 1; row >= 0; row--) {
            if (isEmpty(row, col)) {
                return super.placeSymbol(row, col, symbol);
            }
        }
        return false;  // Column full
    }
    
    @Override
    public boolean checkWin(Symbol symbol) {
        // Check 4 in a row (horizontal, vertical, diagonal)
        return checkFourInARow(symbol);
    }
}
```

### Q2: How would you implement undo?

```java
public class Game {
    private final Stack<Move> moveStack;
    
    public boolean undoMove() {
        if (moveStack.isEmpty()) return false;
        
        Move lastMove = moveStack.pop();
        board.removeSymbol(lastMove.getRow(), lastMove.getCol());
        switchPlayer();
        status = GameStatus.IN_PROGRESS;
        return true;
    }
}
```

### Q3: How would you add difficulty levels?

```java
public class AIPlayer {
    private final Difficulty difficulty;
    
    public Move findBestMove(Board board) {
        switch (difficulty) {
            case EASY:
                return findRandomMove(board);
            case MEDIUM:
                // 50% optimal, 50% random
                return Math.random() < 0.5 ? 
                    findOptimalMove(board) : findRandomMove(board);
            case HARD:
                return findOptimalMove(board);
        }
    }
}
```

### Q4: How would you add multiplayer over network?

```java
public class NetworkGame extends Game {
    private final Socket socket;
    
    public void sendMove(Move move) {
        out.writeObject(move);
    }
    
    public Move receiveMove() {
        return (Move) in.readObject();
    }
    
    @Override
    public boolean makeMove(int row, int col) {
        boolean success = super.makeMove(row, col);
        if (success) {
            sendMove(new Move(row, col, getCurrentPlayer().getSymbol()));
        }
        return success;
    }
}
```

