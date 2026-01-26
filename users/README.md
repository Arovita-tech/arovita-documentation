Below is a **clean, production-ready `README.md`** that you can directly drop into your repo.
It includes **base URL everywhere**, **all endpoints**, **request/response**, and **user appointment views (upcoming + history)**.

---

# 🩺 Arovita Backend APIs — README

## 🌐 Base URL

```
https://toqj4qcitk.execute-api.ap-south-1.amazonaws.com/dev
```

All endpoints below are relative to this base URL.

---

# 🔑 Auth Service (`user-auth`)

---

## 1️⃣ Send OTP

**POST**

```
/user/get-otp
```

### Request Body

```json
{
  "phone_number": "+919876543210"
}
```

### Response

```json
{
  "message": "otp_sent"
}
```

---

## 2️⃣ Signup – Verify OTP & Create User

**POST**

```
/user/signup/verify
```

### Request Body

```json
{
  "phone_number": "+919876543210",
  "otp": "123456",
  "password": "StrongPass@123",
  "email": "user@example.com",
  "first_name": "Tony",
  "last_name": "Stark",
  "gender": "MALE",
  "dob": "1999-08-12"
}
```

### Response

```json
{
  "AccessToken": "ACCESS_TOKEN",
  "IdToken": "ID_TOKEN",
  "RefreshToken": "REFRESH_TOKEN",
  "ExpiresIn": 3600,
  "TokenType": "Bearer"
}
```

---

## 3️⃣ Login (Password)

**POST**

```
/user/login/password
```

### Request Body

```json
{
  "phone_number": "+919876543210",
  "password": "StrongPass@123"
}
```

### Response

```json
{
  "AccessToken": "ACCESS_TOKEN",
  "IdToken": "ID_TOKEN",
  "RefreshToken": "REFRESH_TOKEN",
  "ExpiresIn": 3600
}
```

---

## 4️⃣ Refresh Token

**POST**

```
Authorization: Bearer <ID_TOKEN>
```

```
/user/token/refresh
```

### Request Body

```json
{
  "refresh_token": "REFRESH_TOKEN"
}
```

---

## 5️⃣ Logout

**POST**

```
/user/logout
```


### Headers

```
Authorization: Bearer <ACCESS_TOKEN>
```
### Request Body

```json
{}
```

### Response

```json
{
  "message": "logged_out"
}
```

---

## 6️⃣ Change Password

**POST**

```
/user/change-password
```


### Headers

```
Authorization: Bearer <ACCESS_TOKEN>
```

### Request Body

```json
{
    "old_password": "Vinay@tcs43",
    "new_password": "KN@tcsvinay43"
}
```

### Response

```json
{
  "message": "password_changed"
}
```

---

# 👤 User Profile Service (`user-profile`)

---

## 7️⃣ Get Logged-in User Profile

**GET**

```
/user/me
```

### Headers

```
Authorization: Bearer <ID_TOKEN>
```

### Response

```json
{
  "id": "uuid",
  "first_name": "Tony",
  "last_name": "Stark",
  "gender": "MALE",
  "dob": "1999-08-12",
  "profile_completed": false
}
```

---

# 🩺 Health Survey Service (`user-health`)

---

## 8️⃣ Save / Update Health Survey

**POST**

```
/user/health
```

### Headers

```
Authorization: Bearer <ID_TOKEN>
```

### Request Body

```json
{
  "food_allergies": ["peanuts"],
  "medicine_allergies": ["penicillin"],
  "pollen": ["grass"],
  "others": [],
  "height_cm": 175,
  "weight_kg": 70,
  "smoking_status": "NEVER",
  "alcohol_consumption": "OCCASIONALLY",
  "sleep_hours_per_night": 8,
  "physical_activity_level": "MODERATE",
  "stress_level": "LOW",
  "consent_medical": true,
  "consent_data_processing": true,
  "consent_privacy_policy": true,
  "consent_notifications": false
}
```

### Response

```json
{
  "message": "health_profile_saved"
}
```

---

## 9️⃣ Get Health Survey Details

**GET**

```
/user/health/details
```

### Headers

```
Authorization: Bearer <ID_TOKEN>
```

### Response

```json
{
  "completed": true,
  "survey": {
    "food_allergies": ["peanuts"],
    "height_cm": 175,
    "weight_kg": 70,
    "smoking_status": "NEVER",
    "alcohol_consumption": "OCCASIONALLY",
    "physical_activity_level": "MODERATE",
    "stress_level": "LOW"
  }
}
```

---

## 🔟 Health Survey Status

**GET**

```
/user/health/status
```

### Headers

```
Authorization: Bearer <ID_TOKEN>
```

### Response

```json
{
  "completed": true
}
```

---

# 🧠 Consultation Service (`consultation-service`)

---

## 11 Instant Consultation – List Doctors

**GET**

```
Authorization: Bearer <ID_TOKEN>
```

```
/consultation/instant/doctors?speciality=General Medicine
```

### Response

```json
{
  "doctors": [
    {
      "doctor_id": "uuid",
      "full_name": "Dr. Strange",
      "years_experience": 10,
      "consultation_rate": {
        "video": 500
      }
    }
  ]
}
```

---

## 12️⃣ Scheduled Consultation – List Doctors

**GET**

```
/consultation/scheduled/doctors?speciality=General Medicine&date=2026-01-20
```

---

## 13️⃣ Create Instant Consultation

**POST**

```
/consultation/instant/create
```

### Request Body

```json
{
  "doctor_id": "uuid",
  "consultation_mode": "VIDEO",
  "language": "EN",
  "intake_snapshot": {
    "symptoms": ["fever", "cough"],
    "severity": "moderate"
  }
}
```

### Response

```json
{
  "appointment_id": "uuid",
  "consultation_id": "uuid",
  "status": "INITIATED"
}
```

---

## 14️⃣ Create Scheduled Consultation

**POST**

```
/consultation/scheduled/create
```

### Request Body

```json
{
  "doctor_id": "uuid",
  "consultation_mode": "VIDEO",
  "appointment_date": "2026-01-20",
  "slot_start": "10:00",
  "slot_end": "10:30",
  "language": "EN",
  "intake_snapshot": {
    "symptoms": ["headache"]
  }
}
```

---

## 15️⃣ Start Consultation (Doctor)   Strictly to done from the doctor side not the user side.

**POST**

```
Authorization: Bearer <ID_TOKEN>
```

```
/consultation/start
```

### Request Body

```json
{
  "consultation_id": "uuid"
}
```

---

## 16️⃣ End Consultation

**POST**

```
Authorization: Bearer <ID_TOKEN>
```

```
/consultation/end
```

### Request Body

```json
{
  "consultation_id": "uuid"
}
```

---

# 💳 Payment Service (`payment-service`)  Ignored for the beta version.

---

## 17️⃣ Razorpay Webhook (Backend Only)

**POST**

```
/payments/webhook
```

✔ Verifies payment signature
✔ Confirms appointment
✔ Moves consultation to `WAITING`

---

# 📅 User Appointment Views

---

## 18️⃣ Upcoming Appointments (User)

**GET**

```
/user/consultations/upcoming
```

### Headers

```
Authorization: Bearer <ID_TOKEN>
```

### Response

```json
{
  "appointments": [
    {
      "appointment_id": "uuid",
      "consultation_id": "uuid",
      "doctor_name": "Dr. Strange",
      "doctor_specialty": "Cardiology",
      "appointment_date": "2026-01-20",
      "slot_start": "10:00",
      "consultation_mode": "VIDEO",
      "status": "CONFIRMED"
    }
  ]
}
```

---

## 19️⃣ Appointment History (User)

**GET**

```
Authorization: Bearer <ID_TOKEN>
```

```
/user/consultations/history
```

### Response

```json
{
  "appointments": [
    {
      "appointment_id": "uuid",
      "doctor_name": "Dr. House",
      "appointment_date": "2025-12-10",
      "consultation_mode": "VIDEO",
      "status": "COMPLETED"
    }
  ]
}
```

---

```



