# 21. CDN & Edge

CDN (Content Delivery Network) คือ Service กลุ่มที่กระจาย content ไปยัง edge server ทั่วโลก ทำให้ user ได้รับ content จาก server ที่ใกล้ที่สุด ลด latency และลด load บน origin server

---

## Content Delivery Network (CDN)

### คืออะไร

CDN คือเครือข่ายของ edge server ที่กระจายอยู่ทั่วโลก cache content เช่น static file, image, video, API response เพื่อให้ user ได้รับ response จาก edge ที่ใกล้ที่สุดแทนการดึงจาก origin ทุกครั้ง

### ใช้งานแบบไหน

ชี้ DNS ไปยัง CDN endpoint แล้วกำหนด origin ว่าเป็น S3, ALB หรือ web server กำหนด cache behavior ว่า path ไหน cache นานแค่ไหน

### เหมาะกับงานแบบไหน

เหมาะกับ static website, media streaming, large file download, API ที่มี response cacheable, หรือ global audience ที่ต้องการ low latency

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ API response ที่ personalized ต่อ user แต่ละคนและ cache ไม่ได้ หรือ real-time data ที่ต้องการ fresh ทุกครั้ง

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                   |
| -------------- | ------------------------------ |
| AWS            | Amazon CloudFront              |
| GCP            | Cloud CDN                      |
| Azure          | Azure Front Door, Azure CDN    |
| Huawei Cloud   | Content Delivery Network (CDN) |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                                   |
| -------------------- | ----------------------------------------- | ---------------------------------------- |
| Origin               | server ต้นทางที่ CDN ดึง content มา           | S3 bucket, ALB, custom domain            |
| Cache Behavior       | กำหนด cache policy ต่อ path pattern         | `/static/*` cache 1 ปี, `/api/*` ไม่ cache |
| TTL (Time to Live)   | ระยะเวลา cache อยู่ที่ edge                   | 86400 seconds (1 วัน)                     |
| Cache Invalidation   | การลบ cache ก่อน TTL หมด                   | invalidate `/index.html` หลัง deploy      |
| HTTPS / SSL          | การ force HTTPS                           | redirect HTTP → HTTPS                    |
| Custom Domain        | domain name ที่ใช้กับ CDN                     | `cdn.example.com`                        |
| Geo Restriction      | block หรือ allow access จาก country ที่กำหนด  | เฉพาะ APAC countries                     |
| Origin Shield        | cache layer กลางเพื่อลด request ไปยัง origin | ลด origin load                           |
| WAF Integration      | เชื่อมต่อ WAF บน CDN layer                   | ป้องกันก่อนถึง origin                        |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB egress + request count

|                         | AWS CloudFront              | GCP Cloud CDN | Azure Front Door Standard | Huawei CDN  |
| ----------------------- | --------------------------- | ------------- | ------------------------- | ----------- |
| Egress (0–10 TB/month)  | $0.0085/GB                  | $0.0080/GB    | $0.0075/GB                | $0.0070/GB  |
| Egress (10–50 TB/month) | $0.0080/GB                  | $0.0060/GB    | $0.0060/GB                | $0.0060/GB  |
| HTTP/S Request          | $0.0100/10K                 | $0.0075/10K   | $0.0090/10K               | $0.0080/10K |
| Cache Invalidation      | $0.005/path (after 1K free) | ฟรี            | ฟรี                        | ฟรี          |
| Origin Shield           | $0.0075/10K requests        | —             | รวมใน tier                | —           |

> CDN egress ($0.0085/GB) ถูกกว่า serve จาก origin S3/EC2 โดยตรง ($0.09/GB) ~10 เท่า — ใช้ CDN นำหน้า origin เสมอสำหรับ static/public content

### ตัวอย่างการใช้งานใน Project

```
User (Bangkok) → CloudFront Edge (Singapore)
    → /static/* : cache 1 ปี จาก S3 bucket
    → /api/* : forward ไป ALB ใน ap-southeast-1
    → /* : serve index.html จาก S3 (SPA)
```

### Best Practice

- ใช้ Cache-Control header ที่ถูกต้องใน origin response
- ตั้งชื่อ file ด้วย content hash เพื่อ long cache สำหรับ versioned asset
- ใช้ CDN เป็น WAF integration point
- invalidate cache เฉพาะ path ที่จำเป็นไม่ invalidate ทั้งหมด (แพงและช้า)

### Common Mistakes

- ไม่ได้ set Cache-Control header ทำให้ CDN cache ตาม default ที่ไม่เหมาะสม
- invalidate `/*` ทุก deploy ทำให้ cache miss ทั้งหมด
- เปิด caching สำหรับ endpoint ที่มี user-specific response
