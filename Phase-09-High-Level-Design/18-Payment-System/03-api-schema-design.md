# Payment System - API & Schema Design

## API Design Philosophy

Payment system APIs prioritize:

1. **Idempotency**: All write operations must be idempotent
2. **Security**: PCI DSS compliance, encryption, tokenization
3. **Reliability**: Strong consistency, zero data loss
4. **Auditability**: Complete transaction history
5. **Compliance**: SOX, PCI DSS, financial regulations

---

## Base URL Structure

```
REST API:     https://api.payments.example.com/v1
Webhooks:     https://webhooks.payments.example.com/v1
```

---

## API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) because:
- Easy to understand and implement
- Clear in logs and documentation
- Allows running multiple versions simultaneously

**Backward Compatibility Rules:**

Non-breaking changes (no version bump):
- Adding new optional fields
- Adding new endpoints
- Adding new error codes

Breaking changes (require new version):
- Removing fields
- Changing field types
- Changing endpoint paths

**Deprecation Policy:**
1. Announce deprecation 12 months in advance
2. Return Deprecation header
3. Maintain old version for 24 months after new version release

---

## Authentication

### API Key Authentication

```http
Authorization: Bearer sk_live_abc123def456...
```

**Key Types:**
- **Live keys**: Production transactions
- **Test keys**: Sandbox testing
- **Restricted keys**: Limited permissions

---

## Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Tier | Requests/minute |
|------|-----------------|
| Free | 100 |
| Pro | 1,000 |
| Enterprise | 10,000 |

---

## Error Model

All error responses follow this standard envelope structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "type": "card_error",
    "details": {
      "field": "card_number",
      "reason": "Card declined"
    },
    "request_id": "req_123456"
  }
}
```

**Error Codes Reference:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_REQUEST | Request validation failed |
| 400 | CARD_DECLINED | Card was declined |
| 400 | INSUFFICIENT_FUNDS | Insufficient funds |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Resource not found |
| 409 | CONFLICT | Duplicate transaction |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

---

## Idempotency Implementation

**Idempotency-Key Header:**

Clients must include `Idempotency-Key` header for all write operations:
```http
Idempotency-Key: <uuid-v4>
```

**Deduplication Storage:**

Idempotency keys are stored in Redis with 24-hour TTL:
```
Key: idempotency:{key}
Value: Serialized response
TTL: 24 hours
```

**Retry Semantics:**

1. Client sends request with Idempotency-Key
2. Server checks Redis for existing key
3. If found: Return cached response (same status code + body)
4. If not found: Process request, cache response, return result
5. Retries with same key within 24 hours return cached response

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/payments | Yes | Idempotency-Key header |
| POST /v1/refunds | Yes | Idempotency-Key header |
| POST /v1/payment_methods | Yes | Idempotency-Key header |
| PUT /v1/payments/{id} | Yes | Idempotency-Key or version-based |

---

## Core API Endpoints

### 1. Process Payment

**Create Payment**

`POST /v1/payments`

**Request:**
```http
POST /v1/payments HTTP/1.1
Host: api.payments.example.com
Authorization: Bearer sk_live_abc123
Content-Type: application/json
Idempotency-Key: idemp_xyz789

{
  "amount": 10000,
  "currency": "usd",
  "payment_method_id": "pm_abc123",
  "description": "Subscription payment for Premium plan",
  "metadata": {
    "order_id": "order_123",
    "customer_id": "cust_456"
  },
  "capture": true,
  "statement_descriptor": "PREMIUM SUBSCRIPTION"
}
```

**Response (200 OK):**
```json
{
  "payment_id": "pay_abc123",
  "status": "succeeded",
  "amount": 10000,
  "currency": "usd",
  "payment_method": {
    "id": "pm_abc123",
    "type": "card",
    "card": {
      "last4": "4242",
      "brand": "visa",
      "exp_month": 12,
      "exp_year": 2025
    }
  },
  "description": "Subscription payment for Premium plan",
  "metadata": {
    "order_id": "order_123",
    "customer_id": "cust_456"
  },
  "created_at": "2024-01-20T10:00:00Z",
  "updated_at": "2024-01-20T10:00:01Z"
}
```

**Error Response (400 Bad Request):**
```json
{
  "error": {
    "code": "CARD_DECLINED",
    "message": "Your card was declined.",
    "type": "card_error",
    "details": {
      "decline_code": "insufficient_funds",
      "reason": "Insufficient funds in account"
    },
    "request_id": "req_abc123"
  }
}
```

---

### 2. Get Payment

**Retrieve Payment**

`GET /v1/payments/{payment_id}`

**Response (200 OK):**
```json
{
  "payment_id": "pay_abc123",
  "status": "succeeded",
  "amount": 10000,
  "currency": "usd",
  "payment_method": {
    "id": "pm_abc123",
    "type": "card",
    "card": {
      "last4": "4242",
      "brand": "visa"
    }
  },
  "created_at": "2024-01-20T10:00:00Z"
}
```

---

### 3. Process Refund

**Create Refund**

`POST /v1/refunds`

**Request:**
```json
{
  "payment_id": "pay_abc123",
  "amount": 5000,
  "reason": "customer_request",
  "metadata": {
    "refund_reason": "Product defect",
    "support_ticket": "ticket_789"
  },
  "idempotency_key": "idemp_refund_xyz"
}
```

**Response (200 OK):**
```json
{
  "refund_id": "ref_xyz789",
  "payment_id": "pay_abc123",
  "amount": 5000,
  "currency": "usd",
  "status": "succeeded",
  "reason": "customer_request",
  "created_at": "2024-01-20T11:00:00Z"
}
```

---

### 4. Payment Methods

**Create Payment Method**

`POST /v1/payment_methods`

**Request:**
```json
{
  "type": "card",
  "card": {
    "number": "4242424242424242",
    "exp_month": 12,
    "exp_year": 2025,
    "cvc": "123"
  },
  "billing_details": {
    "name": "John Doe",
    "email": "john@example.com",
    "address": {
      "line1": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postal_code": "94102",
      "country": "US"
    }
  }
}
```

**Response (200 OK):**
```json
{
  "payment_method_id": "pm_abc123",
  "type": "card",
  "card": {
    "last4": "4242",
    "brand": "visa",
    "exp_month": 12,
    "exp_year": 2025
  },
  "billing_details": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "created_at": "2024-01-20T10:00:00Z"
}
```

---

### 5. List Payments

**List Payments**

`GET /v1/payments`

**Query Parameters:**
- `limit` (optional): Max results (default: 10, max: 100)
- `starting_after` (optional): Cursor for pagination
- `ending_before` (optional): Cursor for pagination
- `status` (optional): Filter by status (succeeded, pending, failed)
- `created` (optional): Filter by date range

**Response (200 OK):**
```json
{
  "data": [
    {
      "payment_id": "pay_abc123",
      "status": "succeeded",
      "amount": 10000,
      "currency": "usd",
      "created_at": "2024-01-20T10:00:00Z"
    }
  ],
  "has_more": true,
  "next_cursor": "pay_xyz789"
}
```

---

## Database Schema Design

### Transactions Table

```sql
CREATE TABLE transactions (
    transaction_id VARCHAR(50) PRIMARY KEY,
    payment_id VARCHAR(50) NOT NULL UNIQUE,
    
    -- Amount and currency
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    
    -- Payment method
    payment_method_id VARCHAR(50) NOT NULL,
    payment_method_type VARCHAR(20) NOT NULL,
    
    -- Status
    status VARCHAR(20) NOT NULL,
    failure_code VARCHAR(50),
    failure_message TEXT,
    
    -- Metadata
    description TEXT,
    metadata JSONB,
    statement_descriptor VARCHAR(22),
    
    -- Payment processor
    processor_transaction_id VARCHAR(100),
    processor_response JSONB,
    
    -- Fraud check
    fraud_check_id VARCHAR(50),
    fraud_score DECIMAL(5, 2),
    fraud_status VARCHAR(20),
    
    -- Idempotency
    idempotency_key VARCHAR(100) UNIQUE,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processed_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'succeeded', 'failed', 'canceled', 'refunded')),
    CONSTRAINT valid_currency CHECK (currency IN ('usd', 'eur', 'gbp', 'jpy', 'cad', 'aud'))
);

-- Indexes
CREATE INDEX idx_transactions_payment_id ON transactions(payment_id);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_created_at ON transactions(created_at DESC);
CREATE INDEX idx_transactions_idempotency_key ON transactions(idempotency_key);
CREATE INDEX idx_transactions_payment_method ON transactions(payment_method_id);
CREATE INDEX idx_transactions_processor_id ON transactions(processor_transaction_id);
```

---

### Ledger Entries Table

```sql
CREATE TABLE ledger_entries (
    entry_id BIGSERIAL PRIMARY KEY,
    transaction_id VARCHAR(50) NOT NULL,
    
    -- Account information
    account_id VARCHAR(50) NOT NULL,
    account_type VARCHAR(20) NOT NULL,  -- asset, liability, equity, revenue, expense
    
    -- Amounts
    debit DECIMAL(12, 2),
    credit DECIMAL(12, 2),
    balance DECIMAL(12, 2) NOT NULL,
    
    -- Metadata
    description TEXT,
    reference VARCHAR(100),
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT fk_transaction FOREIGN KEY (transaction_id) 
        REFERENCES transactions(transaction_id),
    CONSTRAINT valid_account_type CHECK (account_type IN ('asset', 'liability', 'equity', 'revenue', 'expense')),
    CONSTRAINT debit_or_credit CHECK (
        (debit IS NOT NULL AND credit IS NULL) OR 
        (debit IS NULL AND credit IS NOT NULL)
    )
);

-- Indexes
CREATE INDEX idx_ledger_transaction ON ledger_entries(transaction_id);
CREATE INDEX idx_ledger_account ON ledger_entries(account_id, created_at DESC);
CREATE INDEX idx_ledger_created_at ON ledger_entries(created_at DESC);
```

---

### Payment Methods Table

```sql
CREATE TABLE payment_methods (
    payment_method_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,
    
    -- Type
    type VARCHAR(20) NOT NULL,  -- card, ach, wallet
    
    -- Card details (tokenized)
    card_last4 VARCHAR(4),
    card_brand VARCHAR(20),
    card_exp_month SMALLINT,
    card_exp_year SMALLINT,
    card_token VARCHAR(100),  -- Encrypted token
    
    -- ACH details
    ach_account_type VARCHAR(20),  -- checking, savings
    ach_last4 VARCHAR(4),
    ach_routing_number VARCHAR(9),
    
    -- Billing details
    billing_name VARCHAR(255),
    billing_email VARCHAR(255),
    billing_address JSONB,
    
    -- Status
    is_default BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT valid_type CHECK (type IN ('card', 'ach', 'wallet'))
);

-- Indexes
CREATE INDEX idx_payment_methods_user ON payment_methods(user_id, is_active);
CREATE INDEX idx_payment_methods_default ON payment_methods(user_id, is_default) WHERE is_default = TRUE;
```

---

### Refunds Table

```sql
CREATE TABLE refunds (
    refund_id VARCHAR(50) PRIMARY KEY,
    payment_id VARCHAR(50) NOT NULL,
    transaction_id VARCHAR(50) NOT NULL,
    
    -- Amount
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    
    -- Status
    status VARCHAR(20) NOT NULL,
    reason VARCHAR(50),
    failure_code VARCHAR(50),
    failure_message TEXT,
    
    -- Metadata
    metadata JSONB,
    
    -- Processor
    processor_refund_id VARCHAR(100),
    processor_response JSONB,
    
    -- Idempotency
    idempotency_key VARCHAR(100) UNIQUE,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processed_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT fk_payment FOREIGN KEY (payment_id) 
        REFERENCES transactions(payment_id),
    CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'succeeded', 'failed', 'canceled'))
);

-- Indexes
CREATE INDEX idx_refunds_payment ON refunds(payment_id);
CREATE INDEX idx_refunds_status ON refunds(status);
CREATE INDEX idx_refunds_created_at ON refunds(created_at DESC);
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐
│   payment_methods   │
├─────────────────────┤
│ payment_method_id   │
│ user_id             │
│ type                │
│ card_last4          │
└──────────┬──────────┘
           │
           │ 1:N
           ▼
┌─────────────────────┐
│   transactions      │
├─────────────────────┤
│ transaction_id (PK) │
│ payment_id          │
│ payment_method_id   │
│ amount              │
│ status              │
│ idempotency_key     │
└──────────┬──────────┘
           │
           │ 1:2
           ▼
┌─────────────────────┐
│  ledger_entries     │
├─────────────────────┤
│ entry_id (PK)       │
│ transaction_id (FK) │
│ account_id          │
│ debit               │
│ credit              │
│ balance             │
└─────────────────────┘

┌─────────────────────┐
│     refunds         │
├─────────────────────┤
│ refund_id (PK)      │
│ payment_id (FK)     │
│ amount              │
│ status              │
└─────────────────────┘
```

---

## Summary

| Component | Technology/Approach |
| ----------------- | --------------------------------------- |
| API | REST with idempotency keys |
| Database | PostgreSQL (ACID transactions) |
| Ledger | Double-entry bookkeeping |
| Idempotency | Redis with 24h TTL |
| Security | PCI DSS Level 1, tokenization |
