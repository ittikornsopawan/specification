# 8. Database

Database คือ Service กลุ่มที่ให้บริการ managed database ประเภทต่าง ๆ Cloud Provider จัดการ provisioning, patching, backup และ high availability ให้ ทำให้ทีมโฟกัสที่ data และ schema แทนการจัดการ infrastructure

---

## Relational Database (RDS / Managed SQL)

### คืออะไร

Relational Database Service คือ managed database สำหรับ SQL engine ต่าง ๆ เช่น MySQL, PostgreSQL, MariaDB, SQL Server, Oracle Cloud Provider จัดการ installation, patching, backup, failover และ monitoring ให้

### ใช้งานแบบไหน

ใช้เป็น primary database ของ application ที่ต้องการ ACID transaction, structured data, หรือ complex query ด้วย SQL

### เหมาะกับงานแบบไหน

เหมาะกับ transactional workload, e-commerce, ERP, CRM, application ที่ต้องการ relational data model และ ACID compliance

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ document data ที่ structure เปลี่ยนบ่อย, time-series data ปริมาณมาก, หรือ graph data ที่มี complex relationship

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                            |
| -------------- | ------------------------------------------------------- |
| AWS            | Amazon RDS, Amazon Aurora                               |
| GCP            | Cloud SQL, AlloyDB                                      |
| Azure          | Azure Database for PostgreSQL/MySQL, Azure SQL Database |
| Huawei Cloud   | Relational Database Service (RDS)                       |

### Spec / Configuration

#### AWS RDS Instance Class

| Instance Class   | vCPU | Memory | เหมาะกับงานแบบไหน                           | On-demand ($/hr) | RI 1yr No Upfront | RI 1yr All Upfront | RI 3yr All Upfront |
| ---------------- | ---: | -----: | ------------------------------------------ | ---------------: | ----------------: | -----------------: | -----------------: |
| `db.t3.medium`   |    2 |   4 GB | dev/test, small application                |            0.068 |             0.044 |              0.042 |              0.030 |
| `db.t3.large`    |    2 |   8 GB | dev/test, medium application               |            0.136 |             0.088 |              0.084 |              0.060 |
| `db.m6g.large`   |    2 |   8 GB | general purpose production                 |            0.150 |             0.098 |              0.094 |              0.067 |
| `db.m6g.xlarge`  |    4 |  16 GB | general purpose production, medium traffic |            0.300 |             0.196 |              0.188 |              0.134 |
| `db.m6g.2xlarge` |    8 |  32 GB | general purpose production, high traffic   |            0.600 |             0.392 |              0.376 |              0.268 |
| `db.r6g.large`   |    2 |  16 GB | memory-heavy workload                      |            0.240 |             0.157 |              0.150 |              0.107 |
| `db.r6g.xlarge`  |    4 |  32 GB | memory-heavy workload, large database      |            0.480 |             0.314 |              0.300 |              0.214 |

#### GCP Cloud SQL Tier

| Machine Type       |   vCPU | Memory | เหมาะกับงานแบบไหน                           | On-demand ($/hr) | CUD 1yr |
| ------------------ | -----: | -----: | ------------------------------------------ | ---------------: | ------: |
| `db-f1-micro`      | shared | 0.6 GB | dev/test เท่านั้น                             |            0.013 |       — |
| `db-g1-small`      | shared | 1.7 GB | small dev/test                             |            0.025 |       — |
| `db-n1-standard-2` |      2 | 7.5 GB | general purpose production                 |            0.096 |   0.061 |
| `db-n1-standard-4` |      4 |  15 GB | general purpose production, medium traffic |            0.192 |   0.122 |
| `db-n1-highmem-4`  |      4 |  26 GB | memory-heavy workload                      |            0.256 |   0.162 |

#### Azure Database Instance (PostgreSQL Flexible)

| SKU Name           | vCPU | Memory | เหมาะกับงานแบบไหน                      | PAYG ($/hr) | Reserved 1yr |
| ------------------ | ---: | -----: | ------------------------------------- | ----------: | -----------: |
| `Standard_B2s`     |    2 |   4 GB | dev/test                              |       0.068 |        0.044 |
| `Standard_D2ds_v4` |    2 |   8 GB | general purpose production            |       0.150 |        0.095 |
| `Standard_D4ds_v4` |    4 |  16 GB | medium traffic                        |       0.300 |        0.190 |
| `Standard_E2ds_v4` |    2 |  16 GB | memory-heavy workload                 |       0.300 |        0.190 |
| `Standard_E4ds_v4` |    4 |  32 GB | memory-heavy workload, large database |       0.600 |        0.380 |

#### Huawei Cloud RDS Flavor

| Flavor                | vCPU | Memory | เหมาะกับงานแบบไหน           | On-demand ($/hr) | Reserved 1yr |
| --------------------- | ---: | -----: | -------------------------- | ---------------: | -----------: |
| `rds.mysql.c2.medium` |    2 |   4 GB | dev/test                   |           ~0.060 |       ~0.039 |
| `rds.mysql.c2.large`  |    2 |   8 GB | general purpose production |           ~0.130 |       ~0.085 |
| `rds.mysql.c2.xlarge` |    4 |  16 GB | medium traffic             |           ~0.260 |       ~0.169 |
| `rds.mysql.m2.xlarge` |    4 |  32 GB | memory-heavy workload      |           ~0.400 |       ~0.260 |

### Database Configuration

| Spec / Configuration | ความหมาย                                           | ตัวอย่าง                          |
| -------------------- | -------------------------------------------------- | ------------------------------- |
| Engine & Version     | database engine และ version                        | PostgreSQL 16, MySQL 8.0        |
| Multi-AZ             | deploy database แบบ synchronous replication ข้าม AZ | เปิดสำหรับ production              |
| Read Replica         | copy ที่ read-only สำหรับ scale read workload          | สร้าง 1-2 read replica           |
| Storage Type         | ประเภทของ storage                                  | gp3, io1                        |
| Storage Size         | ขนาด disk                                          | 100 GB เริ่มต้น                    |
| Automated Backup     | backup อัตโนมัติพร้อม retention                        | 7 วัน                            |
| Parameter Group      | ค่า database configuration                          | max_connections, shared_buffers |
| Maintenance Window   | ช่วงเวลาสำหรับ patch อัตโนมัติ                           | อาทิตย์ 02:00-03:00 UTC           |
| Deletion Protection  | ป้องกันการลบ database โดยบังเอิญ                       | เปิดสำหรับ production              |

### ตัวอย่างการใช้งานใน Project

```
Application Server → RDS PostgreSQL Primary (Multi-AZ)
                   → RDS Read Replica (สำหรับ analytics query)
```

### Best Practice

- เปิด Multi-AZ ทุก production database เสมอ
- สร้าง Read Replica สำหรับ analytics หรือ reporting query แยกออกจาก primary
- ตั้ง Deletion Protection เพื่อป้องกันการลบโดยบังเอิญ
- monitor connection count เพราะ database มี connection limit

### Common Mistakes

- ใช้ database เดียวกันสำหรับทั้ง application และ analytics
- ไม่ได้เปิด Multi-AZ ทำให้ downtime นานเมื่อ AZ มีปัญหา
- ไม่ได้ตั้ง Parameter Group ที่เหมาะสม ใช้ค่า default ทั้งหมด

---

## NoSQL Database

### คืออะไร

NoSQL Database คือ database ที่ไม่ใช้ relational model มีหลายประเภทเช่น Document Store, Key-Value Store, Wide Column, Graph database แต่ละประเภทเหมาะกับ data model และ access pattern ที่ต่างกัน

### ใช้งานแบบไหน

เลือกประเภท NoSQL ตาม data model เช่น ใช้ Document Store สำหรับ user profile หรือ product catalog, ใช้ Key-Value Store สำหรับ session หรือ feature flag, ใช้ Wide Column สำหรับ time-series data

### เหมาะกับงานแบบไหน

เหมาะกับ workload ที่ต้องการ horizontal scaling สูง, schema ยืดหยุ่น, document data, time-series, หรือ simple key-value lookup

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ complex JOIN หรือ multi-table transaction แบบ ACID อย่างเข้มงวด

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                                                 |
| -------------- | ---------------------------------------------------------------------------- |
| AWS            | Amazon DynamoDB (Key-Value/Document), Amazon DocumentDB (MongoDB-compatible) |
| GCP            | Cloud Firestore (Document), Cloud Bigtable (Wide Column)                     |
| Azure          | Azure Cosmos DB (Multi-model)                                                |
| Huawei Cloud   | Document Database Service (DDS), GaussDB NoSQL                               |

### Spec / Configuration

| Spec / Configuration         | ความหมาย                                | ตัวอย่าง                                   |
| ---------------------------- | --------------------------------------- | ---------------------------------------- |
| Read / Write Capacity        | ความสามารถในการรับ read/write            | On-demand หรือ Provisioned (RCU/WCU)      |
| Partition Key / Sort Key     | primary key structure ของ table         | `userId` (partition), `timestamp` (sort) |
| Global Secondary Index (GSI) | index เพิ่มเติมสำหรับ query pattern ต่างออกไป | query by email แทน userId                |
| Consistency Level            | ระดับความ consistent ของ read            | Eventual Consistency, Strong Consistency |
| TTL (Time to Live)           | กำหนดวันหมดอายุของ item อัตโนมัติ             | session หมดอายุใน 24 ชั่วโมง                |
| Replication                  | จำนวน region ที่ replicate ข้อมูล            | Global Tables ใน 3 Region                |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม read/write operation และ storage

**AWS DynamoDB**

| Billing Mode | Write           | Read             | Storage        |
| ------------ | --------------- | ---------------- | -------------- |
| On-demand    | $1.25/1M WRU    | $0.25/1M RRU     | $0.25/GB-month |
| Provisioned  | $0.47/WCU-month | $0.047/RCU-month | $0.25/GB-month |

**GCP Firestore**

| Operation | ราคา                  |
| --------- | --------------------- |
| Write     | $0.18/100K operations |
| Read      | $0.06/100K operations |
| Storage   | $0.18/GB-month        |

**Azure Cosmos DB**

| Billing Mode          | ราคา                    | Storage        |
| --------------------- | ----------------------- | -------------- |
| Serverless            | $0.25/1M RU             | $0.25/GB-month |
| Autoscale Provisioned | $0.012/100 RU/sec-month | $0.25/GB-month |

**Huawei DDS (MongoDB-compatible)**

| Instance       | ราคา/hour |
| -------------- | --------- |
| 2 vCPU / 8 GB  | ~$0.100   |
| 4 vCPU / 16 GB | ~$0.200   |

### ตัวอย่างการใช้งานใน Project

E-commerce ใช้ DynamoDB เก็บ shopping cart session โดยใช้ `userId` เป็น partition key และตั้ง TTL 7 วัน ให้ item ลบตัวเองอัตโนมัติเมื่อหมดอายุ

### Best Practice

- ออกแบบ data model จาก access pattern ก่อน ไม่ใช่จาก entity model
- เลือก partition key ที่กระจาย load ได้ดี ไม่ควรเป็น key ที่มีค่าซ้ำกันมาก
- ใช้ TTL สำหรับ data ที่มีอายุ เช่น session, cache

### Common Mistakes

- ออกแบบ schema โดยคิดแบบ relational ทำให้ query ไม่ efficient
- เลือก partition key ที่ไม่กระจาย เช่นใช้ date เป็น partition key ทำให้ hot partition
