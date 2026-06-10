# 20. Backup & Disaster Recovery

Backup และ Disaster Recovery คือ Service กลุ่มที่ช่วยปกป้องข้อมูลและให้ระบบสามารถ recover ได้เมื่อเกิด failure ตั้งแต่ human error ไปจนถึง region-wide outage

---

## Backup Service

### คืออะไร

Backup Service คือบริการที่ automate การ backup resource ต่าง ๆ เช่น database, volume, file system ตามตารางเวลาที่กำหนด พร้อม retention policy และ restore capability

### ใช้งานแบบไหน

สร้าง backup plan กำหนดว่า resource ใดต้อง backup บ่อยแค่ไหน เก็บ backup ไว้นานแค่ไหน และต้องการ cross-region หรือ cross-account copy หรือไม่

### เหมาะกับงานแบบไหน

เหมาะกับทุก production data ที่ไม่ยอมสูญเสียได้ โดยเฉพาะ database, critical file และ configuration

### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรมี backup ใน production

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                    |
| -------------- | ------------------------------- |
| AWS            | AWS Backup                      |
| GCP            | Cloud Backup and DR             |
| Azure          | Azure Backup                    |
| Huawei Cloud   | Cloud Backup and Recovery (CBR) |

### Spec / Configuration

| Spec / Configuration           | ความหมาย                                    | ตัวอย่าง                                      |
| ------------------------------ | ------------------------------------------- | ------------------------------------------- |
| Backup Plan                    | กำหนด rule ว่า backup อะไร เมื่อไหร่ เก็บนานแค่ไหน | daily backup, retention 30 days             |
| Recovery Point Objective (RPO) | ข้อมูลที่ยอมสูญเสียได้มากที่สุด                       | 1 ชั่วโมง = backup ทุกชั่วโมง                    |
| Recovery Time Objective (RTO)  | เวลาสูงสุดที่ยอมรับในการ restore                 | 4 ชั่วโมง                                     |
| Backup Frequency               | ความถี่ในการ backup                           | hourly, daily, weekly                       |
| Retention Period               | ระยะเวลาเก็บ backup                          | 7 วัน daily, 4 สัปดาห์ weekly, 12 เดือน monthly |
| Cross-Region Copy              | copy backup ไปยัง Region อื่น                  | เพื่อ DR ใน Region ที่สอง                       |
| Backup Vault                   | ที่เก็บ backup พร้อม access control             | encrypted, immutable backup                 |
| Restore Test                   | การทดสอบ restore backup จริง                 | ทดสอบทุกไตรมาส                               |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB ที่ backup จริง (incremental หลังแรก)

| Backup Type             | AWS Backup           | GCP Cloud Backup | Azure Backup    | Huawei CBR      |
| ----------------------- | -------------------- | ---------------- | --------------- | --------------- |
| VM Disk / EBS Snapshot  | $0.05/GB-month       | $0.026/GB-month  | $0.05/GB-month  | ~$0.04/GB-month |
| RDS / Database Backup   | $0.095/GB-month      | $0.08/GB-month   | $0.095/GB-month | ~$0.04/GB-month |
| File System / S3 Backup | $0.05/GB-month       | included         | $0.02/GB-month  | ~$0.03/GB-month |
| Cross-Region Copy       | $0.02/GB transferred | $0.02/GB         | $0.02/GB        | ~$0.015/GB      |

> RDS automated backup ใน AWS ฟรีสำหรับ 0–7 วัน (ขนาดเท่า storage instance) — ค่าใช้จ่าย Backup ส่วนใหญ่เกิดจาก long-term retention และ cross-region copy

### ตัวอย่างการใช้งานใน Project

RDS กำหนด daily backup เวลา 02:00 UTC retention 30 วัน และ cross-region copy ไปยัง secondary region เพื่อ DR พร้อมทดสอบ restore ทุก quarter

### Best Practice

- กำหนด RPO และ RTO ก่อนออกแบบ backup strategy
- ทดสอบ restore จริงสม่ำเสมอ ไม่ใช่แค่มี backup
- เก็บ backup ใน account หรือ region ที่แยกจาก production
- ใช้ immutable backup เพื่อป้องกัน ransomware ลบ backup

### Common Mistakes

- มี backup แต่ไม่เคยทดสอบ restore ทำให้ไม่รู้ว่า restore ได้จริงหรือไม่
- เก็บ backup ใน account เดียวกัน ทำให้ ransomware โจมตีได้ทั้ง production และ backup
- ตั้ง retention สั้นเกินไป ทำให้ไม่มี backup เก่าพอสำหรับ compliance
