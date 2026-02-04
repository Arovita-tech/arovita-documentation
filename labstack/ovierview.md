## ovierview

# Diagnostic / LabStack Integration – LLD (Demo Version)


---

## 1️⃣ Starting Point – Network / Catalog Sync

### What problem this solves

* LabStack owns **tests, packages, prices, metadata**
* We **do NOT fetch this on every user request**
* This data changes **slowly** (weekly / bi-weekly)

### What we do

* A **Catalog Sync Lambda** runs:

  * once a week (cron)

### APIs used (LabStack)

* `network/getPackages`
* `network/getPackage`
* `network/getTests`
* `network/getPackagePreparations`

### What gets stored

* `diagnostic_packages`
* `diagnostic_tests`
* `package_tests`
* `package_lab_prices`
* `package_preparations`

### Why this is important

* User browsing is **fast**
* No external dependency for catalog
* Prices & metadata are **consistent**

📌 **Demo line:**

> “We treat LabStack network APIs as *reference data* and sync them periodically.”

---

## 2️⃣ User Entry – Catalog Browsing (Sync APIs)

When user opens **Labs / Diagnostics page**:

### Backend APIs (our system)

* `GET /catalog/packages`
* `GET /catalog/tests`
* `GET /catalog/search?q=thyroid`

### Data source

* **Postgres only**
* No LabStack calls here

### Characteristics

* Paginated (cursor-based)
* Stateless
* Fast
* Safe to scale


---

## 3️⃣ Availability & Slots (Real-Time, Sync)

Once user selects:

* location (pincode / lat-long)
* tests or package

### Backend API

* `GET /availability`

### What happens

* Backend calls **LabStack availability APIs**

  * `availability/checkServiceability`
  * `availability/getSlots`

### Why this is real-time

* Availability depends on:

  * location
  * date
  * lab workload

### What we return

* serviceable labs
* slot windows
* service type (home / center)


---

## 4️⃣ Quote & Selection (Sync)

User:

* selects tests/packages
* sees final price
* picks preferred slot

### Backend API

* `GET /quote`

### Data sources

* Prices → DB (from catalog sync)
* Availability → already known

📌 **Important**

* **No booking yet**
* **No payment yet**
* Just a **calculated view**

---

## 5️⃣ Order Creation (Sync)

### User action

Clicks **“Book Now”**

### Backend API

* `POST /orders`

### What we do

1. Validate user (Cognito)
2. Create internal order
3. Store:

   * selected tests
   * address snapshot
   * preferred slot
4. Status → `CREATED`

### DB tables involved

* `diagnostic_orders`
* `diagnostic_order_events`


> “At this stage, it’s only our internal order — no lab interaction.”

---

## 6️⃣ Payment Flow (Sync + Async)

### Step A: Payment Initiation (Sync)

* `POST /payments/initiate`
* Creates Razorpay order
* Stores payment intent

### Step B: User Pays (External)

* Razorpay Checkout (FE)

### Step C: Webhook (Async)

* Razorpay → `/payments/webhook`

### What webhook does

* Verify signature
* Update payment status
* Mark order as **PAID**
* Emit internal event

📌 **Key idea**

> Webhook is the **source of truth**, not FE callback.

---

## 7️⃣ Order Processor (Async)

Triggered **after payment success**

### What it does

1. Calls LabStack:

   * `order/placeOrder`
2. Stores:

   * labstack order id
   * lab assigned
3. Updates order status

### DB updates

* `diagnostic_orders`
* `diagnostic_order_events`


> “This is where our system hands off to LabStack.”

---

## 8️⃣ Lab Execution & Updates (Async)

### LabStack actions

* assigns phlebo
* collects sample
* processes tests
* generates report

### How we know

* **LabStack Webhooks**

### What webhook updates

* phlebo details
* order status
* report links

### Stored in

* `diagnostic_order_events`
* `diagnostic_reports`


> “From this point on, the flow is event-driven.”

---

## 9️⃣ User Tracking & Reports (Sync)

User checks:

* order status
* phlebo details
* reports

### Backend APIs

* `GET /orders`
* `GET /orders/{id}`
* `GET /orders/{id}/reports`

### Data source

* Our DB only

📌 **No LabStack dependency for reads**

---

## 10️⃣ Sync vs Async Summary (Very Important Slide)

### Synchronous

* Catalog browsing
* Availability
* Quote
* Order creation
* Payment initiation

### Asynchronous

* Payment confirmation (webhook)
* Lab order placement
* Lab status updates
* Report delivery


> “Anything that can fail or retry is async.”

---

## 11️⃣ Services Used

### Core

* API Gateway
* Lambda
* Postgres
* Cognito

### External

* LabStack APIs
* Razorpay

### Async

* Webhooks
* EventBridge / SQS later

---


---

## Finally

> “We periodically sync catalog data from LabStack.
> Users browse everything from our DB.
> Availability is checked in real time.
> Orders and payments are synchronous up to payment.
> Everything after payment is async and webhook-driven.
> This keeps the system fast, reliable, and scalable.”

---

