
# 17. Tracing

Tracing คือ Service กลุ่มที่ช่วย track request ตลอดเส้นทางที่ผ่าน microservice หลาย ๆ ตัว ทำให้เห็นว่า bottleneck อยู่ที่ service ใด และ latency เกิดขึ้นที่ไหน

---

## Distributed Tracing

### คืออะไร

Distributed Tracing คือบริการที่ track request journey ข้าม service หลายตัวโดยแสดงผลเป็น trace ที่ประกอบด้วย span แต่ละ span แสดง latency ของ operation หนึ่ง ๆ เช่น database query, API call, หรือ service call

### ใช้งานแบบไหน

Instrument application ด้วย tracing SDK (OpenTelemetry) แล้วส่ง trace ไปยัง tracing backend จากนั้น query trace เพื่อดู waterfall view ของ request และระบุ bottleneck

### เหมาะกับงานแบบไหน

เหมาะกับ microservice architecture ที่มีหลาย service request ข้ามหลาย service ทำให้ยาก debug ด้วย log อย่างเดียว

### ไม่เหมาะกับงานแบบไหน

อาจมี overhead เกินจำเป็นสำหรับ monolithic application ที่ง่ายมาก

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                             |
| -------------- | ---------------------------------------- |
| AWS            | AWS X-Ray                                |
| GCP            | Cloud Trace                              |
| Azure          | Azure Monitor (Application Insights)     |
| Huawei Cloud   | Application Performance Management (APM) |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                     | ตัวอย่าง                                    |
| -------------------- | -------------------------------------------- | ----------------------------------------- |
| Trace                | การ track request หนึ่งตลอดเส้นทาง              | request ID: abc-123 จาก API จนถึง database |
| Span                 | unit ย่อยของ trace แสดง latency ของ operation | span: DynamoDB Query ใช้เวลา 12ms          |
| Trace ID             | ID ที่ผูก span ทุกตัวใน trace เดียวกัน              | propagate ผ่าน HTTP header `X-Trace-Id`    |
| Sampling Rate        | สัดส่วนของ request ที่ trace                     | 1% (high traffic), 100% (debug)           |
| Instrumentation      | การ inject tracing code เข้า application      | OpenTelemetry SDK, AWS X-Ray SDK          |
| Service Map          | diagram แสดง dependency ระหว่าง service       | auto-generated จาก trace data             |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม trace/span ที่ record และ retrieve

|                          | AWS X-Ray         | GCP Cloud Trace  | Azure Application Insights | Huawei APM                |
| ------------------------ | ----------------- | ---------------- | -------------------------- | ------------------------- |
| Trace Recording          | $5.00/1M traces   | $0.20/1M spans   | $2.30/GB ingested          | ~$2.00/month (basic plan) |
| Trace Retrieval/Scan     | $0.50/1M traces   | ฟรี               | รวมใน ingestion            | รวมใน plan                |
| Free Tier                | 100K traces/month | 2.5M spans/month | 5 GB/month                 | —                         |
| ตัวอย่าง (1M traces/month) | ~$5.50            | ~$0.20           | ~$2.30/GB                  | ~$2.00+                   |

### ตัวอย่างการใช้งานใน Project

API Gateway → Lambda → DynamoDB trace แสดงว่า latency 450ms มาจาก DynamoDB query 400ms ทำให้รู้ว่าต้องเพิ่ม index

### Best Practice

- ใช้ OpenTelemetry เป็น standard เพื่อ vendor-neutral instrumentation
- propagate trace context ทุก service call ผ่าน HTTP header
- ตั้ง sampling rate ให้เหมาะสมกับ traffic volume
- ผูก trace ID กับ log และ metric เพื่อ full observability

### Common Mistakes

- instrument เฉพาะ service บางตัว ทำให้ trace ขาดกลางทาง
- sampling rate สูงเกินใน high-traffic production ทำให้ค่าใช้จ่ายสูง

---
