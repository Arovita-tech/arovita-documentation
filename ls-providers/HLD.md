                                   ┌──────────────────────┐
                                   │     EventBridge      │
                                   │  (Scheduled Cron)    │
                                   └──────────┬───────────┘
                                              │
                                              ▼
                                   ┌──────────────────────┐
                                   │    provider-sync     │
                                   │      Lambda          │
                                   └──────────┬───────────┘
                                              │
                                              ▼
                                        LabStack APIs
                                              │
                                              ▼
                                       ┌───────────────┐
                                       │   RDS (PG)    │
                                       │  Teleconsult  │
                                       └───────────────┘



Frontend App
      │
      ▼
┌──────────────────────┐
│      API Gateway     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   teleconsult-api    │
│       Lambda         │
└──────────┬───────────┘
           │
           ├──────────────▶ LabStack (Availability Proxy)
           │
           └──────────────▶ RDS (Appointments / Providers)



Payment Initiate
      │
      ▼
┌──────────────────────┐
│     payment-api      │
│       Lambda         │
└──────────┬───────────┘
           │
           ├──────────────▶ Razorpay API
           │
           ▼
   teleconsult-booking-queue
           │
           ▼
┌──────────────────────┐
│   async-processor    │
│       Lambda         │
└──────────┬───────────┘
           │
           ├──────────────▶ LabStack (bookAppointment)
           │
           └──────────────▶ RDS



LabStack Webhook
      │
      ▼
┌──────────────────────┐
│   vendor-webhook     │
│       Lambda         │
└──────────┬───────────┘
           │
           ▼
   labstack-webhook-queue
           │
           ▼
┌──────────────────────┐
│   async-processor    │
│       Lambda         │
└──────────┬───────────┘
           │
           └──────────────▶ RDS (Update status, documents, prescription)