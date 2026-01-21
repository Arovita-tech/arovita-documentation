## 📂 API Categories

---
Authentication

Doctor Profile & Settings

Availability Management

MRN & Bank Verification

Doctor Home Dashboard

Consultations

Patients

Prescriptions
--


## 🌐 Base URL
```
  https://ijcm73njbl.execute-api.ap-south-1.amazonaws.com/dev/
```

1️⃣ Authentication APIs
POST /doctor/get-otp

Send OTP for signup.

Request

{
  "phone_number": "+919845672310"
}


Response

{ "message": "otp_sent" }

--

POST /doctor/verify-otp

Verify OTP & create doctor.

Response

{
  "id_token": "eyJraWQiOi...",
  "access_token": "eyJraWQiOi...",
  "refresh_token": "..."
}

POST /doctor/login/get-otp

Send OTP for existing doctor login.

POST /doctor/login/verify-otp

Verify OTP and login.

POST /doctor/token/refresh

Refresh tokens.

POST /doctor/logout

Logout doctor session.

2️⃣ Doctor Profile APIs
GET /doctor/me

Fetch doctor profile.

Response

{
  "doctor": {
    "id": "b7e2c9fa-7f8b-4c2e-b7a8-9b4e92f0e4a3",
    "phone": "+919845672310",
    "full_name": "Dr. Arjun Raghav Rao",
    "years_experience": 13,
    "gender": "MALE",
    "verification_status": "VERIFIED",
    "is_active": true,
    "profile_data": {
      "expertise": ["General Medicine", "Diabetology"],
      "languages": ["English", "Hindi"]
    },
    "availability": {
      "instant_available": true,
      "weekly_availability": {
        "monday": [{"from": "10:00", "to": "13:00"}]
      }
    },
    "consultation_rate": {
      "amount": 700,
      "currency": "INR",
      "is_free": false
    },
    "status": "PROFILE_SET"
  }
}

POST /doctor/profile

Main onboarding/update endpoint.

Rules

Partial updates allowed

Sets status = PROFILE_SET

Request

{
  "gender": "MALE",
  "profile_data": {
    "expertise": ["Cardiology", "Internal Medicine"],
    "languages": ["English", "Hindi"]
  },
  "availability": {
    "instant_available": true,
    "weekly_availability": {
      "monday": [{"from": "10:00", "to": "13:00"}]
    }
  },
  "consultation_rate": {
    "amount": 800,
    "currency": "INR",
    "is_free": false
  }
}


Response

{ "message": "profile_updated" }

3️⃣ Availability APIs
PATCH /doctor/availability/instant

Toggle instant consultation availability.

Request

{ "instant_available": true }

PATCH /doctor/availability/weekly

Update weekly slots.

Request

{
  "weekly_availability": {
    "monday": [{"from": "10:00", "to": "13:00"}],
    "wednesday": [{"from": "14:00", "to": "18:00"}]
  }
}

4️⃣ MRN & Bank APIs
POST /doctor/mrn/verify

Submit MRN for verification (IDfy).

Request

{
  "mrn_number": "SMC/2018/81497",
  "council": "UTTAR PRADESH MEDICAL COUNCIL",
  "year_of_registration": 2018
}


Response

{ "status": "MRN_VERIFICATION_STARTED" }

GET /doctor/mrn/status

Fetch MRN verification status.

GET /doctor/bank

Get bank details.

PUT /doctor/bank/update

Update bank details.

Request

{
  "account_name": "Arjun Rao",
  "account_number": "123456789012",
  "ifsc": "HDFC0001234",
  "bank_name": "HDFC Bank"
}

5️⃣ Doctor Home
GET /doctor/home

Response

{
  "doctor_name": "Dr. Arjun Raghav Rao",
  "active_consultations": 1,
  "completed_consultations": 12
}

6️⃣ Consultation APIs
GET /doctor/consultations/upcoming
GET /doctor/consultations/active
GET /doctor/consultations/history
POST /doctor/consultations/start

---
Rules

Only doctor can start

Consultation must be CONFIRMED

Payment must be successful

---

POST /doctor/consultations/end

Rules

Only doctor can end

Status must be IN_PROGRESS

7️⃣ Patient APIs
GET /doctor/patients

List all patients consulted by doctor.

GET /doctor/patients/{user_id}

---

Fetch patient profile.

GET /doctor/patients/{user_id}/consultations

Fetch all consultations (past + upcoming) between doctor and patient.

---
8️⃣ Prescription APIs
🔒 Prescription Rules (STRICT)
Rule	Enforced
Consultation must be IN_PROGRESS or COMPLETED	✅
Only one prescription per consultation	✅
Doctor must own consultation	✅
POST /doctor/prescriptions

Request

{
  "consultation_id": "ffffffff-ffff-ffff-ffff-ffffffffffff",
  "user_id": "9c2b8d5e-1f64-4c4b-9e9c-3cbf0b98b712",
  "diagnosis": "Migraine",
  "medications": [
    {
      "name": "Paracetamol",
      "dosage": "500mg",
      "frequency": "Twice daily",
      "duration": "5 days"
    }
  ],
  "tests": "CT scan if pain persists",
  "advice": "Hydration, avoid stress",
  "follow_up": "After 7 days",
  "doctor_signature": "Dr. Arjun Raghav Rao"
}


Response

{
  "message": "prescription_saved",
  "prescription_id": "99999999-9999-9999-9999-999999999999"
}

---
GET /doctor/consultations/{consultation_id}/prescription

Response

{
  "prescription": {
    "id": "99999999-9999-9999-9999-999999999999",
    "doctor_name": "Dr. Arjun Raghav Rao",
    "diagnosis": "Migraine",
    "medications": [
      {
        "name": "Paracetamol",
        "dosage": "500mg",
        "frequency": "Twice daily",
        "duration": "5 days"
      }
    ],
    "tests": "CT scan if pain persists",
    "advice": "Hydration, avoid stress",
    "follow_up": "After 7 days",
    "created_at": "2026-01-20T17:48:29Z"
  }
}