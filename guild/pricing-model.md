# Pricing Model / Billing Model

การเข้าใจ Pricing Model ช่วยให้ตัดสินใจได้ว่าจะใช้รูปแบบการจ่ายเงินแบบไหนให้เหมาะกับ workload และงบประมาณของ project

> **หมายเหตุ:** ราคาทั้งหมดในเอกสารนี้เป็นราคาโดยประมาณ ณ ปี 2025 อ้างอิงจาก Region: **AWS us-east-1 / GCP us-central1 / Azure East US / Huawei Cloud AP-Southeast** ราคาจริงอาจแตกต่างตาม Region, configuration และข้อตกลงพิเศษ ควรตรวจสอบจาก pricing calculator ของแต่ละ Cloud Provider เสมอ

---

## Pay-as-you-go / On-demand

### คืออะไร

รูปแบบจ่ายตามการใช้งานจริง ไม่มีการ commit ล่วงหน้า ใช้เท่าไรจ่ายเท่านั้น

### เหมาะกับงานแบบไหน

dev/test environment, workload ที่มี traffic spike ไม่สม่ำเสมอ, project ใหม่ที่ยังไม่รู้ usage pattern

### ไม่เหมาะกับงานแบบไหน

steady-state production workload ที่ run 24/7 เป็นเดือน ๆ เพราะแพงกว่า Reserved อย่างมีนัยสำคัญ

### ตัวอย่างราคาต่อหน่วย (On-demand)

---

#### Networking

**NAT Gateway**

|                   | AWS         | GCP        | Azure       | Huawei Cloud |
| ----------------- | ----------- | ---------- | ----------- | ------------ |
| ค่า Gateway        | $0.045/hour | $0.01/hour | $0.045/hour | ~$0.030/hour |
| ค่า Data Processed | $0.045/GB   | $0.045/GB  | $0.045/GB   | ~$0.040/GB   |

**VPN Gateway**

|      | AWS Site-to-Site VPN  | GCP Cloud VPN     | Azure VPN Gateway (VpnGw1) | Huawei VPN Gateway |
| ---- | --------------------- | ----------------- | -------------------------- | ------------------ |
| ราคา | $0.05/hour/connection | $0.20/tunnel/hour | $0.190/hour                | ~$0.050/hour       |

**Direct Connect / Dedicated Line**

|                     | AWS Direct Connect | GCP Cloud Interconnect | Azure ExpressRoute    | Huawei Direct Connect |
| ------------------- | ------------------ | ---------------------- | --------------------- | --------------------- |
| 1 Gbps port         | $0.30/hour         | N/A                    | ~$220/month (50 Mbps) | ~$0.25/hour           |
| 10 Gbps port        | $1.60/hour         | $1.735/hour            | ~$5,000/month         | ~$1.20/hour           |
| Data Transfer (in)  | ฟรี                 | ฟรี                     | ฟรี                    | ฟรี                    |
| Data Transfer (out) | $0.02/GB           | $0.02/GB               | $0.025/GB             | ~$0.02/GB             |

**Data Transfer**

| ทิศทาง                                  | AWS       | GCP       | Azure     | Huawei Cloud |
| -------------------------------------- | --------- | --------- | --------- | ------------ |
| Ingress (internet → cloud)             | ฟรี        | ฟรี        | ฟรี        | ฟรี           |
| Egress to internet (first 10 TB/month) | $0.090/GB | $0.085/GB | $0.087/GB | $0.072/GB    |
| Egress to internet (next 40 TB/month)  | $0.085/GB | $0.065/GB | $0.083/GB | $0.060/GB    |
| Same Region, cross-AZ                  | $0.010/GB | $0.010/GB | $0.010/GB | ฟรี           |
| Cross-Region (within same continent)   | $0.020/GB | $0.020/GB | $0.020/GB | ~$0.015/GB   |

---

#### Compute / Virtual Machine

**On-demand Instance Price (per hour)**

| Instance Type (AWS/GCP/Azure/Huawei)                | vCPU | Memory | AWS     | GCP    | Azure   | Huawei  |
| --------------------------------------------------- | ---: | -----: | ------- | ------ | ------- | ------- |
| t3.medium / e2-medium / B2s / s6.large.2            |    2 |   4 GB | $0.0416 | $0.033 | $0.0416 | ~$0.040 |
| t3.large / e2-standard-2 / B2ms / s6.large.4        |    2 |   8 GB | $0.0832 | $0.067 | $0.083  | ~$0.075 |
| m6i.large / n2-standard-2 / D2s v5 / c6.large.4     |    2 |   8 GB | $0.096  | $0.097 | $0.096  | ~$0.085 |
| m6i.xlarge / n2-standard-4 / D4s v5 / c6.xlarge.4   |    4 |  16 GB | $0.192  | $0.194 | $0.192  | ~$0.170 |
| m6i.2xlarge / n2-standard-8 / D8s v5 / c6.2xlarge.4 |    8 |  32 GB | $0.384  | $0.388 | $0.384  | ~$0.340 |
| c6i.large / c2-standard-2 / F2s v2 / c6.large.2     |    2 |   4 GB | $0.085  | $0.105 | $0.085  | ~$0.075 |
| c6i.xlarge / c2-standard-4 / F4s v2 / c6.xlarge.2   |    4 |   8 GB | $0.170  | $0.209 | $0.169  | ~$0.150 |
| r6i.large / n2-highmem-2 / E2s v5 / m6.large.8      |    2 |  16 GB | $0.126  | $0.131 | $0.126  | ~$0.120 |
| r6i.xlarge / n2-highmem-4 / E4s v5 / m6.xlarge.8    |    4 |  32 GB | $0.252  | $0.262 | $0.252  | ~$0.240 |

---

#### Container Instance / Serverless Container

**AWS Fargate (per hour)**

| Resource              | ราคา               |
| --------------------- | ------------------ |
| vCPU                  | $0.04048/vCPU-hour |
| Memory                | $0.004445/GB-hour  |
| ตัวอย่าง: 1 vCPU + 2 GB | ~$0.049/hour       |
| ตัวอย่าง: 2 vCPU + 4 GB | ~$0.099/hour       |

**GCP Cloud Run (per second, หลัง Free Tier)**

| Resource  | ราคา                                             |
| --------- | ------------------------------------------------ |
| vCPU      | $0.00002400/vCPU-second ($0.0864/vCPU-hour)      |
| Memory    | $0.00000250/GB-second ($0.009/GB-hour)           |
| Request   | $0.40/1M requests                                |
| Free Tier | 2M requests + 360K vCPU-sec + 180K GB-sec /month |

**Azure Container Apps (per second)**

| Resource | ราคา                                      |
| -------- | ----------------------------------------- |
| vCPU     | $0.000024/vCPU-second ($0.0864/vCPU-hour) |
| Memory   | $0.000003/GB-second ($0.0108/GB-hour)     |
| Request  | $0.40/1M requests                         |

**Huawei Cloud CCI (per hour)**

| Resource | ราคา              |
| -------- | ----------------- |
| vCPU     | ~$0.035/vCPU-hour |
| Memory   | ~$0.004/GB-hour   |

---

#### Serverless / FaaS

|                            | AWS Lambda              | GCP Cloud Functions  | Azure Functions (Consumption) | Huawei FunctionGraph  |
| -------------------------- | ----------------------- | -------------------- | ----------------------------- | --------------------- |
| Invocation                 | $0.20/1M requests       | $0.40/1M requests    | $0.20/1M requests             | $0.20/1M requests     |
| Compute                    | $0.0000166667/GB-second | $0.0000100/GB-second | $0.000016/GB-second           | $0.00001667/GB-second |
| Free Tier (monthly)        | 1M req + 400K GB-sec    | 2M req + 400K GB-sec | 1M req + 400K GB-sec          | 1M req + 400K GB-sec  |
| ตัวอย่าง 1M req × 512MB × 1s | ~$0.208                 | ~$0.405              | ~$0.208                       | ~$0.208               |

---

#### Load Balancing

**Application Load Balancer (ALB)**

|                                | AWS ALB         | GCP HTTP(S) LB | Azure Application Gateway (v2) | Huawei ELB Dedicated |
| ------------------------------ | --------------- | -------------- | ------------------------------ | -------------------- |
| ค่า Gateway/hour                | $0.008          | $0.008         | $0.246 (small)                 | ~$0.007              |
| ค่า Usage                       | $0.008/LCU-hour | $0.006/GB      | $0.008/CU-hour                 | ~$0.003/GB           |
| ตัวอย่าง (1 Gbps medium traffic) | ~$15–30/month   | ~$18–35/month  | ~$200–250/month                | ~$10–20/month        |

**Network Load Balancer (NLB)**

|                 | AWS NLB          | GCP Network LB | Azure Standard Load Balancer | Huawei ELB Network |
| --------------- | ---------------- | -------------- | ---------------------------- | ------------------ |
| ค่า Gateway/hour | $0.008           | $0.008         | $0.025                       | ~$0.006            |
| ค่า Usage        | $0.006/NLCU-hour | $0.006/GB      | $0.005/GB (rules)            | ~$0.003/GB         |

---

#### API Management

|           | AWS API Gateway (REST) | GCP Apigee (Eval)  | Azure API Management (Consumption) | Huawei APIG  |
| --------- | ---------------------- | ------------------ | ---------------------------------- | ------------ |
| Request   | $3.50/1M               | $0.03/1K API calls | $3.50/1M                           | ~$3.00/1M    |
| Cache     | $0.020/hour (0.5 GB)   | N/A                | included                           | ~$0.015/hour |
| WebSocket | $1.00/1M messages      | N/A                | included                           | ~$1.00/1M    |

---

#### Object Storage

| Storage Class              | AWS S3            | GCP Cloud Storage | Azure Blob Storage | Huawei OBS      |
| -------------------------- | ----------------- | ----------------- | ------------------ | --------------- |
| Standard                   | $0.023/GB-month   | $0.020/GB-month   | $0.018/GB-month    | $0.020/GB-month |
| Infrequent Access          | $0.0125/GB-month  | $0.010/GB-month   | $0.010/GB-month    | $0.010/GB-month |
| Archive/Glacier Instant    | $0.004/GB-month   | $0.004/GB-month   | $0.002/GB-month    | $0.002/GB-month |
| Deep Archive               | $0.00099/GB-month | $0.0012/GB-month  | $0.00099/GB-month  | $0.001/GB-month |
| PUT/POST (per 1K requests) | $0.005            | $0.005            | $0.055             | $0.004          |
| GET (per 1K requests)      | $0.0004           | $0.0004           | $0.0044            | $0.0004         |

---

#### Block Storage

| Volume Type                   | AWS EBS                 | GCP Persistent Disk | Azure Managed Disk    | Huawei EVS       |
| ----------------------------- | ----------------------- | ------------------- | --------------------- | ---------------- |
| SSD General Purpose (gp3/SSD) | $0.08/GB-month          | $0.170/GB-month     | $0.084/GB-month (P10) | ~$0.075/GB-month |
| SSD High IOPS (io2/Extreme)   | $0.125/GB + $0.065/IOPS | $0.187/GB-month     | $0.127/GB-month (P20) | ~$0.120/GB-month |
| HDD (st1/Standard)            | $0.045/GB-month         | $0.040/GB-month     | $0.045/GB-month       | ~$0.035/GB-month |
| Snapshot                      | $0.05/GB-month          | $0.026/GB-month     | $0.05/GB-month        | ~$0.04/GB-month  |

---

#### Shared File Storage (NFS)

|                   | AWS EFS             | GCP Filestore  | Azure Files (Premium) | Huawei SFS       |
| ----------------- | ------------------- | -------------- | --------------------- | ---------------- |
| Standard Tier     | $0.30/GB-month      | $0.20/GB-month | $0.06/GB-month        | ~$0.060/GB-month |
| Infrequent Access | $0.025/GB-month     | N/A            | N/A                   | N/A              |
| ค่า Read/Write     | $0.01/GB (IA reads) | รวมอยู่ใน tier   | รวมอยู่ใน tier          | รวมอยู่ใน tier     |

---

#### Relational Database (RDS / Managed SQL)

**On-demand Instance Price (per hour, Single-AZ)**

| Instance Class            | AWS RDS MySQL | GCP Cloud SQL (MySQL) | Azure DB for MySQL Flex | Huawei RDS MySQL |
| ------------------------- | ------------- | --------------------- | ----------------------- | ---------------- |
| Small dev (2 vCPU / 4 GB) | $0.068        | $0.050                | $0.068                  | ~$0.060          |
| General (2 vCPU / 8 GB)   | $0.150        | $0.096                | $0.150                  | ~$0.135          |
| General (4 vCPU / 16 GB)  | $0.300        | $0.192                | $0.300                  | ~$0.270          |
| General (8 vCPU / 32 GB)  | $0.600        | $0.384                | $0.600                  | ~$0.540          |
| Memory (4 vCPU / 32 GB)   | $0.480        | $0.304                | $0.480                  | ~$0.430          |
| Memory (8 vCPU / 64 GB)   | $0.960        | $0.608                | $0.960                  | ~$0.860          |

> Multi-AZ / HA แพงกว่า Single-AZ ประมาณ 2 เท่า

**Database Storage & I/O**

|                     | AWS RDS               | GCP Cloud SQL   | Azure DB Flex   | Huawei RDS       |
| ------------------- | --------------------- | --------------- | --------------- | ---------------- |
| Storage (SSD gp3)   | $0.115/GB-month       | $0.170/GB-month | $0.115/GB-month | ~$0.100/GB-month |
| Storage (High IOPS) | $0.125/GB-month       | $0.187/GB-month | $0.127/GB-month | ~$0.120/GB-month |
| Automated Backup    | ฟรี (retention 0–7 วัน) | $0.08/GB-month  | $0.095/GB-month | ~$0.04/GB-month  |
| I/O (gp2/standard)  | $0.20/1M I/O          | included        | included        | included         |

---

#### NoSQL Database

**AWS DynamoDB**

| Billing Mode                 | Write                                | Read                                   | Storage               |
| ---------------------------- | ------------------------------------ | -------------------------------------- | --------------------- |
| On-demand                    | $1.25/1M WRU                         | $0.25/1M RRU                           | $0.25/GB-month        |
| Provisioned                  | $0.00065/WCU-hour (~$0.47/WCU-month) | $0.000065/RCU-hour (~$0.047/RCU-month) | $0.25/GB-month        |
| Global Tables (multi-region) | $1.875/1M rWCU                       | $0.375/1M rRCU                         | $0.25/GB-month/region |

**GCP Firestore**

| Operation      | ราคา                   |
| -------------- | ---------------------- |
| Write          | $0.18/100K operations  |
| Read           | $0.06/100K operations  |
| Delete         | $0.02/100K operations  |
| Storage        | $0.18/GB-month         |
| Network Egress | ตาม Data Transfer rate |

**Azure Cosmos DB**

| Billing Mode         | Throughput                       | Storage        |
| -------------------- | -------------------------------- | -------------- |
| Serverless           | $0.25/1M RU                      | $0.25/GB-month |
| Provisioned (Manual) | $0.008/100 RU/second/month       | $0.25/GB-month |
| Autoscale            | $0.012/100 RU/second/month (max) | $0.25/GB-month |

**Huawei Cloud DDS (MongoDB-compatible)**

| Instance                                | ราคา         |
| --------------------------------------- | ------------ |
| dds.mongodb.c3.medium.4 (2 vCPU / 8 GB) | ~$0.100/hour |
| dds.mongodb.c3.large.4 (4 vCPU / 16 GB) | ~$0.200/hour |

---

#### Cache (Redis / Memcached)

**On-demand Node Price (per hour)**

| Node Type          | AWS ElastiCache      | GCP Memorystore | Azure Cache for Redis | Huawei DCS |
| ------------------ | -------------------- | --------------- | --------------------- | ---------- |
| Small dev (~1 GB)  | $0.034 (t3.small)    | $0.049/GB × 1   | $0.055 (C1 Basic)     | ~$0.040    |
| Medium (6–8 GB)    | $0.068 (t3.large)    | $0.049/GB × 8   | $0.101 (C1 Standard)  | ~$0.082    |
| Production (13 GB) | $0.166 (r6g.large)   | $0.049/GB × 13  | $0.202 (C2 Standard)  | ~$0.120    |
| Large (26 GB)      | $0.332 (r6g.xlarge)  | $0.049/GB × 26  | $0.403 (C3 Standard)  | ~$0.240    |
| XLarge (52 GB)     | $0.664 (r6g.2xlarge) | $0.049/GB × 52  | $0.806 (C4 Standard)  | ~$0.480    |

---

#### Message Queue

|                     | AWS SQS           | GCP Cloud Tasks     | Azure Service Bus (Standard) | Huawei DMS RocketMQ      |
| ------------------- | ----------------- | ------------------- | ---------------------------- | ------------------------ |
| Standard Queue      | $0.40/1M requests | $0.40/1M operations | $10/month + $0.013/1M        | ~$0.006/hour + $0.020/GB |
| FIFO Queue          | $0.50/1M requests | N/A                 | $10/month + $0.013/1M        | ~$0.008/hour             |
| Free Tier (monthly) | 1M requests ฟรี    | 1M operations ฟรี    | 10M operations ฟรี            | —                        |
| Message Size (max)  | 256 KB            | 1 MB                | 256 KB                       | 4 MB                     |
| Retention (max)     | 14 วัน             | 30 วัน               | 14 วัน                        | 3 วัน                     |

---

#### Pub/Sub

|                     | AWS SNS                | AWS EventBridge | GCP Cloud Pub/Sub  | Azure Event Grid    | Huawei DMS |
| ------------------- | ---------------------- | --------------- | ------------------ | ------------------- | ---------- |
| ราคา                | $0.50/1M notifications | $1.00/1M events | $0.04/GB processed | $0.60/1M operations | ~$0.020/GB |
| HTTP/HTTPS delivery | $0.60/1M               | —               | รวมใน data         | —                   | —          |
| Email delivery      | $2.00/1M               | —               | N/A                | —                   | —          |
| Free Tier (monthly) | 1M ฟรี                  | —               | 10 GB ฟรี           | 100K ฟรี             | —          |

---

#### Event Streaming (Kafka / Kinesis)

**AWS**

| Service                      | ราคา                                      |
| ---------------------------- | ----------------------------------------- |
| MSK Broker (kafka.m5.large)  | $0.210/hour/broker                        |
| MSK Broker (kafka.m5.xlarge) | $0.420/hour/broker                        |
| MSK Storage                  | $0.100/GB-month                           |
| Kinesis Data Streams         | $0.015/shard-hour + $0.014/1M PUT records |
| Kinesis Firehose             | $0.029/GB ingested                        |

**GCP**

| Service                          | ราคา               |
| -------------------------------- | ------------------ |
| Managed Kafka (small)            | ~$0.20/hour/broker |
| Cloud Pub/Sub (Kafka-compatible) | $0.04/GB processed |

**Azure**

| Service                    | ราคา                              |
| -------------------------- | --------------------------------- |
| Event Hubs Standard (1 TU) | $0.015/TU-hour + $0.028/1M events |
| Event Hubs Premium (1 PU)  | $0.927/PU-hour                    |
| Event Hubs Dedicated       | $6.617/CU-hour                    |

**Huawei Cloud**

| Service           | ราคา                |
| ----------------- | ------------------- |
| DMS Kafka (small) | ~$0.120/hour/broker |
| DMS Kafka Storage | ~$0.05/GB-month     |

---

#### Security — WAF

|                    | AWS WAF                       | GCP Cloud Armor             | Azure WAF (App Gateway WAF v2) | Huawei WAF         |
| ------------------ | ----------------------------- | --------------------------- | ------------------------------ | ------------------ |
| Web ACL / Policy   | $5.00/month                   | $5.00/policy/month          | รวมใน Gateway fee              | ~$30/month (basic) |
| Rule               | $1.00/rule/month              | —                           | รวมใน Gateway fee              | รวมใน plan         |
| Request            | $0.60/1M requests             | $0.75/1M requests evaluated | รวมใน CU charge                | ~$0.30/1M          |
| Managed Rule Group | $20/group/month               | —                           | —                              | —                  |
| Bot Control        | $10/month + $1.00/1M requests | $1.00/1M requests           | $0.004/1K requests             | —                  |

---

#### Security — DDoS Protection

|          | AWS Shield Standard | AWS Shield Advanced | GCP Cloud Armor   | Azure DDoS Basic | Azure DDoS Standard        | Huawei Anti-DDoS |
| -------- | ------------------- | ------------------- | ----------------- | ---------------- | -------------------------- | ---------------- |
| ราคา     | ฟรี (auto)           | $3,000/month        | รวมใน Cloud Armor | ฟรี (auto)        | $2,944/month/protected VIP | ~$1,500/month    |
| Coverage | L3/L4 auto          | L3/L4/L7 + DRT      | L3/L4/L7          | L3/L4 basic      | L3/L4/L7 + SLA             | L3/L4            |

---

#### Identity & Access Management (IAM)

| Service                                      | AWS | GCP                 | Azure              | Huawei Cloud |
| -------------------------------------------- | --- | ------------------- | ------------------ | ------------ |
| IAM Users/Roles/Policies                     | ฟรี  | ฟรี                  | ฟรี                 | ฟรี           |
| IAM Identity Center (SSO)                    | ฟรี  | ฟรี (Cloud Identity) | ฟรี (Entra ID Free) | ฟรี           |
| MFA                                          | ฟรี  | ฟรี                  | ฟรี                 | ฟรี           |
| Azure Entra ID P1 (Conditional Access, MFA)  | N/A | N/A                 | $6/user/month      | N/A          |
| Azure Entra ID P2 (PIM, Identity Protection) | N/A | N/A                 | $9/user/month      | N/A          |

---

#### Key Management Service (KMS)

|                           | AWS KMS            | GCP Cloud KMS           | Azure Key Vault (Standard) | Huawei DEW/KMS   |
| ------------------------- | ------------------ | ----------------------- | -------------------------- | ---------------- |
| Customer Managed Key      | $1.00/key/month    | $0.06/key version/month | $0.03/10K transactions     | ~$1.00/key/month |
| AWS Managed Key           | ฟรี                 | N/A                     | N/A                        | N/A              |
| Cryptographic Operations  | $0.03/10K requests | $0.03/10K operations    | $0.03/10K transactions     | ~$0.03/10K       |
| Asymmetric Key Operations | $0.15/10K requests | $0.06/10K               | $0.15/10K                  | ~$0.10/10K       |

---

#### Secrets Manager

|                           | AWS Secrets Manager       | GCP Secret Manager          | Azure Key Vault (Secrets) | Huawei DEW Secrets  |
| ------------------------- | ------------------------- | --------------------------- | ------------------------- | ------------------- |
| Secret Storage            | $0.40/secret/month        | $0.06/secret version/month  | $0.03/10K transactions    | ~$0.10/secret/month |
| API Call                  | $0.05/10K API calls       | $0.03/10K access operations | รวมใน transactions        | ~$0.03/10K          |
| Cross-Account Replication | $0.40/secret/month/Region | N/A                         | N/A                       | N/A                 |

---

#### Monitoring & Observability

|                   | AWS CloudWatch                       | GCP Cloud Monitoring                | Azure Monitor          | Huawei Cloud Eye    |
| ----------------- | ------------------------------------ | ----------------------------------- | ---------------------- | ------------------- |
| Custom Metric     | $0.30/metric/month (first 10K)       | $0.18/metric/month (after 150 free) | $0.25/metric/month     | ~$0.20/metric/month |
| Dashboard         | $3.00/dashboard/month (after 3 free) | ฟรี                                  | ฟรี                     | ฟรี                  |
| Alarm             | $0.10/alarm/month                    | ฟรี (first 5)                        | $0.10/alert rule/month | ~$0.05/alarm/month  |
| GetMetricData API | $0.01/1K metrics requested           | ฟรี                                  | ฟรี                     | ฟรี                  |

---

#### Logging

|                  | AWS CloudWatch Logs | GCP Cloud Logging           | Azure Monitor Logs      | Huawei LTS      |
| ---------------- | ------------------- | --------------------------- | ----------------------- | --------------- |
| Log Ingestion    | $0.50/GB            | $0.01/GB (after 50 GB free) | $2.30/GB                | ~$0.04/GB       |
| Log Storage      | $0.03/GB-month      | $0.01/GB-month              | รวมใน ingestion (90 วัน) | ~$0.02/GB-month |
| Log Query        | $0.005/GB scanned   | $0.01/GB scanned            | รวมใน workspace fee     | ~$0.003/GB      |
| Retention (free) | N/A                 | 30 วันสำหรับ _Default bucket   | 31 วัน                   | 7 วัน            |

---

#### Tracing

|                     | AWS X-Ray                 | GCP Cloud Trace                  | Azure Application Insights | Huawei APM           |
| ------------------- | ------------------------- | -------------------------------- | -------------------------- | -------------------- |
| Trace Recording     | $5.00/1M traces           | $0.20/1M spans (after 2.5M free) | $2.30/GB ingested          | ~$2.00/month (basic) |
| Trace Retrieval     | $0.50/1M traces retrieved | ฟรี                               | รวมใน ingestion            | รวมใน plan           |
| Free Tier (monthly) | 100K traces ฟรี            | 2.5M spans ฟรี                    | 5 GB ฟรี                    | —                    |

---

#### DevOps & CI/CD

|                     | AWS CodeBuild           | AWS CodePipeline            | GCP Cloud Build          | Azure Pipelines                 | Huawei CodeArts    |
| ------------------- | ----------------------- | --------------------------- | ------------------------ | ------------------------------- | ------------------ |
| Build/minute        | $0.005 (general1.small) | —                           | $0.003/build-minute      | $0.008/minute (after free tier) | รวมใน plan         |
| Pipeline            | —                       | $1.00/active pipeline/month | —                        | ฟรี (1 parallel job)             | รวมใน plan         |
| Free Tier (monthly) | 100 build-minutes ฟรี    | 1 pipeline ฟรี               | 120 build-minutes/day ฟรี | 1,800 minutes ฟรี                | —                  |
| Subscription Plan   | —                       | —                           | —                        | —                               | ~$15/month (basic) |

---

#### Container Registry

|                                    | AWS ECR        | GCP Artifact Registry | Azure Container Registry (Basic) | Huawei SWR      |
| ---------------------------------- | -------------- | --------------------- | -------------------------------- | --------------- |
| Storage                            | $0.10/GB-month | $0.10/GB-month        | $0.10/GB-day (~$3/GB-month)      | ~$0.05/GB-month |
| Data Transfer (pull, same region)  | ฟรี             | ฟรี                    | ฟรี                               | ฟรี              |
| Data Transfer (pull, cross-region) | $0.09/GB       | $0.08/GB              | $0.087/GB                        | ~$0.07/GB       |
| Registry Fee                       | ฟรี             | ฟรี                    | $0.167/day (~$5/month)           | ฟรี              |

---

#### Backup & Disaster Recovery

|                                    | AWS Backup           | GCP Cloud Backup | Azure Backup    | Huawei CBR      |
| ---------------------------------- | -------------------- | ---------------- | --------------- | --------------- |
| EBS / Disk Snapshot (warm)         | $0.05/GB-month       | $0.026/GB-month  | $0.05/GB-month  | ~$0.04/GB-month |
| RDS Backup (beyond free retention) | $0.095/GB-month      | $0.08/GB-month   | $0.095/GB-month | ~$0.04/GB-month |
| S3 / Object Backup                 | $0.05/GB-month       | included         | $0.02/GB-month  | ~$0.03/GB-month |
| Cross-Region Copy                  | $0.02/GB transferred | $0.02/GB         | $0.02/GB        | ~$0.015/GB      |

---

#### CDN & Edge

|                            | AWS CloudFront                    | GCP Cloud CDN     | Azure Front Door (Standard) | Huawei CDN |
| -------------------------- | --------------------------------- | ----------------- | --------------------------- | ---------- |
| Egress (first 10 TB/month) | $0.0085/GB                        | $0.0080/GB        | $0.0075/GB                  | $0.0070/GB |
| Egress (next 40 TB/month)  | $0.0080/GB                        | $0.0060/GB        | $0.0060/GB                  | $0.0060/GB |
| HTTP Request (per 10K)     | $0.0100                           | $0.0075           | $0.0090                     | $0.0080    |
| HTTPS Request (per 10K)    | $0.0100                           | $0.0090           | รวมใน request               | $0.0090    |
| Cache Invalidation         | $0.005/path (after 1K free/month) | ฟรี                | ฟรี                          | ฟรี         |
| WAF add-on                 | $5/policy + $0.60/1M req          | รวมใน Cloud Armor | รวมใน WAF rule              | แยกซื้อ      |

---

#### DNS

|                         | AWS Route 53                | GCP Cloud DNS    | Azure DNS        | Huawei DNS        |
| ----------------------- | --------------------------- | ---------------- | ---------------- | ----------------- |
| Hosted Zone             | $0.50/zone/month (first 25) | $0.20/zone/month | $0.90/zone/month | ~$0.60/zone/month |
| DNS Query (Standard)    | $0.40/1M queries            | $0.40/1M queries | $0.40/1M queries | ~$0.30/1M queries |
| DNS Query (Latency/Geo) | $0.70/1M queries            | $0.70/1M queries | $0.70/1M queries | —                 |
| Health Check            | $0.50/check/month           | —                | —                | —                 |

---

#### Search

|                                     | AWS OpenSearch  | GCP (Elastic on Marketplace) | Azure AI Search          | Huawei CSS       |
| ----------------------------------- | --------------- | ---------------------------- | ------------------------ | ---------------- |
| Small dev (t3.small.search)         | $0.036/hour     | varies                       | —                        | —                |
| General (m6g.large.search / 8 GB)   | $0.148/hour     | ~$0.150/hour                 | $245/month (Standard S1) | ~$0.120/hour     |
| General (m6g.xlarge.search / 16 GB) | $0.296/hour     | ~$0.300/hour                 | $245/month (S1, 25 GB)   | ~$0.240/hour     |
| Memory (r6g.large.search / 16 GB)   | $0.219/hour     | ~$0.250/hour                 | $981/month (Standard S2) | ~$0.200/hour     |
| Storage                             | $0.135/GB-month | ~$0.100/GB-month             | รวมใน tier               | ~$0.100/GB-month |
| Free Tier                           | —               | —                            | Basic ~$73/month         | —                |

---

#### Cost Management

|                                    | AWS Cost Explorer     | GCP Cost Management | Azure Cost Management | Huawei Cost Center |
| ---------------------------------- | --------------------- | ------------------- | --------------------- | ------------------ |
| Cost Visibility                    | ฟรี                    | ฟรี                  | ฟรี                    | ฟรี                 |
| Budget Alert                       | ฟรี                    | ฟรี                  | ฟรี                    | ฟรี                 |
| Advanced Query (Cost Explorer API) | $0.01/paginated query | ฟรี                  | ฟรี                    | ฟรี                 |
| Savings Plans Recommendations      | ฟรี                    | ฟรี                  | ฟรี                    | ฟรี                 |
| Anomaly Detection                  | ฟรี                    | ฟรี                  | ฟรี                    | ฟรี                 |

---

### Best Practice

- ใช้ On-demand ใน dev/test เพราะ environment เหล่านี้ไม่ได้ run ตลอด
- ติดตาม usage ด้วย Cost Explorer และ Budget Alert ตั้งแต่วันแรก
- มองข้าม data transfer cost ไม่ได้ โดยเฉพาะระบบที่มี cross-region หรือ egress สูง
- หลังจาก run production 2–3 เดือน evaluate ว่าคุ้มค่าไหมที่จะเปลี่ยนเป็น Reserved

### Common Mistakes

- ใช้ On-demand กับ production instance ที่ run 24/7 เป็นปี โดยไม่ได้ evaluate Reserved
- มองข้าม data transfer cost จนเกิด bill shock
- ไม่ได้ตั้ง Budget Alert ทำให้ไม่รู้ว่า usage พุ่งขึ้น

---

## Reserved / Committed Use

### คืออะไร

รูปแบบที่ commit ว่าจะใช้ resource ต่อเนื่อง 1 ปี หรือ 3 ปี แลกกับส่วนลด 35–72% เมื่อเทียบกับ On-demand

### เหมาะกับงานแบบไหน

production workload ที่ run ตลอด 24/7 อย่างน้อย 1 ปี

### ไม่เหมาะกับงานแบบไหน

dev/test environment, workload ที่ยังไม่แน่ใจ pattern หรือ project ที่อาจ shutdown ก่อน commitment หมด

### ตัวอย่างราคาต่อหน่วย (Reserved vs On-demand)

---

#### AWS EC2 Reserved Instance (us-east-1)

| Instance Type | On-demand/hr | 1yr No Upfront/hr | 1yr All Upfront/hr | 3yr All Upfront/hr | ประหยัด (3yr) |
| ------------- | -----------: | ----------------: | -----------------: | -----------------: | -----------: |
| `t3.medium`   |      $0.0416 |            $0.028 |             $0.026 |             $0.018 |         ~57% |
| `m6i.large`   |       $0.096 |            $0.065 |             $0.060 |             $0.042 |         ~56% |
| `m6i.xlarge`  |       $0.192 |            $0.131 |             $0.121 |             $0.085 |         ~56% |
| `m6i.2xlarge` |       $0.384 |            $0.262 |             $0.242 |             $0.170 |         ~56% |
| `c6i.large`   |       $0.085 |            $0.057 |             $0.053 |             $0.036 |         ~58% |
| `c6i.xlarge`  |       $0.170 |            $0.114 |             $0.105 |             $0.073 |         ~57% |
| `r6i.large`   |       $0.126 |            $0.086 |             $0.079 |             $0.055 |         ~56% |
| `r6i.xlarge`  |       $0.252 |            $0.171 |             $0.158 |             $0.110 |         ~56% |

#### AWS Savings Plans (us-east-1)

| Type                       | Flexibility                                     | ส่วนลดสูงสุด (1yr) | ส่วนลดสูงสุด (3yr) |
| -------------------------- | ----------------------------------------------- | --------------: | --------------: |
| Compute Savings Plans      | EC2 ทุก family + Lambda + Fargate                |            ~66% |            ~66% |
| EC2 Instance Savings Plans | EC2 ใน instance family ที่ commit (ยืดหยุ่น size/OS) |            ~72% |            ~72% |

#### AWS RDS Reserved (us-east-1, Single-AZ)

| Instance Class (MySQL/PostgreSQL) | On-demand/hr | 1yr All Upfront/hr | 3yr All Upfront/hr | ประหยัด (3yr) |
| --------------------------------- | -----------: | -----------------: | -----------------: | -----------: |
| `db.t3.medium` (2/4 GB)           |       $0.068 |             $0.046 |             $0.033 |         ~51% |
| `db.m6g.large` (2/8 GB)           |       $0.150 |             $0.100 |             $0.072 |         ~52% |
| `db.m6g.xlarge` (4/16 GB)         |       $0.300 |             $0.200 |             $0.144 |         ~52% |
| `db.m6g.2xlarge` (8/32 GB)        |       $0.600 |             $0.400 |             $0.288 |         ~52% |
| `db.r6g.large` (2/16 GB)          |       $0.240 |             $0.160 |             $0.115 |         ~52% |
| `db.r6g.xlarge` (4/32 GB)         |       $0.480 |             $0.320 |             $0.230 |         ~52% |

> Multi-AZ แพงกว่า Single-AZ ประมาณ 2 เท่า

#### AWS ElastiCache Reserved (us-east-1)

| Node Type                   | On-demand/hr | 1yr All Upfront/hr | 3yr All Upfront/hr | ประหยัด (3yr) |
| --------------------------- | -----------: | -----------------: | -----------------: | -----------: |
| `cache.t3.medium` (3 GB)    |       $0.068 |             $0.046 |             $0.033 |         ~51% |
| `cache.r6g.large` (13 GB)   |       $0.166 |             $0.112 |             $0.080 |         ~52% |
| `cache.r6g.xlarge` (26 GB)  |       $0.332 |             $0.224 |             $0.160 |         ~52% |
| `cache.r6g.2xlarge` (52 GB) |       $0.664 |             $0.448 |             $0.320 |         ~52% |

#### AWS OpenSearch Reserved (us-east-1)

| Instance Type                 | On-demand/hr | 1yr All Upfront/hr | 3yr All Upfront/hr | ประหยัด (3yr) |
| ----------------------------- | -----------: | -----------------: | -----------------: | -----------: |
| `m6g.large.search` (2/8 GB)   |       $0.148 |             $0.100 |             $0.071 |         ~52% |
| `m6g.xlarge.search` (4/16 GB) |       $0.296 |             $0.200 |             $0.143 |         ~52% |
| `r6g.large.search` (2/16 GB)  |       $0.219 |             $0.148 |             $0.106 |         ~52% |

#### GCP Committed Use Discount (us-central1)

| Machine Type            | On-demand/hr | 1yr CUD/hr | 3yr CUD/hr | ประหยัด (3yr) |
| ----------------------- | -----------: | ---------: | ---------: | -----------: |
| `n2-standard-2`         |       $0.097 |     $0.067 |     $0.048 |         ~51% |
| `n2-standard-4`         |       $0.194 |     $0.134 |     $0.096 |         ~51% |
| `n2-standard-8`         |       $0.388 |     $0.268 |     $0.192 |         ~51% |
| `c2-standard-4`         |       $0.209 |     $0.149 |     $0.107 |         ~49% |
| `n2-highmem-4`          |       $0.262 |     $0.181 |     $0.130 |         ~50% |
| Cloud SQL n1-standard-2 |       $0.096 |     $0.067 |     $0.048 |         ~50% |
| Cloud SQL n1-standard-4 |       $0.192 |     $0.134 |     $0.096 |         ~50% |

#### Azure Reserved VM Instance (East US)

| VM Size  | Pay-as-you-go/hr | 1yr Reserved/hr | 3yr Reserved/hr | ประหยัด (3yr) |
| -------- | ---------------: | --------------: | --------------: | -----------: |
| `B2s`    |          $0.0416 |          $0.028 |          $0.018 |         ~57% |
| `D2s v5` |           $0.096 |          $0.061 |          $0.043 |         ~55% |
| `D4s v5` |           $0.192 |          $0.122 |          $0.086 |         ~55% |
| `D8s v5` |           $0.384 |          $0.244 |          $0.172 |         ~55% |
| `E2s v5` |           $0.126 |          $0.080 |          $0.056 |         ~56% |
| `E4s v5` |           $0.252 |          $0.160 |          $0.112 |         ~56% |

#### Huawei Cloud Reserved Instance (AP-Southeast)

| Flavor                  | Pay-per-use/hr | 1yr Reserved/hr | ประหยัด |
| ----------------------- | -------------: | --------------: | -----: |
| `s6.large.2` (2/4 GB)   |        ~$0.040 |         ~$0.026 |   ~35% |
| `c6.large.4` (2/8 GB)   |        ~$0.085 |         ~$0.055 |   ~35% |
| `c6.xlarge.4` (4/16 GB) |        ~$0.170 |         ~$0.110 |   ~35% |
| `m6.large.8` (2/16 GB)  |        ~$0.120 |         ~$0.078 |   ~35% |
| RDS MySQL c6.large.4    |        ~$0.135 |         ~$0.088 |   ~35% |

### Best Practice

- ซื้อ Savings Plans แทน Reserved Instance เมื่อเป็นไปได้เพราะยืดหยุ่นกว่า
- commit เฉพาะ baseline capacity อย่า commit รวม peak
- review utilization ทุกไตรมาสว่า Reserved ที่ซื้อถูกใช้งานอยู่ไหม (target ≥ 80%)
- ใช้ 1yr term ก่อนถ้าไม่แน่ใจ อย่า commit 3yr ตั้งแต่ต้น

### Common Mistakes

- commit เป็น specific instance type แล้ว type นั้น end-of-life
- commit เกินกว่า baseline จริง ทำให้จ่ายค่า Reserved ที่ไม่ได้ใช้
- ลืม renew ก่อน expiry ทำให้ราคาเด้งกลับเป็น On-demand

---

## Subscription

### คืออะไร

รูปแบบจ่ายแบบรายเดือนหรือรายปีในแบบ package หรือ plan ที่กำหนด feature และขีดจำกัดชัดเจน ไม่ผันตาม usage มากนัก

### เหมาะกับงานแบบไหน

Support plan, security service, managed SLA, compliance tool, managed Kubernetes cluster fee

### ไม่เหมาะกับงานแบบไหน

workload ที่ usage ผันผวนมาก เพราะ subscription จ่ายแม้จะ idle

### ตัวอย่างราคาต่อหน่วย (Subscription)

#### Support Plan

| Provider | Plan                | ราคา                                                | SLA หลัก                        |
| -------- | ------------------- | --------------------------------------------------- | ------------------------------ |
| AWS      | Developer           | max($29, 3% of bill)/month                          | Email, business hours          |
| AWS      | Business            | max(10%≤$10K / 7% $10K–$80K / 5% >$80K, $100)/month | 24/7, <1hr critical            |
| AWS      | Enterprise On-Ramp  | max(10% of bill, $5,500)/month                      | TAM pool, <30min critical      |
| AWS      | Enterprise          | max(10% of bill, $15,000)/month                     | Dedicated TAM, <15min critical |
| GCP      | Standard            | $150/month min + % of spend                         | Business hours                 |
| GCP      | Enhanced            | $500/month min + % of spend                         | 24/7, <1hr critical            |
| GCP      | Premium             | $12,500/month min + % of spend                      | Dedicated TAM, <15min critical |
| Azure    | Developer           | $29/month                                           | Email, business hours          |
| Azure    | Standard            | $300/month                                          | 24/7, <2hr critical            |
| Azure    | Professional Direct | $1,000/month                                        | <1hr critical                  |
| Huawei   | Business Support    | ~$200/month                                         | 24/7                           |

#### Managed Kubernetes Cluster Fee

| Provider | Service       | ราคา                                    |
| -------- | ------------- | --------------------------------------- |
| AWS      | EKS Cluster   | $0.10/hour (~$73/month) per cluster     |
| GCP      | GKE Standard  | $0.10/hour (ฟรีสำหรับ 1 zonal cluster แรก) |
| GCP      | GKE Autopilot | ฟรี — จ่ายเฉพาะ pod resource              |
| Azure    | AKS           | ฟรี — จ่ายเฉพาะ VM/resource               |
| Huawei   | CCE           | ~$0.05/hour (~$36/month) per cluster    |

#### DDoS Protection (Advanced Tier)

| Provider | Service                  | ราคา                              |
| -------- | ------------------------ | --------------------------------- |
| AWS      | Shield Advanced          | $3,000/month + data transfer fees |
| GCP      | Cloud Armor WAF+DDoS     | $5/policy/month + request fees    |
| Azure    | DDoS Protection Standard | $2,944/month per protected VIP    |
| Huawei   | Advanced Anti-DDoS       | ~$1,500/month                     |

### Best Practice

- เปรียบเทียบ feature ระหว่าง plan tier ก่อน subscribe
- ประเมินว่า annual plan คุ้มกว่า monthly หรือไม่
- ติดตาม renewal date เพื่อ review ก่อนต่ออายุ

### Common Mistakes

- subscribe plan ใหญ่กว่าที่จำเป็น
- ลืม cancel subscription ของ service ที่ไม่ได้ใช้แล้ว

---

## Flat Rate

### คืออะไร

รูปแบบจ่ายเหมาราคาเดียวโดยไม่ผันตาม usage มากนัก พบใน dedicated capacity หรือ Enterprise Agreement (EA) ที่เจรจากับ Cloud Provider โดยตรง

### เหมาะกับงานแบบไหน

enterprise ที่ต้องการ dedicated capacity, compliance ที่ห้ามแชร์ hardware (PCI DSS, HIPAA), หรือ Cloud spend สูงพอจะเจรจา EA

### ไม่เหมาะกับงานแบบไหน

startup หรือ project ขนาดเล็ก เพราะ Flat Rate มักต้องการ minimum commitment สูง

### ตัวอย่างราคาต่อหน่วย (Flat Rate)

#### AWS Dedicated Host (us-east-1)

| Instance Family | ราคา Dedicated Host | instance สูงสุด  | เทียบกับ On-demand |
| --------------- | ------------------: | -------------- | ---------------- |
| `m6i`           |         $4.992/hour | 16× m6i.xlarge | คุ้มเมื่อ fill >70%  |
| `c6i`           |         $4.250/hour | 16× c6i.xlarge | คุ้มเมื่อ fill >70%  |
| `r6i`           |         $6.624/hour | 8× r6i.xlarge  | คุ้มเมื่อ fill >70%  |

> Dedicated Host เหมาะกับ BYOL license (Windows Server, Oracle) ที่ผูกกับ physical core

#### Enterprise Agreement — ประมาณการส่วนลดตาม Annual Spend

| Annual Cloud Spend | ส่วนลดโดยประมาณ | รูปแบบ                             |
| ------------------ | -------------- | --------------------------------- |
| < $100K            | 0%             | On-demand / Savings Plans         |
| $100K – $500K      | 5–10%          | Private Pricing Agreement         |
| $500K – $1M        | 10–15%         | Enterprise Discount Program (EDP) |
| $1M – $5M          | 15–25%         | EDP / Enterprise Agreement        |
| > $5M              | 25–40%+        | Custom Enterprise Agreement       |

> ส่วนลด EA จริงต้องเจรจากับ Cloud Provider โดยตรง

### Best Practice

- เจรจา EA เมื่อ Cloud spend มีนัยสำคัญและมีแผนระยะยาวชัดเจน
- ให้ฝ่าย finance และ legal ร่วม review term ก่อน sign

### Common Mistakes

- commit spend สูงเกินกว่าที่ใช้จริง
- ไม่ได้ review termination clause อย่างละเอียด

---

## Spot / Preemptible

### คืออะไร

Instance ที่ใช้ spare capacity ของ Cloud Provider ในราคาถูกกว่า On-demand 60–90% แต่สามารถถูกดึงคืนได้ตลอดเวลา (2 นาที notice บน AWS, 30 วินาทีบน GCP)

### เหมาะกับงานแบบไหน

batch processing, ML training, CI/CD build worker, rendering, Kubernetes worker node สำหรับ non-critical pod

### ไม่เหมาะกับงานแบบไหน

stateful service ที่ไม่มี checkpoint เช่น primary database หรือ workload ที่ต้องการ uptime สูง

### ตัวอย่างราคาต่อหน่วย (Spot vs On-demand)

#### AWS EC2 Spot (us-east-1, ราคาผันผวน)

| Instance Type       | On-demand/hr | Spot (ประมาณ)/hr |  ประหยัด | เหมาะกับ Spot Use Case  |
| ------------------- | -----------: | ---------------: | ------: | ---------------------- |
| `m6i.large`         |       $0.096 |    $0.029–$0.050 | ~48–70% | general batch worker   |
| `m6i.xlarge`        |       $0.192 |    $0.058–$0.100 | ~48–70% | medium batch, K8s node |
| `m6i.2xlarge`       |       $0.384 |    $0.115–$0.200 | ~48–70% | large batch workload   |
| `c6i.xlarge`        |       $0.170 |    $0.051–$0.090 | ~47–70% | compute-heavy batch    |
| `c6i.2xlarge`       |       $0.340 |    $0.102–$0.180 | ~47–70% | large compute batch    |
| `r6i.large`         |       $0.126 |    $0.038–$0.065 | ~48–70% | memory-heavy batch     |
| `g4dn.xlarge` (GPU) |       $0.526 |    $0.158–$0.250 | ~52–70% | ML training            |

#### GCP Spot VM (us-central1)

| Machine Type    | On-demand/hr | Spot/hr | ประหยัด |
| --------------- | -----------: | ------: | -----: |
| `n2-standard-2` |       $0.097 |  $0.024 |   ~75% |
| `n2-standard-4` |       $0.194 |  $0.048 |   ~75% |
| `n2-standard-8` |       $0.388 |  $0.097 |   ~75% |
| `c2-standard-4` |       $0.209 |  $0.049 |   ~77% |
| `n2-highmem-4`  |       $0.262 |  $0.065 |   ~75% |

#### Azure Spot VM (East US)

| VM Size  | Pay-as-you-go/hr | Spot (ประมาณ)/hr |  ประหยัด |
| -------- | ---------------: | ---------------: | ------: |
| `D2s v5` |           $0.096 |    $0.015–$0.034 | ~65–84% |
| `D4s v5` |           $0.192 |    $0.030–$0.068 | ~65–84% |
| `D8s v5` |           $0.384 |    $0.058–$0.136 | ~65–85% |
| `F4s v2` |           $0.169 |    $0.025–$0.060 | ~65–85% |

#### Huawei Cloud Spot ECS (AP-Southeast)

| Flavor         | Pay-per-use/hr | Spot (ประมาณ)/hr |  ประหยัด |
| -------------- | -------------: | ---------------: | ------: |
| `c6.large.4`   |        ~$0.085 |   ~$0.025–$0.040 | ~53–71% |
| `c6.xlarge.4`  |        ~$0.170 |   ~$0.050–$0.080 | ~53–71% |
| `c6.2xlarge.4` |        ~$0.340 |   ~$0.100–$0.160 | ~53–71% |

#### Interruption Rate vs Savings Strategy

| Interruption Rate | ส่วนลดโดยประมาณ | กลยุทธ์ที่แนะนำ                                      |
| ----------------- | -------------- | ----------------------------------------------- |
| Low (<5%/month)   | 60–75%         | long-running batch, ML training ยาวหลายชั่วโมง    |
| Medium (5–20%)    | 50–65%         | checkpoint บ่อย ทุก 10–30 นาที                     |
| High (>20%)       | 40–60%         | เฉพาะ short job (<1 ชั่วโมง) หรือ stateless worker |

### Best Practice

- ใช้ instance type หลายแบบใน Spot Fleet/Pool เพื่อเพิ่มโอกาสได้ capacity
- กระจาย Spot request หลาย AZ เสมอ
- ออกแบบ workload ให้ resume จาก checkpoint ได้เมื่อถูก interrupt

### Common Mistakes

- ใช้ Spot กับ single instance type และ AZ เดียว ทำให้ไม่มี fallback
- run stateful service บน Spot โดยไม่มี mechanism ย้าย state ก่อน interrupt

---

## สรุปเปรียบเทียบ Pricing Model

| Pricing Model             | ราคาเทียบ On-demand | ความยืดหยุ่น             | เหมาะกับ                                      | ไม่เหมาะกับ                    |
| ------------------------- | ------------------ | --------------------- | -------------------------------------------- | ---------------------------- |
| On-demand / Pay-as-you-go | 100% (baseline)    | สูงสุด                  | dev/test, spike traffic, project ใหม่         | steady-state 24/7 production |
| Reserved 1yr All Upfront  | ~40–55%            | ต่ำ (commit 1 ปี)        | production workload ที่ run ตลอด               | workload ที่ pattern ยังไม่นิ่ง    |
| Reserved 3yr All Upfront  | ~30–45%            | ต่ำมาก (commit 3 ปี)     | long-term stable workload                    | project ที่อาจ re-architect    |
| Savings Plans (AWS)       | ~35–55%            | ปานกลาง               | production ที่ต้องการยืดหยุ่น                      | N/A                          |
| Subscription              | Fixed monthly      | ปานกลาง               | support plan, security service, SLA          | workload usage ผันผวนมาก      |
| Flat Rate / Enterprise    | ~60–75%            | ต่ำมาก                  | enterprise large-scale spend, Dedicated Host | startup, project ขนาดเล็ก     |
| Spot / Preemptible        | ~10–40%            | สูง (แต่ interruptible) | batch, ML training, CI/CD worker             | stateful production service  |
