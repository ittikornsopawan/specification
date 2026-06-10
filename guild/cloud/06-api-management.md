
# 6. API Management

API Management คือ Service กลุ่มที่ช่วย manage, secure, monitor และ publish API ให้กับ consumer ภายในและภายนอกองค์กร ช่วยลด boilerplate ที่ต้องเขียนซ้ำในทุก service เช่น authentication, rate limiting และ logging

---

## API Gateway

### คืออะไร

API Gateway คือ service ที่ทำหน้าที่เป็น single entry point สำหรับ API ทั้งหมด รับ request จาก client แล้ว route ไปยัง backend service ที่เหมาะสม พร้อมจัดการ authentication, rate limiting, caching, request/response transformation และ logging

### ใช้งานแบบไหน

ใช้เป็น front door ของ microservice architecture กำหนด API route แต่ละ endpoint ว่า forward ไปยัง backend service ใด ตั้ง authentication policy, rate limit และ quota ต่อ API key หรือ consumer

### เหมาะกับงานแบบไหน

เหมาะกับ microservice architecture, public API ที่ต้องการ security, mobile backend, หรือ system ที่ต้องการ centralize API policy

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ internal service-to-service communication ที่ไม่ต้องการ overhead ของ API Gateway อาจใช้ service mesh แทน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                 |
| -------------- | -------------------------------------------- |
| AWS            | Amazon API Gateway (REST / HTTP / WebSocket) |
| GCP            | Apigee, Cloud Endpoints                      |
| Azure          | Azure API Management                         |
| Huawei Cloud   | API Gateway (APIG)                           |

### Spec / Configuration

| Spec / Configuration              | ความหมาย                           | ตัวอย่าง                          |
| --------------------------------- | ---------------------------------- | ------------------------------- |
| API Type                          | ประเภทของ API                      | REST, HTTP, WebSocket, GraphQL  |
| Stage / Environment               | environment สำหรับ deploy API        | dev, staging, prod              |
| Authorization / Authentication    | วิธี auth ที่ใช้                        | API Key, JWT, OAuth 2.0, IAM    |
| Rate Limiting / Throttling        | จำกัดจำนวน request ต่อหน่วยเวลา         | 1000 req/min ต่อ API key         |
| Usage Plan / Quota                | กำหนด limit การใช้งานต่อ consumer     | 10,000 req/day                  |
| Request / Response Transformation | แปลง request หรือ response          | เพิ่ม header, แปลง JSON format    |
| Caching                           | cache response เพื่อลด backend load  | TTL 60 seconds                  |
| CORS Policy                       | กำหนด Cross-Origin Resource Sharing | อนุญาต `https://app.example.com` |
| Integration Type                  | วิธีเชื่อมต่อกับ backend                 | Lambda, HTTP endpoint, VPC Link |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม request count + optional cache fee

|                   | AWS API GW (REST)        | GCP API Gateway | Azure API Mgmt (Consumption) | Huawei APIG  |
| ----------------- | ------------------------ | --------------- | ---------------------------- | ------------ |
| REST/HTTP Request | $3.50/1M                 | $3.50/1M        | $3.50/1M                     | ~$3.00/1M    |
| WebSocket msg     | $1.00/1M msgs            | —               | included                     | ~$1.00/1M    |
| Cache (0.5 GB)    | $0.020/hour              | N/A             | included                     | ~$0.015/hour |
| Free Tier         | 1M req/month (12 months) | 2M calls/month  | 1M calls/month               | —            |

> GCP Apigee X สำหรับ enterprise API management มีค่าใช้จ่ายตามแพลน ($600–$6,000+/month) แตกต่างจาก API Gateway ทั่วไปมาก

### ตัวอย่างการใช้งานใน Project

```
Mobile App → API Gateway → JWT Validation → Rate Limiter
    ├── GET /users → Lambda: user-service
    ├── POST /orders → HTTP: order-service (ECS)
    └── WebSocket /chat → Lambda: chat-handler
```

### Best Practice

- ใช้ JWT หรือ OAuth 2.0 แทน API Key สำหรับ user authentication
- ตั้ง rate limit ทุก endpoint เพื่อป้องกัน abuse
- เปิด access log และ execution log เพื่อ troubleshoot
- ใช้ stage variables แยก configuration ระหว่าง environment

### Common Mistakes

- ไม่ตั้ง rate limit ทำให้ backend ถูก flood จาก API call
- expose internal error message ออกสู่ client โดยไม่ mask
- ไม่ได้ version API ทำให้ยาก maintain เมื่อ breaking change

---
