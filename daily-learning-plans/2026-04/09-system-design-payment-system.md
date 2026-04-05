# Day 33 — System Design Case Study: Payment System
**Date:** 2026-04-09
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**System Design Case Study: Payment System — When Correctness Is Non-Negotiable**

Payments are where distributed systems engineering meets hard real-world consequences. A duplicate charge, a missed transfer, or a lost payment directly harms users and creates legal liability. Every technique you've learned — idempotency, distributed transactions, event sourcing, circuit breakers — was designed for situations like this. Today you design a payment system where "exactly once" is not optional.

---

## 2. Requirements

**Functional:**
- User initiates payment (debit card, credit card, bank transfer)
- System charges the user and credits the merchant
- View payment history and statements
- Refunds
- Support multiple currencies

**Non-Functional:**
- 1M transactions/day = ~12 writes/second (peak: 3× = 36 TPS)
- Latency: p99 < 3 seconds for synchronous payment confirmation
- Correctness: ZERO duplicate charges, ZERO lost payments
- Auditability: every state change must be logged permanently
- Availability: 99.99% (4 minutes downtime/month max)
- PCI-DSS compliance (card data must be tokenised, never stored raw)

---

## 3. The Idempotency Problem

```
User clicks "Pay" → Network timeout → Did payment process?
Browser retries → Duplicate charge? ☠️

Solution: Idempotency Keys

Client generates unique key: idempotency_key = UUID per payment attempt
POST /v1/payments
  X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
  { amount: 100, currency: USD, ... }

Server stores: idempotency_key → (status, response)
  On first request: process payment, store result
  On duplicate request (same key): return stored result (no re-processing)

This guarantees: process exactly once, but client can safely retry.
```

```java
@RestController
@RequiredArgsConstructor
public class PaymentController {

    private final PaymentService paymentService;
    private final IdempotencyService idempotencyService;

    @PostMapping("/v1/payments")
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("X-Idempotency-Key") String idempotencyKey,
            @RequestBody PaymentRequest request) {

        // Check if this key was already processed
        Optional<PaymentResponse> cached = idempotencyService.get(idempotencyKey);
        if (cached.isPresent()) {
            return ResponseEntity.ok(cached.get());  // Return cached result
        }

        // Process payment
        PaymentResponse response = paymentService.process(request);

        // Cache result (TTL: 24h)
        idempotencyService.store(idempotencyKey, response);

        return ResponseEntity.ok(response);
    }
}

@Service
@RequiredArgsConstructor
public class IdempotencyService {

    private final RedisTemplate<String, PaymentResponse> redis;

    public Optional<PaymentResponse> get(String key) {
        PaymentResponse value = redis.opsForValue().get("idempotency:" + key);
        return Optional.ofNullable(value);
    }

    public void store(String key, PaymentResponse response) {
        redis.opsForValue().set("idempotency:" + key, response, Duration.ofHours(24));
    }
}
```

---

## 4. Double-Entry Bookkeeping

The foundation of all financial systems since 1494:

```
Every transaction = two equal and opposite entries

Transfer $100: Alice → Bob
  DEBIT  Alice's account: -$100
  CREDIT Bob's account:   +$100

Sum of all entries always = 0 ✅
Auditors can verify system integrity by checking sum

Database schema:
  account_id | balance
  alice      | 900
  bob        | 1100

  ledger_entry_id | account_id | amount | transaction_id | type
  1               | alice      | -100   | txn-001        | DEBIT
  2               | bob        | +100   | txn-001        | CREDIT
```

```java
@Service
@Transactional   // CRITICAL: both entries or neither
@RequiredArgsConstructor
public class LedgerService {

    private final LedgerRepository ledger;
    private final AccountRepository accounts;

    public void transfer(Long fromAccountId, Long toAccountId,
                         BigDecimal amount, String transactionId) {
        // Lock rows in consistent order (lower ID first) to prevent deadlock
        Long lockId1 = Math.min(fromAccountId, toAccountId);
        Long lockId2 = Math.max(fromAccountId, toAccountId);

        Account from = accounts.findByIdForUpdate(lockId1);
        Account to   = accounts.findByIdForUpdate(lockId2);

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException(fromAccountId);
        }

        // Atomic update
        accounts.debit(fromAccountId, amount);
        accounts.credit(toAccountId, amount);

        // Immutable audit log
        ledger.save(new LedgerEntry(fromAccountId, amount.negate(), transactionId, "DEBIT"));
        ledger.save(new LedgerEntry(toAccountId, amount, transactionId, "CREDIT"));
    }
}
```

---

## 5. Payment State Machine

```
PENDING → PROCESSING → SUCCEEDED
    │           │
    │      FAILED (e.g., card declined)
    │
CANCELLED (user cancelled before processing)

SUCCEEDED → REFUND_REQUESTED → REFUNDED

State stored in database, transitions are append-only events (event sourcing).
```

```java
@Entity
public class Payment {
    @Id private Long id;
    private String idempotencyKey;
    private Long userId;
    private Long merchantId;
    private BigDecimal amount;
    private String currency;

    @Enumerated(EnumType.STRING)
    private PaymentStatus status;

    private String failureCode;     // INSUFFICIENT_FUNDS, CARD_DECLINED, etc.
    private String paymentGatewayId; // Stripe charge ID for reconciliation

    // Version for optimistic locking (prevent concurrent state changes)
    @Version
    private Long version;
}
```

---

## 6. Architecture

```
Client
  │
  │ POST /v1/payments (with Idempotency-Key)
  ▼
API Gateway
  │ Rate limit: 10 payment requests/min per user
  │ Auth: JWT validation
  ▼
Payment Service
  │
  ├─── Idempotency check → Redis
  ├─── Validate: sufficient funds, valid card token
  ├─── Create PENDING payment record
  │
  ▼
Payment Gateway Integration
  │ (Stripe/Adyen/Braintree)
  │ Circuit breaker: Resilience4j
  │ Timeout: 10s
  │ Fallback: queue for async processing
  ▼
  Success/Failure → Update payment status

  → Publish PaymentCompletedEvent (Kafka)
    ├── Ledger Service: debit/credit accounts
    ├── Notification Service: email/push confirmation
    └── Fraud Detection: async scoring
```

### 6.1 Handling Gateway Timeouts (The Hard Case)

```
Payment Service → Stripe timeout (10s) → Status unknown!

Problem: Did Stripe charge the card? We don't know.
Solution: Reconciliation

1. On timeout: mark payment as PENDING (not FAILED)
2. Start reconciliation job (every 5 minutes):
   - Query Stripe for all payments in PENDING state
   - If Stripe says SUCCESS → mark SUCCESS in our DB
   - If Stripe says FAILED → mark FAILED
   - If Stripe has no record → payment didn't go through → mark FAILED
3. Use idempotencyKey as Stripe's idempotency key too:
   - Stripe will return same result for same idempotency key
```

---

## 7. Fraud Detection (Async)

```
NEVER block payment on fraud check (adds latency, availability risk)

Instead:
  1. Process payment immediately (< 3s SLA)
  2. Publish PaymentCompletedEvent to Kafka
  3. Fraud service consumes async:
     - ML model scores transaction
     - If HIGH RISK: flag account, trigger review
     - If CONFIRMED_FRAUD: initiate refund, notify user
     - Update user's fraud score for future transactions

Light real-time checks (block immediately):
  - Card in known stolen list
  - Transaction amount > configured limit for new accounts
  - Country mismatch (card issued in Germany, IP in Nigeria)
```

---

## 8. PCI-DSS Compliance

```
NEVER store raw card numbers (PAN) in your database.

Tokenisation flow:
  1. Client enters card details in Stripe.js (data goes directly to Stripe)
  2. Stripe returns a card token (tok_xxx) → send this to your backend
  3. Your backend stores: tok_xxx (safe, useless outside Stripe)
  4. For payment: send tok_xxx to Stripe → Stripe charges real card

Your database never sees: card number, CVV, expiry date
Your servers never touch raw card data → PCI scope reduced dramatically
```

---

## 9. Key Takeaways

1. **Idempotency keys are mandatory for payment APIs** — every client must send them, every server must honour them.
2. **Double-entry bookkeeping ensures financial integrity** — the ledger never lies.
3. **Gateway timeouts require reconciliation, not retry** — retrying a payment without idempotency = potential double charge.
4. **Fraud detection must be async** — never block the payment path on ML scoring.
5. **Never store raw card data** — tokenise via payment processor to reduce PCI scope.

---

## 10. Connection to Previous Days

Idempotency was introduced in Day 20 (API Design). The Saga pattern (Day 21) handles distributed transactions across payment + inventory + order services. Circuit breakers (Day 25) protect against Stripe unavailability. Event sourcing — storing immutable events (ledger entries) — is the natural model for financial data.

---

## 11. Resources

- **[Stripe API documentation](https://stripe.com/docs/api)** — Best-in-class API design for payments.
- **[Stripe Engineering Blog — Idempotency](https://stripe.com/blog/idempotency)** — How Stripe implements idempotency.
- **[The Payments System Architecture](https://gist.github.com/spirosoik/fde56ec580f25c5b3d6e)** — Overview of modern payment architecture.

---

*Day 33. In payments, "almost correct" is the same as "wrong" — the margin for error is zero.*
