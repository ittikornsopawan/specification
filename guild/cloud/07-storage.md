
# 7. Storage

Storage คือ Service กลุ่มที่ให้บริการเก็บข้อมูลรูปแบบต่าง ๆ ตั้งแต่ file, object, block และ archive แต่ละประเภทเหมาะกับ use case ที่ต่างกัน

---

## Object Storage

### คืออะไร

Object Storage คือบริการเก็บไฟล์แบบ unstructured data ในรูปแบบ object ซึ่งแต่ละ object ประกอบด้วย data, metadata และ unique key เหมาะกับการเก็บ static file, media, backup, log, dataset ขนาดใหญ่ โดย scale ได้ไม่จำกัดและทนทานสูง (typically 99.999999999% durability)

### ใช้งานแบบไหน

ใช้เก็บ static file ของ web application เช่น รูปภาพ, video, document, เก็บ log file จาก application, เก็บ backup ของ database, หรือใช้เป็น data lake สำหรับ analytics

### เหมาะกับงานแบบไหน

เหมาะกับ static website hosting, media storage, backup storage, data lake, log archive, artifact storage สำหรับ CI/CD pipeline

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ read/write แบบ random access เร็ว เช่น database หรือ application ที่ต้องการ POSIX file system

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                 |
| -------------- | ---------------------------- |
| AWS            | Amazon S3                    |
| GCP            | Google Cloud Storage         |
| Azure          | Azure Blob Storage           |
| Huawei Cloud   | Object Storage Service (OBS) |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                            | ตัวอย่าง                                       |
| -------------------- | --------------------------------------------------- | -------------------------------------------- |
| Storage Class        | ระดับ tier ของ storage ที่ส่งผลต่อ access speed และ cost | Standard, Infrequent Access, Archive/Glacier |
| Bucket Policy / ACL  | กำหนดสิทธิ์การเข้าถึง bucket                              | Public read สำหรับ static website              |
| Versioning           | เก็บหลาย version ของ object เดียวกัน                   | เปิดเพื่อป้องกันการลบโดยผิดพลาด                    |
| Lifecycle Policy     | กำหนด rule เปลี่ยน storage class หรือลบ object อัตโนมัติ   | ย้าย log เก่ากว่า 30 วันไป Infrequent Access     |
| Encryption           | การเข้ารหัส object ที่เก็บ                               | SSE-S3, SSE-KMS                              |
| Replication          | copy object ไปยัง bucket อื่นหรือ Region อื่น             | Cross-Region Replication                     |
| Access Logging       | บันทึก request ที่เข้าถึง bucket                          | เพื่อ audit และ security                       |
| Pre-signed URL       | URL ชั่วคราวสำหรับให้ access object โดยไม่ต้องมี credential | Download link หมดอายุใน 1 ชั่วโมง               |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB-month ที่เก็บ + request count + egress

**Storage Cost**

| Storage Class     | AWS S3   | GCP Cloud Storage | Azure Blob | Huawei OBS | หน่วย      |
| ----------------- | -------- | ----------------- | ---------- | ---------- | --------- |
| Standard          | $0.023   | $0.020            | $0.018     | $0.020     | /GB-month |
| Infrequent Access | $0.0125  | $0.010            | $0.010     | $0.010     | /GB-month |
| Archive (Instant) | $0.004   | $0.004            | $0.002     | $0.002     | /GB-month |
| Deep Archive      | $0.00099 | $0.0012           | $0.00099   | $0.001     | /GB-month |

**Request Cost**

|                        | AWS S3   | GCP Cloud Storage | Azure Blob | Huawei OBS | หน่วย         |
| ---------------------- | -------- | ----------------- | ---------- | ---------- | ------------ |
| PUT/COPY/POST/LIST     | $0.005   | $0.005            | $0.055     | $0.004     | /1K requests |
| GET/SELECT/other       | $0.0004  | $0.0004           | $0.0044    | $0.0004    | /1K requests |
| Data Egress (internet) | $0.09/GB | $0.085/GB         | $0.087/GB  | ~$0.072/GB | first 10 TB  |

> ค่า **Data Egress** คือกับดักหลัก — serve object โดยตรงจาก S3 อาจแพงกว่า serve ผ่าน CloudFront CDN เกือบ 10 เท่า ควรใช้ CDN นำหน้า S3 เสมอสำหรับ public content

### ตัวอย่างการใช้งานใน Project

Application ให้ผู้ใช้ upload รูปภาพ → API server สร้าง Pre-signed URL → Client upload โดยตรงไปยัง S3 (ไม่ผ่าน server) → Lambda process thumbnail → รูปพร้อม serve ผ่าน CloudFront CDN

### Best Practice

- ปิด public access โดย default เปิดเฉพาะที่จำเป็น
- เปิด Versioning สำหรับ bucket ที่เก็บข้อมูลสำคัญ
- ตั้ง Lifecycle Policy ลบหรือย้าย tier data เก่าที่ไม่ใช้แล้ว
- ใช้ Pre-signed URL แทนการทำ bucket public

### Common Mistakes

- ทำ bucket เป็น public access โดยไม่ตั้งใจ
- ไม่ได้เปิด Versioning ทำให้ข้อมูลหายเมื่อถูกเขียนทับหรือลบ
- เก็บ secret หรือ credential ใน bucket ที่ไม่ได้ encrypt

---

## Block Storage

### คืออะไร

Block Storage คือบริการ disk เสมือนที่ attach กับ VM หรือ container ทำงานเหมือน hard disk ทั่วไป รองรับ random read/write access มี latency ต่ำ เหมาะกับ database, OS disk, และ application ที่ต้องการ POSIX file system

### ใช้งานแบบไหน

ใช้เป็น disk ของ VM สำหรับ OS, application code หรือ database data directory สามารถ attach/detach จาก instance ได้และ take snapshot สำหรับ backup

### เหมาะกับงานแบบไหน

เหมาะกับ database server, application server ที่ต้องการ fast local disk, หรือ workload ที่ต้องการ POSIX-compliant file system

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับการ share file ระหว่างหลาย instance พร้อมกัน ควรใช้ Shared File Storage แทน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                     |
| -------------- | -------------------------------- |
| AWS            | Amazon EBS (Elastic Block Store) |
| GCP            | Persistent Disk, Hyperdisk       |
| Azure          | Azure Managed Disks              |
| Huawei Cloud   | Elastic Volume Service (EVS)     |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                                                |
| -------------------- | ----------------------------------------- | ----------------------------------------------------- |
| Volume Type          | ประเภทของ disk ส่งผลต่อ IOPS และ throughput | gp3 (General Purpose SSD), io2 (High IOPS), st1 (HDD) |
| Size                 | ขนาดของ volume                            | 100 GB, 500 GB, 2 TB                                  |
| IOPS                 | จำนวน I/O operations ต่อวินาที                | 3000 IOPS (gp3), 64000 IOPS (io2)                     |
| Throughput           | ความเร็วในการ transfer data                | 125 MB/s, 1000 MB/s                                   |
| Snapshot             | การ backup disk ณ จุดเวลาหนึ่ง               | Snapshot ทุกคืนก่อน maintenance                          |
| Encryption           | การเข้ารหัส disk                            | AES-256 ด้วย KMS key                                   |
| Multi-Attach         | attach volume เดียวกันกับหลาย instance       | ใช้กับ cluster database บางประเภท                       |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB ที่ **provision** (ไม่ใช่ที่ใช้จริง)

| Volume Type                       | AWS EBS              | GCP Persistent Disk | Azure Managed Disk | Huawei EVS | หน่วย      |
| --------------------------------- | -------------------- | ------------------- | ------------------ | ---------- | --------- |
| SSD General (gp3 / Balanced)      | $0.080               | $0.170              | $0.084             | ~$0.075    | /GB-month |
| SSD High IOPS (io2 / Extreme)     | $0.125 + $0.065/IOPS | $0.187              | $0.127             | ~$0.120    | /GB-month |
| HDD Throughput (st1 / Throughput) | $0.045               | $0.040              | $0.045             | ~$0.035    | /GB-month |
| Snapshot                          | $0.050               | $0.026              | $0.050             | ~$0.040    | /GB-month |

> Provision ต้องจ่ายเต็มแม้ใช้บางส่วน — ตั้งขนาดให้เหมาะสมและเปิด auto-extend แทนการ over-provision ตั้งแต่ต้น

### ตัวอย่างการใช้งานใน Project

PostgreSQL database server ใช้ io2 volume ขนาด 500 GB เพื่อให้ได้ IOPS สูงพอสำหรับ production workload พร้อมตั้ง automated snapshot ทุกคืน

### Best Practice

- เลือก volume type ให้ตรงกับ workload pattern เช่น database ควรใช้ SSD
- เปิด encryption โดย default
- ตั้ง automated snapshot policy

### Common Mistakes

- ใช้ gp2 (เก่า) แทน gp3 ทั้งที่ gp3 ให้ IOPS มากกว่าในราคาเท่ากัน
- ไม่ได้ตั้ง snapshot ทำให้ไม่มี backup เมื่อ disk เสีย

---

## Shared File Storage (NFS/SMB)

### คืออะไร

Shared File Storage คือบริการ file system ที่ share ระหว่างหลาย instance พร้อมกันได้ ใช้โปรโตคอล NFS หรือ SMB เหมาะกับ application ที่ต้องการ shared storage เช่น content management system, shared media library

### ใช้งานแบบไหน

mount file system บน instance หลายตัวพร้อมกัน ทุก instance เห็น file เดียวกัน เหมาะกับ application ที่ยังไม่ได้ออกแบบให้ store file ใน Object Storage

### เหมาะกับงานแบบไหน

เหมาะกับ legacy application ที่ต้องการ shared disk, CMS เช่น WordPress ที่ต้องการ shared media directory, หรือ Kubernetes persistent volume ที่ต้องการ ReadWriteMany

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ high-throughput workload เพราะ latency สูงกว่า Block Storage หรือ workload ที่ทำได้ด้วย Object Storage

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                     |
| -------------- | -------------------------------- |
| AWS            | Amazon EFS (Elastic File System) |
| GCP            | Cloud Filestore                  |
| Azure          | Azure Files                      |
| Huawei Cloud   | Scalable File Service (SFS)      |

### Spec / Configuration

| Spec / Configuration | ความหมาย                            | ตัวอย่าง                              |
| -------------------- | ----------------------------------- | ----------------------------------- |
| Performance Mode     | ระดับ performance ของ file system    | General Purpose, Max I/O            |
| Throughput Mode      | วิธีกำหนด throughput                   | Bursting, Provisioned               |
| Storage Class        | tier ของ storage                    | Standard, Infrequent Access         |
| Mount Target         | endpoint ที่ใช้ mount จาก instance     | สร้างใน Subnet ที่ต้องการ mount         |
| Access Point         | entry point สำหรับ application แต่ละตัว | กำหนด root path และ permission แยกกัน |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — EFS จ่ายตาม GB ที่ใช้จริง, Filestore/Azure จ่ายตาม capacity ที่ provision

|                   | AWS EFS           | GCP Filestore        | Azure Files (Premium) | Huawei SFS | หน่วย      |
| ----------------- | ----------------- | -------------------- | --------------------- | ---------- | --------- |
| Standard Tier     | $0.300            | $0.200               | $0.060                | ~$0.060    | /GB-month |
| Infrequent Access | $0.025            | N/A                  | N/A                   | N/A        | /GB-month |
| Minimum Size      | ไม่มี (pay per use) | 1 TB (Basic HDD)     | 100 GB                | ไม่มี        | —         |
| Billing basis     | ใช้จริง (GB)        | Provisioned capacity | Provisioned capacity  | ใช้จริง      | —         |

> EFS แพงกว่า S3 Standard ~13 เท่า และแพงกว่า EBS gp3 ~3.75 เท่า ใช้เฉพาะเมื่อต้องการ shared POSIX filesystem จริง ๆ เช่น legacy app หรือ shared config

### ตัวอย่างการใช้งานใน Project

WordPress cluster ที่ run บน EC2 หลาย instance ใช้ EFS เป็น shared `/var/www/html/wp-content/uploads` ทำให้ทุก instance เห็น media file เดียวกัน

### Best Practice

- ใช้ Lifecycle Policy ย้าย file ที่ไม่ได้ access ไป Infrequent Access เพื่อลดค่าใช้จ่าย
- ใช้ Access Point เพื่อแยก root directory ต่อ application

### Common Mistakes

- ใช้ Shared File Storage กับ workload ที่ควรใช้ Object Storage เพราะ Shared File Storage แพงกว่า
- ไม่ได้ตั้ง Security Group ควบคุมการ mount
