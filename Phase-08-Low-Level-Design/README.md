# 🏗️ PHASE 8: LOW-LEVEL DESIGN (Weeks 8-10)

**Goal**: Master object-oriented design for interviews

**Learning Objectives**:
- Apply SOLID principles to real design problems
- Design extensible and maintainable systems
- Code any LLD problem in 45 minutes
- Explain design decisions clearly

**Estimated Time**: 45-60 hours

---

## LLD Problems (3 files each):
Each problem gets:
- `01-problem-solution.md` - Complete code
- `02-design-explanation.md` - SOLID principles, design decisions
- `03-code-walkthrough.md` - Line-by-line teaching

---

## Problems List:

### Core Problems (Must Master)

1. **Parking Lot System**
   - Multiple vehicle types (car, bike, truck)
   - Parking spots (compact, large, handicapped)
   - Entry/exit, payment
   - Availability tracking
   - Multi-floor support

2. **Elevator System**
   - Multiple elevators
   - Scheduling algorithms (SCAN, LOOK, FCFS)
   - Request queue
   - Up/down states
   - Weight limits

3. **Library Management System**
   - Books, members, lending
   - Search, reservations
   - Fine calculations
   - Multiple copies
   - Notifications

4. **ATM System**
   - Account operations
   - Cash dispensing (denomination handling)
   - Card validation
   - Transaction logging
   - Concurrent access

5. **LRU Cache**
   - O(1) get/put
   - HashMap + Doubly Linked List
   - Thread-safe version
   - Eviction callbacks

6. **File System**
   - Directory tree
   - File operations (create, delete, move, copy)
   - Search by name/extension
   - Size calculations
   - Permissions

7. **Rate Limiter**
   - Token bucket implementation
   - Sliding window
   - Fixed window
   - Distributed version (Redis)

8. **In-Memory Pub/Sub**
   - Topics and subscribers
   - Message delivery
   - Thread-safe implementation
   - Message filtering

### Booking & Reservation Systems

9. **Restaurant Reservation System**
   - Table management
   - Booking system
   - Waitlist
   - Time slots
   - Party size handling

10. **Car Rental System**
    - Vehicle inventory
    - Booking and returns
    - Pricing (daily, hourly)
    - Late fees
    - Vehicle types

11. **Hotel Booking System**
    - Room types
    - Availability search
    - Booking management
    - Pricing strategies
    - Cancellation policies

12. **Movie Ticket Booking (BookMyShow)**
    - Theater and screen management
    - Seat selection and locking
    - Concurrent booking handling
    - Show scheduling
    - Payment integration

### Financial Systems

13. **Digital Wallet**
    - Account balance
    - Transactions (credit/debit)
    - Transfer between wallets
    - Transaction history
    - Fraud checks

14. **Splitwise (Expense Sharing)**
    - Group expenses
    - Split strategies (equal, exact, percentage)
    - Debt simplification
    - Settlement suggestions
    - Expense history

15. **Stock Exchange / Order Matching**
    - Order book management
    - Buy/sell order matching
    - Price-time priority
    - Order types (market, limit)
    - Trade execution

### Games

16. **Tic-Tac-Toe Game**
    - Game board representation
    - Win condition checking
    - Player turns
    - AI opponent (minimax algorithm)
    - Extensible to NxN board

17. **Snake Game**
    - Game board and snake representation
    - Movement logic
    - Collision detection
    - Score tracking
    - Food generation

18. **Chess Game**
    - Piece movement rules
    - Check/checkmate detection
    - Castling, en passant
    - Move validation
    - Game state management

### Social Media (Simplified)

19. **Design Twitter (Simplified)**
    - User, Tweet, Timeline entities
    - Follow/unfollow functionality
    - Post tweet
    - Get user timeline
    - Retweet functionality

20. **Design Instagram (Simplified)**
    - User, Photo entities
    - Follow/unfollow
    - Post photo
    - News feed generation
    - Like/comment

### Utilities

21. **Design a Logger**
    - Log levels (DEBUG, INFO, WARN, ERROR)
    - Multiple appenders (file, console, remote)
    - Thread-safe logging
    - Log formatting
    - Log rotation

22. **Design a Task Scheduler**
    - Task representation
    - Scheduling algorithms (FIFO, Priority, Round Robin)
    - Task execution
    - Retry mechanism
    - Task dependencies

23. **Vending Machine**
    - Product inventory
    - Coin/cash handling
    - State machine pattern
    - Change calculation
    - Refund handling

---

## Design Principles Checklist:
For each problem, verify:
- [ ] Single Responsibility: Each class has one reason to change
- [ ] Open/Closed: Extensible without modification
- [ ] Liskov Substitution: Subtypes are substitutable
- [ ] Interface Segregation: No unused interface methods
- [ ] Dependency Inversion: Depend on abstractions

---

## Practice Tips:
1. **Time yourself**: 45 minutes per problem
2. **Start with requirements**: Clarify before coding
3. **Draw class diagrams**: Before writing code
4. **Explain trade-offs**: Why this design over alternatives
5. **Consider extensibility**: How would you add features?

---

## Common Interview Questions:
- "How would you add a new vehicle type?"
- "What if we need to support multiple currencies?"
- "How do you handle concurrent access?"
- "What design patterns did you use and why?"

---

## Deliverable
Can code any LLD problem in 45 minutes
