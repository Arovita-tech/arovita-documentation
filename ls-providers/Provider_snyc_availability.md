
# 📘 Teleconsultation – Provider Sync & Availability Services

**Version:** 1.0
**Scope:** Teleconsultation – Discovery & Availability Layer
**Owner:** Backend Team

---

# 1️⃣ Introduction

This document explains the architecture and design of:

1. **Provider Sync Service**
2. **Availability Service**

These two services together power:

* Doctor discovery
* Speciality filtering
* Consultation pricing
* Real-time slot availability

This document does NOT cover:

* Booking service
* Payment integration
* Webhook handling
* Diagnostics module

Those are separate modules.

---

# 2️⃣ Why We Need These Services

LabStack does NOT provide:

* A bulk "get all doctors" API
* Pricing information
* Master provider export

Their APIs are location-based and availability-based.

Therefore:

We must build a **local provider metadata layer** and a **real-time availability proxy layer**.

This ensures:

* Fast browsing experience
* Internal pricing control
* Vendor independence
* Scalable system design

---

# 3️⃣ System Overview

We divide the system into two clear components:

---

## 🟢 A. Provider Sync Service (Background Job)

Purpose:

* Discover providers from LabStack
* Store metadata locally
* Sync specialities and procedures
* Attach pricing

Runs:

* Daily via EventBridge
* Or manually (triggered)

---

## 🟡 B. Availability Service (User-Facing)

Purpose:

* Handle user search by pincode
* Fetch real-time appointment slots
* Validate provider serviceability

Runs:

* On every frontend request

---

# 4️⃣ Architecture Diagram

```plaintext
EventBridge (Cron)
        ↓
Provider Sync Lambda
        ↓
LabStack APIs
        ↓
PostgreSQL (RDS)


Frontend
    ↓
API Gateway
    ↓
Availability Lambda
    ↓
LabStack (real-time calls)
```

---

# 5️⃣ LabStack APIs Used

We use only these APIs for this layer:

### Sync Service:

* GET /availability/getSpecialities
* GET /availability/getProviders
* GET /availability/getProcedures

### Availability Service:

* GET /availability/getProviders
* GET /availability/getAppointmentSlots

We DO NOT call:

* Booking APIs
* Admin APIs
* Document APIs
* Webhooks (separate module)

---

# 6️⃣ Provider Sync Service (Detailed Design)

---

## 🎯 Objective

Build a local, queryable provider database.

Since LabStack does not provide a master provider export, we use a **Coverage-Based Sync Strategy**.

---

## 🔄 Sync Strategy

We define a list of operational coverage cities:

Example:

```python
COVERAGE_CITIES = [
    {"city": "Bangalore", "pincode": "560001"},
    {"city": "Mumbai", "pincode": "400001"}
]
```

For each:

* Provider Type (DOCTOR, PSYCHOLOGIST, etc.)
* City / Pincode

We call:

```plaintext
GET /availability/getProviders
```

Extract unique providers and upsert into DB.

---

## 🧠 Why This Works

LabStack returns providers serviceable in a location.

By querying for all operational cities, we gradually build a complete dataset for our business coverage.

---

## 🗄 Database Tables Used

### providers

Stores provider metadata:

* ls_provider_id
* provider_type
* name
* gender
* years_experience
* specialities
* languages
* supported_appointment_types
* profile_url
* active flag
* last_synced_at

---

### provider_specialities

Stores:

* speciality
* sub-specialities
* tags
* provider_type

---

### provider_procedures

Stores:

* lsProcedureId
* name
* speciality
* instructions

---

### provider_sync_logs

Tracks:

* provider_type
* city
* status
* number of providers synced
* error messages

---

## 🔁 Sync Flow

Pseudo Logic:

```python
for provider_type in PROVIDER_TYPES:

    sync_specialities(provider_type)
    sync_procedures(provider_type)

    for city in COVERAGE_CITIES:
        providers = get_providers(provider_type, city.pincode)

        for provider in providers:
            upsert_provider(provider)

mark_missing_providers_inactive()
```

---

## 🛑 Important Rules

* Always use UPSERT (ON CONFLICT DO UPDATE)
* Never delete providers
* Mark inactive if not seen for 7 days
* Log every sync execution
* Retry failed calls (max 3 attempts)

---

# 7️⃣ Availability Service (Detailed Design)

---

## 🎯 Objective

Handle real-time availability and slot queries.

This service does NOT store availability permanently.

Availability is dynamic and must always be fetched live.

---

## 🟢 API 1: GET /providers

Returns:

* Provider metadata
* Pricing
* Experience
* Supported appointment types

Data Source:

* PostgreSQL only

No LabStack call.

---

## 🟡 API 2: GET /availability/providers

Purpose:
Check if providers are serviceable in specific pincode.

Flow:

1. Validate request
2. Call LabStack getProviders
3. Cross-check with local DB (active providers only)
4. Return filtered result

---

## 🟡 API 3: GET /availability/slots

Purpose:
Fetch real-time appointment slots.

Flow:

1. Validate provider exists locally
2. Call LabStack getAppointmentSlots
3. Transform response
4. Return slot timings

No DB write happens here.

---

# 8️⃣ Code Structure

```plaintext
teleconsult/
  provider_sync/
    handler.py
    labstack_client.py
    sync_providers.py
    sync_specialities.py
    db_repository.py

  availability_api/
    handler.py
    labstack_client.py
    provider_service.py
    slot_service.py
    validation.py
```

---

# 9️⃣ Error Handling Strategy

### Sync Failures:

* Log error
* Continue next provider type
* Do not block entire job

### Availability Failures:

* Return HTTP 503
* Log LabStack error
* Do not crash Lambda

---

# 🔟 Security

* API Gateway protected by Cognito
* LabStack API key stored in Secrets Manager
* Lambda inside VPC
* RDS accessible only via security group
* No direct frontend access to LabStack

---

# 1️⃣1️⃣ Why Two Lambdas?

We separate because:

| Sync Lambda     | Availability Lambda |
| --------------- | ------------------- |
| Batch job       | Real-time           |
| Heavy DB writes | Read-only           |
| Runs daily      | Runs per request    |
| Not user-facing | User-facing         |

This prevents:

* Performance interference
* Deployment risk overlap
* Scaling conflict

---

# 1️⃣2️⃣ Scalability Plan

For MVP:

* Daily sync
* Single availability Lambda

At scale:

* Parallel sync per provider type
* Add Redis for provider caching
* Add monitoring alarms
* Add rate limiting

---

# 1️⃣3️⃣ What We Do NOT Sync

We DO NOT sync:

* Appointment slots
* Real-time availability
* Follow-up eligibility
* Recent appointment data

Those must always be live.

---

# 1️⃣4️⃣ Summary

The Provider Sync & Availability Services:

* Build local provider metadata layer
* Attach internal pricing
* Remove LabStack dependency for browsing
* Fetch availability in real-time
* Provide clean separation of concerns

This is the foundation for Teleconsultation.

---
