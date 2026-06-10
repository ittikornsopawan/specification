
# 15. Monitoring & Observability

Monitoring และ Observability คือ Service กลุ่มที่ช่วยให้เห็น health และ performance ของระบบแบบ real-time ช่วย detect ปัญหาก่อนที่จะส่งผลกระทบต่อ user และช่วย debug เมื่อเกิดปัญหา

---

## Cloud Monitoring (Metrics & Alerting)

### คืออะไร

Cloud Monitoring คือบริการเก็บ metric จาก Cloud resource ต่าง ๆ เช่น CPU, memory, disk, network และ custom metric จาก application แล้วแสดงผลใน dashboard และ alert เมื่อ metric เกิน threshold

### ใช้งานแบบไหน

ตั้ง alarm บน metric ที่สำคัญเช่น CPU > 80%, error rate > 1%, p99 latency > 500ms เชื่อมต่อกับ notification เช่น email, Slack, PagerDuty เมื่อ alarm trigger

### เหมาะกับงานแบบไหน

เหมาะกับทุก production system ที่ต้องการ observability

### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ monitoring ใน production

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                     |
| -------------- | ------------------------------------------------ |
| AWS            | Amazon CloudWatch                                |
| GCP            | Cloud Monitoring (Google Cloud Operations Suite) |
| Azure          | Azure Monitor                                    |
| Huawei Cloud   | Cloud Eye                                        |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                | ตัวอย่าง                                  |
| -------------------- | --------------------------------------- | --------------------------------------- |
| Metric               | ข้อมูลตัวเลขที่วัดตามเวลา                     | CPUUtilization, RequestCount, ErrorRate |
| Namespace            | กลุ่มของ metric                           | `AWS/EC2`, `MyApp/API`                  |
| Dimension            | attribute ที่ใช้ filter metric             | InstanceId, Environment                 |
| Period               | ช่วงเวลาของแต่ละ data point               | 60 seconds                              |
| Statistics           | วิธีคำนวณ aggregate                        | Average, Sum, Maximum, p99              |
| Alarm                | rule ที่ trigger เมื่อ metric เกิน threshold | CPU > 80% นาน 5 นาที                     |
| Alarm Action         | สิ่งที่เกิดขึ้นเมื่อ alarm trigger               | SNS notification, Auto Scaling          |
| Dashboard            | หน้าแสดงผล metric หลายตัวในที่เดียว          | Production Overview Dashboard           |
| Retention            | ระยะเวลาเก็บ metric data                 | 15 months (CloudWatch)                  |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี (platform metrics) + On-demand (custom metrics, dashboards, alerts)

| Feature                         | AWS CloudWatch                       | GCP Cloud Monitoring                | Azure Monitor      | Huawei Cloud Eye    |
| ------------------------------- | ------------------------------------ | ----------------------------------- | ------------------ | ------------------- |
| Platform Metrics (CPU/Disk ฯลฯ) | ฟรี                                   | ฟรี                                  | ฟรี                 | ฟรี                  |
| Custom Metric                   | $0.30/metric/month (first 10K)       | $0.18/metric/month (after 150 free) | $0.25/metric/month | ~$0.20/metric/month |
| Dashboard                       | $3.00/dashboard/month (after 3 free) | ฟรี                                  | ฟรี                 | ฟรี                  |
| Alert Rule                      | $0.10/alarm/month                    | ฟรี (first 5)                        | $0.10/rule/month   | ~$0.05/alarm/month  |
| GetMetricData API               | $0.01/1K metrics requested           | ฟรี                                  | ฟรี                 | ฟรี                  |

### ตัวอย่างการใช้งานใน Project

กำหนด alarm บน API Gateway: `5xx error rate > 1%` ส่ง SNS notification ไปยัง Slack channel ทีมพร้อม PagerDuty สำหรับ on-call engineer

### Best Practice

- monitor ทั้ง infrastructure metric และ application metric (custom metric)
- ตั้ง alarm ในระดับ warning ก่อน critical เพื่อให้มีเวลาตอบสนอง
- สร้าง dashboard สำหรับ daily review
- ทดสอบ alarm ว่า fire ไปถึง on-call จริง

### Common Mistakes

- ตั้ง alarm เยอะเกินไปทำให้ alert fatigue ทีมไม่สนใจ alarm
- ไม่ได้ monitor application metric เช่น error rate, latency เฉพาะ infra metric
- ลืม test ว่า notification ส่งถึงคนที่ใช่จริง
