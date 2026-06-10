# 5. Load Balancing

Load Balancing คือ Service ที่กระจาย network traffic ไปยัง backend server หลายตัว เพื่อให้ระบบรองรับ load ได้มากขึ้น ไม่มี single point of failure และ scale ได้อย่างมีประสิทธิภาพ

---

## Application Load Balancer (ALB)

### คืออะไร

Application Load Balancer (ALB) คือ Load Balancer ที่ทำงานในระดับ Layer 7 (HTTP/HTTPS) สามารถ route traffic ตาม URL path, hostname, HTTP header หรือ query string ได้ รองรับ WebSocket, HTTP/2 และ SSL termination

### ใช้งานแบบไหน

ใช้เป็น entry point ของ web application โดย terminate SSL/TLS แล้ว forward request ไปยัง backend service ตาม routing rule เช่น `/api/*` ไปหา API service, `/static/*` ไปหา CDN origin

### เหมาะกับงานแบบไหน

เหมาะกับ web application, REST API, microservice ที่ต้องการ path-based routing, หรือ blue/green deployment

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ non-HTTP protocol เช่น TCP/UDP game server หรือ custom binary protocol ควรใช้ Network Load Balancer แทน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                 |
| -------------- | -------------------------------------------- |
| AWS            | Application Load Balancer (ALB)              |
| GCP            | Cloud Load Balancing (HTTP(S) Load Balancer) |
| Azure          | Azure Application Gateway                    |
| Huawei Cloud   | Elastic Load Balance (ELB) - Dedicated       |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                              |
| -------------------- | ----------------------------------------- | ----------------------------------- |
| Listener             | port และ protocol ที่ ALB รับ traffic        | HTTPS:443                           |
| SSL/TLS Certificate  | certificate สำหรับ terminate HTTPS          | ACM Certificate, Let's Encrypt      |
| Target Group         | กลุ่ม backend ที่รับ traffic                   | EC2 instances, IP addresses, Lambda |
| Health Check         | วิธีตรวจสอบ backend ที่ยังทำงานปกติ              | GET /health → 200 OK                |
| Routing Rule         | เงื่อนไขการ route request                   | path `/api/*` → target-group-api    |
| Sticky Session       | ส่ง request จาก client เดิมไปยัง backend เดิม | Duration-based cookie               |
| WAF Integration      | เชื่อมต่อ WAF เพื่อกรอง malicious request      | AWS WAF, Azure WAF                  |
| Access Log           | บันทึก request ทุกรายการ                     | บันทึกไปยัง S3                         |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม hour + usage unit (LCU / CU / GB)

|                          | AWS ALB         | GCP HTTPS LB | Azure App Gateway WAF v2 | Huawei ELB Dedicated |
| ------------------------ | --------------- | ------------ | ------------------------ | -------------------- |
| Gateway/hour             | $0.008          | $0.008       | $0.246                   | ~$0.007              |
| Usage unit               | $0.008/LCU-hour | $0.006/GB    | $0.008/CU-hour           | ~$0.003/GB           |
| Free Tier                | —               | —            | —                        | —                    |
| ตัวอย่าง (ปานกลาง, 1 เดือน) | ~$20–40         | ~$20–40      | ~$180–250                | ~$15–25              |

> Azure App Gateway WAF v2 มีค่า gateway สูงกว่า AWS/GCP ~30× เหมาะกับ enterprise ที่ต้องการ WAF รวมอยู่ด้วย

### ตัวอย่างการใช้งานใน Project

```
Internet → Route 53 → ALB (HTTPS:443)
    ├── Rule: /api/* → Target Group: api-servers (ECS Fargate)
    ├── Rule: /admin/* → Target Group: admin-servers (EC2)
    └── Rule: /* → Target Group: frontend-servers (EC2)
```

### Best Practice

- ใช้ ALB เป็น SSL termination point เสมอ ไม่ส่ง HTTPS ผ่านต่อถึง backend
- ตั้ง Health Check ที่เหมาะสม ไม่ใช้ `/` แต่ใช้ endpoint เฉพาะเช่น `/health`
- เปิด Access Log เพื่อ troubleshoot และ audit
- ตั้ง Idle Timeout ให้เหมาะสมกับ application

### Common Mistakes

- ไม่ได้ตั้ง Health Check ทำให้ traffic ถูกส่งไปยัง backend ที่ dead
- ใช้ self-signed certificate ใน production
- ลืม redirect HTTP → HTTPS

---

## Network Load Balancer (NLB)

### คืออะไร

Network Load Balancer (NLB) คือ Load Balancer ที่ทำงานในระดับ Layer 4 (TCP/UDP) มี latency ต่ำมาก สามารถรับ traffic ได้หลายล้าน request ต่อวินาที เหมาะกับ protocol ที่ไม่ใช่ HTTP

### ใช้งานแบบไหน

ใช้สำหรับ distribute TCP/UDP traffic ไปยัง backend โดยไม่ inspect HTTP content ใช้กับ game server, IoT protocol, database proxy หรือ application ที่ต้องการ static IP

### เหมาะกับงานแบบไหน

เหมาะกับ TCP/UDP workload, game server, real-time communication, application ที่ต้องการ static IP สำหรับ whitelist, หรือ throughput สูงมาก

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ HTTP routing ที่ต้องการ path-based rule เพราะ NLB ไม่ inspect HTTP header

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                 |
| -------------- | -------------------------------------------- |
| AWS            | Network Load Balancer (NLB)                  |
| GCP            | Cloud Load Balancing (TCP/UDP Load Balancer) |
| Azure          | Azure Load Balancer (Standard)               |
| Huawei Cloud   | Elastic Load Balance (ELB) - Network         |

### Spec / Configuration

| Spec / Configuration      | ความหมาย                         | ตัวอย่าง                                  |
| ------------------------- | -------------------------------- | --------------------------------------- |
| Protocol                  | โปรโตคอลที่ Listener รับ            | TCP, UDP, TLS                           |
| Static IP / Elastic IP    | IP address คงที่สำหรับ whitelist     | สำคัญสำหรับ partner integration             |
| Target Type               | ประเภทของ backend                | Instance, IP, Application Load Balancer |
| Cross-Zone Load Balancing | กระจาย traffic ข้าม AZ            | เปิดเพื่อ even distribution                |
| Health Check              | ตรวจสอบ backend ด้วย TCP หรือ HTTP | TCP connect check                       |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม hour + NLCU/GB

|                          | AWS NLB          | GCP Network LB | Azure Standard Load Balancer | Huawei ELB Network |
| ------------------------ | ---------------- | -------------- | ---------------------------- | ------------------ |
| Gateway/hour             | $0.008           | $0.008         | $0.025                       | ~$0.006            |
| Usage unit               | $0.006/NLCU-hour | $0.006/GB      | $0.005/GB processed          | ~$0.003/GB         |
| ตัวอย่าง (ปานกลาง, 1 เดือน) | ~$20–30          | ~$15–25        | ~$18–25                      | ~$10–15            |

### ตัวอย่างการใช้งานใน Project

Game server ที่ใช้ UDP protocol ใช้ NLB เพื่อ distribute player connection ไปยัง game server instance หลายตัว โดยที่ static IP ช่วยให้ player ไม่ต้อง update IP ใน client

### Best Practice

- ใช้ NLB เมื่อต้องการ static IP หรือ protocol ที่ไม่ใช่ HTTP
- เปิด Cross-Zone Load Balancing เพื่อกระจาย traffic อย่างสม่ำเสมอ

### Common Mistakes

- ใช้ NLB กับ HTTP workload ทั้งที่ ALB เหมาะกว่า เพราะ ALB มี feature มากกว่า
