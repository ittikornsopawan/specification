# 12. Security

Security คือ Service กลุ่มที่ช่วยปกป้องระบบจากการโจมตีและการเข้าถึงที่ไม่ได้รับอนุญาต ครอบคลุมตั้งแต่ firewall ระดับ application ไปจนถึงการตรวจจับ intrusion และ DDoS protection

---

## Web Application Firewall (WAF)

### คืออะไร

WAF (Web Application Firewall) คือ firewall ระดับ Layer 7 ที่ตรวจสอบ HTTP/HTTPS traffic เพื่อกรองการโจมตีประเภทต่าง ๆ เช่น SQL Injection, Cross-Site Scripting (XSS), Remote Code Execution และ OWASP Top 10

### ใช้งานแบบไหน

ติดตั้งหน้า Load Balancer หรือ API Gateway กำหนด rule set เพื่อ block หรือ monitor request ที่ pattern เข้าข่ายการโจมตี

### เหมาะกับงานแบบไหน

เหมาะกับทุก web application ที่ expose สู่ internet โดยเฉพาะ application ที่รับ user input หรืออยู่ภายใต้ compliance เช่น PCI DSS, HIPAA

### ไม่เหมาะกับงานแบบไหน

WAF ไม่ใช่ substitute สำหรับ secure coding practice ควรใช้ร่วมกับ application security ไม่ใช่แทนกัน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                         |
| -------------- | ------------------------------------ |
| AWS            | AWS WAF                              |
| GCP            | Cloud Armor                          |
| Azure          | Azure Web Application Firewall (WAF) |
| Huawei Cloud   | Web Application Firewall (WAF)       |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                                        |
| -------------------- | ----------------------------------------- | --------------------------------------------- |
| Rule Group           | ชุด rule สำเร็จรูปสำหรับการโจมตีประเภทต่าง ๆ      | AWS Managed Rules: CommonRuleSet, SQLiRuleSet |
| Custom Rule          | rule ที่กำหนดเองตาม business logic           | block IP range ของคู่แข่ง                        |
| Action               | สิ่งที่ทำเมื่อ request match rule                | Allow, Block, Count, CAPTCHA                  |
| Rate-Based Rule      | จำกัด request จาก IP เดียวตามเวลา            | block IP ที่ส่ง > 1000 req ใน 5 นาที              |
| Geo-Restriction      | block หรือ allow traffic จาก country ที่กำหนด | อนุญาตเฉพาะ Thailand, Singapore                |
| Bot Control          | จัดการ bot traffic                         | block bad bot, allow Google bot               |
| IP Set               | กลุ่ม IP ที่ whitelist หรือ blacklist          | whitelist office IP                           |
| Logging              | บันทึก request ที่ match rule                 | ส่ง log ไป S3 หรือ CloudWatch                   |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** Subscription (per ACL/policy) + On-demand (per request)

|                                  | AWS WAF          | GCP Cloud Armor    | Azure WAF on App GW v2    | Huawei WAF         |
| -------------------------------- | ---------------- | ------------------ | ------------------------- | ------------------ |
| Web ACL / Policy                 | $5.00/ACL/month  | $5.00/policy/month | รวมใน App GW (~$0.246/hr) | ~$30/month (basic) |
| Rule                             | $1.00/rule/month | —                  | รวมใน Gateway             | รวมใน plan         |
| Request                          | $0.60/1M         | $0.75/1M evaluated | รวมใน CU charge           | ~$0.30/1M          |
| Managed Rule Group               | $20/group/month  | —                  | —                         | —                  |
| ตัวอย่าง (10 rules + 5M req/month) | ~$43/month       | ~$8/month          | ~$180/month               | ~$30/month         |

### ตัวอย่างการใช้งานใน Project

```
Internet → CloudFront → WAF → ALB → Application
WAF Rules:
- AWS Managed: CommonRuleSet (block OWASP Top 10)
- Rate-Based: block IP > 2000 req/5min
- Geo Block: block traffic จาก high-risk countries
```

### Best Practice

- เริ่มต้นด้วย Count mode ก่อน Block mode เพื่อดูว่า rule block request ที่ถูกต้องหรือไม่
- ใช้ managed rule group ของ Cloud Provider เป็นพื้นฐาน
- เปิด logging เพื่อ analyze attack pattern

### Common Mistakes

- เปิด Block mode ทันทีโดยไม่ได้ test ทำให้ block legitimate user
- ไม่ได้ update managed rule group หลังจาก subscribe

---

## DDoS Protection

### คืออะไร

DDoS Protection คือบริการที่ตรวจจับและบรรเทาการโจมตีแบบ Distributed Denial of Service (DDoS) ที่พยายาม flood ระบบด้วย traffic ปริมาณมากจนทำให้ service ไม่สามารถทำงานได้ตามปกติ

### ใช้งานแบบไหน

เปิดใช้งานระดับ network และ application layer ป้องกันทั้ง volumetric attack (bandwidth flooding) และ application layer attack (HTTP flood)

### เหมาะกับงานแบบไหน

เหมาะกับทุก public-facing service โดยเฉพาะ e-commerce, gaming, financial service, หรือ service ที่เป็น target ของ DDoS

### ไม่เหมาะกับงานแบบไหน

DDoS Protection ระดับสูงมี cost สูง อาจไม่คุ้มค่าสำหรับ internal service ที่ไม่ expose สู่ internet

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                        |
| -------------- | --------------------------------------------------- |
| AWS            | AWS Shield Standard (ฟรี), AWS Shield Advanced       |
| GCP            | Cloud Armor (รวม DDoS protection)                   |
| Azure          | Azure DDoS Protection (Basic ฟรี, Standard มีค่าใช้จ่าย) |
| Huawei Cloud   | Anti-DDoS                                           |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                          | ตัวอย่าง                                   |
| -------------------- | ------------------------------------------------- | ---------------------------------------- |
| Protection Tier      | ระดับการป้องกัน                                      | Standard (ฟรี, auto), Advanced (มีค่าใช้จ่าย) |
| Mitigation Capacity  | bandwidth ที่รับมือได้ก่อน mitigate                     | Tbps level                               |
| Attack Visibility    | dashboard แสดง attack metric                      | CloudWatch metrics, attack report        |
| Response Team        | ทีม support ระหว่างการโจมตี                          | AWS Shield Advanced มี DRT                |
| Rate Limiting        | จำกัด request rate เพื่อรับมือ application layer attack | ร่วมกับ WAF rate-based rule                |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี (basic auto-protection) / Subscription (advanced)

| Tier                  | AWS Shield               | GCP Cloud Armor DDoS     | Azure DDoS Protection    | Huawei Anti-DDoS |
| --------------------- | ------------------------ | ------------------------ | ------------------------ | ---------------- |
| Basic                 | ฟรี (auto L3/L4)          | ฟรี (auto L3/L4)          | ฟรี (auto basic)          | ฟรี (basic)       |
| Advanced / Standard   | $3,000/month (org-level) | $5/policy + request fees | $2,944/month/VIP         | ~$1,500/month    |
| DRT / Expert Response | Shield Advanced only     | N/A                      | Azure DDoS Standard only | Advanced only    |
| SLA / refund          | มี                        | มี                        | มี                        | มี                |

> Basic tier ปกป้อง L3/L4 ได้โดยอัตโนมัติโดยไม่ต้องเปิด Advanced — Advanced เหมาะสำหรับ enterprise ที่ต้องการ 24/7 expert response และ cost protection guarantee

### ตัวอย่างการใช้งานใน Project

เปิด AWS Shield Standard บน CloudFront และ ALB โดยอัตโนมัติ ถ้า service มีความเสี่ยง DDoS สูงอาจ upgrade เป็น Shield Advanced เพื่อ SLA protection และ cost protection

### Best Practice

- ใช้ CDN เป็น DDoS absorption layer แรก
- ร่วมกับ WAF rate limiting เพื่อรับมือ application layer attack
- มี runbook สำหรับขั้นตอนรับมือเมื่อเกิด DDoS

### Common Mistakes

- ไม่ได้เปิด protection จนกว่าจะถูก attack แล้ว
- พึ่ง DDoS Protection อย่างเดียวโดยไม่มี WAF และ rate limiting
