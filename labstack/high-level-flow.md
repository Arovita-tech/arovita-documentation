This is your **LabStack Diagnostic Booking Platform — End-to-End Overview**.

---

# Big Picture

We built a **Diagnostic Booking Engine** that integrates with **LabStack APIs**.

The system allows:

1. Users to browse tests/packages
2. Check availability
3. Lock slots
4. Create orders
5. Pay
6. Lab processes the sample
7. Reports delivered
8. Status tracked via webhooks

Everything is split into clean microservices.

---

# 🏗 System Overview (High Level)

```
User (Mobile/Web App)
        ↓
API Gateway
        ↓
Multiple Lambdas (Microservices)
        ↓
PostgreSQL DB
        ↓
LabStack APIs
        ↓
Razorpay (Payments)
```

---

# 🔹 Microservices We Built

Here are the microservices and what they do:

---

## 1️⃣ Catalog Sync Service (Backend Data Loader)

### Purpose:

Fetch catalog data from LabStack and store in our DB.

### Calls LabStack APIs:

* `/v1.3/network/getPackages`
* `/v1.3/network/getPackage`
* `/v1.3/network/getTests`

### Runs:

* Once/twice a week (cron)
* Or manually

### Stores data in:

* `diagnostic_packages`
* `diagnostic_tests`
* `package_tests`
* `package_lab_prices`

📌 explanation:

> This service loads all available tests and packages from LabStack so our app can show them instantly without calling LabStack every time.

---

## 2️⃣ User Catalog Service (User-Facing)

### APIs:

* `GET /catalog/packages`
* `GET /catalog/tests`
* `GET /catalog/search`
* `GET /availability`
* `GET /quote`

### What it does:

* Shows packages/tests
* Pagination
* Search
* Check serviceability
* Quote pricing

### Reads from:

* Our PostgreSQL DB

📌 explanation:

> This is what the mobile app calls to show tests, packages, and availability.

---

## 3️⃣ Orders Service (User Creates Order)

### APIs:

* `POST /orders`
* `GET /orders`

### What it does:

* Validates Cognito user
* Validates slot availability
* Locks slot for 10 minutes
* Creates internal order
* Sets status = `CREATED`

### Tables:

* `diagnostic_orders`
* `diagnostic_slot_locks`
* `diagnostic_order_events`

📌 explanation:

> This creates the booking inside our system. No lab interaction yet.

---

## 4️⃣ Slot Locking Mechanism

When user selects slot:

* We lock slot for 10 minutes
* If payment not completed → expires

### Table:

`diagnostic_slot_locks`

### Status:

* TEMP_LOCKED
* CONFIRMED
* EXPIRED

📌 explanation:

> This prevents two users from booking same slot at same time.

---

## 5️⃣ Payment Service (Razorpay Integration)

### APIs:

* `POST /payments/initiate`
* `POST /payments/webhook`

### What it does:

Step 1 – Initiate

* Validates order
* Creates Razorpay order
* Stores idempotency key

Step 2 – Webhook

* Razorpay calls us
* We verify signature
* Update payment_status
* Update order status → `PAID`

### Tables:

* `diagnostic_payments`
* `diagnostic_orders`
* `diagnostic_order_events`

📌 explanation:

> Payment is confirmed only through webhook, not frontend.

---

## 6️⃣ Order Processor Service (Async)

Triggered after payment success.

### What it does:

* Calls LabStack API:

  * `/v1.3/order/placeOrder`
* Stores:

  * labstack_order_id
* Updates status → `LAB_PENDING`

📌 explanation:

> This is where we officially send order to LabStack.

---

## 7️⃣ LabStack Webhook Processor

LabStack sends updates:

* PHLEBO_ASSIGNED
* SAMPLE_COLLECTED
* REPORT_DELIVERED
* etc.

### What it does:

* Updates internal status
* Stores report links
* Stores phlebo details
* Adds audit event

### Tables:

* `diagnostic_orders`
* `diagnostic_order_events`
* `diagnostic_reports`

📌 explanation:

> This keeps our system in sync with lab progress.

---

# 🔄 Complete User Journey

Let’s walk through full flow:

---

## Step 1 – Browse

User opens app
→ Calls `/catalog/packages`

---

## Step 2 – Select Slot

User checks availability
→ Calls `/availability`

---

## Step 3 – Create Order

User clicks “Book Now”
→ `POST /orders`

Result:

* Order created
* Slot locked for 10 mins
* Status = CREATED

---

## Step 4 – Payment

User pays via Razorpay
→ Razorpay webhook hits `/payments/webhook`

Result:

* payment_status = SUCCESS
* order status = PAID

---

## Step 5 – Order Processor

Triggered automatically
→ Calls LabStack placeOrder

Result:

* labstack_order_id stored
* status = LAB_PENDING

---

## Step 6 – Lab Processes

LabStack webhook sends updates:

* PHLEBO_ASSIGNED
* SAMPLE_PROCESSED
* REPORT_DELIVERED

We update DB accordingly.

---

## Step 7 – User Checks Status

User calls:

* `GET /orders`

All data served from our DB.

No direct LabStack dependency for reads.

---

# 📦 Final Microservices Summary

| Microservice     | Purpose                |
| ---------------- | ---------------------- |
| Catalog Sync     | Load LabStack catalog  |
| User Catalog     | Show packages/tests    |
| Orders Service   | Create & lock orders   |
| Payment Service  | Handle Razorpay        |
| Order Processor  | Send order to LabStack |
| LabStack Webhook | Sync lab updates       |

---

# 🔐 Security Components

* Cognito JWT validation
* DB constraints
* Unique slot locking
* Idempotency key for payments
* Status state machine
* Webhook idempotency handling

---

# 📊 State Machine Summary

```
CREATED
   ↓ (payment success)
PAID
   ↓ (processor)
LAB_PENDING
   ↓ (lab scheduled)
CONFIRMED
   ↓ (report delivered)
COMPLETED
```

---

# 🗂 Tables Used

Core tables:

* `diagnostic_orders`
* `diagnostic_slot_locks`
* `diagnostic_payments`
* `diagnostic_order_events`
* `diagnostic_reports`
* `diagnostic_packages`
* `diagnostic_tests`

---

# What Is Required To Run This System

1. PostgreSQL
2. AWS Lambda
3. API Gateway
4. Cognito
5. Razorpay
6. LabStack API credentials
7. Secrets Manager
8. Scheduled cron for catalog sync

---


> We built a fully event-driven diagnostic booking system.

* Users browse from cached LabStack catalog.
* Orders are created internally.
* Slots are locked for 10 minutes.
* Payment is confirmed via webhook.
* Order sent to LabStack asynchronously.
* LabStack updates via webhook.
* Users always read from our database.
* System is idempotent, scalable, and production-ready.

---

