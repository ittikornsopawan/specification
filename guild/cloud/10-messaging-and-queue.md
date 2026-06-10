# 10. Messaging & Queue

Messaging และ Queue คือ Service กลุ่มที่ช่วยให้ component ต่าง ๆ ของระบบสื่อสารกันแบบ asynchronous ลดการ coupling ระหว่าง service และช่วยรองรับ load ที่ spike กะทันหัน

---

## Message Queue

### คืออะไร

Message Queue คือบริการ queue ที่เก็บ message ไว้รอให้ consumer มาดึงไป (pull model) ใช้สำหรับ decouple producer และ consumer ให้ทำงานในอัตราที่ต่างกันได้ รองรับ retry และ dead-letter queue

### ใช้งานแบบไหน

Producer ส่ง message เข้า queue, Consumer ดึง message ออกมา process ทีละรายการ ใช้กับ background job, task offloading, หรือ pipeline ที่ต้องการ guaranteed delivery

### เหมาะกับงานแบบไหน

เหมาะกับ background job processing, email/notification sending, order processing, file conversion, หรืองานที่ต้องการ retry mechanism

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ use case ที่ต้องการ publish message ให้หลาย consumer รับพร้อมกัน (fan-out) ควรใช้ Pub/Sub แทน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                   |
| -------------- | ---------------------------------------------- |
| AWS            | Amazon SQS                                     |
| GCP            | Cloud Tasks                                    |
| Azure          | Azure Queue Storage, Azure Service Bus         |
| Huawei Cloud   | Distributed Message Service (DMS) for RocketMQ |

### Spec / Configuration

| Spec / Configuration      | ความหมาย                                               | ตัวอย่าง                                                 |
| ------------------------- | ------------------------------------------------------ | ------------------------------------------------------ |
| Queue Type                | ประเภทของ queue                                        | Standard (at-least-once), FIFO (exactly-once, ordered) |
| Visibility Timeout        | ช่วงเวลาที่ message ถูก "lock" ระหว่าง consumer กำลัง process | 30 seconds                                             |
| Message Retention Period  | ระยะเวลาที่เก็บ message ไว้ใน queue                        | 4 วัน (default), สูงสุด 14 วัน                             |
| Max Message Size          | ขนาดสูงสุดของ message                                    | 256 KB (SQS), 1 MB (Service Bus)                       |
| Dead-Letter Queue (DLQ)   | queue สำหรับเก็บ message ที่ process ไม่สำเร็จหลาย retry       | แยก queue สำหรับ alert ทีม                                |
| Receive Message Wait Time | การทำ long polling เพื่อลด empty receive                  | 20 seconds                                             |
| Max Receive Count         | จำนวน retry ก่อนส่งไป DLQ                                 | 3-5 ครั้ง                                                |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม message/request count

|           | AWS SQS Standard  | AWS SQS FIFO      | GCP Cloud Tasks | Azure Service Bus Standard | Huawei DMS RocketMQ   |
| --------- | ----------------- | ----------------- | --------------- | -------------------------- | --------------------- |
| ราคาหลัก   | $0.40/1M requests | $0.50/1M requests | $0.40/1M ops    | $10/month + $0.013/1M      | ~$0.006/hour/instance |
| Free Tier | 1M/month          | —                 | 1M/month        | 10M/month                  | —                     |

> SQS request = 1 API call (Send, Receive, Delete); ข้อความขนาดใหญ่กว่า 64 KB นับเป็นหลาย request

### ตัวอย่างการใช้งานใน Project

```
API Server → SQS Queue → Lambda Worker → send email via SES
                       ↓ (fail 3x)
                       Dead-Letter Queue → Alert ทีม
```

### Best Practice

- ตั้ง Dead-Letter Queue ทุก queue ใน production
- ออกแบบ consumer ให้ idempotent รับ message เดิมซ้ำแล้วได้ผลเหมือนกัน
- ใช้ FIFO Queue เมื่อต้องการ ordering และ exactly-once processing
- monitor queue depth และ DLQ message count

### Common Mistakes

- ไม่ได้ตั้ง DLQ ทำให้ message หายเมื่อ process ล้มเหลว
- Visibility Timeout สั้นกว่าเวลา process จริง ทำให้ message ถูก process ซ้ำ
- ไม่ได้ทำ consumer ให้ idempotent

---

## Pub/Sub (Publish/Subscribe)

### คืออะไร

Pub/Sub คือ messaging pattern ที่ publisher ส่ง message ไปยัง topic และ subscriber หลายคนรับ message จาก topic นั้นพร้อมกัน (fan-out) ใช้สำหรับ event notification, broadcasting และ event-driven architecture

### ใช้งานแบบไหน

Publisher ส่ง event ไปยัง topic เมื่อเกิด event บางอย่าง เช่น user สร้าง order, subscriber หลายตัว เช่น notification service, inventory service, analytics service รับ event เดียวกันและ process แบบ parallel และอิสระต่อกัน

### เหมาะกับงานแบบไหน

เหมาะกับ event broadcasting, microservice ที่ต้องการ decoupling, notification system, หรือ event-driven architecture ที่มีหลาย consumer

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ message ordering เข้มงวด หรือ consumer เดียวที่ต้องการ guaranteed processing

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                 |
| -------------- | -------------------------------------------- |
| AWS            | Amazon SNS, Amazon EventBridge               |
| GCP            | Cloud Pub/Sub                                |
| Azure          | Azure Service Bus (Topics), Azure Event Grid |
| Huawei Cloud   | Distributed Message Service (DMS) for Kafka  |

### Spec / Configuration

| Spec / Configuration    | ความหมาย                                | ตัวอย่าง                               |
| ----------------------- | --------------------------------------- | ------------------------------------ |
| Topic                   | channel ที่ publisher ส่ง message เข้า      | `user-events`, `order-created`       |
| Subscription            | การ subscribe ของ consumer ไปยัง topic   | push subscription, pull subscription |
| Message Retention       | ระยะเวลาเก็บ message ใน topic            | 7 วัน (GCP Pub/Sub)                   |
| Acknowledgment Deadline | เวลาที่ subscriber ต้อง ack ก่อน re-deliver | 600 seconds                          |
| Filter                  | กรอง message ตาม attribute              | รับเฉพาะ event ที่ `type=ORDER_CREATED` |
| Dead-Letter Topic       | topic สำหรับ message ที่ deliver ไม่สำเร็จ     | เพื่อ monitor และ debug                |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม message/event count หรือ data volume

|                 | AWS SNS                | AWS EventBridge | GCP Cloud Pub/Sub  | Azure Event Grid    | Huawei DMS |
| --------------- | ---------------------- | --------------- | ------------------ | ------------------- | ---------- |
| ราคาหลัก         | $0.50/1M notifications | $1.00/1M events | $0.04/GB processed | $0.60/1M operations | ~$0.020/GB |
| HTTP/S delivery | $0.60/1M deliveries    | —               | รวมใน data GB      | —                   | —          |
| Free Tier       | 1M/month               | —               | 10 GB/month        | 100K ops/month      | —          |

### ตัวอย่างการใช้งานใน Project

```
Order Service → SNS Topic: order-created
    ├── SQS: notification-queue → send push notification
    ├── SQS: inventory-queue → update stock
    └── SQS: analytics-queue → update dashboard
```

### Best Practice

- ออกแบบ event schema ให้ชัดเจน มี versioning
- ใช้ filter subscription เพื่อให้ consumer รับเฉพาะ event ที่ต้องการ
- ตั้ง Dead-Letter Topic เพื่อจับ message ที่ deliver ไม่สำเร็จ

### Common Mistakes

- ส่ง message ขนาดใหญ่ใน Pub/Sub แทนที่จะส่งแค่ reference (S3 key)
- ไม่ได้ออกแบบ consumer ให้ idempotent
