---
type: Solution Architecture's Pricing
description: A solution to the pricing exercise for the e-commerce platform.
status: proposed
updated: 2026-06-08
date: 2026-06-08
---

# Pricing Solutions

> **สมมติฐานการประมาณราคา:** ราคาทั้งหมดเป็นราคาประมาณการแบบ on-demand (USD/เดือน) อ้างอิงจาก public pricing ของผู้ให้บริการแต่ละราย ภูมิภาค Asia Pacific (Singapore / Southeast Asia ใกล้เคียง) ณ กลางปี 2026 โดยคำนวณ
>
> **1 เดือน = 730 ชั่วโมง**
>
> ตัวเลขจริงอาจแตกต่างตาม discount, reserved/savings plan, data transfer และ usage pattern จริง สำหรับ Huawei Cloud ซึ่งมีข้อมูลราคาเปิดเผยต่อสาธารณะค่อนข้างจำกัด ตัวเลขถูกประมาณจาก spec ที่เทียบเคียงได้กับผู้ให้บริการรายอื่นและรูปแบบราคาทั่วไปของภูมิภาค APAC

## Environment & Infrastructure

`03-System-Architecture.md` กำหนด 5 environments (Development, SIT/QA, UAT, Staging, Production) ตารางหลักของเอกสารนี้ยึดสมมติฐาน **"1 ชุด infra ต่อ tier"** ส่วนนี้นำเสนอมุมมองเสริมสำหรับกรณีที่ต้องการแยกงบประมาณตามกลุ่ม environment

### Environment Groups

| Environment Group | Environments          | Multiplier | หลักการ                                          |
| ----------------- | --------------------- | ---------: | ----------------------------------------------- |
| Dev + SIT/QA      | Development, SIT / QA |       ×0.3 | ขนาดเล็กสุด รองรับการพัฒนาและ integration test      |
| UAT + Staging     | UAT, Staging          |       ×0.6 | topology ใกล้เคียง Production แต่ traffic น้อยกว่า   |
| Production        | Production            |       ×1.0 | ขนาดเต็มตาม tier แยก dedicated                   |
| **Total**         |                       |   **×1.9** | เทียบกับ ×5.0 หากแยกเต็มชุดทุก environment (~62% ลด) |

### Cost by Tier (AWS baseline)

**Minimum** — Production only (baseline $986.00)

| Environment Group | Environments | Multiplier | Estimated Cost |
| ----------------- | ------------ | ---------: | -------------: |
| Production        | Production   |       ×1.0 |      ≈ $986.00 |
| **Total**         |              |   **×1.0** |  **≈ $986.00** |

**Recommended and Scaling** — Production + UAT/Staging (baseline $3,244.82)

| Environment Group | Environments             | Multiplier |  Estimated Cost |
| ----------------- | ------------------------ | ---------: | --------------: |
| UAT + Staging     | UAT, Staging             |       ×0.6 |     ≈ $1,946.89 |
| Production        | Production               |       ×1.0 |     ≈ $3,244.82 |
| **Total**         | UAT, Staging, Production |   **×1.6** | **≈ $5,191.71** |

**Long-term** — All environments (baseline $13,761.42)

| Environment Group | Environments          | Multiplier |   Estimated Cost |
| ----------------- | --------------------- | ---------: | ---------------: |
| Dev + SIT/QA      | Development, SIT / QA |       ×0.3 |      ≈ $4,128.43 |
| UAT + Staging     | UAT, Staging          |       ×0.6 |      ≈ $8,256.85 |
| Production        | Production            |       ×1.0 |     ≈ $13,761.42 |
| **Total**         | All 5 environments    |   **×1.9** | **≈ $26,146.70** |

## Cloud: Amazon Web Services (AWS)

AWS คือผู้ให้บริการหลักที่ระบุไว้ใน `03-System-Architecture.md`:

- **Minimum**: ตั้งต้นได้ทันทีด้วยบริการ managed ครบทุกตัวในขนาดเล็กสุด ไม่มี service ที่ต้อง self-manage จึงลดภาระทีม ops ตั้งแต่วันแรก
- **Recommended and Scaling**: ฟีเจอร์ HA (RDS Multi-AZ, ElastiCache replica, OpenSearch multi-node) และ HPA บน EKS ตรงกับ "High Availability & Scalability" ที่ระบุไว้ในสถาปัตยกรรมพอดี ทำให้ scale ได้โดยไม่ต้องเปลี่ยนแพลตฟอร์ม
- **Long-term**: รองรับ Multi-Region Replication, dedicated master node สำหรับ OpenSearch และ MSK ขนาดใหญ่ได้ในระบบนิเวศเดียวกัน ตรงกับแผน Disaster Recovery ที่วางไว้ และมี ecosystem (observability, IAM, IRSA) ที่ทีมเลือกไว้แล้วรองรับ scale ระดับองค์กร

### Minimum

| Service                       | Instance Name    | Specification                 | Quantity |   Unit Price |           Total Price |
| ----------------------------- | ---------------- | ----------------------------- | -------: | -----------: | --------------------: |
| Amazon EKS                    | EKS Cluster      | Managed control plane         |        1 |       $73.00 |                $73.00 |
| Amazon EC2 (Worker Nodes)     | m6i.large        | 2 vCPU, 8 GB RAM              |        3 |       $70.08 |               $210.24 |
| Amazon RDS for PostgreSQL     | db.t4g.medium    | 2 vCPU, 4 GB, Single-AZ       |        1 |       $60.00 |                $60.00 |
| Amazon ElastiCache for Redis  | cache.t4g.medium | 2 vCPU, 3.09 GB               |        1 |       $48.00 |                $48.00 |
| Amazon OpenSearch Service     | t3.medium.search | 2 vCPU, 4 GB, 1 node          |        1 |       $53.29 |                $53.29 |
| Amazon MSK (Apache Kafka)     | kafka.m5.large   | 2 vCPU, 8 GB, broker (min. 3) |        3 |      $153.00 |               $459.00 |
| AWS Application Load Balancer | ALB              | 1 load balancer + base LCU    |        1 |       $35.00 |                $35.00 |
| NAT Gateway                   | NAT Gateway      | Single-AZ                     |        1 |       $32.85 |                $32.85 |
| Amazon S3                     | Standard Storage | ~200 GB                       |      200 |    $0.023/GB |                 $4.60 |
| AWS Secrets Manager           | Secret           | ~25 secrets/keys              |       25 | $0.40/secret |                $10.00 |
|                               |                  |                               |          |    **Total** | **≈ $986.00 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                           |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ---------------------------------------------------------------------------------- |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |   ≈ $1,873.40 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                   |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |   ≈ $1,577.60 | ลดลง ≈ $295.80 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                   |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |     ≈ $986.00 | ลดลงอีก ≈ $591.60 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                         | Instance Name         | Specification                                                         | Quantity | Unit Price |          Total Price |
| ------------------------------- | --------------------- | --------------------------------------------------------------------- | -------: | ---------: | -------------------: |
| Amazon EC2 (Observability Node) | m6i.large             | 2 vCPU, 8 GB — runs OTel Collector, Prometheus, Grafana, Loki, Jaeger |        1 |     $70.08 |               $70.08 |
| Amazon EBS (gp3)                | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~100 GB)    |      100 |   $0.08/GB |                $8.00 |
| Amazon S3                       | Log & Trace Retention | Loki chunks, trace retention (~200 GB)                                |      200 |  $0.023/GB |                $4.60 |
|                                 |                       |                                                                       |          |  **Total** | **≈ $82.68 / month** |

**ตัวเลือก B — Cloud Managed (AWS)**

| Service                               | Instance Name        | Specification                                        | Quantity | Unit Price |           Total Price |
| ------------------------------------- | -------------------- | ---------------------------------------------------- | -------: | ---------: | --------------------: |
| AWS Distro for OpenTelemetry (ADOT)   | Collector Deployment | Telemetry pipeline สำหรับ metrics, logs, traces บน EKS |        1 |      $0.00 |                 $0.00 |
| Amazon Managed Service for Prometheus | Metrics Backend      | Metrics ingestion, query และ storage (scale เล็ก)     |        1 |     $75.00 |                $75.00 |
| Amazon Managed Grafana                | Dashboard            | Dashboard, alerting สำหรับทีม ~5 users                  |        5 | $9.00/user |                $45.00 |
| Amazon CloudWatch Logs                | Log Aggregation      | Centralized application logs ปริมาณเล็ก                |        1 |     $50.00 |                $50.00 |
| AWS X-Ray                             | Distributed Tracing  | Distributed tracing สำหรับ request flow สำคัญ            |        1 |     $10.00 |                $10.00 |
|                                       |                      |                                                      |          |  **Total** | **≈ $180.00 / month** |

### Recommended and Scaling

| Service                       | Instance Name    | Specification                          | Quantity |   Unit Price |             Total Price |
| ----------------------------- | ---------------- | -------------------------------------- | -------: | -----------: | ----------------------: |
| Amazon EKS                    | EKS Cluster      | Managed control plane                  |        1 |       $73.00 |                  $73.00 |
| Amazon EC2 (Worker Nodes)     | m6i.xlarge       | 4 vCPU, 16 GB RAM                      |        6 |      $140.16 |                 $840.96 |
| Amazon RDS for PostgreSQL     | db.r6g.large     | 2 vCPU, 16 GB, Multi-AZ + Read Replica |        2 |      $248.20 |                 $496.40 |
| Amazon ElastiCache for Redis  | cache.r6g.large  | 2 vCPU, 13 GB, primary + replica       |        2 |      $150.38 |                 $300.76 |
| Amazon OpenSearch Service     | r6g.large.search | 2 vCPU, 16 GB, 3-node cluster          |        3 |      $140.00 |                 $420.00 |
| Amazon MSK (Apache Kafka)     | kafka.m5.xlarge  | 4 vCPU, 16 GB, broker                  |        3 |      $306.00 |                 $918.00 |
| AWS Application Load Balancer | ALB              | Higher throughput + LCU                |        1 |       $60.00 |                  $60.00 |
| NAT Gateway                   | NAT Gateway      | Multi-AZ                               |        2 |       $32.85 |                  $65.70 |
| Amazon S3                     | Standard Storage | ~2,000 GB                              |    2,000 |    $0.023/GB |                  $46.00 |
| AWS Secrets Manager           | Secret           | ~60 secrets/keys                       |       60 | $0.40/secret |                  $24.00 |
|                               |                  |                                        |          |    **Total** | **≈ $3,244.82 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                             |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ------------------------------------------------------------------------------------ |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |   ≈ $6,165.16 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                     |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |   ≈ $5,191.71 | ลดลง ≈ $973.45 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                     |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |   ≈ $3,244.82 | ลดลงอีก ≈ $1,946.89 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                         | Instance Name         | Specification                                                         | Quantity | Unit Price |           Total Price |
| ------------------------------- | --------------------- | --------------------------------------------------------------------- | -------: | ---------: | --------------------: |
| Amazon EC2 (Observability Node) | m6i.large             | 2 vCPU, 8 GB — runs OTel Collector, Prometheus, Grafana, Loki, Jaeger |        1 |     $70.08 |                $70.08 |
| Amazon EBS (gp3)                | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~200 GB)    |      200 |   $0.08/GB |                $16.00 |
| Amazon S3                       | Log & Trace Retention | Loki chunks, trace retention (~1,000 GB)                              |    1,000 |  $0.023/GB |                $23.00 |
|                                 |                       |                                                                       |          |  **Total** | **≈ $109.08 / month** |

**ตัวเลือก B — Cloud Managed (AWS)**

| Service                               | Instance Name        | Specification                                                 | Quantity | Unit Price |           Total Price |
| ------------------------------------- | -------------------- | ------------------------------------------------------------- | -------: | ---------: | --------------------: |
| AWS Distro for OpenTelemetry (ADOT)   | Collector Deployment | Telemetry pipeline สำหรับ metrics, logs, traces บน EKS          |        1 |      $0.00 |                 $0.00 |
| Amazon Managed Service for Prometheus | Metrics Backend      | Metrics ingestion, query และ storage สำหรับ production workload |        1 |     $75.00 |                $75.00 |
| Amazon Managed Grafana                | Dashboard            | Dashboard, alerting สำหรับทีม ~5 users                           |        5 | $9.00/user |                $45.00 |
| Amazon CloudWatch Logs                | Log Aggregation      | Centralized application logs ปริมาณกลาง                        |        1 |     $50.00 |                $50.00 |
| AWS X-Ray                             | Distributed Tracing  | Distributed tracing สำหรับ request flow สำคัญ                     |        1 |     $10.00 |                $10.00 |
|                                       |                      |                                                               |          |  **Total** | **≈ $180.00 / month** |

### Long-term

| Service                       | Instance Name                  | Specification                                | Quantity |   Unit Price |              Total Price |
| ----------------------------- | ------------------------------ | -------------------------------------------- | -------: | -----------: | -----------------------: |
| Amazon EKS                    | EKS Cluster                    | Managed control plane (Production + Staging) |        2 |       $73.00 |                  $146.00 |
| Amazon EC2 (Worker Nodes)     | m6i.2xlarge                    | 8 vCPU, 32 GB RAM, multi node-group          |       12 |      $280.32 |                $3,363.84 |
| Amazon RDS for PostgreSQL     | db.r6g.xlarge                  | 4 vCPU, 32 GB, Multi-AZ + Read Replicas      |        3 |      $496.00 |                $1,488.00 |
| Amazon ElastiCache for Redis  | cache.r6g.xlarge               | 4 vCPU, 26 GB, sharded cluster + replicas    |        6 |      $300.03 |                $1,800.18 |
| Amazon OpenSearch Service     | r6g.xlarge.search              | 4 vCPU, 32 GB, 6 data + 3 dedicated master   |        9 |      $280.00 |                $2,520.00 |
| Amazon MSK (Apache Kafka)     | kafka.m5.2xlarge               | 8 vCPU, 32 GB, broker (6-node cluster)       |        6 |      $612.00 |                $3,672.00 |
| AWS Application Load Balancer | ALB                            | Multi-region                                 |        2 |       $80.00 |                  $160.00 |
| NAT Gateway                   | NAT Gateway                    | Multi-AZ × multi-region                      |        4 |       $32.85 |                  $131.40 |
| Amazon S3                     | Standard + Intelligent-Tiering | ~20,000 GB                                   |   20,000 |    $0.021/GB |                  $420.00 |
| AWS Secrets Manager           | Secret                         | ~150 secrets/keys                            |      150 | $0.40/secret |                   $60.00 |
|                               |                                |                                              |          |    **Total** | **≈ $13,761.42 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                             |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ------------------------------------------------------------------------------------ |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |  ≈ $26,146.70 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                     |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |  ≈ $22,018.27 | ลดลง ≈ $4,128.43 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                   |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |  ≈ $13,761.42 | ลดลงอีก ≈ $8,256.85 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                         | Instance Name         | Specification                                                               | Quantity | Unit Price |           Total Price |
| ------------------------------- | --------------------- | --------------------------------------------------------------------------- | -------: | ---------: | --------------------: |
| Amazon EC2 (Observability Node) | m6i.xlarge            | 4 vCPU, 16 GB — runs OTel Collector, Prometheus (HA), Grafana, Loki, Jaeger |        1 |    $140.16 |               $140.16 |
| Amazon EBS (gp3)                | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~500 GB)          |      500 |   $0.08/GB |                $40.00 |
| Amazon S3                       | Log & Trace Retention | Loki chunks, trace retention (~5,000 GB)                                    |    5,000 |  $0.023/GB |               $115.00 |
|                                 |                       |                                                                             |          |  **Total** | **≈ $295.16 / month** |

**ตัวเลือก B — Cloud Managed (AWS)**

| Service                               | Instance Name        | Specification                                          | Quantity | Unit Price |           Total Price |
| ------------------------------------- | -------------------- | ------------------------------------------------------ | -------: | ---------: | --------------------: |
| AWS Distro for OpenTelemetry (ADOT)   | Collector Deployment | Standard telemetry pipeline สำหรับทุก service และ cluster |        1 |      $0.00 |                 $0.00 |
| Amazon Managed Service for Prometheus | Metrics Backend      | Metrics backend สำหรับ scale ระยะยาว                     |        1 |    $350.00 |               $350.00 |
| Amazon Managed Grafana                | Dashboard            | Enterprise dashboard สำหรับทีม ~15 users                  |       15 | $9.00/user |               $135.00 |
| Amazon CloudWatch Logs                | Log Aggregation      | Centralized logs และ query ระยะยาว                     |        1 |    $250.00 |               $250.00 |
| AWS X-Ray                             | Distributed Tracing  | Production tracing สำหรับ request ปริมาณสูง                |        1 |     $50.00 |                $50.00 |
|                                       |                      |                                                        |          |  **Total** | **≈ $785.00 / month** |

## Cloud: Google Cloud Platform (GCP)

GCP เหมาะกับทีมที่เน้น Kubernetes-native และต้นทุนคุ้มค่าต่อ compute (GKE และ Compute Engine ราคาแข่งขันได้):

- **Minimum**: GKE เป็นหนึ่งใน managed Kubernetes ที่โตเต็มที่ที่สุด ต้นทุนรวมต่ำที่สุดในบรรดา 4 ราย (≈ $668/เดือน) เหมาะกับ MVP ที่ต้องการประหยัดต้นทุน
- **Recommended and Scaling**: Cloud SQL Regional และ Memorystore Standard Tier ให้ HA ในรูปแบบ managed เต็มรูปแบบ ส่วน Elasticsearch/Kafka ต้อง self-manage บน Compute Engine ซึ่งเพิ่มภาระ ops เล็กน้อยแต่ยังควบคุมต้นทุนได้ดี
- **Long-term**: ข้อจำกัดคือไม่มี managed OpenSearch/Kafka ของตัวเอง ทำให้ทีมต้องดูแล cluster เหล่านี้เองในระดับใหญ่ขึ้น จึงเหมาะกับองค์กรที่มีทีม SRE/Platform แข็งแรงพอจะรับภาระ self-managed service ที่ scale ขึ้น

### Minimum

| Service                                     | Instance Name         | Specification                          | Quantity |    Unit Price |           Total Price |
| ------------------------------------------- | --------------------- | -------------------------------------- | -------: | ------------: | --------------------: |
| Google Kubernetes Engine                    | GKE Cluster           | Cluster management fee (Standard mode) |        1 |        $73.00 |                $73.00 |
| Compute Engine (Worker Nodes)               | e2-standard-2         | 2 vCPU, 8 GB RAM                       |        3 |        $48.92 |               $146.76 |
| Cloud SQL for PostgreSQL                    | db-custom-2-8192      | 2 vCPU, 8 GB, single-zone              |        1 |       $140.00 |               $140.00 |
| Memorystore for Redis                       | Basic Tier            | 1 GB                                   |        1 |        $50.00 |                $50.00 |
| Compute Engine (self-managed Elasticsearch) | e2-standard-2         | 2 vCPU, 8 GB, single node              |        1 |        $48.92 |                $48.92 |
| Compute Engine (self-managed Kafka broker)  | e2-standard-2         | 2 vCPU, 8 GB, broker (min. 3)          |        3 |        $48.92 |               $146.76 |
| Cloud Load Balancing                        | Application LB        | Forwarding rule + base usage           |        1 |        $25.00 |                $25.00 |
| Cloud NAT                                   | NAT Gateway           | Single gateway + light data processing |        1 |        $32.00 |                $32.00 |
| Cloud Storage                               | Standard Storage      | ~200 GB                                |      200 |     $0.020/GB |                 $4.00 |
| Secret Manager                              | Active secret version | ~25 secrets/month                      |       25 | $0.06/version |                 $1.50 |
|                                             |                       |                                        |          |     **Total** | **≈ $667.94 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                           |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ---------------------------------------------------------------------------------- |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |   ≈ $1,269.09 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                   |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |   ≈ $1,068.70 | ลดลง ≈ $200.38 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                   |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |     ≈ $667.94 | ลดลงอีก ≈ $400.76 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                             | Instance Name         | Specification                                                         | Quantity | Unit Price |          Total Price |
| ----------------------------------- | --------------------- | --------------------------------------------------------------------- | -------: | ---------: | -------------------: |
| Compute Engine (Observability Node) | e2-standard-2         | 2 vCPU, 8 GB — runs OTel Collector, Prometheus, Grafana, Loki, Jaeger |        1 |     $48.92 |               $48.92 |
| Persistent Disk (SSD)               | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~100 GB)    |      100 |   $0.17/GB |               $17.00 |
| Cloud Storage                       | Log & Trace Retention | Loki chunks, trace retention (~200 GB)                                |      200 |  $0.020/GB |                $4.00 |
|                                     |                       |                                                                       |          |  **Total** | **≈ $69.92 / month** |

**ตัวเลือก B — Cloud Managed (GCP)**

| Service                                     | Instance Name        | Specification                                                   | Quantity | Unit Price |          Total Price |
| ------------------------------------------- | -------------------- | --------------------------------------------------------------- | -------: | ---------: | -------------------: |
| Google Cloud OpenTelemetry Collector        | Collector Deployment | Telemetry pipeline สำหรับ metrics, logs, traces บน GKE            |        1 |      $0.00 |                $0.00 |
| Google Cloud Managed Service for Prometheus | Metrics Backend      | Metrics ingestion, query และ storage (scale เล็ก)                |        1 |     $20.00 |               $20.00 |
| Cloud Monitoring                            | Dashboard & Alerting | Dashboard และ alerting พื้นฐาน (free tier รองรับ workload ขนาดเล็ก) |        1 |      $0.00 |                $0.00 |
| Cloud Logging                               | Log Aggregation      | Centralized application logs                                    |        1 |     $50.00 |               $50.00 |
| Cloud Trace                                 | Distributed Tracing  | Distributed tracing สำหรับ request flow สำคัญ                       |        1 |      $5.00 |                $5.00 |
|                                             |                      |                                                                 |          |  **Total** | **≈ $75.00 / month** |

### Recommended and Scaling

| Service                                     | Instance Name         | Specification                          | Quantity |    Unit Price |             Total Price |
| ------------------------------------------- | --------------------- | -------------------------------------- | -------: | ------------: | ----------------------: |
| Google Kubernetes Engine                    | GKE Cluster           | Cluster management fee (Standard mode) |        1 |        $73.00 |                  $73.00 |
| Compute Engine (Worker Nodes)               | n2-standard-4         | 4 vCPU, 16 GB RAM                      |        6 |       $164.16 |                 $984.96 |
| Cloud SQL for PostgreSQL                    | db-custom-4-16384     | 4 vCPU, 16 GB, Regional (HA)           |        2 |       $280.00 |                 $560.00 |
| Memorystore for Redis                       | Standard Tier (HA)    | 5 GB                                   |        1 |       $358.00 |                 $358.00 |
| Compute Engine (self-managed Elasticsearch) | n2-standard-4         | 4 vCPU, 16 GB, 3-node cluster          |        3 |       $164.16 |                 $492.48 |
| Compute Engine (self-managed Kafka broker)  | n2-standard-4         | 4 vCPU, 16 GB, broker                  |        3 |       $164.16 |                 $492.48 |
| Cloud Load Balancing                        | Application LB        | Higher throughput + data processing    |        1 |        $60.00 |                  $60.00 |
| Cloud NAT                                   | NAT Gateway           | Multi-zone                             |        2 |        $32.00 |                  $64.00 |
| Cloud Storage                               | Standard Storage      | ~2,000 GB                              |    2,000 |     $0.020/GB |                  $40.00 |
| Secret Manager                              | Active secret version | ~60 secrets/month                      |       60 | $0.06/version |                   $3.60 |
|                                             |                       |                                        |          |     **Total** | **≈ $3,128.52 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                             |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ------------------------------------------------------------------------------------ |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |   ≈ $5,944.19 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                     |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |   ≈ $5,005.63 | ลดลง ≈ $938.56 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                     |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |   ≈ $3,128.52 | ลดลงอีก ≈ $1,877.11 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                             | Instance Name         | Specification                                                         | Quantity | Unit Price |           Total Price |
| ----------------------------------- | --------------------- | --------------------------------------------------------------------- | -------: | ---------: | --------------------: |
| Compute Engine (Observability Node) | e2-standard-2         | 2 vCPU, 8 GB — runs OTel Collector, Prometheus, Grafana, Loki, Jaeger |        1 |     $48.92 |                $48.92 |
| Persistent Disk (SSD)               | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~200 GB)    |      200 |   $0.17/GB |                $34.00 |
| Cloud Storage                       | Log & Trace Retention | Loki chunks, trace retention (~1,000 GB)                              |    1,000 |  $0.020/GB |                $20.00 |
|                                     |                       |                                                                       |          |  **Total** | **≈ $102.92 / month** |

**ตัวเลือก B — Cloud Managed (GCP)**

| Service                                     | Instance Name        | Specification                                                 | Quantity | Unit Price |           Total Price |
| ------------------------------------------- | -------------------- | ------------------------------------------------------------- | -------: | ---------: | --------------------: |
| Google Cloud OpenTelemetry Collector        | Collector Deployment | Telemetry pipeline สำหรับ metrics, logs, traces บน GKE          |        1 |      $0.00 |                 $0.00 |
| Google Cloud Managed Service for Prometheus | Metrics Backend      | Metrics ingestion, query และ storage สำหรับ production workload |        1 |     $75.00 |                $75.00 |
| Cloud Monitoring                            | Dashboard & Alerting | Dashboard และ alerting ผ่าน Google Cloud Monitoring            |        1 |      $0.00 |                 $0.00 |
| Cloud Logging                               | Log Aggregation      | Centralized application logs ปริมาณกลาง                        |        1 |     $50.00 |                $50.00 |
| Cloud Trace                                 | Distributed Tracing  | Distributed tracing สำหรับ request flow สำคัญ                     |        1 |     $10.00 |                $10.00 |
|                                             |                      |                                                               |          |  **Total** | **≈ $135.00 / month** |
### Long-term

| Service                                     | Instance Name              | Specification                                 | Quantity |    Unit Price |              Total Price |
| ------------------------------------------- | -------------------------- | --------------------------------------------- | -------: | ------------: | -----------------------: |
| Google Kubernetes Engine                    | GKE Cluster                | Cluster management fee (Production + Staging) |        2 |        $73.00 |                  $146.00 |
| Compute Engine (Worker Nodes)               | n2-standard-8              | 8 vCPU, 32 GB RAM, multi node-pool            |       12 |       $328.32 |                $3,939.84 |
| Cloud SQL for PostgreSQL                    | db-custom-8-32768          | 8 vCPU, 32 GB, Regional HA + Read Replica     |        3 |       $560.00 |                $1,680.00 |
| Memorystore for Redis                       | Standard Tier (HA cluster) | 20 GB                                         |        2 |       $700.00 |                $1,400.00 |
| Compute Engine (self-managed Elasticsearch) | n2-standard-8              | 8 vCPU, 32 GB, 6-node cluster                 |        6 |       $328.32 |                $1,969.92 |
| Compute Engine (self-managed Kafka broker)  | n2-standard-8              | 8 vCPU, 32 GB, 6-broker cluster               |        6 |       $328.32 |                $1,969.92 |
| Cloud Load Balancing                        | Application LB             | Multi-region                                  |        2 |        $80.00 |                  $160.00 |
| Cloud NAT                                   | NAT Gateway                | Multi-region / multi-zone                     |        4 |        $32.00 |                  $128.00 |
| Cloud Storage                               | Standard + Nearline        | ~20,000 GB                                    |   20,000 |     $0.018/GB |                  $360.00 |
| Secret Manager                              | Active secret version      | ~150 secrets/month                            |      150 | $0.06/version |                    $9.00 |
|                                             |                            |                                               |          |     **Total** | **≈ $11,762.68 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                             |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ------------------------------------------------------------------------------------ |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |  ≈ $22,349.09 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                     |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |  ≈ $18,820.29 | ลดลง ≈ $3,528.80 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                   |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |  ≈ $11,762.68 | ลดลงอีก ≈ $7,057.61 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                             | Instance Name         | Specification                                                               | Quantity | Unit Price |           Total Price |
| ----------------------------------- | --------------------- | --------------------------------------------------------------------------- | -------: | ---------: | --------------------: |
| Compute Engine (Observability Node) | n2-standard-4         | 4 vCPU, 16 GB — runs OTel Collector, Prometheus (HA), Grafana, Loki, Jaeger |        1 |    $164.16 |               $164.16 |
| Persistent Disk (SSD)               | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~500 GB)          |      500 |   $0.17/GB |                $85.00 |
| Cloud Storage                       | Log & Trace Retention | Loki chunks, trace retention (~5,000 GB)                                    |    5,000 |  $0.020/GB |               $100.00 |
|                                     |                       |                                                                             |          |  **Total** | **≈ $349.16 / month** |

**ตัวเลือก B — Cloud Managed (GCP)**

| Service                                     | Instance Name        | Specification                                          | Quantity | Unit Price |           Total Price |
| ------------------------------------------- | -------------------- | ------------------------------------------------------ | -------: | ---------: | --------------------: |
| Google Cloud OpenTelemetry Collector        | Collector Deployment | Standard telemetry pipeline สำหรับทุก service และ cluster |        1 |      $0.00 |                 $0.00 |
| Google Cloud Managed Service for Prometheus | Metrics Backend      | Metrics backend สำหรับ scale ระยะยาว                     |        1 |    $350.00 |               $350.00 |
| Cloud Monitoring                            | Dashboard & Alerting | Operational dashboard กลางของ platform                 |        1 |      $0.00 |                 $0.00 |
| Cloud Logging                               | Log Aggregation      | Centralized logs พร้อม long-term sink                   |        1 |    $250.00 |               $250.00 |
| Cloud Trace                                 | Distributed Tracing  | Production tracing สำหรับ request ปริมาณสูง                |        1 |     $50.00 |                $50.00 |
|                                             |                      |                                                        |          |  **Total** | **≈ $650.00 / month** |

## Cloud: Microsoft Azure (Azure)

Azure เหมาะกับองค์กรที่มี ecosystem ของ Microsoft อยู่แล้ว (Active Directory, .NET, Office 365) และต้องการ integration ที่ราบรื่น:

- **Minimum**: AKS ไม่คิดค่า cluster management fee เพิ่มเติม (จ่ายเฉพาะ Uptime SLA) และ Azure Database/Cache มี tier เริ่มต้นที่ยืดหยุ่น เหมาะกับการเริ่มต้นแบบประหยัด
- **Recommended and Scaling**: Flexible Server แบบ Zone-Redundant และ Azure Cache for Redis Standard/Premium ให้ HA ระดับ production ได้ในรูปแบบ managed เต็มรูปแบบ ตรงกับความต้องการ Multi-AZ ของสถาปัตยกรรม
- **Long-term**: เช่นเดียวกับ GCP คือไม่มี managed search/streaming ของตัวเอง ต้อง self-manage บน VM แต่ Azure มีจุดแข็งด้าน compliance, hybrid cloud (Azure Arc) และ regional presence ที่กว้าง เหมาะกับองค์กรที่ต้องขยายตลาดในหลายภูมิภาคพร้อม regulatory requirement ที่เข้มงวด

### Minimum

| Service                                       | Instance Name       | Specification                        | Quantity |    Unit Price |           Total Price |
| --------------------------------------------- | ------------------- | ------------------------------------ | -------: | ------------: | --------------------: |
| Azure Kubernetes Service                      | AKS Cluster         | Standard tier (Uptime SLA)           |        1 |        $73.00 |                $73.00 |
| Virtual Machines (Worker Nodes)               | Standard_D2s_v5     | 2 vCPU, 8 GB RAM                     |        3 |        $70.08 |               $210.24 |
| Azure Database for PostgreSQL                 | Flexible Server B2s | 2 vCore, 4 GB, single zone           |        1 |        $60.00 |                $60.00 |
| Azure Cache for Redis                         | Basic C1            | 1 GB                                 |        1 |        $40.00 |                $40.00 |
| Virtual Machines (self-managed Elasticsearch) | Standard_D2s_v5     | 2 vCPU, 8 GB, single node            |        1 |        $70.08 |                $70.08 |
| Virtual Machines (self-managed Kafka broker)  | Standard_D2s_v5     | 2 vCPU, 8 GB, broker (min. 3)        |        3 |        $70.08 |               $210.24 |
| Azure Load Balancer                           | Standard LB         | Basic rules + base usage             |        1 |        $20.00 |                $20.00 |
| Azure NAT Gateway                             | NAT Gateway         | Single zone                          |        1 |        $32.85 |                $32.85 |
| Azure Blob Storage                            | Hot Tier            | ~200 GB                              |      200 |     $0.018/GB |                 $3.60 |
| Azure Key Vault                               | Operations          | ~100,000 ops/month (10× 10k-batches) |       10 | $0.03/10k ops |                 $0.30 |
|                                               |                     |                                      |          |     **Total** | **≈ $720.31 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                           |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ---------------------------------------------------------------------------------- |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |   ≈ $1,368.59 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                   |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |   ≈ $1,152.50 | ลดลง ≈ $216.09 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                   |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |     ≈ $720.31 | ลดลงอีก ≈ $432.19 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                               | Instance Name         | Specification                                                         | Quantity | Unit Price |          Total Price |
| ------------------------------------- | --------------------- | --------------------------------------------------------------------- | -------: | ---------: | -------------------: |
| Virtual Machines (Observability Node) | Standard_D2s_v5       | 2 vCPU, 8 GB — runs OTel Collector, Prometheus, Grafana, Loki, Jaeger |        1 |     $70.08 |               $70.08 |
| Managed Disk (Premium SSD)            | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~100 GB)    |      100 |  $0.135/GB |               $13.50 |
| Azure Blob Storage                    | Log & Trace Retention | Loki chunks, trace retention (~200 GB)                                |      200 |  $0.018/GB |                $3.60 |
|                                       |                       |                                                                       |          |  **Total** | **≈ $87.18 / month** |

**ตัวเลือก B — Cloud Managed (Azure)**

| Service                                      | Instance Name             | Specification                                        | Quantity | Unit Price |           Total Price |
| -------------------------------------------- | ------------------------- | ---------------------------------------------------- | -------: | ---------: | --------------------: |
| Azure Monitor OpenTelemetry                  | Collector Deployment      | Telemetry pipeline สำหรับ metrics, logs, traces บน AKS |        1 |      $0.00 |                 $0.00 |
| Azure Monitor managed service for Prometheus | Metrics Backend           | Metrics ingestion, query และ storage                 |        1 |     $75.00 |                $75.00 |
| Azure Managed Grafana (Essential)            | Dashboard                 | Dashboard, alerting และ operational view             |        1 |     $49.00 |                $49.00 |
| Azure Monitor Logs / Log Analytics           | Log Aggregation           | Centralized application logs ปริมาณเล็ก                |        1 |     $50.00 |                $50.00 |
| Application Insights                         | Distributed Tracing / APM | Distributed tracing และ APM สำหรับ request flow สำคัญ    |        1 |     $10.00 |                $10.00 |
|                                              |                           |                                                      |          |  **Total** | **≈ $184.00 / month** |
### Recommended and Scaling

| Service                                       | Instance Name                  | Specification                        | Quantity |    Unit Price |             Total Price |
| --------------------------------------------- | ------------------------------ | ------------------------------------ | -------: | ------------: | ----------------------: |
| Azure Kubernetes Service                      | AKS Cluster                    | Standard tier (Uptime SLA)           |        1 |        $73.00 |                  $73.00 |
| Virtual Machines (Worker Nodes)               | Standard_D4s_v5                | 4 vCPU, 16 GB RAM                    |        6 |       $140.16 |                 $840.96 |
| Azure Database for PostgreSQL                 | Flexible Server (Gen. Purpose) | 4 vCore, 16 GB, Zone-Redundant HA    |        2 |       $300.00 |                 $600.00 |
| Azure Cache for Redis                         | Standard C2                    | 2.5 GB, primary + replica            |        2 |       $170.00 |                 $340.00 |
| Virtual Machines (self-managed Elasticsearch) | Standard_D4s_v5                | 4 vCPU, 16 GB, 3-node cluster        |        3 |       $140.16 |                 $420.48 |
| Virtual Machines (self-managed Kafka broker)  | Standard_D4s_v5                | 4 vCPU, 16 GB, broker                |        3 |       $140.16 |                 $420.48 |
| Azure Load Balancer                           | Standard LB                    | More rules + data processing         |        1 |        $50.00 |                  $50.00 |
| Azure NAT Gateway                             | NAT Gateway                    | Multi-zone                           |        2 |        $32.85 |                  $65.70 |
| Azure Blob Storage                            | Hot Tier                       | ~2,000 GB                            |    2,000 |     $0.018/GB |                  $36.00 |
| Azure Key Vault                               | Operations                     | ~300,000 ops/month (30× 10k-batches) |       30 | $0.03/10k ops |                   $0.90 |
|                                               |                                |                                      |          |     **Total** | **≈ $2,847.52 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                             |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ------------------------------------------------------------------------------------ |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |   ≈ $5,410.29 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                     |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |   ≈ $4,556.03 | ลดลง ≈ $854.26 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                     |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |   ≈ $2,847.52 | ลดลงอีก ≈ $1,708.51 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                               | Instance Name         | Specification                                                         | Quantity | Unit Price |           Total Price |
| ------------------------------------- | --------------------- | --------------------------------------------------------------------- | -------: | ---------: | --------------------: |
| Virtual Machines (Observability Node) | Standard_D2s_v5       | 2 vCPU, 8 GB — runs OTel Collector, Prometheus, Grafana, Loki, Jaeger |        1 |     $70.08 |                $70.08 |
| Managed Disk (Premium SSD)            | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~200 GB)    |      200 |  $0.135/GB |                $27.00 |
| Azure Blob Storage                    | Log & Trace Retention | Loki chunks, trace retention (~1,000 GB)                              |    1,000 |  $0.018/GB |                $18.00 |
|                                       |                       |                                                                       |          |  **Total** | **≈ $115.08 / month** |

**ตัวเลือก B — Cloud Managed (Azure)**

| Service                                      | Instance Name             | Specification                                                 | Quantity | Unit Price |           Total Price |
| -------------------------------------------- | ------------------------- | ------------------------------------------------------------- | -------: | ---------: | --------------------: |
| Azure Monitor OpenTelemetry                  | Collector Deployment      | Telemetry pipeline สำหรับ metrics, logs, traces บน AKS          |        1 |      $0.00 |                 $0.00 |
| Azure Monitor managed service for Prometheus | Metrics Backend           | Metrics ingestion, query และ storage สำหรับ production workload |        1 |     $75.00 |                $75.00 |
| Azure Managed Grafana (Essential)            | Dashboard                 | Dashboard, alerting และ operational view                      |        1 |     $49.00 |                $49.00 |
| Azure Monitor Logs / Log Analytics           | Log Aggregation           | Centralized application logs ปริมาณกลาง                        |        1 |     $50.00 |                $50.00 |
| Application Insights                         | Distributed Tracing / APM | Distributed tracing และ APM สำหรับ request flow สำคัญ             |        1 |     $10.00 |                $10.00 |
|                                              |                           |                                                               |          |  **Total** | **≈ $184.00 / month** |
### Long-term

| Service                                       | Instance Name                    | Specification                        | Quantity |    Unit Price |              Total Price |
| --------------------------------------------- | -------------------------------- | ------------------------------------ | -------: | ------------: | -----------------------: |
| Azure Kubernetes Service                      | AKS Cluster                      | Standard tier (Production + Staging) |        2 |        $73.00 |                  $146.00 |
| Virtual Machines (Worker Nodes)               | Standard_D8s_v5                  | 8 vCPU, 32 GB RAM, multi node-pool   |       12 |       $280.32 |                $3,363.84 |
| Azure Database for PostgreSQL                 | Flexible Server (Mem. Optimized) | 8 vCore, 64 GB, HA + Read Replica    |        3 |       $620.00 |                $1,860.00 |
| Azure Cache for Redis                         | Premium P2                       | 13 GB, clustered                     |        4 |       $700.00 |                $2,800.00 |
| Virtual Machines (self-managed Elasticsearch) | Standard_D8s_v5                  | 8 vCPU, 32 GB, 6-node cluster        |        6 |       $280.32 |                $1,681.92 |
| Virtual Machines (self-managed Kafka broker)  | Standard_D8s_v5                  | 8 vCPU, 32 GB, 6-broker cluster      |        6 |       $280.32 |                $1,681.92 |
| Azure Load Balancer                           | Standard LB                      | Multi-region                         |        2 |        $80.00 |                  $160.00 |
| Azure NAT Gateway                             | NAT Gateway                      | Multi-region / multi-zone            |        4 |        $32.85 |                  $131.40 |
| Azure Blob Storage                            | Hot + Cool Tier                  | ~20,000 GB                           |   20,000 |     $0.015/GB |                  $300.00 |
| Azure Key Vault                               | Operations                       | ~800,000 ops/month (80× 10k-batches) |       80 | $0.03/10k ops |                    $2.40 |
|                                               |                                  |                                      |          |     **Total** | **≈ $12,127.48 / month** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) | ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                             |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ------------: | ------------------------------------------------------------------------------------ |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 |  ≈ $23,042.21 | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                     |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 |  ≈ $19,403.97 | ลดลง ≈ $3,638.24 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                   |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |  ≈ $12,127.48 | ลดลงอีก ≈ $7,276.49 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                               | Instance Name         | Specification                                                               | Quantity | Unit Price |           Total Price |
| ------------------------------------- | --------------------- | --------------------------------------------------------------------------- | -------: | ---------: | --------------------: |
| Virtual Machines (Observability Node) | Standard_D4s_v5       | 4 vCPU, 16 GB — runs OTel Collector, Prometheus (HA), Grafana, Loki, Jaeger |        1 |    $140.16 |               $140.16 |
| Managed Disk (Premium SSD)            | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~500 GB)          |      500 |  $0.135/GB |                $67.50 |
| Azure Blob Storage                    | Log & Trace Retention | Loki chunks, trace retention (~5,000 GB)                                    |    5,000 |  $0.018/GB |                $90.00 |
|                                       |                       |                                                                             |          |  **Total** | **≈ $297.66 / month** |

**ตัวเลือก B — Cloud Managed (Azure)**

| Service                                      | Instance Name             | Specification                                          | Quantity | Unit Price |           Total Price |
| -------------------------------------------- | ------------------------- | ------------------------------------------------------ | -------: | ---------: | --------------------: |
| Azure Monitor OpenTelemetry                  | Collector Deployment      | Standard telemetry pipeline สำหรับทุก service และ cluster |        1 |      $0.00 |                 $0.00 |
| Azure Monitor managed service for Prometheus | Metrics Backend           | Metrics backend สำหรับ scale ระยะยาว                     |        1 |    $350.00 |               $350.00 |
| Azure Managed Grafana (Standard)             | Dashboard                 | Enterprise dashboard, permission และ operational view  |        1 |     $99.00 |                $99.00 |
| Azure Monitor Logs / Log Analytics           | Log Aggregation           | Centralized logs และ query ระยะยาว                     |        1 |    $250.00 |               $250.00 |
| Application Insights                         | Distributed Tracing / APM | Production tracing และ APM สำหรับ request ปริมาณสูง        |        1 |     $50.00 |                $50.00 |
|                                              |                           |                                                        |          |  **Total** | **≈ $749.00 / month** |

## Cloud: Huawei Cloud (Huawei)

Huawei เหมาะกับธุรกิจที่ต้องการขยายตลาดในจีนแผ่นดินใหญ่และเอเชียตะวันออกเฉียงใต้ ซึ่งมี data center ในภูมิภาคและราคาที่แข่งขันได้:

- **Minimum**: ต้นทุนรวมใกล้เคียง AWS/Azure (≈ $900/เดือน) และมีบริการ managed ครบสำหรับ component หลัก (CCE, RDS, DCS, OBS) จึงเริ่มต้นได้ไม่ต่างจากผู้ให้บริการรายใหญ่
- **Recommended and Scaling**: DCS for Redis และ RDS รองรับ HA ได้แบบ managed เทียบเท่า ElastiCache/RDS ของ AWS ทำให้ปรับ scale ตามสถาปัตยกรรมได้โดยไม่ต้อง self-manage มาก
- **Long-term**: เหมาะกับธุรกิจที่ traffic หลักอยู่ในภูมิภาคที่ Huawei มี presence แข็งแรง (เช่น จีน, เอเชียตะวันออกเฉียงใต้) เพราะ latency ต่ำกว่าและมักมีข้อได้เปรียบด้าน data residency/compliance ในประเทศนั้น ๆ แต่ควรพิจารณาเรื่องความพร้อมของข้อมูลราคาและเอกสารภาษาอังกฤษที่ยังจำกัดกว่า 3 รายข้างต้น

### Minimum

| Service                                     | Instance Name    | Specification                   | Quantity |   Unit Price |                       Total Price |
| ------------------------------------------- | ---------------- | ------------------------------- | -------: | -----------: | --------------------------------: |
| Cloud Container Engine (CCE)                | CCE Cluster      | Standard cluster management fee |        1 |       $70.00 |                            $70.00 |
| Elastic Cloud Server (Worker Nodes)         | s6.xlarge.2      | 2 vCPU, 8 GB RAM                |        3 |       $65.00 |                           $195.00 |
| RDS for PostgreSQL                          | rds.pg.large     | 2 vCore, 4 GB, single-AZ        |        1 |       $55.00 |                            $55.00 |
| Distributed Cache Service (DCS) for Redis   | Single-node      | 1 GB                            |        1 |       $35.00 |                            $35.00 |
| Cloud Search Service (CSS) – Elasticsearch  | css.xlarge.2     | 2 vCPU, 8 GB, single node       |        1 |       $60.00 |                            $60.00 |
| Distributed Message Service (DMS) for Kafka | Broker           | 2 vCPU, 8 GB, broker (min. 3)   |        3 |      $140.00 |                           $420.00 |
| Elastic Load Balance (ELB)                  | Shared LB        | Basic rules + base usage        |        1 |       $20.00 |                            $20.00 |
| NAT Gateway                                 | NAT Gateway      | Single AZ                       |        1 |       $30.00 |                            $30.00 |
| Object Storage Service (OBS)                | Standard Storage | ~200 GB                         |      200 |    $0.023/GB |                             $4.60 |
| Key Management / Secret Management Service  | Secret           | ~25 secrets/keys                |       25 | $0.40/secret |                            $10.00 |
|                                             |                  |                                 |          |    **Total** | **≈ $899.60 / month (estimated)** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) |           ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                           |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ----------------------: | ---------------------------------------------------------------------------------- |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 | ≈ $1,709.24 (estimated) | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                   |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 | ≈ $1,439.36 (estimated) | ลดลง ≈ $269.88 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                   |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 |   ≈ $899.60 (estimated) | ลดลงอีก ≈ $539.76 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                                   | Instance Name         | Specification                                                         | Quantity | Unit Price |                      Total Price |
| ----------------------------------------- | --------------------- | --------------------------------------------------------------------- | -------: | ---------: | -------------------------------: |
| Elastic Cloud Server (Observability Node) | s6.xlarge.2           | 2 vCPU, 8 GB — runs OTel Collector, Prometheus, Grafana, Loki, Jaeger |        1 |     $65.00 |                           $65.00 |
| Elastic Volume Service (EVS)              | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~100 GB)    |      100 |   $0.08/GB |                            $8.00 |
| Object Storage Service (OBS)              | Log & Trace Retention | Loki chunks, trace retention (~200 GB)                                |      200 |  $0.023/GB |                            $4.60 |
|                                           |                       |                                                                       |          |  **Total** | **≈ $77.60 / month (estimated)** |

**ตัวเลือก B — Cloud Managed (Huawei)** _(ไม่มี managed Prometheus/Grafana — ใช้ Cloud Eye + AOM แทน metrics/dashboard, APM แทน Jaeger)_

| Service                                  | Instance Name                   | Specification                                                             | Quantity | Unit Price |                       Total Price |
| ---------------------------------------- | ------------------------------- | ------------------------------------------------------------------------- | -------: | ---------: | --------------------------------: |
| OpenTelemetry Collector on CCE           | Collector Deployment            | Telemetry pipeline สำหรับ metrics, logs, traces บน CCE                      |        1 |      $0.00 |                             $0.00 |
| Cloud Eye                                | Infra Metrics & Alerting        | Infrastructure metrics และ alarm สำหรับ cloud resources                     |        1 |     $30.00 |                            $30.00 |
| Application Operations Management (AOM)  | Application Metrics & Dashboard | Metrics collection, dashboard, alerting (แทน Prometheus + Grafana บางส่วน) |        1 |     $30.00 |                            $30.00 |
| Log Tank Service (LTS)                   | Log Aggregation                 | Centralized application logs (แทน Loki)                                   |        1 |     $30.00 |                            $30.00 |
| Application Performance Management (APM) | Distributed Tracing             | Distributed tracing สำหรับ request flow (แทน Jaeger)                        |        1 |     $30.00 |                            $30.00 |
|                                          |                                 |                                                                           |          |  **Total** | **≈ $120.00 / month (estimated)** |

### Recommended and Scaling

| Service                                     | Instance Name    | Specification                       | Quantity |   Unit Price |                         Total Price |
| ------------------------------------------- | ---------------- | ----------------------------------- | -------: | -----------: | ----------------------------------: |
| Cloud Container Engine (CCE)                | CCE Cluster      | Standard cluster management fee     |        1 |       $70.00 |                              $70.00 |
| Elastic Cloud Server (Worker Nodes)         | s6.2xlarge.2     | 4 vCPU, 16 GB RAM                   |        6 |      $125.00 |                             $750.00 |
| RDS for PostgreSQL                          | rds.pg.xlarge    | 4 vCore, 16 GB, HA (active/standby) |        2 |      $270.00 |                             $540.00 |
| Distributed Cache Service (DCS) for Redis   | Master/Standby   | 4 GB                                |        2 |      $150.00 |                             $300.00 |
| Cloud Search Service (CSS) – Elasticsearch  | css.2xlarge.2    | 4 vCPU, 16 GB, 3-node cluster       |        3 |      $125.00 |                             $375.00 |
| Distributed Message Service (DMS) for Kafka | Broker           | 4 vCPU, 16 GB, broker               |        3 |      $270.00 |                             $810.00 |
| Elastic Load Balance (ELB)                  | Dedicated LB     | Higher throughput                   |        1 |       $50.00 |                              $50.00 |
| NAT Gateway                                 | NAT Gateway      | Multi-AZ                            |        2 |       $30.00 |                              $60.00 |
| Object Storage Service (OBS)                | Standard Storage | ~2,000 GB                           |    2,000 |    $0.023/GB |                              $46.00 |
| Key Management / Secret Management Service  | Secret           | ~60 secrets/keys                    |       60 | $0.40/secret |                              $24.00 |
|                                             |                  |                                     |          |    **Total** | **≈ $3,025.00 / month (estimated)** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) |           ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                             |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | ----------------------: | ------------------------------------------------------------------------------------ |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 | ≈ $5,747.50 (estimated) | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                     |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 | ≈ $4,840.00 (estimated) | ลดลง ≈ $907.50 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                     |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 | ≈ $3,025.00 (estimated) | ลดลงอีก ≈ $1,815.00 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                                   | Instance Name         | Specification                                                         | Quantity | Unit Price |                       Total Price |
| ----------------------------------------- | --------------------- | --------------------------------------------------------------------- | -------: | ---------: | --------------------------------: |
| Elastic Cloud Server (Observability Node) | s6.xlarge.2           | 2 vCPU, 8 GB — runs OTel Collector, Prometheus, Grafana, Loki, Jaeger |        1 |     $65.00 |                            $65.00 |
| Elastic Volume Service (EVS)              | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~200 GB)    |      200 |   $0.08/GB |                            $16.00 |
| Object Storage Service (OBS)              | Log & Trace Retention | Loki chunks, trace retention (~1,000 GB)                              |    1,000 |  $0.023/GB |                            $23.00 |
|                                           |                       |                                                                       |          |  **Total** | **≈ $104.00 / month (estimated)** |

**ตัวเลือก B — Cloud Managed (Huawei)** _(ไม่มี managed Prometheus/Grafana — ใช้ Cloud Eye + AOM แทน metrics/dashboard, APM แทน Jaeger)_

| Service                                  | Instance Name                   | Specification                                                             | Quantity | Unit Price |                       Total Price |
| ---------------------------------------- | ------------------------------- | ------------------------------------------------------------------------- | -------: | ---------: | --------------------------------: |
| OpenTelemetry Collector on CCE           | Collector Deployment            | Telemetry pipeline สำหรับ metrics, logs, traces บน CCE                      |        1 |      $0.00 |                             $0.00 |
| Cloud Eye                                | Infra Metrics & Alerting        | Infrastructure metrics และ alarm สำหรับ production workload                 |        1 |     $50.00 |                            $50.00 |
| Application Operations Management (AOM)  | Application Metrics & Dashboard | Metrics collection, dashboard, alerting (แทน Prometheus + Grafana บางส่วน) |        1 |     $50.00 |                            $50.00 |
| Log Tank Service (LTS)                   | Log Aggregation                 | Centralized application logs ปริมาณกลาง (แทน Loki)                         |        1 |    $100.00 |                           $100.00 |
| Application Performance Management (APM) | Distributed Tracing             | Distributed tracing สำหรับ request flow สำคัญ (แทน Jaeger)                    |        1 |     $50.00 |                            $50.00 |
|                                          |                                 |                                                                           |          |  **Total** | **≈ $250.00 / month (estimated)** |

### Long-term

| Service                                     | Instance Name         | Specification                                          | Quantity |   Unit Price |                          Total Price |
| ------------------------------------------- | --------------------- | ------------------------------------------------------ | -------: | -----------: | -----------------------------------: |
| Cloud Container Engine (CCE)                | CCE Cluster           | Standard cluster management fee (Production + Staging) |        2 |       $70.00 |                              $140.00 |
| Elastic Cloud Server (Worker Nodes)         | c6.4xlarge.2          | 8 vCPU, 32 GB RAM, multi node-pool                     |       12 |      $250.00 |                            $3,000.00 |
| RDS for PostgreSQL                          | rds.pg.4xlarge        | 8 vCore, 32 GB, HA + Read Replica                      |        3 |      $560.00 |                            $1,680.00 |
| Distributed Cache Service (DCS) for Redis   | Cluster               | 16 GB                                                  |        4 |      $620.00 |                            $2,480.00 |
| Cloud Search Service (CSS) – Elasticsearch  | css.4xlarge.2         | 8 vCPU, 32 GB, 6-node cluster                          |        6 |      $250.00 |                            $1,500.00 |
| Distributed Message Service (DMS) for Kafka | Broker                | 8 vCPU, 32 GB, 6-broker cluster                        |        6 |      $530.00 |                            $3,180.00 |
| Elastic Load Balance (ELB)                  | Dedicated LB          | Multi-region                                           |        2 |       $80.00 |                              $160.00 |
| NAT Gateway                                 | NAT Gateway           | Multi-region / multi-AZ                                |        4 |       $30.00 |                              $120.00 |
| Object Storage Service (OBS)                | Standard + IA Storage | ~20,000 GB                                             |   20,000 |    $0.020/GB |                              $400.00 |
| Key Management / Secret Management Service  | Secret                | ~150 secrets/keys                                      |      150 | $0.40/secret |                               $60.00 |
|                                             |                       |                                                        |          |    **Total** | **≈ $12,720.00 / month (estimated)** |

**Environment Breakdown** _(ราคา "1 ชุด" ด้านบน × multiplier — ดู §Environment & Infrastructure)_

| ขอบเขต Environment                                     | Environment ที่รวมอยู่                            | สัดส่วน (multiplier) |            ราคาประมาณการ | เปลี่ยนแปลงจากขอบเขตก่อนหน้า                                                             |
| ------------------------------------------------------ | --------------------------------------------- | -----------------: | -----------------------: | ------------------------------------------------------------------------------------ |
| ครบทุกกลุ่ม (Dev+SIT/QA + UAT+Staging + Production)       | Development, SIT/QA, UAT, Staging, Production |               ×1.9 | ≈ $24,168.00 (estimated) | ฐานเริ่มต้น — กรณีแยก infra ตามกลุ่ม environment ทั้งหมด                                     |
| ตัดกลุ่ม Dev + SIT/QA ออก → เหลือ UAT+Staging + Production | UAT, Staging, Production                      |               ×1.6 | ≈ $20,352.00 (estimated) | ลดลง ≈ $3,816.00 (ส่วนของ Dev + SIT/QA, ×0.3 หายไป)                                   |
| ตัดกลุ่ม UAT + Staging ออกด้วย → เหลือเฉพาะ Production      | Production                                    |               ×1.0 | ≈ $12,720.00 (estimated) | ลดลงอีก ≈ $7,632.00 (ส่วนของ UAT + Staging, ×0.6 หายไป) — เท่ากับยอด "1 ชุด" ในตารางข้างต้น |

#### OpenTelemetry

**ตัวเลือก A — Self-hosted** _(OTel Collector + Prometheus + Grafana + Loki + Jaeger)_

| Service                                   | Instance Name         | Specification                                                               | Quantity | Unit Price |                       Total Price |
| ----------------------------------------- | --------------------- | --------------------------------------------------------------------------- | -------: | ---------: | --------------------------------: |
| Elastic Cloud Server (Observability Node) | s6.2xlarge.2          | 4 vCPU, 16 GB — runs OTel Collector, Prometheus (HA), Grafana, Loki, Jaeger |        1 |    $125.00 |                           $125.00 |
| Elastic Volume Service (EVS)              | Observability Volume  | Prometheus TSDB, Grafana DB, Loki buffer, Jaeger storage (~500 GB)          |      500 |   $0.08/GB |                            $40.00 |
| Object Storage Service (OBS)              | Log & Trace Retention | Loki chunks, trace retention (~5,000 GB)                                    |    5,000 |  $0.023/GB |                           $115.00 |
|                                           |                       |                                                                             |          |  **Total** | **≈ $280.00 / month (estimated)** |

**ตัวเลือก B — Cloud Managed (Huawei)** _(ไม่มี managed Prometheus/Grafana — ใช้ Cloud Eye + AOM แทน metrics/dashboard, APM แทน Jaeger)_

| Service                                  | Instance Name                   | Specification                                                                             | Quantity | Unit Price |                       Total Price |
| ---------------------------------------- | ------------------------------- | ----------------------------------------------------------------------------------------- | -------: | ---------: | --------------------------------: |
| OpenTelemetry Collector on CCE           | Collector Deployment            | Standard telemetry pipeline สำหรับทุก service และ cluster                                    |        1 |      $0.00 |                             $0.00 |
| Cloud Eye                                | Infra Metrics & Alerting        | Infrastructure metrics และ alarm สำหรับหลาย environment / region                            |        1 |    $100.00 |                           $100.00 |
| Application Operations Management (AOM)  | Application Metrics & Dashboard | Metrics collection, dashboard, alerting ระดับ enterprise (แทน Prometheus + Grafana บางส่วน) |        1 |    $100.00 |                           $100.00 |
| Log Tank Service (LTS)                   | Log Aggregation                 | Centralized logs และ query ระยะยาว (แทน Loki)                                             |        1 |    $250.00 |                           $250.00 |
| Application Performance Management (APM) | Distributed Tracing             | Production tracing สำหรับ request ปริมาณสูง (แทน Jaeger)                                      |        1 |    $100.00 |                           $100.00 |
|                                          |                                 |                                                                                           |          |  **Total** | **≈ $550.00 / month (estimated)** |

## DevOps & Observability Tooling

### Minimum

| Service        | Item                                        | Quantity | Unit Price           |                Price |
| -------------- | ------------------------------------------- | -------- | -------------------- | -------------------: |
| GitHub         | Team Plan (seats)                           | 5        | $4.00 / user / month |               $20.00 |
| GitHub Actions | CI runner minutes (Linux, ส่วนเกิน free tier) | 3,000    | $0.008 / minute      |               $24.00 |
| ArgoCD         | GitOps Deployment                           | -        |                      |                $0.00 |
| Helm           | Chart Packaging & Repository (OCI)          | -        |                      |                $0.00 |
|                |                                             | -        | **Subtotal**         | **≈ $44.00 / month** |

### Recommended and Scaling

| Service        | Item                                        | Quantity | Unit Price           |                 Price |
| -------------- | ------------------------------------------- | -------- | -------------------- | --------------------: |
| GitHub         | Team Plan (seats)                           | 10       | $4.00 / user / month |                $40.00 |
| GitHub Actions | CI runner minutes (Linux, ส่วนเกิน free tier) | 10,000   | $0.008 / minute      |                $80.00 |
| ArgoCD         | GitOps Deployment                           | -        |                      |                 $0.00 |
| Helm           | Chart Packaging & Repository (OCI)          | -        |                      |                 $0.00 |
|                |                                             | -        | **Subtotal**         | **≈ $120.00 / month** |

### Long-term

| Service        | Item                                        | Quantity | Unit Price           |                 Price |
| -------------- | ------------------------------------------- | -------- | -------------------- | --------------------: |
| GitHub         | Team Plan (seats)                           | 20       | $4.00 / user / month |                $80.00 |
| GitHub Actions | CI runner minutes (Linux, ส่วนเกิน free tier) | 30,000   | $0.008 / minute      |               $240.00 |
| ArgoCD         | GitOps Deployment                           | -        |                      |                 $0.00 |
| Helm           | Chart Packaging & Repository (OCI)          | -        |                      |                 $0.00 |
|                |                                             | -        | **Subtotal**         | **≈ $320.00 / month** |
