# ♟️ Chess Game - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Scholar's Mate

```
1. e4 e5 (Pawns forward)
2. Bc4 Nc6 (Bishop out, Knight defends)
3. Qh5 Nf6?? (Queen threatens, Knight blocks wrong)
4. Qxf7# CHECKMATE

King at e8 is in check from Qf7
No legal moves: blocked by own pieces
Game over, White wins
```

**Final State:**
```
Game Status: CHECKMATE (White wins)
All moves executed correctly
Game ended with checkmate detection
```

---

### Scenario 2: Failure/Invalid Input - Illegal Move

**Initial State:**
```
Chess board: Standard starting position
White's turn to move
```

**Step-by-step:**

1. `game.makeMove(Position.fromNotation("e2"), Position.fromNotation("e5"))`
   - Pawn at e2 attempts to move to e5
   - Check: Pawn can move 2 squares from starting position
   - But e5 is blocked by black pawn at e7? No, e5 is empty
   - Actually valid move (2-square move from start)
   - Move succeeds

2. `game.makeMove(Position.fromNotation("a1"), Position.fromNotation("a3"))` (invalid)
   - Rook at a1 attempts to move to a3
   - Check: Rook can move horizontally/vertically
   - But path blocked by pawn at a2
   - Move rejected
   - Throws IllegalArgumentException("Move is blocked")

3. `game.makeMove(Position.fromNotation("e1"), Position.fromNotation("e2"))` (invalid)
   - King at e1 attempts to move to e2
   - Check: King can move one square
   - But e2 is occupied by own pawn
   - Move rejected
   - Throws IllegalArgumentException("Cannot capture own piece")

**Final State:**
```
Board: Only valid moves applied
Invalid moves rejected
Game state consistent
```

---

### Scenario 3: Edge Case - Stalemate

**Initial State:**
```
Chess board: King and Rook vs King
Black's turn to move
Black King not in check
Black has no legal moves (all squares attacked)
```

**Step-by-step:**

1. `game.makeMove(Position.fromNotation("h8"), Position.fromNotation("g8"))` (attempt move)
   - Black King at h8 attempts to move to g8
   - Check if move is legal: g8 is attacked by White Rook
   - Move would put king in check → illegal
   - Try other squares: All adjacent squares attacked
   - No legal moves available

2. `game.updateGameStatus()`
   - Check: Is Black King in check? → No
   - Check: Does Black have any legal moves? → No
   - Condition: Not in check + No legal moves = STALEMATE
   - Game status: STALEMATE
   - Game ends in draw

**Final State:**
```
Game Status: STALEMATE (draw)
Black King not in check but has no legal moves
Game ended correctly with stalemate detection
```

---

### Scenario 4: Concurrency/Race Condition - Concurrent Move Attempts

**Initial State:**
```
Chess board: Standard starting position
Game Status: IN_PROGRESS
Current Turn: WHITE
Thread A: Attempt move e2 to e4
Thread B: Attempt move e2 to e3 (concurrent, same piece)
```

**Step-by-step (simulating concurrent move attempts):**

**Thread A:** `game.move(Position.fromNotation("e2"), Position.fromNotation("e4"))` at time T0
**Thread B:** `game.move(Position.fromNotation("e2"), Position.fromNotation("e3"))` at time T0 (concurrent)

1. **Thread A:** Enters `move()` method
   - Checks game status: IN_PROGRESS → valid
   - Checks current turn: WHITE → valid
   - Checks piece at e2: Pawn (WHITE) → valid
   - Validates move e2→e4: Legal (pawn double move) → valid
   - Acquires lock on game object
   - Executes move: Moves pawn from e2 to e4
   - Updates current turn to BLACK
   - Updates game state
   - Releases lock
   - Returns true

2. **Thread B:** Enters `move()` method (concurrent)
   - Checks game status: IN_PROGRESS → valid
   - Checks current turn: WHITE → valid (before Thread A completes)
   - Attempts to acquire lock (waits because Thread A holds it)
   - After Thread A releases, acquires lock
   - Checks piece at e2: Empty (Thread A already moved it)
   - Move rejected
   - Throws IllegalArgumentException("No piece at source position")
   - Releases lock
   - Returns false

**Final State:**
```
Board: Pawn at e4 (Thread A succeeded)
Game Status: IN_PROGRESS
Current Turn: BLACK (Thread A's move completed)
Thread B's move rejected (source piece already moved)
No race condition, proper synchronization ensures atomicity
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions
- **Castling Through Check**: Illegal
- **En Passant Timing**: Only immediately after
- **Pawn Promotion**: Must choose piece
- **Stalemate**: No legal moves, not in check

---

## Testing Approach

### Unit Tests

```java
// PieceMovementTest.java
public class PieceMovementTest {
    
    @Test
    void testRookMoves() {
        Board board = new Board();
        // Clear board except rook at d4
        clearBoard(board);
        board.setPiece(Position.fromNotation("d4"), 
            new Rook(Color.WHITE, Position.fromNotation("d4")));
        
        Piece rook = board.getPiece(Position.fromNotation("d4"));
        List<Position> moves = rook.getPossibleMoves(board);
        
        // Should have 14 moves (7 vertical + 7 horizontal)
        assertEquals(14, moves.size());
    }
    
    @Test
    void testKnightMoves() {
        Board board = new Board();
        clearBoard(board);
        board.setPiece(Position.fromNotation("d4"),
            new Knight(Color.WHITE, Position.fromNotation("d4")));
        
        Piece knight = board.getPiece(Position.fromNotation("d4"));
        List<Position> moves = knight.getPossibleMoves(board);
        
        // Knight at d4 has 8 possible moves
        assertEquals(8, moves.size());
        assertTrue(moves.contains(Position.fromNotation("c6")));
        assertTrue(moves.contains(Position.fromNotation("e6")));
    }
}
```

```java
// CheckDetectionTest.java
public class CheckDetectionTest {
    
    @Test
    void testCheckByQueen() {
        ChessGame game = new ChessGame();
        // Setup: White Queen attacks Black King
        
        assertTrue(game.isKingInCheck(Color.BLACK));
    }
    
    @Test
    void testCheckmate() {
        ChessGame game = new ChessGame();
        // Setup: Scholar's mate position
        
        assertEquals(GameStatus.CHECKMATE, game.getStatus());
    }
    
    @Test
    void testStalemate() {
        ChessGame game = new ChessGame();
        // Setup: King has no legal moves but not in check
        
        assertEquals(GameStatus.STALEMATE, game.getStatus());
    }
}
```


### getPossibleMoves

```
King: O(8) - 8 adjacent squares
Queen: O(27) - max squares on empty board
Rook: O(14) - max squares
Bishop: O(13) - max squares
Knight: O(8) - 8 L-shaped moves
Pawn: O(4) - forward, double, 2 captures
```

### isKingInCheck

```
Time: O(P × M)
  - P = number of opponent pieces (max 16)
  - M = max moves per piece (Queen: 27)
  - Worst case: O(16 × 27) = O(432) = O(1)
```

### getValidMoves

```
Time: O(M × P × M)
  - For each possible move M
  - Simulate and check isKingInCheck: O(P × M)
  - Worst case: O(27 × 16 × 27) ≈ O(12,000) = O(1)
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.

```java
public class ChessGame {
    private final Map<String, Integer> positionCounts;
    
    public boolean makeMove(Position from, Position to) {
        // ... execute move
        
        String positionKey = getBoardState();
        int count = positionCounts.merge(positionKey, 1, Integer::sum);
        
        if (count >= 3) {
            status = GameStatus.DRAW;
        }
    }
    
    private String getBoardState() {
        StringBuilder sb = new StringBuilder();
        for (int row = 0; row < 8; row++) {
            for (int col = 0; col < 8; col++) {
                Piece p = board.getPiece(new Position(row, col));
                sb.append(p != null ? p.getSymbol() : '.');
            }
        }
        sb.append(currentTurn);
        return sb.toString();
    }
}
```

### Q2: How would you implement the 50-move rule?

```java
public class ChessGame {
    private int halfMoveClock;  // Moves since pawn move or capture
    
    public boolean makeMove(Position from, Position to) {
        Piece piece = board.getPiece(from);
        Piece captured = board.getPiece(to);
        
        // Reset clock on pawn move or capture
        if (piece.getType() == PieceType.PAWN || captured != null) {
            halfMoveClock = 0;
        } else {
            halfMoveClock++;
        }
        
        if (halfMoveClock >= 100) {  // 50 moves = 100 half-moves
            status = GameStatus.DRAW;
        }
    }
}
```

### Q3: How would you implement time control?

```java
public class ChessGame {
    private final long[] timeRemaining;  // [WHITE, BLACK]
    private final long increment;
    private long lastMoveTime;
    
    public boolean makeMove(Position from, Position to) {
        long now = System.currentTimeMillis();
        long elapsed = now - lastMoveTime;
        
        int playerIndex = currentTurn.ordinal();
        timeRemaining[playerIndex] -= elapsed;
        
        if (timeRemaining[playerIndex] <= 0) {
            status = currentTurn == Color.WHITE ? 
                GameStatus.BLACK_WINS_TIME : GameStatus.WHITE_WINS_TIME;
            return false;
        }
        
        // Add increment
        timeRemaining[playerIndex] += increment;
        
        // ... execute move
        
        lastMoveTime = System.currentTimeMillis();
    }
}
```

### Q4: What would you do differently with more time?

1. **Add FEN support** - Standard position notation
2. **Add PGN support** - Game notation import/export
3. **Add opening book** - Common opening moves
4. **Add endgame tablebase** - Perfect endgame play
5. **Add analysis mode** - Show best moves
6. **Add online play** - Multiplayer support

