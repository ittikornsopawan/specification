# 3. Container & Kubernetes

Container และ Kubernetes เป็น Service กลุ่มที่ช่วยให้ deploy, manage และ scale application แบบ containerized ได้อย่างมีประสิทธิภาพ ลด overhead ในการจัดการ infrastructure และรองรับ microservice architecture ได้ดี

---

## Kubernetes (Managed Kubernetes Service)

### คืออะไร

Managed Kubernetes Service คือบริการ Kubernetes cluster ที่ Cloud Provider จัดการ control plane ให้ ผู้ใช้ต้องดูแลเฉพาะ worker node และ workload ที่ deploy บน cluster ลด operational overhead จากการจัดการ etcd, API server และ scheduler เอง

### ใช้งานแบบไหน

สร้าง cluster พร้อม Node Pool แล้ว deploy workload ผ่าน Kubernetes manifest (YAML) หรือ Helm Chart จัดการ scaling ผ่าน Horizontal Pod Autoscaler (HPA) และ Cluster Autoscaler

### เหมาะกับงานแบบไหน

เหมาะกับ microservice architecture, application ที่ต้องการ scale อย่างอิสระ, CI/CD pipeline ที่ deploy บ่อย, หรือ workload ที่ต้องการ high availability

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ project เล็กที่มี service เดียว เพราะ Kubernetes มี overhead ด้าน complexity สูง อาจใช้ Container service แบบ serverless หรือ VM แทน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                            |
| -------------- | --------------------------------------- |
| AWS            | Amazon EKS (Elastic Kubernetes Service) |
| GCP            | Google Kubernetes Engine (GKE)          |
| Azure          | Azure Kubernetes Service (AKS)          |
| Huawei Cloud   | Cloud Container Engine (CCE)            |

### Spec / Configuration

#### Cluster Configuration

| Spec / Configuration    | ความหมาย                           | ตัวอย่าง                      |
| ----------------------- | ---------------------------------- | --------------------------- |
| Kubernetes Version      | version ของ cluster                | 1.29, 1.30                  |
| Control Plane Mode      | แบบ Managed หรือ Self-managed       | Managed (ค่า default)        |
| Network Plugin (CNI)    | plugin สำหรับ pod networking         | Calico, Cilium, AWS VPC CNI |
| Cluster Endpoint Access | การเข้าถึง API server                | Public, Private, หรือ Both   |
| OIDC Provider           | สำหรับ IAM Role for Service Accounts | เปิดใช้เพื่อ pod-level IAM      |

#### Node Pool Configuration (AWS EKS)

| Instance Type | vCPU | Memory | เหมาะกับงานแบบไหน                 | On-demand ($/hr) | RI 1yr No Upfront | RI 1yr All Upfront | RI 3yr All Upfront | Spot avg |
| ------------- | ---: | -----: | -------------------------------- | ---------------: | ----------------: | -----------------: | -----------------: | -------: |
| `t3.medium`   |    2 |   4 GB | dev/test cluster, small workload |           0.0416 |            0.0280 |             0.0267 |             0.0188 |  ~0.0125 |
| `m6i.large`   |    2 |   8 GB | general purpose worker node      |           0.0960 |            0.0629 |             0.0604 |             0.0430 |  ~0.0288 |
| `m6i.xlarge`  |    4 |  16 GB | medium traffic workload          |           0.1920 |            0.1258 |             0.1208 |             0.0860 |  ~0.0576 |
| `m6i.2xlarge` |    8 |  32 GB | high traffic workload            |           0.3840 |            0.2516 |             0.2416 |             0.1720 |  ~0.1152 |
| `c6i.xlarge`  |    4 |   8 GB | compute-heavy workload           |           0.1700 |            0.1098 |             0.1054 |             0.0748 |  ~0.0510 |
| `r6i.xlarge`  |    4 |  32 GB | memory-heavy workload            |           0.2520 |            0.1650 |             0.1584 |             0.1122 |  ~0.0756 |

#### Node Pool Configuration (GKE)

| Machine Type    | vCPU | Memory | เหมาะกับงานแบบไหน            | On-demand ($/hr) | CUD 1yr | CUD 3yr | Spot avg |
| --------------- | ---: | -----: | --------------------------- | ---------------: | ------: | ------: | -------: |
| `e2-medium`     |    2 |   4 GB | dev/test cluster            |           0.0335 |       — |       — |  ~0.0101 |
| `n2-standard-2` |    2 |   8 GB | general purpose worker node |           0.0971 |  0.0613 |  0.0476 |  ~0.0243 |
| `n2-standard-4` |    4 |  16 GB | medium traffic workload     |           0.1942 |  0.1226 |  0.0952 |  ~0.0486 |
| `n2-standard-8` |    8 |  32 GB | high traffic workload       |           0.3884 |  0.2452 |  0.1904 |  ~0.0972 |
| `c2-standard-4` |    4 |  16 GB | compute-heavy workload      |           0.2088 |  0.1318 |  0.1024 |  ~0.0522 |

#### Node Pool Configuration (AKS)

| VM Size  | vCPU | Memory | เหมาะกับงานแบบไหน            | PAYG ($/hr) | Reserved 1yr | Reserved 3yr | Spot avg |
| -------- | ---: | -----: | --------------------------- | ----------: | -----------: | -----------: | -------: |
| `B2s`    |    2 |   4 GB | dev/test cluster            |      0.0416 |       0.0267 |       0.0188 |  ~0.0083 |
| `D2s v5` |    2 |   8 GB | general purpose worker node |      0.0960 |       0.0620 |       0.0432 |  ~0.0192 |
| `D4s v5` |    4 |  16 GB | medium traffic workload     |      0.1920 |       0.1240 |       0.0864 |  ~0.0384 |
| `D8s v5` |    8 |  32 GB | high traffic workload       |      0.3840 |       0.2480 |       0.1728 |  ~0.0768 |
| `F4s v2` |    4 |   8 GB | compute-heavy workload      |      0.1690 |       0.1090 |       0.0768 |  ~0.0254 |

#### Node Pool Configuration (Huawei Cloud CCE)

| Flavor         | vCPU | Memory | เหมาะกับงานแบบไหน            | On-demand ($/hr) | Reserved 1yr | Spot avg |
| -------------- | ---: | -----: | --------------------------- | ---------------: | -----------: | -------: |
| `c6.large.4`   |    2 |   8 GB | general purpose worker node |           ~0.085 |       ~0.055 |   ~0.026 |
| `c6.xlarge.4`  |    4 |  16 GB | medium traffic workload     |           ~0.170 |       ~0.111 |   ~0.051 |
| `c6.2xlarge.4` |    8 |  32 GB | high traffic workload       |           ~0.340 |       ~0.221 |   ~0.102 |
| `c6.xlarge.2`  |    4 |   8 GB | compute-heavy workload      |           ~0.160 |       ~0.104 |   ~0.048 |

### ตัวอย่างการใช้งานใน Project

```
EKS Cluster
├── Node Pool: general (m6i.large × 3-10 nodes)
│   ├── Namespace: production
│   │   ├── Deployment: api-service (3 replicas)
│   │   ├── Deployment: worker-service (2 replicas)
│   │   └── HPA: api-service (min=3, max=20)
│   └── Namespace: monitoring
│       └── Deployment: prometheus, grafana
└── Node Pool: compute (c6i.xlarge × 1-5 nodes)
    └── Namespace: batch
        └── Job: data-processing
```

### Best Practice

- แยก Node Pool ตาม workload type (general, compute, memory)
- ใช้ Horizontal Pod Autoscaler (HPA) และ Cluster Autoscaler ร่วมกัน
- กำหนด Resource Requests และ Limits ทุก Pod เสมอ
- ใช้ Namespace แยก environment หรือ team
- update Kubernetes version สม่ำเสมอก่อน version หมด support

### Common Mistakes

- ไม่ได้กำหนด Resource Requests/Limits ทำให้ Pod หนึ่ง consume resource จน Pod อื่น evict
- deploy workload ใน `default` namespace ทั้งหมด
- ไม่ได้ตั้ง Pod Disruption Budget (PDB) ทำให้ maintenance ทำให้ downtime

---

## Container Instance / Serverless Container

### คืออะไร

Container Instance หรือ Serverless Container คือบริการ run container โดยไม่ต้องจัดการ cluster หรือ node เองเลย ผู้ใช้ระบุแค่ container image, CPU, Memory และ environment variable แล้ว Cloud จะ run ให้ทันที

### ใช้งานแบบไหน

ใช้ deploy container ที่ไม่ต้องการ orchestration ซับซ้อน เช่น API service เดี่ยว, background worker, task ที่ run แบบ one-off, หรือ microservice ขนาดเล็กที่ต้องการ deploy เร็ว

### เหมาะกับงานแบบไหน

เหมาะกับ stateless microservice, background job, event-driven worker, หรือ project ที่ต้องการ deploy container โดยไม่อยาก manage Kubernetes

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ inter-container communication ซับซ้อน หรือ workload ที่ต้องการ custom node configuration

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                    |
| -------------- | ----------------------------------------------- |
| AWS            | AWS Fargate (บน ECS หรือ EKS), AWS App Runner    |
| GCP            | Cloud Run                                       |
| Azure          | Azure Container Apps, Azure Container Instances |
| Huawei Cloud   | Cloud Container Instance (CCI)                  |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                                    | ตัวอย่าง                       |
| -------------------- | ----------------------------------------------------------- | ---------------------------- |
| vCPU                 | จำนวน virtual CPU ที่จัดสรรให้ container                         | 0.5, 1, 2, 4 vCPU            |
| Memory               | ขนาด memory ที่จัดสรรให้ container                              | 1 GB, 2 GB, 4 GB             |
| Container Image      | Docker image ที่ใช้ run                                        | `my-registry/api:v1.2.3`     |
| Min / Max Instances  | จำนวน container ต่ำสุดและสูงสุดสำหรับ scaling                       | min=1, max=10                |
| Concurrency          | จำนวน request ที่ container instance หนึ่งรับพร้อมกันได้ (Cloud Run) | 80 concurrent requests       |
| Timeout              | เวลาสูงสุดที่ request สามารถ process ได้                         | 300 seconds                  |
| Environment Variable | ค่าที่ inject เข้า container ตอน runtime                        | `DATABASE_URL`, `SECRET_KEY` |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม vCPU-second และ GB-second ที่ใช้จริง ไม่มีค่า idle

|                                     | AWS Fargate           | GCP Cloud Run | Azure Container Apps | Huawei CCI |
| ----------------------------------- | --------------------- | ------------- | -------------------- | ---------- |
| vCPU/hour                           | $0.04048              | $0.0864       | $0.0864              | ~$0.035    |
| GB Memory/hour                      | $0.004445             | $0.009        | $0.0108              | ~$0.004    |
| Request (per 1M)                    | —                     | $0.40         | $0.40                | —          |
| Scale-to-zero                       | ไม่รองรับ (ECS Fargate) | รองรับ         | รองรับ                | รองรับ      |
| ตัวอย่าง 1 vCPU + 2 GB RAM, 24/7/30วัน | ~$35/month            | ~$69/month    | ~$69/month           | ~$28/month |

> Cloud Run / Container Apps scale to zero — ช่วง idle ไม่มีค่าใช้จ่าย เหมาะสำหรับ workload ที่ traffic ไม่สม่ำเสมอ

### ตัวอย่างการใช้งานใน Project

Deploy REST API บน Cloud Run โดยไม่ต้องจัดการ cluster โดย Cloud Run จะ scale จาก 0 ถึง N instance ตาม traffic อัตโนมัติ ลดค่าใช้จ่ายเมื่อไม่มี request

### Best Practice

- เก็บ secret ใน Secret Manager ไม่ใช่ environment variable โดยตรง
- ตั้ง health check endpoint ให้ container ทุกตัว
- ใช้ min instances ≥ 1 เพื่อหลีกเลี่ยง cold start ถ้า latency สำคัญ

### Common Mistakes

- เก็บ database credential ใน environment variable แทน Secret Manager
- ไม่ได้กำหนด resource limit ทำให้ container ใช้ resource เกินที่คาด
- ไม่ทดสอบ cold start behavior
