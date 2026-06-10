# 4. Serverless

Serverless คือรูปแบบการ run code โดยไม่ต้องจัดการ server เลย ผู้ใช้เขียนแค่ function หรือ logic แล้ว Cloud จะ provision, scale และ manage infrastructure ให้ทั้งหมด จ่ายเฉพาะเวลาที่ code ทำงานจริง

---

## Function as a Service (FaaS)

### คืออะไร

FaaS (Function as a Service) คือบริการ run code เป็น function ขนาดเล็ก ถูก trigger โดย event ต่าง ๆ เช่น HTTP request, message queue, file upload หรือ scheduled timer โดยไม่ต้องจัดการ server หรือ OS เลย

### ใช้งานแบบไหน

เขียน function สำหรับ handle event เฉพาะเจาะจง เช่น process image เมื่อมี upload ขึ้น S3, handle webhook, หรือ run scheduled job ทุกคืน

### เหมาะกับงานแบบไหน

เหมาะกับ event-driven workload, scheduled task, webhook handler, data transformation pipeline, หรืองานที่มี traffic ไม่สม่ำเสมอและต้องการ cost efficiency สูง

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ execution นาน (มักมี timeout 15-60 นาที), workload ที่ต้องการ persistent connection หรือ in-memory state ระหว่าง invocation

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                         |
| -------------- | ------------------------------------ |
| AWS            | AWS Lambda                           |
| GCP            | Cloud Functions, Cloud Run Functions |
| Azure          | Azure Functions                      |
| Huawei Cloud   | FunctionGraph                        |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                 | ตัวอย่าง                                        |
| -------------------- | ---------------------------------------- | --------------------------------------------- |
| Runtime              | ภาษาและ version ที่ function ใช้            | Node.js 20, Python 3.12, Go 1.21              |
| Memory               | ขนาด memory ที่จัดสรร (ส่งผลต่อ CPU ด้วย)      | 128 MB, 512 MB, 1024 MB                       |
| Timeout              | เวลา execution สูงสุด                      | 30 seconds, 15 minutes                        |
| Concurrency          | จำนวน function instance ที่ run พร้อมกันได้    | Reserved Concurrency, Provisioned Concurrency |
| Trigger Type         | สิ่งที่ trigger function ให้ทำงาน              | HTTP, S3 Event, SQS, EventBridge, Schedule    |
| Environment Variable | ค่า configuration ที่ inject เข้า function   | `DB_HOST`, `API_KEY`                          |
| Layer / Dependency   | code หรือ library ที่ share ระหว่าง function | Lambda Layers                                 |
| VPC Integration      | การให้ function เข้าถึง VPC resource ได้     | เพื่อเชื่อมต่อ database ใน Private Subnet          |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม invocation และ compute time (GB-second)

|                               | AWS Lambda           | GCP Cloud Functions (2nd gen) | Azure Functions (Consumption) | Huawei FunctionGraph |
| ----------------------------- | -------------------- | ----------------------------- | ----------------------------- | -------------------- |
| Invocations                   | $0.20/1M             | $0.40/1M                      | $0.20/1M                      | $0.20/1M             |
| Compute (GB-sec)              | $0.0000166667        | $0.0000100                    | $0.000016                     | $0.00001667          |
| Free Tier/month               | 1M req + 400K GB-sec | 2M req + 400K GB-sec          | 1M req + 400K GB-sec          | 1M req + 400K GB-sec |
| ตัวอย่าง 1M req × 512MB × 200ms | ~$0.21               | ~$0.41                        | ~$0.21                        | ~$0.21               |

> หลัง free tier, FaaS ถูกที่สุดสำหรับ workload < 10M invocation/เดือน เมื่อเทียบกับการ run container 24/7

### ตัวอย่างการใช้งานใน Project

เมื่อผู้ใช้ upload รูปภาพขึ้น S3 ระบบ trigger Lambda function เพื่อ resize, compress และสร้าง thumbnail แล้วบันทึกกลับ S3 โดยที่ไม่ต้องมี server รอรับ event ตลอดเวลา

### Best Practice

- ออกแบบ function ให้ idempotent รับ event เดียวกันหลายครั้งแล้วได้ผลลัพธ์เหมือนกัน
- เก็บ secret ใน Secrets Manager ไม่ใช่ environment variable
- ใช้ Provisioned Concurrency สำหรับ function ที่ latency สำคัญ เพื่อหลีกเลี่ยง cold start
- แยก function ให้เล็กและทำสิ่งเดียว (single responsibility)

### Common Mistakes

- เขียน function ที่ทำงานนาน แล้ว hit timeout
- ไม่ได้จัดการ error และ retry ทำให้ event ถูก process ซ้ำหรือหาย
- เปิด VPC integration โดยไม่จำเป็น ทำให้ cold start นานขึ้น
