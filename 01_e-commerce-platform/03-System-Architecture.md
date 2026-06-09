---
type: System Architecture
status: proposed
updated: 2026-04-24
date: 2026-04-24
---

# TECHNICAL DESIGN: e-Commerce Platform

## CLOUD ARCHITECTURE

![CLOUD ARCHITECTURE](asset/cloud-architecture-v2.drawio.png)

### Environment Strategy

- Development
- SIT / QA
- UAT
- Staging
- Production

### Network Architecture

ออกแบบ network architecture ให้แยก public/private boundary ชัดเจน รองรับ high availability, security และการ scale ของ microservices บน Kubernetes

- VPC (Virtual Private Cloud):
  - แยก environment ชัดเจน เช่น dev / staging / production
  - ใช้ Multi-AZ เพื่อรองรับ high availability และลดความเสี่ยงจาก AZ failure
  - แยก network boundary ระหว่าง internet-facing components, application workloads และ data layer

- Subnets:
  - Public Subnet:
    - Application Load Balancer (ALB): รับ traffic จาก internet และทำหน้าที่เป็น entry point หลักของระบบ
    - NAT Gateway: ใช้ให้ private workloads สามารถออก internet ได้โดยไม่ต้อง expose public IP

  - Private Application Subnet:
    - Kubernetes Nodes (EKS): ใช้ run microservices และ internal workloads
    - API Gateway (Kong): ทำหน้าที่ routing, authentication, rate limiting และ policy enforcement ก่อนส่งต่อไปยัง internal services
    - Internal Services: microservices ภายในระบบที่ไม่ควรถูกเรียกตรงจาก internet

  - Private Data Subnet:
    - Databases: เช่น PostgreSQL, Redis หรือ data stores อื่นๆ
    - Message Broker / Streaming Platform: เช่น Kafka, RabbitMQ หรือ AWS MSK ถ้ามีการใช้งาน event-driven architecture
    - จำกัดการเข้าถึงเฉพาะ service ที่จำเป็นเท่านั้น

- Security:
  - ใช้ Security Groups เพื่อควบคุม traffic ระดับ resource/service
  - ใช้ NACLs เพื่อควบคุม traffic ระดับ subnet เป็น defense-in-depth
  - ใช้ private communication ระหว่าง services ผ่าน internal network
  - ไม่ expose database หรือ internal services ออก public internet
  - จำกัด inbound traffic จาก internet ให้เข้าผ่าน ALB เท่านั้น
  - ใช้ IAM Roles for Service Accounts (IRSA) เพื่อให้ Kubernetes workload เข้าถึง AWS services แบบ least privilege

**Reason**:

- ALB ควรเป็น public-facing entry point หลัก ส่วน Kong/API Gateway ควรอยู่หลัง ALB เพื่อให้ควบคุม traffic และ security policy ได้ปลอดภัยกว่า
- แยก Private Application Subnet และ Private Data Subnet ช่วยลด blast radius หาก application layer มีปัญหาหรือถูกโจมตี
- EKS workloads ควรอยู่ใน private subnet เพื่อไม่ให้ service ภายในถูกเข้าถึงจาก internet โดยตรง
- Databases ควรอยู่ใน private data subnet และเปิดให้เข้าถึงเฉพาะ service ที่จำเป็นเท่านั้น
- Multi-AZ ช่วยให้ระบบยังทำงานได้หาก Availability Zone ใด Zone หนึ่งมีปัญหา
- NAT Gateway จำเป็นสำหรับ private workloads ที่ต้องออก internet เช่น pull image, call external API หรือ update package โดยไม่ต้องเปิด public IP
- Security Groups เหมาะกับการควบคุม access ราย service ส่วน NACLs เหมาะเป็น subnet-level protection เพิ่มเติม

### Compute Layer

ใช้ Kubernetes เป็น compute platform หลักสำหรับ deploy และ scale microservices โดยรองรับ workload ที่ต้องการ high availability, auto-scaling และแยก resource ตาม domain หรือ environment

- Stack:
  - [X] Amazon EKS: เหมาะกับระบบ microservices เพราะช่วยจัดการ Kubernetes control plane ให้, รองรับ auto-scaling, service discovery และ deployment pattern ที่เหมาะกับ distributed services
  - [X] containerd: เหมาะเป็น container runtime สำหรับ EKS เพราะเป็น runtime มาตรฐานที่ Kubernetes ใช้ในการ run container workload จริง
  - [ ] AWS Fargate for EKS: เหมาะเป็น option สำหรับ workload บางประเภทที่ไม่อยาก manage worker nodes เอง เช่น background jobs, lightweight services หรือ workload ที่ scale ไม่สม่ำเสมอ

**Reason**:

- Amazon EKS เหมาะกับการ deploy microservices หลายตัว เพราะรองรับ orchestration, rolling deployment, service discovery และ workload isolation
- HPA ช่วย scale pod ตาม metric เช่น CPU, memory หรือ custom metric ทำให้แต่ละ service scale ได้ตาม load ของตัวเอง
- Cluster Autoscaler หรือ Karpenter ช่วยเพิ่ม/ลด worker nodes ตาม resource ที่ pod ต้องการ ทำให้ใช้ infrastructure ได้คุ้มขึ้น
- ควรแยก namespace ตาม domain หรือ environment เพื่อช่วยจัดการสิทธิ์, resource quota, deployment และ monitoring ได้ง่ายขึ้น
- containerd ควรใช้เป็น container runtime แทน Docker ใน Kubernetes runtime layer
- Docker ยังใช้ได้ในฝั่ง development หรือ CI/CD สำหรับ build container image แต่ไม่ควรถูกระบุเป็น runtime หลักของ EKS
- AWS Fargate for EKS เป็น option ที่ดีถ้าต้องการลดภาระการดูแล node สำหรับ workload บางกลุ่ม แต่ไม่จำเป็นต้องใช้กับทุก microservice

### API & TRAFFIC MANAGEMENT

จัดการ traffic จาก Web/Mobile ผ่าน API Gateway, BFF และ service-to-service communication ภายใน EKS โดยแยกหน้าที่ของแต่ละ layer ให้ชัดเจน

#### 1. Edge / Public Entry Point

- Technology:
  - [X] AWS Application Load Balancer (ALB): เหมาะสำหรับรับ traffic จาก Web/Mobile และ forward เข้า API Gateway layer
  - [ ] AWS CloudFront: เหมาะเป็น option หากต้องการ CDN, caching และลด latency
  - [ ] AWS Global Accelerator: เหมาะเป็น option หากต้องการเพิ่ม availability และลด latency สำหรับ multi-region traffic

#### 2. API Gateway / Client-Facing Policy

- Technology:
  - [X] Kong API Gateway: เหมาะสำหรับ routing, authentication, rate limiting และ request validation ก่อนส่งต่อเข้า BFF
  - [ ] AWS API Gateway: เหมาะเป็น option หากต้องการ managed API Gateway ของ AWS
  - [ ] NGINX Ingress Controller: เหมาะเป็น option หากต้องการ ingress routing ที่เรียบง่ายกว่า Kong

#### 3. Backend for Frontend / Client Aggregation

- Technology:
  - [X] Node.js + Apollo GraphQL: เหมาะกับ BFF เพราะ aggregate data จากหลาย microservices ได้ดี และ optimize response ให้ Web/Mobile ได้
  - [ ] Node.js + NestJS GraphQL: เหมาะถ้าต้องการโครงสร้าง BFF ที่เป็นระบบและ maintain ง่าย
  - [ ] .NET + Hot Chocolate GraphQL: เหมาะถ้า backend team ใช้ .NET เป็นหลัก

#### 4. Internal Service Communication

- Technology:
  - [X] Istio Service Mesh: เหมาะสำหรับ service-to-service communication ภายใน EKS เช่น mTLS, retry, timeout และ circuit breaking
  - [ ] Linkerd: เหมาะเป็น option หากต้องการ service mesh ที่เบากว่า Istio
  - [ ] AWS App Mesh: เหมาะเป็น option หากต้องการ service mesh ที่ integrate กับ AWS ecosystem

#### Traffic Flow

Client (Web / Mobile)
→ ALB
→ Kong API Gateway
→ BFF (GraphQL)
→ Internal Microservices
→ Databases / Message Broker

**Reason**:

- ALB เป็น public entry point สำหรับรับ traffic จาก client
- Kong ใช้ควบคุม API policy ก่อน request เข้า BFF
- BFF ใช้ aggregate data และ optimize response ตาม client type
- Istio ใช้ดูแล internal service-to-service traffic ภายใน EKS
- แยก API Gateway และ Service Mesh ออกจากกัน ทำให้ responsibility ชัดเจนและลดการ expose microservices ออก internet

### DATA LAYER

จัดการข้อมูลหลักของระบบ โดยแยก database, replication, search และ cache ตาม workload เพื่อให้แต่ละส่วน scale, replicate และ optimize ได้เหมาะสม

#### 1. Relational Database

- Technology:
  - [X] Amazon RDS for PostgreSQL: เหมาะสำหรับ service หลักที่ต้องการ transaction และ data consistency เช่น Order, Payment, User และ Financial
  - [ ] Amazon Aurora PostgreSQL: เหมาะเป็น option หากต้องการ performance, high availability และ scalability สูงกว่า RDS PostgreSQL ทั่วไป
  - [ ] Amazon DynamoDB: เหมาะเป็น option สำหรับข้อมูลที่ access pattern ชัดเจน และต้องการ scale แบบ high throughput

**Reason**:

- PostgreSQL เหมาะกับข้อมูลหลักของระบบที่ต้องการ transaction, relational data และ data integrity
- Multi-AZ ช่วยเพิ่ม high availability สำหรับ production workload
- Automated backup และ point-in-time recovery ช่วยลดความเสี่ยงจากข้อมูลเสียหายหรือ human error

#### 2. Database Replication

- Technology:
  - [X] RDS Multi-AZ Replication: แนะนำให้ใช้กับ Amazon RDS เพื่อ replicate database ไปยัง standby instance ใน Availability Zone อื่น และรองรับ automatic failover เมื่อ primary database มีปัญหา
  - [X] RDS Read Replica: แนะนำให้ใช้เมื่อมี read workload สูง เช่น report, dashboard, product listing หรือ read-heavy service เพื่อลด load จาก primary database
  - [ ] Cross-Region Read Replica: เหมาะเป็น option หากต้องการ disaster recovery หรือเตรียม database สำรองใน region อื่น

**Reason**:

- ถ้าใช้ Amazon RDS ใน production ควรเปิด Multi-AZ เพื่อเพิ่ม availability และลดความเสี่ยงจาก AZ failure
- Read Replica ช่วยแยก read workload ออกจาก primary database ทำให้ primary รับภาระ write transaction ได้ดีขึ้น
- Cross-Region Read Replica เหมาะสำหรับระบบที่ต้องการ disaster recovery ระดับ region
- การแยก replication strategy ออกจาก database หลักช่วยให้เห็นชัดว่า database ไม่ได้มีแค่ storage แต่ต้องรองรับ availability, scalability และ recovery ด้วย

### SEARCH LAYER

แยก search workload ออกจาก database หลัก เพื่อรองรับ product search, filtering, sorting และ full-text search ได้มีประสิทธิภาพมากขึ้น

- Technology:
  - [X] Amazon OpenSearch: เหมาะสำหรับ product search, full-text search, filtering, sorting และ search analytics
  - [ ] Elasticsearch Self-Managed: เหมาะเป็น option หากต้องการควบคุม cluster configuration เอง
  - [ ] Meilisearch: เหมาะเป็น option สำหรับ search ที่ต้องการ setup ง่ายและ lightweight กว่า OpenSearch

**Reason**:

- OpenSearch เหมาะกับ product search เพราะรองรับ full-text search, filtering, sorting และ autocomplete ได้ดี
- ช่วยลดภาระ relational database ไม่ให้ต้องรับ search query ที่ซับซ้อน
- สามารถ scale search workload แยกจาก transactional workload ได้

### CACHE LAYER

จัดการข้อมูลอายุสั้นหรือข้อมูลที่ถูกเรียกบ่อย เพื่อเพิ่มความเร็วของระบบและลด load ที่ database หรือ internal services

- Technology:
  - [X] Amazon ElastiCache for Redis: เหมาะสำหรับ caching, session, token metadata, rate limit counter และ temporary data ที่ต้องการ response เร็ว
  - [ ] Amazon ElastiCache for Memcached: เหมาะเป็น option สำหรับ simple cache ที่ไม่ต้องใช้ data structure ของ Redis
  - [ ] Redis Self-Managed: เหมาะเป็น option หากต้องการควบคุม configuration เอง แต่ต้องรับภาระ operation เพิ่มขึ้น

**Reason**:

- Redis เหมาะกับข้อมูลที่ต้องการ response เร็ว เช่น cache, session และ token metadata
- ช่วยลด load ที่ database และ internal services
- รองรับ data structure ที่ยืดหยุ่นกว่าสำหรับ use case เช่น rate limiting, queue เบา ๆ หรือ temporary state

### MESSAGING & EVENT STREAMING

รองรับ event-driven architecture และ async communication ระหว่าง microservices เพื่อลด coupling และเพิ่ม reliability ของระบบ

- Technology:
  - [X] Amazon MSK (Apache Kafka): เหมาะสำหรับ event-driven architecture, async communication และ event streaming ระหว่าง microservices
  - [ ] Amazon SQS: เหมาะเป็น option สำหรับ simple queue และ async job ที่ไม่ต้องการ event streaming ซับซ้อน
  - [ ] RabbitMQ: เหมาะเป็น option สำหรับ message queue pattern ที่ต้องการ routing flexible และ command-based messaging

**Reason**:

- Kafka/MSK เหมาะกับ microservices ที่ต้องการ publish/subscribe event เช่น OrderCreated, PaymentCompleted หรือ InventoryReserved
- รองรับ async communication ทำให้ services ไม่ต้องเรียกกันแบบ synchronous ตลอดเวลา
- ใช้ร่วมกับ Outbox Pattern เพื่อให้ database transaction และ event publishing reliable มากขึ้น

### SECURITY & SECRETS

จัดการ security foundation ของระบบ ตั้งแต่ secrets, encryption keys, service identity และ access control เพื่อให้ทุก service เข้าถึง resource ได้อย่างปลอดภัยตามหลัก least privilege

#### 1. Secrets Management

- Technology:
  - [X] AWS Secrets Manager: เหมาะสำหรับเก็บ secrets เช่น database credentials, API keys, OAuth secrets และรองรับ secret rotation
  - [ ] AWS Systems Manager Parameter Store: เหมาะเป็น option สำหรับ configuration หรือ secret ที่ไม่ซับซ้อน และต้องการ cost ต่ำกว่า
  - [ ] HashiCorp Vault: เหมาะเป็น option หากต้องการ secrets management กลางที่รองรับ multi-cloud หรือ on-premise

**Reason**:

- ใช้เก็บข้อมูลลับที่ไม่ควรอยู่ใน source code หรือ environment file ตรงๆ
- เหมาะกับ secrets ที่ต้องการ rotation เช่น database credentials หรือ external API keys
- ช่วยลดความเสี่ยงจาก credential leak และทำให้การจัดการ secrets เป็นมาตรฐานเดียวกันทั้งระบบ

#### 2. Key Management & Encryption

- Technology:
  - [X] AWS KMS: เหมาะสำหรับจัดการ encryption keys ที่ใช้กับ AWS services เช่น RDS, S3, EBS, Secrets Manager และ application-level encryption
  - [ ] AWS CloudHSM: เหมาะเป็น option หากต้องการ dedicated hardware security module สำหรับ compliance ระดับสูง
  - [ ] External Key Store (XKS): เหมาะเป็น option หากองค์กรต้องการควบคุม encryption key จากระบบภายนอก AWS

**Reason**:

- ใช้จัดการ encryption keys สำหรับข้อมูลที่ต้องเข้ารหัสทั้ง at-rest และบางกรณีใน application layer
- Integrate กับ AWS services ได้โดยตรง ทำให้ enforce encryption ได้ง่าย
- รองรับ audit trail ผ่าน CloudTrail เพื่อดูการใช้งาน key ย้อนหลังได้

#### 3. AWS Access Control (If Required)

- Technology:
  - [X] AWS IAM: เหมาะสำหรับควบคุมสิทธิ์ของ AWS users, roles, policies และ service access ตามหลัก least privilege
  - [ ] AWS IAM Identity Center: เหมาะเป็น option สำหรับจัดการ SSO และ user access ของทีมภายในองค์กร
  - [ ] AWS Organizations SCP: เหมาะเป็น option สำหรับควบคุม permission boundary ระดับ account หรือ organization

**Reason**:

- ใช้ควบคุมว่า user, role หรือ service ใดสามารถเข้าถึง AWS resource อะไรได้บ้าง
- ช่วยลดความเสี่ยงจาก permission ที่กว้างเกินจำเป็น
- เหมาะกับการแยกสิทธิ์ตาม environment เช่น dev, staging และ production

#### 4. Kubernetes Workload Identity (If Required)

- Technology:
  - [X] IAM Roles for Service Accounts (IRSA): เหมาะสำหรับให้ workload บน EKS เข้าถึง AWS services โดยไม่ต้องฝัง access key ไว้ใน pod
  - [ ] EKS Pod Identity: เหมาะเป็น option ใหม่สำหรับจัดการ AWS permission ให้ Kubernetes workload ง่ายขึ้น
  - [ ] Kubernetes RBAC: เหมาะสำหรับควบคุมสิทธิ์ภายใน Kubernetes cluster เช่น namespace, pod, deployment และ service account

**Reason**:

- ช่วยให้แต่ละ microservice บน EKS ได้ permission เฉพาะ resource ที่ต้องใช้จริง
- ลดความเสี่ยงจากการเก็บ AWS credential ไว้ใน container หรือ config
- แยก responsibility ระหว่าง AWS permission และ Kubernetes permission ได้ชัดเจน

### OBSERVABILITY

ติดตามสุขภาพของระบบผ่าน metrics, logs และ distributed tracing เพื่อให้ตรวจสอบ performance, error และ request flow ของ microservices ได้ชัดเจน

#### 1. Metrics & Dashboard

- Technology:
  - [X] Prometheus + Grafana: เหมาะสำหรับเก็บ metrics, สร้าง dashboard และ monitor service health ของ microservices บน EKS
  - [ ] Amazon Managed Service for Prometheus: เหมาะเป็น option หากต้องการ Prometheus แบบ managed service ลดภาระ operation
  - [ ] Amazon CloudWatch Metrics: เหมาะเป็น option สำหรับ monitor AWS resources และ infrastructure metrics

**Reason**:

- Prometheus เหมาะกับ Kubernetes workload เพราะ scrape metrics จาก service และ pod ได้ดี
- Grafana ช่วย visualize metrics เป็น dashboard สำหรับดู health, latency, throughput และ error rate
- เหมาะสำหรับทำ alerting จาก metric สำคัญ เช่น CPU, memory, request rate และ service error

#### 2. Logging

- Technology:
  - [X] Loki: เหมาะสำหรับ centralized logging ที่ทำงานร่วมกับ Grafana ได้ดี และเหมาะกับ Kubernetes logs
  - [ ] Amazon CloudWatch Logs: เหมาะเป็น option หากต้องการ managed logging ที่ integrate กับ AWS โดยตรง
  - [ ] OpenSearch Logs: เหมาะเป็น option หากต้องการ log search และ analytics ที่ซับซ้อนกว่า

**Reason**:

- Loki เหมาะกับการรวม logs จาก microservices หลายตัวไว้ที่เดียว
- ทำงานร่วมกับ Grafana ได้ดี ทำให้ดู metrics และ logs ใน context เดียวกันได้
- ช่วย debug incident ได้ง่ายขึ้นจากการค้นหา logs ตาม service, namespace, pod หรือ correlation id

#### 3. Distributed Tracing

- Technology:
  - [X] OpenTelemetry + Jaeger: เหมาะสำหรับ trace request flow ข้ามหลาย microservices และช่วยวิเคราะห์ latency bottleneck
  - [ ] AWS X-Ray: เหมาะเป็น option หากต้องการ tracing ที่ integrate กับ AWS ecosystem โดยตรง
  - [ ] Grafana Tempo: เหมาะเป็น option หากต้องการ tracing ที่ integrate กับ Grafana stack

**Reason**:

- OpenTelemetry เป็นมาตรฐานกลางสำหรับเก็บ traces, metrics และ logs จาก application
- Jaeger ช่วยดู request path ข้ามหลาย services ทำให้เห็นว่า latency หรือ error เกิดที่ service ไหน
- เหมาะกับ microservices เพราะ request หนึ่งรายการอาจผ่านหลาย service ก่อนจบ flow

### CI/CD

จัดการ source code, build pipeline และ deployment ไปยัง Kubernetes โดยใช้แนวทาง GitOps เพื่อให้ deployment ตรวจสอบย้อนหลังได้และลด manual operation

#### 1. Source Control

- Technology:
  - [X] GitHub: เหมาะสำหรับจัดการ source code, pull request, code review และทำงานร่วมกับ GitHub Actions ได้โดยตรง
  - [ ] GitLab: เหมาะเป็น option หากต้องการ platform ที่รวม source control, CI/CD และ DevOps workflow ไว้ในตัว
  - [ ] AWS CodeCommit: เหมาะเป็น option หากต้องการ source control ที่อยู่ใน AWS ecosystem โดยตรง

**Reason**:

- Source control เป็นจุดเริ่มต้นของ development workflow และ code review
- GitHub เหมาะกับทีมที่ต้องการ ecosystem กว้าง ใช้งานง่าย และ integrate กับ CI/CD tools ได้ดี
- ช่วยให้ทุก change ผ่าน pull request และตรวจสอบย้อนหลังได้

#### 2. CI Pipeline

- Technology:
  - [X] GitHub Actions: เหมาะสำหรับ build, test, scan และ publish container image จาก repository ได้โดยตรง
  - [ ] GitLab CI: เหมาะเป็น option หากใช้ GitLab เป็น source control หลัก
  - [ ] AWS CodeBuild: เหมาะเป็น option หากต้องการ build pipeline ที่อยู่ใน AWS ecosystem

**Reason**:

- CI Pipeline ใช้ตรวจสอบคุณภาพ code ก่อน deploy เช่น build, unit test, lint และ security scan
- GitHub Actions เหมาะถ้าใช้ GitHub เพราะ setup ง่ายและทำงานร่วมกับ repository ได้โดยตรง
- ช่วยลดความเสี่ยงจากการ deploy code ที่ยังไม่ผ่าน quality gate

#### 3. GitOps Deployment

- Technology:
  - [X] ArgoCD: เหมาะสำหรับ deploy application เข้า Kubernetes ด้วย GitOps โดยให้ Git เป็น source of truth
  - [ ] FluxCD: เหมาะเป็น option หากต้องการ GitOps tool ที่ lightweight และ Kubernetes-native
  - [ ] AWS CodePipeline: เหมาะเป็น option หากต้องการ managed deployment pipeline ใน AWS

**Reason**:

- ArgoCD เหมาะกับ EKS เพราะช่วย sync manifest จาก Git ไปยัง Kubernetes cluster ได้ชัดเจน
- GitOps ทำให้ deployment history, rollback และ environment state ตรวจสอบได้จาก Git
- ลด manual deployment และช่วยให้ dev / staging / production ควบคุม version ได้เป็นระบบ

#### 4. Kubernetes Packaging

- Technology:
  - [X] Helm Charts: เหมาะสำหรับ package Kubernetes manifest และจัดการ configuration แยกตาม environment
  - [ ] Kustomize: เหมาะเป็น option หากต้องการ customize manifest แบบไม่ต้องใช้ template engine
  - [ ] Plain Kubernetes YAML: เหมาะเป็น option สำหรับระบบเล็กที่ manifest ยังไม่ซับซ้อน

**Reason**:

- Helm Charts ช่วยจัดการ Kubernetes deployment ที่มีหลาย service และหลาย environment ได้ง่ายขึ้น
- แยก values ของ dev / staging / production ได้ชัดเจน
- ใช้ร่วมกับ ArgoCD ได้ดีสำหรับ GitOps workflow

### HIGH AVAILABILITY & SCALABILITY

ออกแบบระบบให้รองรับ traffic สูง, scale ได้ตาม load และลดผลกระทบเมื่อ infrastructure บางส่วนมีปัญหา โดยใช้ Multi-AZ, auto scaling, load balancing และ stateless service design

#### 1. High Availability

- Technology:
  - [X] Multi-AZ Deployment: เหมาะสำหรับกระจาย workload ข้ามหลาย Availability Zones เพื่อลดความเสี่ยงจาก AZ failure
  - [ ] Multi-Region Deployment: เหมาะเป็น option หากต้องการ disaster recovery หรือรองรับผู้ใช้หลาย region
  - [ ] Active-Passive DR: เหมาะเป็น option หากต้องการ recovery environment ที่ cost ต่ำกว่า active-active

**Reason**:

- Multi-AZ ช่วยให้ระบบยังทำงานได้เมื่อ Availability Zone ใด Zone หนึ่งมีปัญหา
- เหมาะกับ production workload ที่ต้องการ high availability
- ใช้ร่วมกับ ALB, EKS, RDS และ MSK เพื่อกระจาย workload และลด single point of failure

#### 2. Auto Scaling

- Technology:
  - [X] EKS + HPA: เหมาะสำหรับ scale pod ของ microservices ตาม load เช่น CPU, memory หรือ custom metrics
  - [ ] Karpenter: เหมาะเป็น option สำหรับ scale worker nodes ได้เร็วและยืดหยุ่นกว่า Cluster Autoscaler
  - [ ] Cluster Autoscaler: เหมาะเป็น option พื้นฐานสำหรับเพิ่ม/ลด worker nodes ตาม resource ที่ pod ต้องการ

**Reason**:

- HPA ช่วยให้แต่ละ service scale ได้ตาม workload ของตัวเอง
- EKS รองรับการเพิ่ม/ลด compute capacity ตาม traffic ที่เปลี่ยนแปลง
- Node autoscaling ช่วยให้ cluster มี resource เพียงพอสำหรับ pod ที่เพิ่มขึ้น

#### 3. Load Balancing

- Technology:
  - [X] AWS Application Load Balancer (ALB): เหมาะสำหรับกระจาย external traffic จาก Web/Mobile เข้า API Gateway หรือ service entry point
  - [X] Istio Service Mesh: เหมาะสำหรับกระจาย internal service-to-service traffic พร้อม retry, timeout และ circuit breaking
  - [ ] NGINX Ingress Controller: เหมาะเป็น option หากต้องการ ingress routing ที่เรียบง่ายกว่า ALB/Kong setup

**Reason**:

- ALB ช่วยกระจาย traffic จาก client ไปยัง entry point ของระบบ
- Istio ช่วยจัดการ traffic ภายในระหว่าง microservices ให้ resilient มากขึ้น
- แยก external load balancing และ internal traffic management ทำให้ responsibility ชัดเจน

#### 4. Stateless Service Design

- Technology:
  - [X] Stateless Services + Externalized State: เหมาะสำหรับ microservices เพราะช่วยให้ scale horizontal ได้ง่ายและไม่ผูก state กับ pod ใด pod หนึ่ง
  - [ ] Sticky Session: เหมาะเป็น option เฉพาะกรณีที่มี legacy workload หรือ session state ที่ยังย้ายออกไม่ได้
  - [ ] StatefulSet: เหมาะเป็น option สำหรับ workload ที่จำเป็นต้องมี identity หรือ persistent state เช่น database บางประเภท

**Reason**:

- Stateless service ทำให้ pod ถูกเพิ่ม/ลด/replace ได้ง่ายโดยไม่กระทบ user session หรือ business state
- Externalized state เช่น database, Redis, S3 หรือ message broker ช่วยให้ service scale ได้อย่างปลอดภัย
- เหมาะกับ Kubernetes เพราะ workload สามารถ reschedule ไป node อื่นได้โดยไม่สูญเสีย state

### DISASTER RECOVERY

ออกแบบแนวทางกู้คืนระบบและข้อมูลเมื่อเกิด incident เช่น database failure, data corruption, accidental delete หรือ region-level outage

#### 1. Database Backup & Recovery

- Technology:
  - [X] RDS Automated Backup / Snapshot: เหมาะสำหรับ backup database หลัก และรองรับ point-in-time recovery
  - [ ] Cross-Region Read Replica: เหมาะเป็น option หากต้องการเตรียม database สำรองข้าม region
  - [ ] AWS Backup: เหมาะเป็น option หากต้องการจัดการ backup policy หลาย resource แบบรวมศูนย์

**Reason**:

- RDS automated backup ช่วยลดความเสี่ยงจากข้อมูลเสียหายหรือ human error
- Snapshot ใช้สำหรับ restore database กลับมาในช่วงเวลาที่ต้องการ
- Point-in-time recovery เหมาะกับระบบ transaction เช่น Order, Payment และ Financial

#### 2. Object Storage Protection

- Technology:
  - [X] S3 Versioning: เหมาะสำหรับป้องกันการลบหรือแก้ไขไฟล์ผิดพลาด เช่น product images, documents และ exported files
  - [ ] S3 Cross-Region Replication: เหมาะเป็น option หากต้องการ replicate object ไปยัง region สำรอง
  - [ ] S3 Object Lock: เหมาะเป็น option หากต้องการป้องกันการลบหรือแก้ไขไฟล์ตาม compliance requirement

**Reason**:

- S3 Versioning ช่วยให้สามารถกู้คืนไฟล์เวอร์ชันก่อนหน้าได้
- ลดความเสี่ยงจาก accidental delete หรือ overwrite
- เหมาะกับไฟล์สำคัญ เช่น documents, invoices, reports และ product assets

#### 3. Multi-Region Recovery

- Technology:
  - [X] Multi-Region Replication (Optional): เหมาะเป็น option สำหรับระบบที่ต้องการ disaster recovery ระดับ region
  - [ ] Active-Passive DR: เหมาะหากต้องการ standby environment ที่พร้อมใช้งานเมื่อ primary region มีปัญหา
  - [ ] Active-Active DR: เหมาะหากต้องการให้หลาย region รับ traffic ได้พร้อมกัน แต่มี complexity และ cost สูงกว่า

**Reason**:

- Multi-region replication ช่วยลดผลกระทบจาก region-level outage
- Active-Passive เหมาะกับระบบที่ต้องการ DR แต่ยังควบคุม cost ได้
- Active-Active เหมาะกับระบบ mission-critical ที่ต้องการ availability สูงมาก แต่ต้องออกแบบ data consistency และ traffic routing ให้รอบคอบ

---
