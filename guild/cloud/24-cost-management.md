# 24. Cost Management

Cost Management คือ Service กลุ่มที่ช่วย monitor, analyze และ optimize ค่าใช้จ่ายบน Cloud ช่วยให้ทีมเข้าใจว่าเงินถูกใช้ไปกับ resource อะไรและมีโอกาส optimize ที่ไหนบ้าง

---

## Cloud Cost Management & Billing

### คืออะไร

Cloud Cost Management คือบริการที่แสดงรายละเอียดค่าใช้จ่ายตาม service, account, tag, region และ time period พร้อม budget alert และ recommendation สำหรับ cost optimization

### ใช้งานแบบไหน

ตั้ง budget สำหรับแต่ละ account หรือ project กำหนด alert เมื่อค่าใช้จ่ายเกิน threshold review cost report รายสัปดาห์ ใช้ tag กำกับ resource เพื่อ track cost ต่อ team หรือ feature

### เหมาะกับงานแบบไหน

เหมาะกับทุก Cloud environment การไม่ monitor cost มักทำให้ค่าใช้จ่ายบานปลาย

### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ cost management

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                              |
| -------------- | --------------------------------------------------------- |
| AWS            | AWS Cost Explorer, AWS Budgets, AWS Cost and Usage Report |
| GCP            | Cloud Billing, Cost Management                            |
| Azure          | Azure Cost Management + Billing                           |
| Huawei Cloud   | Cost Center                                               |

### Spec / Configuration

| Spec / Configuration               | ความหมาย                                 | ตัวอย่าง                                     |
| ---------------------------------- | ---------------------------------------- | ------------------------------------------ |
| Budget                             | วงเงินที่กำหนดต่อ period                      | $500/เดือน ต่อ account                       |
| Budget Alert                       | alert เมื่อค่าใช้จ่ายเกิน % ของ budget         | alert เมื่อถึง 80% ของ budget                 |
| Cost Allocation Tag                | tag สำหรับ group cost ตาม dimension        | `Environment: production`, `Team: backend` |
| Savings Plans / Reserved Instances | commitment ล่วงหน้าเพื่อลดค่าใช้จ่าย            | 1 ปี Savings Plans ลด 30-40%                |
| Spot / Preemptible Instance        | instance ราคาถูกที่ interruptible ได้        | ลดค่าใช้จ่าย batch workload 60-90%            |
| Right Sizing                       | การ resize resource ให้เหมาะกับการใช้งานจริง | ลด instance ที่ CPU < 5% ตลอด                |
| Idle Resource                      | resource ที่ไม่ได้ใช้งานแต่ยังถูก charge         | ลบ unattached EBS volume                   |
| Cost Anomaly Detection             | ตรวจหาค่าใช้จ่ายที่ผิดปกติ                      | alert เมื่อค่าใช้จ่ายเพิ่มขึ้น 50% จากสัปดาห์ก่อน      |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี (core features) + On-demand (advanced API)

| Feature                            | AWS                   | GCP | Azure | Huawei Cloud |
| ---------------------------------- | --------------------- | --- | ----- | ------------ |
| Cost Dashboard / Bills             | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| Budget Alert                       | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| Cost Explorer UI                   | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| Cost Explorer API                  | $0.01/paginated query | ฟรี  | ฟรี    | ฟรี           |
| Anomaly Detection                  | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| Savings Recommendations            | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| 3rd-party Cost Tools (Spot.io ฯลฯ) | ค่าใช้จ่ายแยกต่างหาก      | —   | —     | —            |

### ตัวอย่างการใช้งานใน Project

กำหนด Budget $1000/เดือน ต่อ account ตั้ง alert ที่ 70%, 90%, 100% ใช้ tag `Environment` และ `Team` ทุก resource review Cost Explorer ทุกสัปดาห์เพื่อดู trend

### Best Practice

- ใช้ tag ทุก resource ตั้งแต่เริ่มต้น ไม่ใช่ย้อนหลัง
- ตั้ง budget alert ทุก account ทุก environment
- review unused resource สม่ำเสมอ เช่น idle instance, unattached volume, old snapshot
- ใช้ Savings Plans หรือ Reserved Instance สำหรับ steady-state workload
- เปิด Cost Anomaly Detection เพื่อจับ cost spike กะทันหัน

### Common Mistakes

- ไม่ได้ tag resource ทำให้ไม่รู้ว่า cost ไปกับ team หรือ service ใด
- ไม่ได้ตั้ง budget alert ทำให้รู้ค่าใช้จ่ายหลังจาก bill มาแล้ว
- ลืมลบ resource ที่ไม่ใช้แล้วเช่น dev environment ที่ไม่มีใครใช้
- ใช้ On-Demand pricing สำหรับ resource ที่ run 24/7 โดยไม่ได้ซื้อ Savings Plans
