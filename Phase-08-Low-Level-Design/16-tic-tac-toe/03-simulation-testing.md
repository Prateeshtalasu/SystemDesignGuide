# ⭕ Tic-Tac-Toe Game - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - X Wins

```
Move 1: X at (0,0)    Move 2: O at (1,1)
X | - | -             X | - | -
- | - | -             - | O | -
- | - | -             - | - | -

Move 3: X at (0,1)    Move 4: O at (2,0)
X | X | -             X | X | -
- | O | -             - | O | -
- | - | -             O | - | -

Move 5: X at (0,2) → X WINS (top row)
X | X | X
- | O | -
O | - | -
```

**Final State:**
```
Game Status: X_WON
Board: X wins with top row
Game completed successfully
```

---

### Scenario 2: Failure/Invalid Input - Occupied Cell

**Initial State:**
```
Board: 3x3 grid
Position (0,0): X (occupied)
Position (0,1): Empty
```

**Step-by-step:**

1. `game.makeMove(0, 0)` (attempt to place O at occupied cell)
   - Check: `board.getSymbol(0, 0)` → X (not null)
   - Cell is occupied
   - `board.placeSymbol(0, 0, O)` returns false
   - Move rejected
   - Throws IllegalArgumentException("Cell already occupied")
   - Game state unchanged

2. `game.makeMove(-1, 0)` (invalid input)
   - Row out of bounds → throws IllegalArgumentException("Row out of bounds")
   - No state change

3. `game.makeMove(0, 5)` (invalid input)
   - Column out of bounds → throws IllegalArgumentException("Column out of bounds")
   - No state change

**Final State:**
```
Board: (0,0) = X (unchanged)
Game state unchanged
Invalid moves properly rejected
```

---

### Scenario 3: Edge Case - Draw Game

**Initial State:**
```
Board: Near full, no winner yet
Last move: O at (2,1)
Next move: X at (2,2) - last empty cell
```

**Step-by-step:**

1. `game.makeMove(2, 2)` (X's move, last empty cell)
   - Place X at (2, 2)
   - Check win: `board.checkWin(X)` → false (no row/col/diagonal)
   - Check board full: `board.isFull()` → true
   - Game status: DRAW
   - Game ends

2. `game.makeMove(0, 0)` (attempt move after game over)
   - Check game status: DRAW (not IN_PROGRESS)
   - Move rejected
   - Throws IllegalStateException("Game is over")
   - Board unchanged

**Final State:**
```
Board: Full, no winner
Game Status: DRAW
Game ended correctly with draw detection
```

---

### Scenario 4: Concurrency/Race Condition - Concurrent Move Attempts

**Initial State:**
```
Board: 3x3 grid
Game Status: IN_PROGRESS
Current Player: X
Thread A: Attempt move at (1,1)
Thread B: Attempt move at (1,1) (concurrent, same cell)
```

**Step-by-step (simulating concurrent move attempts):**

**Thread A:** `game.makeMove(1, 1)` at time T0
**Thread B:** `game.makeMove(1, 1)` at time T0 (concurrent)

1. **Thread A:** Enters `makeMove()` method
   - Checks game status: IN_PROGRESS → valid
   - Checks current player: X → valid
   - Checks cell (1,1): Empty → valid
   - Acquires lock on game object
   - Places X at (1,1)
   - Updates current player to O
   - Releases lock
   - Returns true

2. **Thread B:** Enters `makeMove()` method (concurrent)
   - Checks game status: IN_PROGRESS → valid
   - Checks current player: X → valid (before Thread A completes)
   - Attempts to acquire lock (waits because Thread A holds it)
   - After Thread A releases, acquires lock
   - Checks cell (1,1): X (occupied by Thread A's move)
   - Move rejected
   - Throws IllegalArgumentException("Cell already occupied")
   - Releases lock
   - Returns false

**Final State:**
```
Board: X at (1,1) (Thread A succeeded)
Game Status: IN_PROGRESS
Current Player: O (Thread A's move completed)
Thread B's move rejected (cell occupied)
No race condition, proper synchronization ensures atomicity
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions
- **Invalid Position**: Out of bounds
- **Occupied Cell**: Already has piece
- **Game Already Over**: No more moves
- **Draw Detection**: Board full, no winner

---

## Visual Trace: Minimax

```
Current board:        AI = O, Human = X
┌───┬───┬───┐
│ X │ O │ X │
├───┼───┼───┤
│   │ O │   │
├───┼───┼───┤
│   │ X │   │
└───┴───┴───┘

AI's turn (O). Empty cells: (1,0), (1,2), (2,0), (2,2)

Evaluating move (2,0):
┌───────────────────────────────────────────────────────────────┐
│ Place O at (2,0)                                              │
│ ┌───┬───┬───┐                                                 │
│ │ X │ O │ X │                                                 │
│ ├───┼───┼───┤                                                 │
│ │   │ O │   │  ← O has diagonal threat!                      │
│ ├───┼───┼───┤                                                 │
│ │ O │ X │   │                                                 │
│ └───┴───┴───┘                                                 │
│                                                               │
│ Opponent (X) plays (2,2) to block:                           │
│ ┌───┬───┬───┐                                                 │
│ │ X │ O │ X │                                                 │
│ ├───┼───┼───┤                                                 │
│ │   │ O │   │                                                 │
│ ├───┼───┼───┤                                                 │
│ │ O │ X │ X │  ← X blocks but O can still win                │
│ └───┴───┴───┘                                                 │
│                                                               │
│ AI plays (1,0):                                               │
│ ┌───┬───┬───┐                                                 │
│ │ X │ O │ X │                                                 │
│ ├───┼───┼───┤                                                 │
│ │ O │ O │   │  ← O wins with column!                         │
│ ├───┼───┼───┤                                                 │
│ │ O │ X │ X │                                                 │
│ └───┴───┴───┘                                                 │
│                                                               │
│ Score: +10 - 2 = +8 (win at depth 2)                         │
└───────────────────────────────────────────────────────────────┘

Best move: (2,0) with score +8
```

---

## Testing Approach

### Unit Tests

```java
// BoardTest.java
public class BoardTest {
    
    @Test
    void testPlaceSymbol() {
        Board board = new Board(3);
        
        assertTrue(board.placeSymbol(0, 0, Symbol.X));
        assertEquals(Symbol.X, board.getSymbol(0, 0));
        
        // Can't place on occupied cell
        assertFalse(board.placeSymbol(0, 0, Symbol.O));
    }
    
    @Test
    void testRowWin() {
        Board board = new Board(3);
        board.placeSymbol(0, 0, Symbol.X);
        board.placeSymbol(0, 1, Symbol.X);
        board.placeSymbol(0, 2, Symbol.X);
        
        assertTrue(board.checkWin(Symbol.X));
        assertFalse(board.checkWin(Symbol.O));
    }
    
    @Test
    void testDiagonalWin() {
        Board board = new Board(3);
        board.placeSymbol(0, 0, Symbol.O);
        board.placeSymbol(1, 1, Symbol.O);
        board.placeSymbol(2, 2, Symbol.O);
        
        assertTrue(board.checkWin(Symbol.O));
    }
    
    @Test
    void testNoWin() {
        Board board = new Board(3);
        board.placeSymbol(0, 0, Symbol.X);
        board.placeSymbol(0, 1, Symbol.O);
        board.placeSymbol(0, 2, Symbol.X);
        
        assertFalse(board.checkWin(Symbol.X));
        assertFalse(board.checkWin(Symbol.O));
    }
}
```

```java
// AIPlayerTest.java
public class AIPlayerTest {
    
    @Test
    void testAIBlocksWin() {
        Board board = new Board(3);
        // X about to win
        board.placeSymbol(0, 0, Symbol.X);
        board.placeSymbol(0, 1, Symbol.X);
        // X can win at (0,2)
        
        AIPlayer ai = new AIPlayer("AI", Symbol.O);
        Move move = ai.findBestMove(board);
        
        // AI should block at (0,2)
        assertEquals(0, move.getRow());
        assertEquals(2, move.getCol());
    }
    
    @Test
    void testAITakesWin() {
        Board board = new Board(3);
        // O about to win
        board.placeSymbol(1, 0, Symbol.O);
        board.placeSymbol(1, 1, Symbol.O);
        // O can win at (1,2)
        
        AIPlayer ai = new AIPlayer("AI", Symbol.O);
        Move move = ai.findBestMove(board);
        
        // AI should win at (1,2)
        assertEquals(1, move.getRow());
        assertEquals(2, move.getCol());
    }
    
    @Test
    void testPerfectPlayResultsInDraw() {
        AIPlayer ai1 = new AIPlayer("AI-X", Symbol.X);
        AIPlayer ai2 = new AIPlayer("AI-O", Symbol.O);
        Game game = new Game(ai1, ai2);
        
        while (!game.isGameOver()) {
            game.makeAIMove();
        }
        
        assertEquals(GameStatus.DRAW, game.getStatus());
    }
}
```


### checkWin

```
Time: O(N)
  - Check N rows: O(N) each = O(N²)
  - Check N cols: O(N) each = O(N²)
  - Check 2 diagonals: O(N) each = O(N)
  - Total: O(N²) but typically O(N) for 3x3

Space: O(1)
```

### minimax

```
Without pruning:
  Time: O(b^d) where b = branching factor, d = depth
  For 3x3: O(9!) = 362,880 worst case

With alpha-beta pruning:
  Time: O(b^(d/2)) best case
  For 3x3: ~O(9^4.5) ≈ 2,000

Space: O(d) for recursion stack
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.

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

### Q2: How would you add game replay?

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

### Q3: How would you save/load games?

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

### Q4: What would you do differently with more time?

1. **Add GUI** - Swing/JavaFX interface
2. **Add sound effects** - Move sounds, win celebration
3. **Add statistics** - Win/loss tracking
4. **Add tournaments** - Multiple game series
5. **Add custom symbols** - Emoji or images
6. **Add hints** - Show best move for human

