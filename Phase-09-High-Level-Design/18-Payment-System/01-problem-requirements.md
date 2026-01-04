# Payment System (Stripe-like) - Problem & Requirements

## What is a Payment System?

A Payment System processes financial transactions between parties, handling payment methods (credit cards, bank transfers), ensuring security (PCI compliance), providing idempotency guarantees, and maintaining accurate financial records through double-entry bookkeeping.

**Example:**
```
User initiates payment: $100 for subscription
  ↓
System validates payment method
  ↓
Charges payment method via payment processor
  ↓
Records transaction in ledger
  ↓
Confirms payment to user
```

### Why Does This Exist?

1. **E-commerce**: Enable online purchases
2. **Subscriptions**: Recurring billing
3. **Marketplaces**: Split payments between parties
4. **Donations**: Accept charitable contributions
5. **Invoicing**: Bill customers
6. **Refunds**: Process returns and cancellations

### What Breaks Without It?

- No online transactions possible
- Manual payment processing required
- No fraud protection
- No financial audit trail
- Compliance violations (PCI, financial regulations)

---

## Clarifying Questions

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the scale (transactions/day)? | Determines infrastructure | 10M transactions/day |
| What payment methods? | Affects complexity | Credit cards, ACH, digital wallets |
| Do we need PCI compliance? | Security requirement | Yes, mandatory |
| What's the transaction volume? | Cost calculation | $1B/month |
| Do we need reconciliation? | Financial accuracy | Yes, daily reconciliation |
| What's the fraud tolerance? | Risk management | < 0.1% fraud rate |

---

## Functional Requirements

### Core Features

1. **Payment Processing**
   - Charge credit cards
   - Process ACH transfers
   - Handle digital wallets (Apple Pay, Google Pay)
   - Support multiple currencies

2. **Idempotency**
   - Prevent duplicate charges
   - Idempotency keys for retries
   - Idempotent refunds

3. **Ledger System**
   - Double-entry bookkeeping
   - Account balances
   - Transaction history
   - Financial reporting

4. **Fraud Detection**
   - Real-time fraud scoring
   - Block suspicious transactions
   - Manual review queue

5. **Refunds**
   - Full and partial refunds
   - Automatic refund processing
   - Refund status tracking

6. **Reconciliation**
   - Match transactions with processor
   - Identify discrepancies
   - Generate reports

### Secondary Features

7. **Subscriptions**
   - Recurring billing
   - Proration
   - Subscription management

8. **Invoicing**
   - Generate invoices
   - Send to customers
   - Track payment status

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Payment processing | < 2s p95 | User expects quick confirmation |
| Ledger write | < 100ms p95 | Financial accuracy critical |
| Fraud check | < 500ms p95 | Real-time blocking |
| Reconciliation | < 1 hour | Daily batch process |

### Scale

| Metric | Value |
|--------|-------|
| Transactions/day | 10 million |
| QPS (average) | 115 |
| QPS (peak) | 1,000 |
| Transaction volume | $1B/month |

### Reliability

- **Availability**: 99.99% (52 minutes downtime/year)
- **Data Durability**: 100% (zero data loss)
- **Consistency**: Strong consistency (ACID transactions)
- **Idempotency**: 100% (no duplicate charges)

### Compliance

- **PCI DSS Level 1**: Required for card processing
- **SOX Compliance**: Financial reporting accuracy
- **GDPR**: User data protection
- **Audit Trail**: Immutable transaction logs

---

## What's Out of Scope

1. **Cryptocurrency**: Bitcoin, Ethereum payments
2. **International Tax**: Tax calculation and collection
3. **Loyalty Programs**: Points, rewards
4. **Credit Underwriting**: Loan approval
5. **Banking Services**: Accounts, loans

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Payment success rate | > 99% |
| Fraud rate | < 0.1% |
| Reconciliation accuracy | 100% |
| Payment latency p95 | < 2s |
| System availability | 99.99% |

---

## User Stories

### Story 1: Process Payment

```
As a merchant,
I want to charge a customer's credit card,
So that I can receive payment for goods.

Acceptance Criteria:
- Payment processed in < 2 seconds
- Idempotent (safe to retry)
- Transaction recorded in ledger
- Fraud check performed
```

### Story 2: Process Refund

```
As a merchant,
I want to refund a customer,
So that I can process returns.

Acceptance Criteria:
- Refund processed within 24 hours
- Original transaction identified
- Ledger updated correctly
- Customer notified
```

---

## Core Components

1. **Payment Gateway**: Interface to payment processors
2. **Ledger Service**: Double-entry bookkeeping
3. **Fraud Service**: Real-time fraud detection
4. **Reconciliation Service**: Match transactions
5. **Idempotency Service**: Prevent duplicates

---

## Summary

| Aspect | Decision |
|--------|----------|
| Scale | 10M transactions/day |
| Payment latency | < 2s p95 |
| Consistency | Strong (ACID) |
| Compliance | PCI DSS Level 1 |
| Idempotency | 100% required |


