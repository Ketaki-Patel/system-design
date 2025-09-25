
# Payment System (ex Stripe, paypal, square etc)

<img width="820" height="463" alt="image" src="https://github.com/user-attachments/assets/1e3338bb-78d4-4405-aff0-0fd6d1234737" />

Here's how the payment process works step by step:

**1. Customer Initiates Payment**
- Customer enters payment details (card number, CVV, etc.) on the merchant's website
- This happens through a secure form, often embedded via the payment gateway's SDK

**2. Merchant Website to Payment Gateway**
- Merchant's website securely sends payment data to the Payment Gateway API
- The gateway tokenizes sensitive card data for security
- Initial validation happens (format checks, basic fraud screening)

**3. Payment Processing**
- Payment Gateway forwards the transaction to a Payment Processor
- Processor formats the request according to card network standards (Visa, Mastercard, etc.)

**4. Bank Network Authorization**
- Request goes through the card network (Visa/Mastercard) to the customer's bank
- Customer's bank checks account balance, fraud patterns, and approves/declines
- Response travels back through the network

**5. Response Handling**
- Payment Gateway receives the authorization response
- Updates transaction status in database
- Fraud detection systems may flag suspicious transactions
- Notifications are sent to both merchant and customer

**6. Settlement (Later)**
- Approved funds are transferred from customer's bank to merchant's bank
- This typically happens in batches, 1-3 business days later
- Payment gateway handles the settlement process

**Key Components:**
- **Payment Gateway**: Secure interface that handles the entire flow
- **Database**: Stores transaction records and merchant data
- **Fraud Detection**: AI/ML systems that analyze patterns
- **Notifications**: Email/SMS confirmations and webhooks to merchants

This simplified architecture handles billions of transactions by being stateless, highly available, and processing payments asynchronously where possible.

# Payment Gateway vs Payment Processor vs Stripe
Great question! Let me break down the differences between these three concepts:

## Payment Gateway
**What it is:** A service that securely captures and transmits payment data
- **Function:** Acts as the "front door" - handles the initial payment request, tokenization, and basic validation
- **Examples:** Authorize.Net, Square Gateway, PayPal Payments Pro
- **Analogy:** Like a cashier at a store who takes your card and processes it

## Payment Processor  
**What it is:** The backend system that actually moves money between banks
- **Function:** Handles the complex bank-to-bank communication, authorization, and settlement
- **Examples:** First Data, Chase Paymentech, Worldpay
- **Analogy:** Like the bank's computer system that checks your account and moves the money

## Stripe
**What it is:** A **full-stack payment platform** that combines both gateway AND processor
- **Function:** Handles everything from payment capture to bank settlement, plus developer tools
- **Unique value:** Complete solution with APIs, dashboard, fraud protection, and global reach
- **Analogy:** Like having both the cashier AND the bank system in one integrated service

## Key Differences:

**Traditional Setup:**
```
Merchant → Payment Gateway → Payment Processor → Banks
```

**Stripe Setup:**
```
Merchant → Stripe (Gateway + Processor + More) → Banks
```

**Why Stripe is Different:**
- **All-in-one:** You don't need separate gateway and processor contracts
- **Developer-first:** Clean APIs and extensive documentation
- **Global:** Built-in support for international payments and currencies
- **Additional services:** Subscription billing, marketplace payments, financial reporting

**When to use what:**
- **Traditional Gateway + Processor:** When you need maximum control or have specific enterprise requirements
- **Stripe:** When you want simplicity, fast integration, and modern tooling
- **Other full-stack solutions:** PayPal, Square, Adyen (Stripe competitors)

Think of it this way: Payment gateways and processors are like buying separate components, while Stripe is like buying a complete, integrated system.

<img width="812" height="467" alt="image" src="https://github.com/user-attachments/assets/c65e7d84-3e43-462d-b5ea-3eb22546a5f3" />

# More details on how payment system work
---

````markdown
# 💳 Payment System Design: Auth, Capture, Refund, Chargeback

---

## 1. Core Concepts

### Entities

#### 1. Merchant
- The business selling goods/services (e.g., online store).
- Initiates payment requests via a payment gateway.

#### 2. Payment Gateway
- Interface between merchant and financial network.
- Collects card details, sends **authorization/capture** requests, returns responses.
- Examples: Stripe, Razorpay, PayPal.

#### 3. Payment Processor
- Moves actual money between banks.
- Executes card network communications.
- Usually hidden behind the gateway.

#### 4. Customer
- Provides card/payment details to complete purchase.

---

## 2. Key Payment Operations

### Authorization (Auth)
- Reserve funds on the customer’s card.
- Money is not moved yet.
- Status: `authorized`

### Capture
- Finalize and move the reserved funds to merchant.
- Can be **full or partial**.
- Status: `captured`

### Refund
- Return money to customer after capture.
- Status: `refunded`

### Chargeback
* A **chargeback** happens when a customer disputes a transaction with their bank.
* The bank temporarily **withdraws** the disputed funds from the merchant’s account and **puts them on hold**.
* During this process, the payment status changes to **`chargeback`**.
* The merchant is notified and can **respond with evidence** to dispute the chargeback.

---

## Chargeback Resolution and Payment Status

| Resolution Outcome       | Payment Status After Resolution          | Description                                                   |
| ------------------------ | ---------------------------------------- | ------------------------------------------------------------- |
| **In favor of Customer** | `chargeback` (or `chargeback_finalized`) | Merchant loses the disputed amount; customer keeps the money. |
| **In favor of Merchant** | `captured`                               | Chargeback reversed; merchant retains funds.                  |

---

## 3. Database Schema

```sql
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    user_id INT,
    order_id INT,
    amount DECIMAL(10,2),
    currency VARCHAR(3),
    status VARCHAR(20),       -- pending, authorized, captured, refunded, chargeback
    auth_id VARCHAR(50),
    capture_id VARCHAR(50),
    refund_id VARCHAR(50),
    chargeback_id VARCHAR(50),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Sample Data
INSERT INTO payments (user_id, order_id, amount, currency, status, auth_id, capture_id, refund_id, chargeback_id, created_at, updated_at)
VALUES
(1, 101, 100.00, 'USD', 'authorized', 'auth_001', NULL, NULL, NULL, NOW(), NOW()),
(2, 102, 50.00, 'USD', 'captured', 'auth_002', 'cap_002', NULL, NULL, NOW(), NOW()),
(3, 103, 200.00, 'USD', 'chargeback', 'auth_003', 'cap_003', NULL, 'cb_003', NOW(), NOW());
````

---

## 4. API Endpoints with Sample Requests and Responses

### 4.1 Authorize Payment

**POST** `/payments`

#### Request

```json
{
  "user_id": 1,
  "order_id": 101,
  "amount": 100.00,
  "currency": "USD",
  "capture": false
}
```

#### Response

```json
{
  "payment_id": 1,
  "status": "authorized",
  "auth_id": "auth_001",
  "amount": 100.00,
  "currency": "USD",
  "created_at": "2025-09-25T14:00:00Z"
}
```

---

### 4.2 Capture Payment

**POST** `/payments/{payment_id}/capture`

#### Request

```json
{
  "amount": 100.00
}
```

#### Response

```json
{
  "payment_id": 1,
  "status": "captured",
  "auth_id": "auth_001",
  "capture_id": "cap_001",
  "amount": 100.00,
  "currency": "USD",
  "captured_at": "2025-09-25T14:05:00Z"
}
```

---

### 4.3 Refund Payment

**POST** `/payments/{payment_id}/refund`

#### Request

```json
{
  "amount": 100.00
}
```

#### Response

```json
{
  "payment_id": 1,
  "status": "refunded",
  "refund_id": "refund_001",
  "amount": 100.00,
  "currency": "USD",
  "refunded_at": "2025-09-25T15:00:00Z"
}
```

---

### 4.4 Chargeback (Webhook from Stripe/Bank)

**POST** `/webhooks/chargeback`

#### Request

```json
{
  "payment_id": 3,
  "chargeback_id": "cb_003",
  "amount": 200.00,
  "reason": "fraudulent",
  "created_at": "2025-09-25T16:00:00Z"
}
```

#### Response

```json
{
  "payment_id": 3,
  "status": "chargeback",
  "chargeback_id": "cb_003",
  "amount": 200.00,
  "currency": "USD",
  "notified_at": "2025-09-25T16:01:00Z"
}
```

---

## 5. Payment Status Flow (State Machine)

```text
pending -> authorized -> captured -> refunded
                             -> chargeback
```

### Sample Table

| payment\_id | user\_id | order\_id | status     | auth\_id  | capture\_id | refund\_id | chargeback\_id |
| ----------- | -------- | --------- | ---------- | --------- | ----------- | ---------- | -------------- |
| 1           | 1        | 101       | authorized | auth\_001 | NULL        | NULL       | NULL           |
| 2           | 2        | 102       | captured   | auth\_002 | cap\_002    | NULL       | NULL           |
| 3           | 3        | 103       | chargeback | auth\_003 | cap\_003    | NULL       | cb\_003        |

---

## 6. System Design Considerations

* **Idempotency**: All API endpoints should handle retries gracefully.
* **Webhooks**: Used for asynchronous events like chargebacks and refunds.
* **Event-Driven Architecture**: Event queue (Kafka/RabbitMQ) can log payment events.
* **Auditing & Compliance**: Store immutable logs for PCI DSS compliance.
* **Fraud Detection**: Monitor unusual patterns (e.g., many chargebacks).
* **Timeouts & Retries**: Handle bank/gateway errors gracefully.

---

## 7. 💳 Payment Flow with Roles Explained

```text
Customer        Merchant           Stripe                Bank
(User)          (Store)      (Gateway + Processor)   (Issuer / Acquirer)
   |                |                    |                    |
   |-- Submit Card ->|                  [Gateway]             |
   |                 |-- Auth Request --->|                   |
   |                 |                  [Processor]           |
   |                 |                    |-- Auth Funds ----->|
   |                 |                    |<-- Auth Success ---|
   |<-- Auth Result -|                    |                    |
   |                 |-- Capture Request ->|                  |
   |                 |                  [Processor]           |
   |                 |                    |-- Move Funds ----->|
   |<-- Capture Result-|                  |                    |
   |                 |                    |                    |
   |                 |<-- Refund Request -|                    |
   |                 |                  [Processor]           |
   |                 |-- Refund Funds ---->|                   |
   |<-- Refund Result-|                   |                    |
   |                 |                  [Gateway]             |
   |                 |                    |<-- Chargeback Notice|
   |<-- Notify CB ---|                    |                    |
```

```
### ✅ Role Summary:

* **Customer (User)**: Makes the purchase with card.
* **Merchant (Store)**: Sells product/service and requests payment.
* **Stripe (Gateway + Processor)**:

  * **Gateway**: Accepts and routes card details.
  * **Processor**: Talks to the customer’s and merchant’s bank to move funds.
* **Bank**:

  * **Issuer**: Customer’s bank (auths/declines transactions).
  * **Acquirer**: Merchant’s bank (receives captured funds).
```
# sample java code reprensting 
* Auth
* Capture
* Refund
* Chargeback
* In-memory DB or JPA-ready
* RESTful API endpoints

---

> 🧩 **Tech Assumptions**:

* Using Spring Boot (`@RestController`, `@Service`, `@Repository`)
* Using H2 / JPA for persistence (can be swapped with PostgreSQL or MySQL)
* Focused on clarity, not production-ready security/error handling

---

### 📦 Folder Structure

```
com.example.payment
│
├── controller
│   └── PaymentController.java
├── model
│   └── Payment.java
├── repository
│   └── PaymentRepository.java
├── service
│   └── PaymentService.java
└── PaymentApplication.java
```

---

## 📁 `Payment.java` (Model)

```java
package com.example.payment.model;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
public class Payment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Integer userId;
    private Integer orderId;
    private BigDecimal amount;
    private String currency;
    private String status; // pending, authorized, captured, refunded, chargeback

    private String authId;
    private String captureId;
    private String refundId;
    private String chargebackId;

    private LocalDateTime createdAt = LocalDateTime.now();
    private LocalDateTime updatedAt = LocalDateTime.now();

    // Getters and setters
    // (Can be generated with Lombok @Data in real projects)
}
```

---

## 📁 `PaymentRepository.java`

```java
package com.example.payment.repository;

import com.example.payment.model.Payment;
import org.springframework.data.jpa.repository.JpaRepository;

public interface PaymentRepository extends JpaRepository<Payment, Long> {
}
```

---

## 📁 `PaymentService.java`

```java
package com.example.payment.service;

import com.example.payment.model.Payment;
import com.example.payment.repository.PaymentRepository;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.Optional;
import java.util.UUID;

@Service
public class PaymentService {

    private final PaymentRepository repo;

    public PaymentService(PaymentRepository repo) {
        this.repo = repo;
    }

    public Payment authorize(Payment input) {
        input.setStatus("authorized");
        input.setAuthId("auth_" + UUID.randomUUID());
        return repo.save(input);
    }

    public Payment capture(Long paymentId, BigDecimal amount) {
        Payment payment = repo.findById(paymentId).orElseThrow();
        payment.setStatus("captured");
        payment.setCaptureId("cap_" + UUID.randomUUID());
        payment.setUpdatedAt(java.time.LocalDateTime.now());
        return repo.save(payment);
    }

    public Payment refund(Long paymentId, BigDecimal amount) {
        Payment payment = repo.findById(paymentId).orElseThrow();
        payment.setStatus("refunded");
        payment.setRefundId("refund_" + UUID.randomUUID());
        payment.setUpdatedAt(java.time.LocalDateTime.now());
        return repo.save(payment);
    }

    public Payment handleChargeback(Long paymentId, String reason) {
        Payment payment = repo.findById(paymentId).orElseThrow();
        payment.setStatus("chargeback");
        payment.setChargebackId("cb_" + UUID.randomUUID());
        payment.setUpdatedAt(java.time.LocalDateTime.now());
        return repo.save(payment);
    }
}
```

---

## 📁 `PaymentController.java`

```java
package com.example.payment.controller;

import com.example.payment.model.Payment;
import com.example.payment.service.PaymentService;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.Map;

@RestController
@RequestMapping("/payments")
public class PaymentController {

    private final PaymentService service;

    public PaymentController(PaymentService service) {
        this.service = service;
    }

    @PostMapping
    public Payment authorizePayment(@RequestBody Payment payment) {
        return service.authorize(payment);
    }

    @PostMapping("/{paymentId}/capture")
    public Payment capturePayment(@PathVariable Long paymentId, @RequestBody Map<String, Object> body) {
        BigDecimal amount = new BigDecimal(body.get("amount").toString());
        return service.capture(paymentId, amount);
    }

    @PostMapping("/{paymentId}/refund")
    public Payment refundPayment(@PathVariable Long paymentId, @RequestBody Map<String, Object> body) {
        BigDecimal amount = new BigDecimal(body.get("amount").toString());
        return service.refund(paymentId, amount);
    }

    @PostMapping("/webhooks/chargeback")
    public Payment handleChargeback(@RequestBody Map<String, Object> body) {
        Long paymentId = Long.parseLong(body.get("payment_id").toString());
        String reason = body.get("reason").toString();
        return service.handleChargeback(paymentId, reason);
    }
}
```

---

## 📁 `PaymentApplication.java`

```java
package com.example.payment;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PaymentApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentApplication.class, args);
    }
}
```

---

## ✅ Sample cURL Requests

```bash
# Authorize
curl -X POST http://localhost:8080/payments -H "Content-Type: application/json" -d '{
  "userId": 1,
  "orderId": 101,
  "amount": 100.00,
  "currency": "USD"
}'

# Capture
curl -X POST http://localhost:8080/payments/1/capture -H "Content-Type: application/json" -d '{
  "amount": 100.00
}'

# Refund
curl -X POST http://localhost:8080/payments/1/refund -H "Content-Type: application/json" -d '{
  "amount": 100.00
}'

# Chargeback
curl -X POST http://localhost:8080/payments/webhooks/chargeback -H "Content-Type: application/json" -d '{
  "payment_id": 1,
  "reason": "fraudulent"
}'
```


---

###  **pom.xml**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                        http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example.payment</groupId>
    <artifactId>payment-system</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>Payment System</name>
    <description>Basic Payment System Design</description>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Add dependencies as needed -->
    </dependencies>

</project>
```

---

## 🧠 Next Steps (Optional Enhancements)

* Add validation and exception handling
* Use DTOs instead of raw entity binding
* Add API auth (JWT)
* Add database migration (Flyway)
* Deploy to cloud or containerize (Docker)








