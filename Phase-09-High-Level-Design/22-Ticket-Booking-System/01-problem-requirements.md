# Ticket Booking System (BookMyShow) - Problem & Requirements

## What is a Ticket Booking System?

A ticket booking system allows users to browse events (movies, concerts, sports), select seats, and purchase tickets. It handles concurrent booking requests, seat locking, payment processing, and ticket confirmation.

**Example Flow:**
```
User browses events → Selects show → Chooses seats → 
Locks seats → Pays → Receives ticket confirmation
```

### Why Does This Exist?

1. **Event Management**: Organize and sell tickets for events
2. **Seat Selection**: Visual seat map for user selection
3. **Concurrent Booking**: Handle multiple users booking same seats
4. **Payment Processing**: Secure ticket purchase
5. **Ticket Delivery**: Digital or physical ticket delivery

### What Breaks Without It?

- Users can't book tickets online
- Double booking (same seat sold twice)
- No seat availability visibility
- Manual ticket sales only
- Lost revenue from unsold tickets

---

## Clarifying Questions (Ask the Interviewer)

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (bookings per day)? | Determines infrastructure size | 1 million bookings/day |
| What types of events? | Affects data model | Movies, concerts, sports |
| Seat selection method? | Affects UI/UX | Visual seat map |
| Booking timeout? | Affects seat locking | 5 minutes |
| Payment methods? | Affects payment integration | Credit cards, UPI, wallets |
| Ticket delivery? | Affects post-booking flow | Digital (QR code) |
| Refund policy? | Affects cancellation logic | 24-hour cancellation window |

---

## Functional Requirements

### Core Features (Must Have)

1. **Event Management**
   - Browse events (movies, concerts)
   - View event details (time, venue, price)
   - Search and filter events

2. **Seat Selection**
   - View seat map (available, booked, locked)
   - Select multiple seats
   - See seat pricing (different zones)

3. **Seat Locking**
   - Lock selected seats temporarily
   - Prevent others from booking locked seats
   - Auto-release on timeout

4. **Booking Management**
   - Create booking with locked seats
   - Process payment
   - Confirm booking
   - Generate ticket

5. **Payment Processing**
   - Accept multiple payment methods
   - Process payment securely
   - Handle payment failures

6. **Ticket Management**
   - Generate digital tickets (QR code)
   - Send ticket via email/SMS
   - View booking history

### Secondary Features (Nice to Have)

7. **Waitlist**
   - Join waitlist for sold-out shows
   - Notify when seats available

8. **Cancellation**
   - Cancel booking (within policy)
   - Refund processing
   - Release seats

9. **Recommendations**
   - Suggest similar events
   - Personalized recommendations

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Seat map load time | < 500ms (p95) | Fast seat selection |
| Seat locking | < 100ms | Real-time availability |
| Booking completion | < 30 seconds | Fast checkout |
| Payment processing | < 3 seconds | Quick confirmation |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Bookings/day | 1 million | Given assumption |
| Bookings/second (peak) | 500 | 1M / 86,400 × 50x peak |
| Concurrent seat selections | 10,000 | Multiple users selecting |
| Events | 10,000 active | Various venues |

### Reliability

- **Availability**: 99.99% (52 minutes downtime/year)
- **Durability**: 99.999999999% (11 nines)
- **Consistency**: Strong consistency for seat availability (cannot double-book)

### Security

- **Payment Security**: PCI-DSS compliance
- **Ticket Security**: QR code encryption
- **Fraud Prevention**: Rate limiting, velocity checks

---

## What's Out of Scope

1. **Event Creation**: Event management (assume separate service)
2. **Venue Management**: Venue setup (assume separate service)
3. **User Authentication**: Login/signup (assume separate service)
4. **Recommendations**: ML-based recommendations
5. **Analytics**: Event analytics dashboard

---

## System Constraints

### Technical Constraints

1. **Seat Locking**: 5-minute timeout
   - Must release seats if booking not completed
   - Prevent seat hoarding

2. **Concurrent Bookings**: Handle race conditions
   - Multiple users selecting same seat
   - Distributed locking required

3. **Seat Availability**: Real-time updates
   - Must reflect current availability
   - Low latency required

### Business Constraints

1. **No Double Booking**: Critical requirement
   - Same seat cannot be sold twice
   - Strong consistency required

2. **Booking Timeout**: 5 minutes
   - Prevents seat hoarding
   - Releases seats for others

---

## Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Booking completion rate | > 40% | Completed bookings / Seat locks |
| Seat locking latency | < 100ms | Time to lock seats |
| Double booking incidents | 0 | No duplicate seat bookings |
| Payment success rate | > 98% | Successful payments / Attempts |

---

## User Stories

### Story 1: Book Movie Tickets

```
As a user,
I want to book movie tickets for a show,
So that I can attend the movie.

Acceptance Criteria:
- Browse available shows
- Select seats on seat map
- Lock seats for 5 minutes
- Complete payment
- Receive ticket confirmation
```

### Story 2: Handle Concurrent Seat Selection

```
As a system,
I want to prevent double booking,
So that the same seat is not sold twice.

Acceptance Criteria:
- Lock seat when user selects
- Prevent others from selecting locked seat
- Release lock on timeout or booking completion
- No double bookings
```

---

## Core Components Overview

### 1. Event Service
- Manages events and shows
- Event details and pricing

### 2. Seat Service
- Manages seat availability
- Seat locking and unlocking
- Seat map generation

### 3. Booking Service
- Creates bookings
- Manages booking lifecycle
- Handles cancellations

### 4. Payment Service
- Processes payments
- Handles payment failures
- Refund processing

### 5. Ticket Service
- Generates tickets
- QR code generation
- Ticket delivery

---

## Summary

| Aspect | Decision |
|-------|----------|
| Primary use case | Book tickets for events with seat selection |
| Scale | 1 million bookings/day, 500 bookings/second peak |
| Key challenge | Concurrent seat booking, preventing double booking |
| Architecture pattern | Distributed locking for seat management |
| Consistency | Strong for seat availability |
| Booking timeout | 5 minutes |

