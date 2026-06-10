# 22. DNS

DNS (Domain Name System) คือ Service ที่แปลง domain name เป็น IP address ช่วยให้ user เข้าถึง application ด้วย domain ที่จำง่าย และช่วย manage traffic routing ระหว่าง region หรือ environment

---

## Managed DNS

### คืออะไร

Managed DNS คือบริการ DNS ที่ Cloud Provider จัดการ name server ให้มีความพร้อมใช้งานสูง รองรับ routing policy ต่าง ๆ เช่น latency-based, geolocation, failover และ weighted routing

### ใช้งานแบบไหน

สร้าง hosted zone สำหรับ domain แล้วสร้าง DNS record เพื่อชี้ domain หรือ subdomain ไปยัง IP address, Load Balancer หรือ CloudFront distribution

### เหมาะกับงานแบบไหน

เหมาะกับทุก project ที่ต้องการ domain name และ routing ขั้นสูง

### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ managed DNS สำหรับ production

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name              |
| -------------- | ------------------------- |
| AWS            | Amazon Route 53           |
| GCP            | Cloud DNS                 |
| Azure          | Azure DNS                 |
| Huawei Cloud   | Domain Name Service (DNS) |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                                           |
| -------------------- | ----------------------------------------- | ------------------------------------------------ |
| Record Type          | ประเภทของ DNS record                      | A, AAAA, CNAME, MX, TXT, NS, Alias               |
| TTL                  | เวลาที่ resolver cache record ก่อน query ใหม่ | 300 seconds                                      |
| Routing Policy       | วิธีการตัดสินใจว่าจะ return IP ใด              | Simple, Weighted, Latency, Geolocation, Failover |
| Health Check         | ตรวจสอบ endpoint ก่อน DNS ส่ง traffic ให้    | HTTP health check ทุก 30 วินาที                     |
| Hosted Zone          | container ของ DNS record ทั้งหมดของ domain  | `example.com`                                    |
| Alias Record         | record ที่ชี้ไปยัง AWS resource โดยตรง         | Alias ไป ALB หรือ CloudFront                      |
| Private Hosted Zone  | DNS ที่ใช้งานได้เฉพาะใน VPC                   | internal service discovery                       |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม hosted zone/month และ query count

|                           | AWS Route 53      | GCP Cloud DNS    | Azure DNS        | Huawei DNS        |
| ------------------------- | ----------------- | ---------------- | ---------------- | ----------------- |
| Hosted Zone               | $0.50/zone/month  | $0.20/zone/month | $0.90/zone/month | ~$0.60/zone/month |
| Standard Query            | $0.400/1M         | $0.400/1M        | $0.400/1M        | ~$0.300/1M        |
| Latency/Geo Routing Query | $0.700/1M         | $0.700/1M        | $0.700/1M        | —                 |
| Health Check              | $0.50/check/month | —                | —                | —                 |

### ตัวอย่างการใช้งานใน Project

```
example.com → Alias → CloudFront Distribution
api.example.com → Latency-based routing
    → ap-southeast-1 ALB (Thailand, Singapore users)
    → us-east-1 ALB (US users)
    Health Check → failover อัตโนมัติเมื่อ region มีปัญหา
```

### Best Practice

- ตั้ง Health Check กับ DNS record เพื่อ automatic failover
- ใช้ low TTL (60-300s) สำหรับ record ที่อาจเปลี่ยนบ่อย
- ใช้ Alias record แทน CNAME สำหรับ AWS resource (ไม่มีค่าใช้จ่ายและ query เร็วกว่า)
- ใช้ Private Hosted Zone สำหรับ internal service communication

### Common Mistakes

- TTL สูงเกินไปทำให้ DNS change ช้าเมื่อต้องการ failover
- ไม่มี Health Check ทำให้ DNS ยังชี้ไปยัง endpoint ที่ dead
