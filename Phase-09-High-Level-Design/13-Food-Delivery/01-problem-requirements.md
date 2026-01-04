# Food Delivery (DoorDash/UberEats) - Problem & Requirements

## What is a Food Delivery System?

A Food Delivery System is a platform that connects customers who want to order food with restaurants that prepare it and delivery partners who transport it. The system handles restaurant discovery, menu browsing, order placement, payment processing, real-time order tracking, and delivery coordination.

**Example Flow:**

1. Customer opens app, sees nearby restaurants
2. Customer browses menu, adds items to cart
3. Customer places order and pays
4. Restaurant receives and confirms order
5. Restaurant prepares food
6. Delivery partner picks up order
7. Customer tracks delivery in real-time
8. Order delivered, both parties rate each other

### Why Does This Exist?

1. **Convenience**: Order food from anywhere without leaving home
2. **Discovery**: Find new restaurants and cuisines easily
3. **Time-saving**: No need to travel, wait in line, or cook
4. **Restaurant reach**: Restaurants can serve customers beyond walk-in capacity
5. **Flexible work**: Delivery partners work on their own schedule

### What Breaks Without It?

- Customers limited to nearby restaurants or cooking
- Restaurants lose potential delivery revenue
- No real-time visibility into order status
- Manual coordination between restaurant and delivery
- Cash handling creates friction and safety concerns

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the geographic scope? | Affects infrastructure and logistics | Single country (USA) with 100 cities |
| How many concurrent orders? | Determines infrastructure scale | 500K orders/day, 50K concurrent at peak |
| Do we support scheduled orders? | Adds scheduling complexity | Yes, up to 7 days in advance |
| What payment methods? | Affects payment integration | Cards, digital wallets, cash on delivery |
| Do we handle restaurant inventory? | Adds real-time sync complexity | Yes, basic inventory management |
| Multiple delivery partners per order? | Affects assignment logic | No, single partner per order |
| Do we support order modifications? | Adds state complexity | Limited, before restaurant confirms |

---

## Functional Requirements

### Core Features (Must Have)

1. **Customer Features**
   - Browse restaurants by location, cuisine, rating
   - View restaurant menus with prices and photos
   - Add items to cart with customizations
   - Place orders (immediate or scheduled)
   - Track order status in real-time
   - Pay via multiple methods
   - Rate restaurant and delivery partner

2. **Restaurant Features**
   - Manage menu items (add, update, remove)
   - Set availability and operating hours
   - Receive and confirm orders
   - Mark orders as ready for pickup
   - View order history and analytics
   - Manage inventory (optional items)

3. **Delivery Partner Features**
   - Go online/offline for deliveries
   - Receive delivery assignments
   - Navigate to restaurant and customer
   - Mark pickup and delivery completion
   - View earnings and history
   - Rate customers

4. **Order Management**
   - Order creation with validation
   - Payment authorization and capture
   - Status tracking through lifecycle
   - Estimated delivery time calculation
   - Order cancellation handling

5. **Search & Discovery**
   - Search restaurants by name, cuisine
   - Filter by rating, price, delivery time
   - Personalized recommendations
   - Promoted/featured restaurants

### Secondary Features (Nice to Have)

6. **Promotions & Discounts**
   - Coupon codes
   - Restaurant-specific deals
   - First-order discounts
   - Loyalty programs

7. **Group Orders**
   - Multiple people add to same order
   - Split payment

8. **Subscription Service**
   - Free delivery for subscribers
   - Exclusive deals

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Search response | < 500ms | Fast restaurant discovery |
| Order placement | < 2 seconds | Critical user action |
| Location update | < 3 seconds | Real-time tracking feel |
| Availability | 99.9% | Revenue-critical, especially during meals |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Daily orders | 500,000 | Business requirement |
| Peak concurrent orders | 50,000 | During dinner rush |
| Restaurants | 200,000 | Across 100 cities |
| Delivery partners | 100,000 | Active at any time: 30,000 |
| Menu items | 20 million | 100 items × 200K restaurants |

### Reliability

- **No order loss**: Every placed order must be processed
- **Payment accuracy**: Never double-charge, never lose payment
- **Graceful degradation**: If tracking fails, orders still work

### Security

- Payment card data protection (PCI compliance)
- User data privacy (GDPR/CCPA)
- Restaurant data isolation
- Delivery partner background verification

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **Grocery delivery**: Different inventory and logistics model
2. **Alcohol delivery**: Requires age verification, special licensing
3. **Catering/bulk orders**: Different pricing and logistics
4. **In-restaurant dining**: Focus on delivery only
5. **Kitchen management**: Restaurant's internal operations
6. **Driver fleet management**: Focus on gig workers

---

## System Constraints

### Technical Constraints

1. **Location accuracy**: GPS accuracy of 5-10 meters
2. **Delivery radius**: Typically 5-10 km from restaurant
3. **Order timeout**: Restaurant has 5 minutes to confirm
4. **Delivery window**: 30-60 minutes typical
5. **Menu size**: Up to 500 items per restaurant

### Business Constraints

1. **Commission model**: Platform takes 15-30% of order value
2. **Minimum order**: Often $10-15 minimum
3. **Delivery fee**: $2-5 based on distance
4. **Peak hours**: Lunch (11am-2pm), Dinner (5pm-9pm)
5. **Restaurant hours**: Must respect operating hours

---

## Success Metrics

How do we know if the system is working well?

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Order completion rate | > 95% | Delivered orders / placed orders |
| Average delivery time | < 45 minutes | Order placed to delivered |
| Customer satisfaction | > 4.5/5 | Average rating |
| Restaurant acceptance rate | > 90% | Accepted / received orders |
| Delivery partner utilization | > 70% | Time with orders / online time |
| Search conversion | > 15% | Orders / search sessions |

---

## User Stories

### Story 1: Customer Orders Food

```
As a hungry customer,
I want to order food from a nearby restaurant,
So that I can enjoy a meal without cooking or going out.

Acceptance Criteria:
- I can see restaurants near my location
- I can browse menus with prices and photos
- I can customize items (e.g., no onions)
- I see estimated delivery time before ordering
- I can track my order in real-time
- I receive notifications at key stages
```

### Story 2: Restaurant Receives Order

```
As a restaurant owner,
I want to receive and manage delivery orders,
So that I can serve more customers and increase revenue.

Acceptance Criteria:
- I receive new orders with audio/visual alert
- I can see order details and special instructions
- I can accept or reject orders within 5 minutes
- I can mark orders as being prepared
- I can mark orders as ready for pickup
- I can temporarily pause new orders if busy
```

### Story 3: Delivery Partner Completes Delivery

```
As a delivery partner,
I want to pick up and deliver food orders,
So that I can earn money on my own schedule.

Acceptance Criteria:
- I receive delivery assignments based on my location
- I can accept or decline assignments
- I see pickup and delivery addresses with navigation
- I can contact customer if needed
- I can mark pickup and delivery completion
- I see my earnings after each delivery
```

---

## Order Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ORDER STATE MACHINE                              │
└─────────────────────────────────────────────────────────────────────────┘

  ┌──────────┐
  │  PLACED  │ ← Customer places order
  └────┬─────┘
       │
       ▼ (5 min timeout)
  ┌──────────────┐     ┌──────────────┐
  │  CONFIRMED   │     │  REJECTED    │ → Refund customer
  │ (Restaurant) │     │              │
  └──────┬───────┘     └──────────────┘
         │
         ▼
  ┌──────────────┐
  │  PREPARING   │ ← Restaurant starts cooking
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │    READY     │ ← Food ready for pickup
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  PICKED_UP   │ ← Delivery partner has food
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  DELIVERED   │ ← Customer received food
  └──────────────┘

  At any stage:
  ┌──────────────┐
  │  CANCELLED   │ → Appropriate refund based on stage
  └──────────────┘
```

---

## Interview Tips

### What Interviewers Look For

1. **Multi-sided marketplace**: Balancing customers, restaurants, delivery partners
2. **Real-time systems**: Order tracking, delivery assignment
3. **Search & discovery**: Restaurant search with multiple criteria
4. **Inventory management**: Menu availability, out-of-stock handling
5. **Peak handling**: Dinner rush creates 5-10x traffic spikes

### Common Mistakes

1. **Ignoring restaurant side**: Focus too much on customer app
2. **Simple delivery assignment**: Nearest partner isn't always best
3. **No inventory consideration**: Items can go out of stock
4. **Ignoring peak hours**: System must handle meal-time spikes

### Good Follow-up Questions from Interviewer

- "How do you handle a restaurant that's overwhelmed with orders?"
- "What if a delivery partner's app crashes mid-delivery?"
- "How do you calculate estimated delivery time?"
- "How do you prevent fraud (fake orders, fake deliveries)?"

---

## Summary

| Aspect | Decision |
|--------|----------|
| Primary use case | Connect customers with restaurants via delivery |
| Scale | 500K orders/day, 200K restaurants, 100K delivery partners |
| Key challenge | Real-time coordination of three-sided marketplace |
| Latency target | < 500ms search, < 2s order placement |
| Availability target | 99.9% |
| Peak handling | 5-10x traffic during meal times |
| Revenue model | Commission on orders + delivery fees |

