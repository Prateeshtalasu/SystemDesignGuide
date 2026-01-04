# ♟️ Chess Game - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Chess Game.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Position` | Represent board coordinates | Coordinate system changes |
| `Piece` | Define piece behavior | Piece rules change |
| `Board` | Manage piece positions | Board representation changes |
| `Move` | Record a move | Move notation changes |
| `ChessGame` | Coordinate game flow | Game rules change |

**SRP in Action:**

```java
// Each piece type handles only its own movement
public class Knight extends Piece {
    @Override
    public List<Position> getPossibleMoves(Board board) {
        // Only L-shaped moves
    }
}

// Board only manages piece positions
public class Board {
    public Piece getPiece(Position pos) { }
    public void setPiece(Position pos, Piece piece) { }
}

// ChessGame handles game logic
public class ChessGame {
    public boolean makeMove(Position from, Position to) { }
    public boolean isKingInCheck(Color color) { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Piece Types:**

```java
// No changes to existing code needed
public class Amazon extends Piece {  // Queen + Knight
    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        // Combine Queen and Knight moves
        return moves;
    }
}
```

**Adding New Game Variants:**

```java
public interface ChessVariant {
    void setupBoard(Board board);
    boolean isValidMove(Move move, ChessGame game);
}

public class Chess960 implements ChessVariant { }
public class ThreeCheck implements ChessVariant { }
```

---

### 3. Liskov Substitution Principle (LSP)

**All pieces work interchangeably:**

```java
public class ChessGame {
    public List<Position> getValidMoves(Position from) {
        Piece piece = board.getPiece(from);
        // Works for any piece type
        List<Position> possibleMoves = piece.getPossibleMoves(board);
        // Filter for check
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Could be improved:**

```java
public interface Movable {
    List<Position> getPossibleMoves(Board board);
}

public interface Capturable {
    boolean canBeCaptured();
}

public interface Promotable {
    boolean canPromote();
    Piece promote(PieceType type);
}

public class Pawn implements Movable, Promotable { }
public class King implements Movable { }  // Not Promotable
```

---

### 5. Dependency Inversion Principle (DIP)

**Better with DIP:**

```java
public interface MoveValidator {
    boolean isValid(Move move, Board board);
}

public interface CheckDetector {
    boolean isInCheck(Color color, Board board);
}

public class ChessGame {
    private final MoveValidator moveValidator;
    private final CheckDetector checkDetector;
}
```

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. Board manages board state, Piece represents pieces, Move stores move data, ChessGame coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new piece types, move validators) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All Piece subclasses properly implement the Piece contract. They are substitutable in move validation. | N/A | - |
| **ISP** | PASS | Interfaces are minimal and focused. Piece interface is well-designed. No unused methods. | N/A | - |
| **DIP** | PASS | ChessGame depends on abstractions (MoveValidator interface). DIP is well applied. | N/A | - |

---

## Design Patterns Used

### 1. Template Method Pattern

**Where:** Piece movement

```java
public abstract class Piece {
    // Template method
    protected List<Position> getMovesInDirection(Board board, 
                                                  int rowDir, int colDir) {
        List<Position> moves = new ArrayList<>();
        Position current = position.offset(rowDir, colDir);
        
        while (current.isValid()) {
            Piece piece = board.getPiece(current);
            if (piece == null) {
                moves.add(current);
            } else {
                if (piece.getColor() != this.color) {
                    moves.add(current);
                }
                break;
            }
            current = current.offset(rowDir, colDir);
        }
        return moves;
    }
}

// Subclasses use the template
public class Rook extends Piece {
    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        moves.addAll(getMovesInDirection(board, 1, 0));
        moves.addAll(getMovesInDirection(board, -1, 0));
        moves.addAll(getMovesInDirection(board, 0, 1));
        moves.addAll(getMovesInDirection(board, 0, -1));
        return moves;
    }
}
```

---

### 2. Factory Pattern (Potential)

**Where:** Creating pieces

```java
public class PieceFactory {
    public static Piece create(PieceType type, Color color, Position pos) {
        switch (type) {
            case KING: return new King(color, pos);
            case QUEEN: return new Queen(color, pos);
            case ROOK: return new Rook(color, pos);
            case BISHOP: return new Bishop(color, pos);
            case KNIGHT: return new Knight(color, pos);
            case PAWN: return new Pawn(color, pos);
            default: throw new IllegalArgumentException();
        }
    }
}
```

---

### 3. Command Pattern (Potential)

**Where:** Move execution and undo

```java
public interface Command {
    void execute();
    void undo();
}

public class MoveCommand implements Command {
    private final Board board;
    private final Position from, to;
    private Piece movedPiece;
    private Piece capturedPiece;
    
    @Override
    public void execute() {
        movedPiece = board.getPiece(from);
        capturedPiece = board.getPiece(to);
        board.movePiece(from, to);
    }
    
    @Override
    public void undo() {
        board.movePiece(to, from);
        board.setPiece(to, capturedPiece);
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single Class for All Piece Types

**What it is:**

```java
// Rejected approach
public class Piece {
    private PieceType type;
    private Color color;
    private Position position;
    
    public List<Position> getPossibleMoves(Board board) {
        switch (type) {
            case PAWN:
                return getPawnMoves(board);
            case ROOK:
                return getRookMoves(board);
            case KNIGHT:
                return getKnightMoves(board);
            // ... all piece types in one class
        }
    }
}
```

**Why rejected:**
- Violates Single Responsibility Principle - one class handles all piece types
- Switch statement grows with each new piece type
- Hard to maintain - all piece logic in one place
- Difficult to test individual piece behaviors
- Not extensible - adding new piece types requires modifying core class

**What breaks:**
- Open/Closed Principle violated (need to modify for extensions)
- Code becomes harder to maintain as complexity grows
- Cannot easily add custom piece types
- Testing requires complex setup for each piece type

---

### Alternative 2: Array-Based Board Representation

**What it is:**

```java
// Rejected approach
public class Board {
    private Piece[] squares;  // 64-element array
    
    public Piece getPiece(int row, int col) {
        return squares[row * 8 + col];
    }
    
    public boolean isValidPosition(int row, int col) {
        // Need to manually check bounds
        return row >= 0 && row < 8 && col >= 0 && col < 8;
    }
}
```

**Why rejected:**
- Less type-safe - requires manual index calculations
- Error-prone - easy to mix up row/col indices
- Harder to read - index arithmetic scattered throughout
- Position validation repeated in many places
- No encapsulation of position logic

**What breaks:**
- Prone to off-by-one errors
- Position validation logic duplicated
- Harder to extend to different board sizes
- Less intuitive API (raw integers vs Position objects)

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you implement threefold repetition?

**Answer:**

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

---

### Q2: How would you implement the 50-move rule?

**Answer:**

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
        
        if (halfMoveClock >= 100) {  // 50 full moves = 100 half moves
            status = GameStatus.DRAW;
        }
    }
}
```

---

### Q3: How would you implement time control?

**Answer:**

```java
public class TimeControl {
    private final Duration initialTime;
    private final Duration increment;
    private final Map<Color, Duration> remainingTime;
    
    public void startTurn(Color color) {
        timer.start();
    }
    
    public void endTurn(Color color) {
        Duration elapsed = timer.stop();
        Duration remaining = remainingTime.get(color);
        remainingTime.put(color, remaining.minus(elapsed).plus(increment));
        
        if (remainingTime.get(color).isNegative()) {
            throw new TimeOutException(color + " ran out of time");
        }
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add AI opponent** - Minimax or neural network player
2. **Add move notation** - Algebraic notation (e.g., e4, Nf3)
3. **Add game history** - Save/load games, replay
4. **Add online multiplayer** - Network play
5. **Add analysis mode** - Show best moves, evaluations
6. **Add puzzles** - Tactical puzzles and training
7. **Add opening book** - Common opening positions
8. **Add endgame tablebase** - Perfect endgame play

---

### Q5: How would you implement move validation optimization?

**Answer:**

```java
public class OptimizedMoveValidator {
    private final Map<Position, List<Position>> validMovesCache;
    
    public List<Position> getValidMoves(Position from) {
        // Check cache first
        if (validMovesCache.containsKey(from)) {
            return validMovesCache.get(from);
        }
        
        // Calculate valid moves
        List<Position> possibleMoves = piece.getPossibleMoves(board);
        List<Position> validMoves = filterLegalMoves(possibleMoves);
        
        // Cache result
        validMovesCache.put(from, validMoves);
        return validMoves;
    }
    
    public void invalidateCache() {
        // Clear cache after each move
        validMovesCache.clear();
    }
}
```

**Benefits:**
- Avoids recalculating moves for same position
- Faster UI updates when highlighting valid squares
- Tradeoff: Memory for cached moves, need cache invalidation

---

### Q6: How would you implement thread-safe chess game?

**Answer:**

```java
public class ThreadSafeChessGame {
    private final ReentrantLock lock = new ReentrantLock();
    private Board board;
    private Color currentTurn;
    private GameStatus status;
    
    public boolean move(Position from, Position to) {
        lock.lock();
        try {
            if (status != GameStatus.IN_PROGRESS) {
                throw new IllegalStateException("Game is not in progress");
            }
            
            Piece piece = board.getPiece(from);
            if (piece == null || piece.getColor() != currentTurn) {
                throw new IllegalArgumentException("Not your turn");
            }
            
            if (!isValidMove(from, to)) {
                return false;
            }
            
            executeMove(from, to);
            updateGameStatus();
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    public List<Position> getValidMoves(Position from) {
        lock.lock();
        try {
            return calculateValidMoves(from);
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
- Fine-grained locking possible for read-heavy scenarios

---

### Q7: How would you implement move history with undo/redo?

**Answer:**

```java
public class MoveHistoryManager {
    private final List<Move> history;
    private final List<Move> undoneMoves;
    private int currentIndex;
    
    public void executeMove(Move move) {
        move.execute();
        
        // Remove any undone moves if we're not at end
        if (currentIndex < history.size()) {
            history.subList(currentIndex, history.size()).clear();
            undoneMoves.clear();
        }
        
        history.add(move);
        currentIndex++;
    }
    
    public boolean undo() {
        if (currentIndex == 0) {
            return false;
        }
        
        Move move = history.get(--currentIndex);
        move.undo();
        undoneMoves.add(move);
        return true;
    }
    
    public boolean redo() {
        if (undoneMoves.isEmpty()) {
            return false;
        }
        
        Move move = undoneMoves.remove(undoneMoves.size() - 1);
        move.execute();
        history.add(move);
        currentIndex++;
        return true;
    }
}
```

**Features:**
- Command pattern for undo/redo
- Maintains history and undone moves separately
- Allows branching history management
- Can extend to save/load game states

---

### Q8: How would you optimize move generation for performance?

**Answer:**

```java
public class OptimizedMoveGenerator {
    // Pre-compute move offsets for sliding pieces
    private static final int[][] ROOK_OFFSETS = {{-1,0}, {1,0}, {0,-1}, {0,1}};
    private static final int[][] BISHOP_OFFSETS = {{-1,-1}, {-1,1}, {1,-1}, {1,1}};
    
    public List<Position> generateRookMoves(Position from, Board board) {
        List<Position> moves = new ArrayList<>();
        
        for (int[] offset : ROOK_OFFSETS) {
            Position pos = from.add(offset[0], offset[1]);
            
            while (board.isValidPosition(pos)) {
                Piece piece = board.getPiece(pos);
                
                if (piece == null) {
                    moves.add(pos);
                } else {
                    if (piece.getColor() != getColor()) {
                        moves.add(pos);  // Can capture
                    }
                    break;  // Blocked
                }
                
                pos = pos.add(offset[0], offset[1]);
            }
        }
        
        return moves;
    }
    
    // Use bitboards for fast position representation
    private long whitePieces;
    private long blackPieces;
    
    public boolean isSquareOccupied(int square) {
        return ((whitePieces | blackPieces) & (1L << square)) != 0;
    }
}
```

**Optimizations:**
- Pre-computed move offsets
- Efficient bitboard representation
- Early termination in sliding piece moves
- Vectorized operations for multiple squares

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `getPossibleMoves` | O(N) | N = max moves (Queen: 27) |
| `isKingInCheck` | O(P × M) | P = pieces, M = moves |
| `getValidMoves` | O(M × P × M) | Check each move |
| `makeMove` | O(P × M) | Validate + check detection |

### Space Complexity

| Component | Space |
|-----------|-------|
| Board | O(64) = O(1) |
| Pieces | O(32) = O(1) |
| Move history | O(moves) |

### Bottlenecks at Scale

**10x Usage (Standard 8×8 board, 1 → 10 concurrent games):**
- Problem: Move generation becomes noticeable (O(P × M) per piece), position evaluation overhead grows, game state storage increases
- Solution: Cache generated moves, optimize position evaluation, use efficient game state representation
- Tradeoff: Cache memory overhead, cache invalidation complexity

**100x Usage (Standard 8×8 board, 1 → 100 concurrent games):**
- Problem: AI move calculation becomes bottleneck, minimax search too expensive for multiple concurrent games, memory for game trees exceeds capacity
- Solution: Use distributed AI engines, implement move time limits, use simplified evaluation for concurrent games
- Tradeoff: Distributed system complexity, may need to compromise AI quality for performance


### Q1: How would you implement en passant?

```java
public class ChessGame {
    private Position enPassantTarget;  // Set after pawn double move
    
    public boolean makeMove(Position from, Position to) {
        Piece piece = board.getPiece(from);
        
        // Check for pawn double move
        if (piece.getType() == PieceType.PAWN && 
            Math.abs(to.getRow() - from.getRow()) == 2) {
            enPassantTarget = new Position(
                (from.getRow() + to.getRow()) / 2, from.getCol());
        } else {
            enPassantTarget = null;
        }
        
        // Handle en passant capture
        if (piece.getType() == PieceType.PAWN && to.equals(enPassantTarget)) {
            Position capturedPawnPos = new Position(from.getRow(), to.getCol());
            board.setPiece(capturedPawnPos, null);
        }
    }
}
```

### Q2: How would you implement pawn promotion?

```java
public boolean makeMove(Position from, Position to, PieceType promotionType) {
    Piece piece = board.getPiece(from);
    
    // Execute move
    board.movePiece(from, to);
    
    // Handle promotion
    if (piece.getType() == PieceType.PAWN) {
        int promotionRow = piece.getColor() == Color.WHITE ? 7 : 0;
        if (to.getRow() == promotionRow) {
            Piece promoted = PieceFactory.create(
                promotionType, piece.getColor(), to);
            board.setPiece(to, promoted);
        }
    }
}
```

### Q3: How would you implement move notation (PGN)?

```java
public class PGNGenerator {
    public String toAlgebraic(Move move, Board board) {
        StringBuilder sb = new StringBuilder();
        
        // Piece letter (except pawn)
        if (move.getPiece().getType() != PieceType.PAWN) {
            sb.append(move.getPiece().getType().getNotation());
        }
        
        // Disambiguation if needed
        sb.append(getDisambiguation(move, board));
        
        // Capture
        if (move.getCaptured() != null) {
            if (move.getPiece().getType() == PieceType.PAWN) {
                sb.append(move.getFrom().toNotation().charAt(0));
            }
            sb.append('x');
        }
        
        // Destination
        sb.append(move.getTo().toNotation());
        
        return sb.toString();
    }
}
```

### Q4: How would you implement a chess engine?

```java
public class ChessEngine {
    public Move findBestMove(ChessGame game, int depth) {
        return minimax(game, depth, Integer.MIN_VALUE, Integer.MAX_VALUE, true).move;
    }
    
    private EvaluatedMove minimax(ChessGame game, int depth, 
                                  int alpha, int beta, boolean maximizing) {
        if (depth == 0 || game.getStatus() != GameStatus.ACTIVE) {
            return new EvaluatedMove(null, evaluate(game));
        }
        
        // Get all legal moves
        List<Move> moves = getAllLegalMoves(game);
        
        // Search...
    }
    
    private int evaluate(ChessGame game) {
        int score = 0;
        // Material value
        // Position value
        // King safety
        // Pawn structure
        return score;
    }
}
```

