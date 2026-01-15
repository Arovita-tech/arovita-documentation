🌐 Base URL
https://z0xmua74i2.execute-api.ap-south-1.amazonaws.com/dev


All endpoints below are relative to this base URL.

🔑 Auth Service (user-auth)
1️⃣ Send OTP

Method: POST
Endpoint:

/user/get-otp


Full URL

https://z0xmua74i2.execute-api.ap-south-1.amazonaws.com/dev/user/get-otp


Request Body

{
  "phone_number": "+919876543210"
}


Response

{
  "message": "otp_sent"
}

2️⃣ Signup – Verify OTP & Create User

Method: POST
Endpoint:

/user/signup/verify


Full URL

https://z0xmua74i2.execute-api.ap-south-1.amazonaws.com/dev/user/signup/verify


Request Body

{
  "phone_number": "+919876543210",
  "otp": "123456",
  "password": "StrongPass@123",
  "email": "user@example.com",
  "first_name": "Tony",
  "last_name": "Stark",
  "gender": "male",
  "dob": "1999-08-12"
}


Response

{
  "AccessToken": "...",
  "IdToken": "...",
  "RefreshToken": "...",
  "ExpiresIn": 3600,
  "TokenType": "Bearer"
}

3️⃣ Login (Password)

Method: POST
Endpoint:

/user/login


Full URL

https://z0xmua74i2.execute-api.ap-south-1.amazonaws.com/dev/user/login


Request Body

{
  "phone_number": "+919876543210",
  "password": "StrongPass@123"
}

4️⃣ Refresh Token

Method: POST
Endpoint:

/user/token/refresh


Full URL

https://z0xmua74i2.execute-api.ap-south-1.amazonaws.com/dev/user/token/refresh


Request Body

{
  "refresh_token": "REFRESH_TOKEN"
}

5️⃣ Logout

Method: POST
Endpoint:

/user/logout


Headers

Authorization: Bearer <ACCESS_TOKEN>


Response

{
  "message": "logged_out"
}

6️⃣ Change Password

Method: POST
Endpoint:

/user/change-password


Headers

Authorization: Bearer <ACCESS_TOKEN>


Request Body

{
  "current_password": "OldPass@123",
  "new_password": "NewStrongPass@456"
}


Response

{
  "message": "password_changed"
}

👤 User Profile Service (user-profile)
7️⃣ Get Logged-in User Profile

Method: GET
Endpoint:

/user/me


Headers

Authorization: Bearer <ACCESS_TOKEN>


Response

{
  "id": "uuid",
  "first_name": "Tony",
  "last_name": "Stark",
  "gender": "MALE",
  "dob": "1999-08-12",
  "profile_completed": false
}

🩺 Health Survey Service
8️⃣ Save / Update Health Survey

Method: POST
Endpoint:

/user/health


Headers

Authorization: Bearer <ACCESS_TOKEN>


Request Body

{
  "food_allergies": ["peanuts"],
  "medicine_allergies": ["penicillin"],
  "pollen": ["grass"],
  "others": [],
  "height_cm": 175,
  "weight_kg": 70,
  "smoking_status": "never",
  "alcohol_consumption": "occasionally",
  "sleep_hours_per_night": 8,
  "physical_activity_level": "moderate",
  "stress_level": "low",
  "consent_medical": true,
  "consent_data_processing": true,
  "consent_privacy_policy": true,
  "consent_notifications": false
}

9️⃣ Get Health Survey Details

Method: GET
Endpoint:

/details


⚠️ NOTE:
This is mounted under the health resource, so the actual path is:

/user/health/details


Headers

Authorization: Bearer <ACCESS_TOKEN>


Response

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

🔟 Health Survey Status

Method: GET
Endpoint:

/status


Actual path

/user/health/status


Headers

Authorization: Bearer <ACCESS_TOKEN>


Response

{
  "completed": true
}

