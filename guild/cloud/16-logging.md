# 16. Logging

Logging คือ Service กลุ่มที่รวบรวม เก็บ และ query log จาก application และ infrastructure เพื่อ troubleshoot ปัญหา audit การเข้าถึง และ detect security incident

---

## Centralized Log Management

### คืออะไร

Centralized Log Management คือบริการที่รวบรวม log จาก source หลาย ๆ ตัวเช่น application server, Load Balancer, database, Cloud service เข้ามาเก็บไว้ที่เดียว พร้อมความสามารถ search, filter และ analyze log

### ใช้งานแบบไหน

ติดตั้ง log agent บน server หรือ configure service ให้ส่ง log ไปยัง centralized log service จากนั้น query ด้วย query language เช่น CloudWatch Logs Insights หรือ Kibana

### เหมาะกับงานแบบไหน

เหมาะกับทุก production system ที่มีมากกว่า 1 service เพราะ distributed log ทำให้ troubleshoot ยากมาก

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับการเก็บ log ขนาดเล็กมากใน managed service ที่แพงเกินจำเป็น

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                       |
| -------------- | ---------------------------------- |
| AWS            | Amazon CloudWatch Logs             |
| GCP            | Cloud Logging                      |
| Azure          | Azure Monitor Logs (Log Analytics) |
| Huawei Cloud   | Log Tank Service (LTS)             |

### Spec / Configuration

| Spec / Configuration       | ความหมาย                                     | ตัวอย่าง                                              |
| -------------------------- | -------------------------------------------- | --------------------------------------------------- |
| Log Group / Bucket         | กลุ่มของ log stream จาก source เดียวกัน          | `/aws/lambda/my-function`, `/app/api`               |
| Log Stream                 | ลำดับของ log event จาก instance/container เดียว | `i-0123456789/application.log`                      |
| Retention Policy           | ระยะเวลาเก็บ log ก่อนลบอัตโนมัติ                  | 30 วัน, 90 วัน, 1 ปี                                   |
| Log Filter / Metric Filter | แปลง log pattern ให้เป็น metric                | นับ error log สร้าง metric ErrorCount                 |
| Subscription Filter        | ส่ง log ไปยัง destination อื่น                   | ส่งไป Kinesis Data Firehose หรือ S3                   |
| Log Agent                  | agent ที่ collect log จาก server               | CloudWatch Agent, Fluentd, Fluent Bit               |
| Structured Logging         | log แบบ JSON format เพื่อ query ง่าย            | `{"level":"error","message":"...","traceId":"..."}` |
| Log Query Language         | ภาษา query สำหรับ analyze log                  | CloudWatch Logs Insights, KQL                       |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB ที่ ingest + store + query

|                   | AWS CloudWatch Logs | GCP Cloud Logging           | Azure Monitor Logs              | Huawei LTS      |
| ----------------- | ------------------- | --------------------------- | ------------------------------- | --------------- |
| Log Ingestion     | $0.50/GB            | $0.01/GB (after 50 GB free) | $2.30/GB                        | ~$0.04/GB       |
| Log Storage       | $0.03/GB-month      | $0.01/GB-month              | รวมใน ingestion (31 วัน default) | ~$0.02/GB-month |
| Log Query         | $0.005/GB scanned   | $0.01/GB scanned            | รวมใน workspace                 | ~$0.003/GB      |
| Default Retention | กำหนดเอง             | 30 วัน (_Default bucket)     | 31 วัน                           | 7 วัน            |

> AWS CloudWatch Logs ingestion ($0.50/GB) แพงที่สุด — ระบบที่มี log volume สูง ควร export ไป S3 ผ่าน Kinesis Firehose (S3 storage $0.023/GB-month) แทน

### ตัวอย่างการใช้งานใน Project

```
Application (JSON log) → Fluent Bit → CloudWatch Logs
CloudWatch Logs → Metric Filter → CloudWatch Alarm (error spike)
CloudWatch Logs → S3 (archive) → Athena (long-term analysis)
```

### Best Practice

- ใช้ structured logging (JSON) เสมอเพื่อ query ได้ง่าย
- กำหนด retention policy ทุก log group ไม่ปล่อยให้เก็บนิรันดร์
- รวม trace ID ทุก log เพื่อ correlate ระหว่าง service
- ส่ง log ไป long-term storage (S3) สำหรับ compliance และ analysis

### Common Mistakes

- log แบบ plain text ทำให้ query ยาก
- ไม่กำหนด retention ทำให้ค่าใช้จ่ายสะสม
- เก็บ sensitive data เช่น password, PII ใน log
