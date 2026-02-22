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

# 3️⃣ Lambda Naming Standard


Examples:

| Domain        | Lambda Name            |
| ------------- | ---------------------- |
| Provider      | provider-sync          |
| Availability  | availability-api       |
| Booking       | booking-initiate       |
| Booking Async | booking-processor      |
| Payment       | payment-webhook        |
| Appointment   | appointment-api        |
| Appointment   | appointment-reschedule |
| Appointment   | appointment-cancel     |
| Refund        | refund-processor       |
| Vendor        | labstack-webhook       |
| Vendor Async  | webhook-processor      |

All lowercase. Hyphen-separated. Two words preferred.

---

# 4️⃣ Queues & DLQs

We separate async processing clearly.

---

## 🔵 1. teleconsult-booking-queue

Purpose:

* Trigger booking after payment confirmation

Triggered by:

* payment-webhook

Consumed by:

* booking-processor

DLQ:

```
teleconsult-booking-dlq
```

---

## 🟡 2. labstack-webhook-queue

Purpose:

* Process LabStack appointment status updates
* Process document notifications

Triggered by:

* labstack-webhook

Consumed by:

* webhook-processor

DLQ:

```
labstack-webhook-dlq
```

---

## 🔴 3. refund-queue (Optional TBC)

Purpose:

* Async refund handling

Triggered by:

* appointment-cancel
* booking-processor (on failure)

Consumed by:

* refund-processor

DLQ:

```
refund-dlq
```

---

# 5️⃣ Complete Lambda Responsibilities

---

# 🔷 1. provider-sync

**Type:** Scheduled Lambda (EventBridge)

**Purpose:**

* Sync providers from LabStack
* Sync procedures
* Store locally

**Calls LabStack:**

* getProviders
* getSpecialities
* getProcedures

---

# 🔷 2. availability-api

**Type:** API Lambda

**Routes:**

```
GET /providers
GET /availability/providers
GET /availability/slots
```

**Responsibilities:**

* Fetch providers from DB
* Proxy real-time availability calls to LabStack
* Transform response
* Never store slots

Acts as proxy.

---

# 🔷 3. booking-initiate

**Route:**

```
POST /bookings/initiate
```

**Responsibilities:**

* Validate request
* Create appointment (internal_status = PAYMENT_PENDING)
* Generate UUIDv7
* Generate idempotency_key
* Create Razorpay order
* Return payment order

Does NOT call LabStack.

---

# 🔷 4. payment-webhook

**Route:**

```
POST /payments/webhook
```

**Responsibilities:**

* Verify Razorpay signature
* Update internal_status = PAYMENT_CONFIRMED
* Push SQS message (booking-queue)
* Idempotent

Never calls LabStack.

---

# 🔷 5. booking-processor (SQS Trigger)

Triggered by:

```
teleconsult-booking-queue
```

**Responsibilities:**

* Fetch appointment
* Validate state
* Optimistic lock update
* Call LabStack bookAppointment
* Update:

  * labstack_appointment_id
  * appointment_status
  * internal_status = BOOKED
* On failure:

  * internal_status = FAILED
  * Trigger refund-queue (optional)

---

# 🔷 6. labstack-webhook

**Route:**

```
POST /labstack/webhook
```

**Responsibilities:**

* Receive LabStack callback
* Log webhook payload
* Compute event_hash
* Push to labstack-webhook-queue
* Return 200 quickly

Never processes heavy logic directly.

---

# 🔷 7. webhook-processor (SQS Trigger)

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

# 🔷 8. appointment-api

**Routes:**

```
GET /appointments
GET /appointments/{id}
GET /appointments/{id}/documents
GET /appointments/{id}/prescription
```

Read-only.
Queries DB only.

Never calls LabStack.

---

# 🔷 9. appointment-reschedule

**Route:**

```
POST /appointments/{id}/reschedule
```

**Responsibilities:**

* Validate eligibility
* Call LabStack rescheduleAppointment
* Update:

  * appointment_datetime
  * appointment_status = RESCHEDULED
* Insert event

---

# 🔷 10. appointment-cancel

**Route:**

```
POST /appointments/{id}/cancel
```

**Responsibilities:**

* Validate eligibility
* Call LabStack cancelAppointment
* Update appointment_status
* Push refund-queue (if required)

---

# 🔷 11. refund-processor

Triggered by:

```
refund-queue
```

**Responsibilities:**

* Check refund eligibility
* Call Razorpay refund API
* Update internal_status = REFUNDED
* Insert refund event

---

# 6️⃣ Proxy Rules (Important)

Only these Lambdas call LabStack:

* availability-api
* booking-processor
* appointment-reschedule
* appointment-cancel
* provider-sync

Frontend NEVER directly calls LabStack.

---

# 7️⃣ State Machine (Complete)

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

# 8️⃣ Async Boundaries

Async happens at:

* Payment confirmation
* Booking execution
* LabStack webhook processing
* Refund processing

Everything that can fail → async.

---

# 9️⃣ Production Safety Features

✔ UUIDv7
✔ Idempotency key
✔ Version column (optimistic locking)
✔ Webhook event_hash
✔ SQS DLQs
✔ Raw payload logging
✔ Snapshot storage

---


