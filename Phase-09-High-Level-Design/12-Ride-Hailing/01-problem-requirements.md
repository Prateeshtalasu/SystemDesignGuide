# Ride Hailing (Uber/Lyft) - Problem & Requirements

## What is a Ride Hailing System?

A Ride Hailing System is a platform that connects passengers who need transportation with drivers who can provide rides. The system handles real-time matching of riders with nearby drivers, tracks vehicle locations, calculates fares, processes payments, and provides ratings for quality assurance.

**Example Flow:**

1. Rider opens app, sees nearby available drivers
2. Rider requests a ride from location A to location B
3. System finds the best matching driver
4. Driver accepts the ride
5. Rider tracks driver's arrival in real-time
6. Trip begins, both parties see live route
7. Trip ends, fare is calculated and charged
8. Both parties rate each other

### Why Does This Exist?

1. **Convenience**: Hail a ride from anywhere without waiting on the street
2. **Transparency**: Know the fare estimate before booking
3. **Safety**: Driver and rider information recorded, GPS tracking throughout
4. **Efficiency**: Optimal matching reduces wait times and empty miles
5. **Cashless**: Automatic payment eliminates cash handling

### What Breaks Without It?

- Riders wait unpredictably for taxis
- Drivers cruise empty looking for passengers (wasted fuel, time)
- No fare transparency leads to disputes
- No accountability for service quality
- Cash transactions create safety risks

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the geographic scope? | Affects infrastructure and regulations | Single country (USA) with 50 cities |
| How many concurrent users? | Determines infrastructure scale | 10M daily active users, 500K concurrent |
| Do we support ride-sharing (pooling)? | Adds matching complexity | Yes, basic pooling support |
| What payment methods? | Affects payment integration | Credit cards, digital wallets |
| Do we need surge pricing? | Adds pricing complexity | Yes, dynamic pricing based on demand |
| What vehicle types? | Affects matching and pricing | Standard, Premium, XL, Pool |
| Is this driver-owned or fleet? | Affects driver management | Driver-owned vehicles |
| Do we need scheduled rides? | Adds scheduling complexity | Yes, up to 7 days in advance |

---

## Functional Requirements

### Core Features (Must Have)

1. **Rider Features**
   - Register/login with phone verification
   - Set pickup and drop-off locations
   - See estimated fare before booking
   - View nearby available drivers on map
   - Request a ride (immediate or scheduled)
   - Track driver en route to pickup
   - Track trip progress in real-time
   - Pay automatically via saved payment method
   - Rate driver after trip

2. **Driver Features**
   - Register with vehicle and license verification
   - Go online/offline to accept rides
   - Receive ride requests with pickup details
   - Accept or decline ride requests
   - Navigate to pickup and destination
   - Start and end trips
   - View earnings and trip history
   - Rate riders after trip

3. **Matching System**
   - Find nearby available drivers for a ride request
   - Match based on proximity, rating, vehicle type
   - Handle driver acceptance/rejection
   - Automatic reassignment if driver doesn't respond

4. **Real-Time Tracking**
   - Track driver location during ride request
   - Track trip progress
   - Update ETA dynamically
   - Show route on map

5. **Pricing System**
   - Calculate fare based on distance, time, vehicle type
   - Apply surge pricing during high demand
   - Show fare estimate before booking
   - Apply promotions and discounts

### Secondary Features (Nice to Have)

6. **Ride Pooling**
   - Match multiple riders going similar directions
   - Split fare among riders
   - Handle pickups and drop-offs in sequence

7. **Scheduled Rides**
   - Book rides up to 7 days in advance
   - Automatic driver assignment before pickup time

8. **Safety Features**
   - Share trip details with contacts
   - Emergency button
   - Driver/rider verification

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Ride request to match | < 10 seconds | User expects quick response |
| Location update latency | < 2 seconds | Real-time tracking feel |
| App response time | < 500ms | Smooth user experience |
| Availability | 99.99% | Critical transportation service |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Daily Active Users | 10 million | Given assumption |
| Concurrent users | 500,000 | 5% of DAU at peak |
| Rides per day | 5 million | 50% of DAU take rides |
| Active drivers | 1 million | 5:1 rider to driver ratio |
| Location updates/second | 500,000 | 500K drivers × 1 update/sec |

### Reliability

- **No ride data loss**: Every ride must be recorded for payment and disputes
- **Graceful degradation**: If matching is slow, queue requests vs. fail
- **Idempotent payments**: Never double-charge riders

### Security

- Phone number verification for all users
- Background checks for drivers
- Encrypted payment information
- Trip data privacy (GDPR compliance)

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **Food/package delivery**: Different matching and routing logic
2. **Autonomous vehicles**: Assumes human drivers
3. **Public transit integration**: Focus on private rides
4. **International expansion**: Single country for simplicity
5. **Driver fleet management**: Focus on independent drivers
6. **Advanced fraud detection**: Basic checks only

---

## System Constraints

### Technical Constraints

1. **Location accuracy**: GPS accuracy of 5-10 meters
2. **Update frequency**: Driver location updates every 3-5 seconds
3. **Matching radius**: Search within 5km radius for drivers
4. **Request timeout**: Driver has 15 seconds to accept
5. **Maximum trip distance**: 100km (long trips need special handling)

### Business Constraints

1. **Regulatory compliance**: Driver licensing, insurance requirements
2. **Commission model**: Platform takes 20-25% of fare
3. **Minimum fare**: $5 minimum to ensure driver profitability
4. **Cancellation policy**: Free cancellation within 2 minutes

---

## Success Metrics

How do we know if the system is working well?

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Ride completion rate | > 95% | Completed rides / requested rides |
| Average wait time | < 5 minutes | Time from request to pickup |
| Driver utilization | > 60% | Time with passenger / online time |
| Customer satisfaction | > 4.5/5 | Average rider rating |
| Driver satisfaction | > 4.3/5 | Average driver rating |
| Matching success rate | > 90% | First match acceptance rate |

---

## User Stories

### Story 1: Rider Requests a Ride

```
As a rider,
I want to request a ride from my current location to my destination,
So that I can get transported quickly and safely.

Acceptance Criteria:
- I can see my current location on the map
- I can enter or select my destination
- I see an estimated fare before confirming
- I see nearby drivers and estimated wait time
- After confirming, I'm matched with a driver within 30 seconds
- I can track the driver coming to pick me up
```

### Story 2: Driver Accepts a Ride

```
As a driver,
I want to receive and accept ride requests,
So that I can earn money by providing rides.

Acceptance Criteria:
- I see ride requests with pickup location and estimated fare
- I have 15 seconds to accept or decline
- After accepting, I see navigation to pickup location
- I can contact the rider if needed
- I can start the trip when rider is picked up
- I can end the trip at destination
```

### Story 3: Real-Time Trip Tracking

```
As a rider,
I want to track my trip in real-time,
So that I know where I am and when I'll arrive.

Acceptance Criteria:
- I see the car moving on the map
- I see updated ETA as trip progresses
- I can share trip details with contacts
- I'm notified when approaching destination
```

---

## Interview Tips

### What Interviewers Look For

1. **Location-based systems**: How do you efficiently find nearby drivers?
2. **Real-time systems**: How do you handle 500K location updates/second?
3. **Matching algorithms**: How do you optimize driver-rider matching?
4. **Consistency vs availability**: What happens during network partitions?

### Common Mistakes

1. **Ignoring location indexing**: Naive lat/long queries don't scale
2. **Synchronous matching**: Blocking on driver response kills throughput
3. **Single point of failure**: Matching service must be highly available
4. **Ignoring surge pricing**: Core to business model

### Good Follow-up Questions from Interviewer

- "How do you handle a driver going offline mid-trip?"
- "What if there are no drivers available?"
- "How does surge pricing work technically?"
- "How do you prevent fraudulent location spoofing?"

---

## Summary

| Aspect | Decision |
|--------|----------|
| Primary use case | Connect riders with drivers for transportation |
| Scale | 10M DAU, 5M rides/day, 500K concurrent users |
| Key challenge | Real-time location tracking and matching |
| Latency target | < 10s for matching, < 2s for location updates |
| Availability target | 99.99% |
| Data model | Geospatial indexing for driver locations |
| Communication | WebSocket for real-time updates |

