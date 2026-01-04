# E-commerce Checkout System - Problem & Requirements

## What is an E-commerce Checkout System?

An e-commerce checkout system is the final step in an online shopping experience where customers review their cart, enter payment and shipping information, and complete their purchase. It handles the critical transaction flow from cart to order confirmation, including inventory management, payment processing, and order fulfillment.

**Example Flow:**
```
User adds items to cart → Reviews cart → Enters shipping info → 
Selects payment method → Confirms order → Receives confirmation
```

### Why Does This Exist?

1. **Transaction Completion**: Converts browsing into sales
2. **Inventory Management**: Ensures products are available before purchase
3. **Payment Processing**: Securely handles financial transactions
4. **Order Fulfillment**: Triggers shipping and delivery processes
5. **Customer Trust**: Provides secure, reliable purchase experience

### What Breaks Without It?

- Customers can't complete purchases
- Inventory overselling (selling items that don't exist)
- Payment fraud and security issues
- Order fulfillment failures
- Lost revenue from abandoned carts

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (orders per day)? | Determines infrastructure size | 10 million orders/day |
| What payment methods? | Affects payment gateway integration | Credit cards, PayPal, digital wallets |
| Do we need inventory reservation? | Prevents overselling | Yes, reserve during checkout |
| What's the cart abandonment rate? | Affects cart persistence strategy | 70% abandonment, 30% completion |
| Do we support guest checkout? | Affects user authentication | Yes, guest and registered users |
| What's the order value range? | Affects fraud detection thresholds | $10 - $10,000 per order |
| Do we need order modifications? | Affects order state management | No, orders are immutable after confirmation |
| What's the refund/return policy? | Affects post-order processing | Handled separately, out of scope |

---

## Functional Requirements

### Core Features (Must Have)

1. **Cart Management**
   - Add/remove items from cart
   - Update quantities
   - Apply discounts and coupons
   - Calculate totals (subtotal, tax, shipping)
   - Persist cart across sessions

2. **Inventory Reservation**
   - Check product availability
   - Reserve inventory during checkout
   - Release reservation on timeout or cancellation
   - Prevent overselling

3. **Shipping Information**
   - Collect shipping address
   - Calculate shipping costs
   - Support multiple shipping options
   - Address validation

4. **Payment Processing**
   - Accept multiple payment methods
   - Process payment securely
   - Handle payment failures
   - Support partial payments (gift cards)

5. **Order Creation**
   - Generate unique order ID
   - Create order record
   - Send confirmation email
   - Update inventory

6. **Order Status Tracking**
   - Track order status (pending, confirmed, shipped, delivered)
   - Provide order history
   - Support order lookup

### Secondary Features (Nice to Have)

7. **Wishlist**
   - Save items for later
   - Move items from wishlist to cart

8. **Order Modifications**
   - Cancel orders (before shipping)
   - Modify shipping address (before shipping)

9. **Split Payments**
   - Multiple payment methods per order
   - Gift card + credit card

10. **Express Checkout**
    - One-click checkout for returning customers
    - Saved payment methods

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Checkout completion time | < 30 seconds | Fast checkout reduces abandonment |
| Cart load time | < 200ms (p95) | Responsive cart experience |
| Payment processing | < 3 seconds | Fast payment confirmation |
| API response time | < 100ms (p99) | Responsive UI |
| Inventory check | < 50ms | Real-time availability |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Orders/day | 10 million | Given assumption |
| Orders/second (peak) | 1,000 | 10M / 86,400 seconds × 10x peak |
| Concurrent checkouts | 50,000 | 5% of daily orders at peak |
| Cart size (avg) | 3 items | Typical e-commerce |
| Cart size (max) | 50 items | Reasonable limit |

### Reliability

- **Availability**: 99.99% (52 minutes downtime/year)
  - Critical for revenue generation
  - Payment processing must be highly available

- **Durability**: 99.999999999% (11 nines)
  - Orders must never be lost
  - Payment transactions must be durable

- **Consistency**: Strong consistency for inventory and payments
  - Cannot oversell inventory
  - Payment must be processed exactly once

### Security

- **PCI Compliance**: Secure payment data handling
- **Encryption**: TLS 1.3 for data in transit, AES-256 for data at rest
- **Fraud Detection**: Real-time fraud screening
- **Input Validation**: Sanitize all user inputs

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **Product Catalog**: Product browsing and search
2. **User Authentication**: Login/signup (assume separate service)
3. **Recommendations**: Product recommendations
4. **Reviews**: Product reviews and ratings
5. **Refunds/Returns**: Post-order refund processing
6. **Inventory Management**: Product stock management (assume separate service)
7. **Shipping Carrier Integration**: Actual shipping label generation
8. **Email Service**: Order confirmation emails (assume separate service)

---

## System Constraints

### Technical Constraints

1. **Payment Gateway Latency**: 1-3 seconds per payment
   - Must handle gateway timeouts gracefully
   - Retry logic for transient failures

2. **Inventory Consistency**: Cannot oversell
   - Strong consistency required
   - Distributed locking for inventory updates

3. **Cart Expiration**: 30 minutes idle timeout
   - Release reserved inventory on timeout
   - Clean up abandoned carts

4. **Order Immutability**: Orders cannot be modified after confirmation
   - Only cancellations allowed (before shipping)
   - Audit trail for all changes

### Business Constraints

1. **Payment Processing Fees**: 2-3% per transaction
   - Minimize unnecessary payment attempts
   - Idempotent payment processing

2. **Inventory Cost**: Holding inventory is expensive
   - Accurate inventory tracking
   - Prevent overselling

3. **Legal**: PCI-DSS compliance required
   - Secure payment data handling
   - Audit logging

---

## Success Metrics

How do we know if the checkout system is working well?

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Checkout completion rate | > 30% | Completed orders / Cart views |
| Average checkout time | < 30 seconds | Time from cart to confirmation |
| Payment success rate | > 98% | Successful payments / Payment attempts |
| Inventory accuracy | 99.9% | No overselling incidents |
| Cart abandonment rate | < 70% | Abandoned carts / Cart creations |
| Order processing latency | < 5 seconds | Time from payment to order confirmation |

---

## User Stories

### Story 1: Complete Purchase Flow

```
As a customer,
I want to add items to my cart and complete checkout,
So that I can purchase products online.

Acceptance Criteria:
- Add items to cart with quantities
- View cart with accurate totals
- Enter shipping information
- Select payment method
- Complete payment securely
- Receive order confirmation
- Order appears in order history
```

### Story 2: Inventory Reservation

```
As a system,
I want to reserve inventory during checkout,
So that items are not oversold.

Acceptance Criteria:
- Check inventory availability before checkout
- Reserve inventory when checkout starts
- Release reservation if checkout abandoned
- Prevent overselling
- Update inventory on order confirmation
```

### Story 3: Handle Payment Failures

```
As a customer,
I want to retry payment if it fails,
So that I can complete my purchase.

Acceptance Criteria:
- Display clear error message on payment failure
- Allow payment retry
- Preserve cart and shipping information
- Support alternative payment methods
- Release inventory reservation if payment fails
```

---

## Core Components Overview

### 1. Cart Service
- Manages shopping cart state
- Calculates totals and taxes
- Applies discounts
- Persists cart data

### 2. Inventory Service
- Checks product availability
- Reserves inventory
- Releases reservations
- Updates stock levels

### 3. Payment Service
- Processes payments
- Integrates with payment gateways
- Handles payment failures
- Manages payment methods

### 4. Order Service
- Creates orders
- Manages order state
- Tracks order history
- Handles order cancellations

### 5. Shipping Service
- Calculates shipping costs
- Validates addresses
- Manages shipping options

### 6. Notification Service
- Sends order confirmations
- Notifies about order status
- Handles email/SMS notifications

---

## Interview Tips

### What Interviewers Look For

1. **Inventory Management**: How to prevent overselling
2. **Payment Processing**: Idempotency and failure handling
3. **Distributed Transactions**: Saga pattern for order creation
4. **Consistency**: Strong consistency for critical operations

### Common Mistakes

1. **Ignoring Inventory**: Not reserving inventory leads to overselling
2. **Payment Idempotency**: Duplicate payments on retries
3. **No Saga Pattern**: Inconsistent state on failures
4. **Weak Consistency**: Inventory overselling

### Good Follow-up Questions from Interviewer

- "How do you handle inventory when checkout times out?"
- "What happens if payment succeeds but order creation fails?"
- "How do you prevent race conditions in inventory updates?"
- "How do you handle partial inventory (some items available, some not)?"

---

## Summary

| Aspect | Decision |
|-------|----------|
| Primary use case | Complete e-commerce purchases securely |
| Scale | 10 million orders/day, 1,000 orders/second peak |
| Key challenge | Inventory consistency and payment idempotency |
| Architecture pattern | Saga pattern for distributed transactions |
| Consistency | Strong for inventory/payments, eventual for cart |
| Payment | Multiple gateways with idempotency |


