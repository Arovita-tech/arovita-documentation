                     ┌──────────────────────┐
                     │      Frontend        │
                     └──────────┬───────────┘
                                │
                         HTTPS (JWT)
                                │
                     ┌──────────▼───────────┐
                     │     API Gateway      │
                     └──────────┬───────────┘
                                │
                       Cognito Authorizer
                                │
                     ┌──────────▼───────────┐
                     │   Backend Service    │
                     │ (Lambda or ECS/Fargate)
                     └───────┬─────────┬────┘
                             │         │
                   ┌─────────▼───┐     │
                   │ PostgreSQL  │     │
                   │ (RDS)       │     │
                   └─────────────┘     │
                             │         │
                   ┌─────────▼───┐     │
                   │  Redis      │     │
                   │ (ElastiCache)│    │
                   └─────────────┘     │
                                       │
                             ┌─────────▼─────────┐
                             │ LabStack Provider │
                             │ API               │
                             └───────────────────┘
                                       │
                             ┌─────────▼─────────┐
                             │  SQS (Webhook Q)  │
                             └─────────┬─────────┘
                                       │
                             ┌─────────▼─────────┐
                             │ Webhook Processor │
                             │ Lambda            │
                             └───────────────────┘


---

# 📘 Teleconsultation – Full Lifecycle Architecture

**Version:** 1.0
**Scope:** Teleconsult End-to-End (Discovery → Booking → Completion → Refund)
**Owner:** Backend Team

---

# 1️⃣ High-Level Lifecycle Overview

Teleconsultation lifecycle consists of:

```plaintext
DISCOVER
   ↓
SELECT SLOT
   ↓
BOOK (Internal)
   ↓
PAY (External)
   ↓
BOOK WITH LABSTACK
   ↓
CONFIRMED
   ↓
RESCHEDULE (optional)
   ↓
CANCEL (optional)
   ↓
COMPLETED
   ↓
DOCUMENTS & PRESCRIPTION
   ↓
REFUND (TBC)
```

---

# 2️⃣ Complete System Architecture Diagram

```plaintext
Frontend
   │
   ▼
API Gateway
   │
   ├────────── availability-api ───────▶ LabStack (proxy)
   │
   ├────────── booking-initiate
   │                │
   │                ▼
   │         PostgreSQL (appointments)
   │                │
   │                ▼
   │          Razorpay (external)
   │
   ├────────── payment-webhook
   │                │
   │                ▼
   │          SQS: booking-queue
   │                │
   │                ▼
   │          booking-processor ───────▶ LabStack (bookAppointment)
   │
   ├────────── appointment-api (read-only)
   │
   ├────────── appointment-reschedule ─▶ LabStack (reschedule)
   │
   ├────────── appointment-cancel ─────▶ LabStack (cancel)
   │
   ├────────── refund-processor ───────▶ Razorpay (refund)
   │
   └────────── labstack-webhook
                     │
                     ▼
               SQS: labstack-webhook-queue
                     │
                     ▼
              webhook-processor
```



---


# 🔷  webhook-processor (SQS Trigger)

Triggered by:

```
labstack-webhook-queue
```

**Responsibilities:**

* Idempotency check
* Update appointment_status
* Update provider snapshot
* Insert documents
* Insert prescription
* Insert events
* Handle:

  * CONFIRMED
  * COMPLETED
  * CANCELED
  * RESCHEDULED

---


---

#  Proxy Rules (Important)

Only these Lambdas call LabStack:

* availability-api
* booking-processor
* appointment-reschedule
* appointment-cancel
* provider-sync

Frontend NEVER directly calls LabStack.

---

#  State Machine (Complete)

```plaintext
PAYMENT_PENDING
   ↓
PAYMENT_CONFIRMED
   ↓
BOOKING_IN_PROGRESS
   ↓
BOOKED
   ↓
CONFIRMED
   ↓
RESCHEDULED (optional)
   ↓
COMPLETED
```

OR

```plaintext
BOOKED → CANCELED → REFUND_PENDING → REFUNDED
```

---

#  Async Boundaries

Async happens at:

* Payment confirmation
* Booking execution
* LabStack webhook processing
* Refund processing

Everything that can fail → async.

---

# Safety Features

✔ UUIDv7
✔ Idempotency key
✔ Version column (optimistic locking)
✔ Webhook event_hash
✔ SQS DLQs
✔ Raw payload logging
✔ Snapshot storage

---

---

# 📘 Teleconsultation Backend – Final Architecture

**Version:** 1.0
**Status:** Implementation Ready
**Goal:** Robust, Minimal, Scalable

---

#  Architecture Principles

1. Do NOT call LabStack for browsing every time.
2. Store provider & procedure data locally.
3. User-facing APIs must be consolidated.
4. Async operations must be isolated.
5. Webhooks must be isolated.
6. Keep Lambda count minimal.
7. Every heavy operation must be async.
8. Every external system must be isolated.

---

#  Final Microservices Layout

We will use **4 Lambdas total**.

---

## 🔵 provider-sync

**Type:** Scheduled (EventBridge)

**Purpose:**
Synchronize provider & procedure data from LabStack into our DB.

**Calls LabStack APIs:**

* getSpecialities
* getProviders
* getProcedures

**Writes To:**

* labstack_providers_cache
* labstack_procedures_cache

**Runs:**

* Daily (or twice daily)

**Never exposed via API Gateway.**

---

## 🟢 teleconsult-api (User-Facing)

**Type:** API Gateway Lambda

Handles ALL frontend APIs:

### Routes

### Provider & Availability

```
GET  /providers
GET  /availability/providers
GET  /availability/slots
```

* Providers → DB only
* Availability → proxy LabStack (real-time)

---

### Booking

```
POST /bookings/initiate
```

* Insert appointment (PAYMENT_PENDING)
* Generate UUIDv7
* Return response

---

### Appointment Queries

```
GET /appointments
GET /appointments/{id}
GET /appointments/{id}/documents
GET /appointments/{id}/prescription
```

* DB only

---

### Online Meeting

```
GET /appointments/{id}/meeting-link
```

* Call LabStack getAuthenticatedMeetingLink

---

### Reschedule

```
POST /appointments/{id}/reschedule
```

* Call LabStack reschedule
* Update DB

---

### Cancel

```
POST /appointments/{id}/cancel
```

* Call LabStack cancel
* Update DB
* Trigger refund (if applicable)

---

## 🟡 payment-api

**Type:** API Gateway Lambda

Isolated for payment security.

### Routes

```
POST /payments/initiate
POST /webhooks/payment
```

Responsibilities:

* Create Razorpay order
* Verify Razorpay webhook
* Update appointment → PAYMENT_CONFIRMED
* Push booking queue message

---

## 🔴 async-processor

**Type:** SQS-triggered Lambda

Handles ALL async work.

Consumes:

* teleconsult-booking-queue
* labstack-webhook-queue
* refund-queue

---

# Queues & DLQs

---

## teleconsult-booking-queue

Trigger:

* payment-api webhook

Consumed by:

* async-processor

Purpose:

* Call LabStack bookAppointment
* Update DB
* Handle booking failures

DLQ:

* teleconsult-booking-dlq

---

## labstack-webhook-queue

Trigger:

* vendor-webhook Lambda

Consumed by:

* async-processor

Purpose:

* Update appointment_status
* Store prescription
* Store documents
* Insert events

DLQ:

* labstack-webhook-dlq

---

## refund-queue

Trigger:

* cancel flow
* booking failure

Consumed by:

* async-processor

DLQ:

* refund-dlq

---

# 4️⃣ Vendor Webhook Lambda

## 🟣 vendor-webhook

**Type:** API Gateway Lambda

Route:

```
POST /webhooks/labstack
```

Responsibilities:

* Validate request
* Store raw payload
* Compute event_hash
* Push message to labstack-webhook-queue
* Return 200 immediately


---

# 5️⃣ Full End-to-End Flow

---

## 🔹 Provider Sync Flow

```
EventBridge → provider-sync
      ↓
LabStack APIs
      ↓
Update DB cache
```

---

## 🔹 User Discovery Flow

```
User → teleconsult-api
      ↓
DB (providers)
```

Availability:

```
User → teleconsult-api
      ↓
LabStack (real-time)
```

---

## 🔹 Booking Flow

```
User → teleconsult-api
      ↓
Insert appointment (PAYMENT_PENDING)
      ↓
payment-api initiate
      ↓
User pays
      ↓
payment-api webhook
      ↓
teleconsult-booking-queue
      ↓
async-processor
      ↓
LabStack bookAppointment
      ↓
Update DB
```

---

## 🔹 Confirmation Flow

```
LabStack → vendor-webhook
           ↓
   labstack-webhook-queue
           ↓
      async-processor
           ↓
Update appointment_status
Store prescription
Store documents
```

---

## 🔹 Online Consultation Flow

```
User → teleconsult-api
GET /meeting-link
      ↓
LabStack getAuthenticatedMeetingLink
      ↓
Return token link
```

---

## 🔹 Reschedule Flow

```
User → teleconsult-api
      ↓
Call LabStack reschedule
      ↓
Update DB
```

---

## 🔹 Cancel Flow

```
User → teleconsult-api
      ↓
Call LabStack cancel
      ↓
Update DB
      ↓
Trigger refund-queue
```

---

#  State Ownership

| Component         | Owner              |
| ----------------- | ------------------ |
| Slot locking      | LabStack           |
| Meeting hosting   | LabStack           |
| Booking lifecycle | Both               |
| Payment           | Us                 |
| Refund            | Us                 |
| Prescription      | LabStack generates |
| Status truth      | LabStack webhook   |
| Internal state    | Our DB             |

---


#  Final Microservice Map

| Lambda          | Purpose                     |
| --------------- | --------------------------- |
| provider-sync   | Sync providers & procedures |
| teleconsult-api | All user APIs               |
| payment-api     | Payment handling            |
| vendor-webhook  | Receive LabStack callbacks  |
| async-processor | All async background work   |

---



