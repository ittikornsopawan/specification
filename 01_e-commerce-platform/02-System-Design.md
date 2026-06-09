---
type: System Design
status: proposed
updated: 2026-04-24
date: 2026-04-24
---

# ARCHITECTURE SNAPSHOT

## MICROSERVICES ARCHITECTURE

### CODE STRUCTURE

![CODE STRUCTURE-](asset/code-structure.drawio.png)

- Domain-Driven Design (DDD) + Clean Architecture
  - ใช้ DDD เพื่อแยก Business Domain ของ e-commerce เช่น Order, Product, Inventory, Payment, Campaign และ Fulfillment ออกเป็น Bounded Context
  - แต่ละ Domain สามารถพัฒนาและ deploy แยกกันได้ ลด coupling และเพิ่ม scalability
  - Clean Architecture ใช้เพื่อแยก layer ของ code ให้ maintain และ test ได้ง่าย
  - Layers:
    - Presentation Layer:
      - รับ request จาก BFF / API Gateway
      - ทำ validation และ map request → use case
      - ไม่มี business logic
    - Application Layer:
      - orchestrate use case ภายใน service เช่น Create Order, Apply Promotion
      - เรียก Domain + Repository
      - publish domain event
    - Domain Layer:
      - เก็บ business logic หลักของระบบ
      - ประกอบด้วย Entity, Value Object, Aggregate และ Domain Service
      - เช่น:
        - Order Aggregate (order, items, pricing, status)
        - Inventory Aggregate (stock, reservation)
    - Infrastructure Layer:
      - เชื่อมต่อ database, cache, message queue และ external services
      - implement repository และ integration
  - Tactical Patterns:
    - เป็น pattern สำหรับจัดโครงสร้าง Domain Model และ Business Logic
    - Entity - Object ที่มี identity และมี lifecycle
      - เช่น Order, Product, User
    - Value Object - Object ที่ไม่มี identity และ immutable
      - เช่น Money, Address, Discount
    - Aggregate - กลุ่มของ Entity/Value Object ที่ควบคุมผ่าน Aggregate Root
      - เช่น Order Aggregate (Order + OrderItem)
    - Domain Event - เหตุการณ์ที่สะท้อนการเปลี่ยนแปลงใน domain
      - เช่น OrderCreated, PaymentCompleted
    - Repository - abstraction สำหรับ persist aggregate
  - CQRS:
    - Command: สำหรับ write เช่น Create Order, Reserve Inventory
    - Query: สำหรับ read เช่น Product Listing, Order History
    - แยก read/write model เพื่อ optimize performance
  - Event-driven:
    - ใช้ Domain Event + Message Queue เพื่อ decouple service
    - รองรับ eventual consistency
    - flow ตัวอย่าง: Order → Inventory → Payment → Shipping

### PROTOCOL

![CODE STRUCTURE-](asset/protocol.drawio.png)

- gRPC (Service-to-Service)
  - ใช้สำหรับ synchronous communication ระหว่าง microservices
  - ใช้ protobuf (schema-first) และ HTTP/2 → latency ต่ำ
  - เหมาะกับ service ที่เรียกกันบ่อย เช่น Order ↔ Inventory ↔ Payment
- RESTful (Client-to-System)
  - ใช้สำหรับ Client (Web / Mobile / Partner) ผ่าน API Gateway หรือ BFF
  - ใช้ resource-based + JSON
  - รองรับ authentication เช่น JWT, OAuth2
- Event-driven (Async Communication)
  - ใช้สำหรับ async workflow ระหว่าง service
  - ใช้ Message Broker เช่น Kafka / RabbitMQ / NATS
  - ตัวอย่าง flow:
    - OrderCreated → InventoryReserved → PaymentCompleted → OrderShipped
  - ลด coupling และรองรับ scalability
- Fault Tolerance
  - Retry: retry เมื่อเกิด transient failure
    - ใช้ exponential backoff และ jitter เพื่อลด load
  - Circuit Breaker: ป้องกัน cascading failure
    - เปิด circuit เมื่อ failure rate สูง และปิดเมื่อระบบกลับมา
  - Timeout: ป้องกัน request ค้าง
    - กำหนด timeout ที่เหมาะสมสำหรับแต่ละ call
  - Idempotency: ป้องกันการประมวลผลซ้ำ (เช่น payment)
    - ใช้ idempotency key และตรวจสอบก่อนประมวลผล

## ARCHITECTURE PATTERN

- Event Driven Architecture
  - ใช้ Domain Event เพื่อสื่อสารระหว่าง Microservices
  - service publish/subscribe event แทน synchronous call
  - ลด coupling และรองรับ scalability
  - Example:
    - OrderCreated → Inventory reserve stock
    - InventoryReserved → Payment charge
    - PaymentCompleted → Shipping process
- Saga Pattern (Choreography-based)
  - ใช้จัดการ distributed transaction ผ่าน event-driven flow
  - แต่ละ service ทำงานเป็น step และ publish event ต่อ
  - หากล้มเหลว ใช้ compensation action
  - Example:
    - Order → Create Order
    - Inventory → Reserve Stock
    - Payment → Process Payment (fail)
    - Compensation:
      - Inventory → Release Stock
      - Order → Cancel Order
- Outbox Pattern
  - บันทึก event ลง database ก่อน (outbox table)
  - worker publish event ไป message broker
  - ป้องกัน event loss และ data inconsistency
- Idempotent Consumer
  - รองรับ at-least-once delivery
  - event ซ้ำต้องไม่ทำให้เกิด side effect
  - ใช้ event id / unique constraint ตรวจสอบความซ้ำ

## BACKEND API

### Identity & Access Layer

Service: **IAM**

จัดการการยืนยันตัวตน การกำหนดสิทธิ์ และการควบคุมการเข้าถึงของผู้ใช้งานและ service ภายในระบบ รวมถึง RBAC, permission, session, token และ audit trail เพื่อให้ระบบมีความปลอดภัย ตรวจสอบย้อนหลังได้ และรองรับการขยายตัว

- Stack:
  - Backend:
    - [X] GoLang: เหมาะกับ service ที่ต้องการ low latency, high concurrency และ resource efficiency โดยเฉพาะ endpoint ที่ถูกเรียกบ่อย เช่น login, token validation, permission check และ session validation
    - [ ] .NET Core C#: เหมาะกับ enterprise microservices โดยเฉพาะองค์กรที่ใช้ Microsoft ecosystem เช่น Azure, SQL Server, Entra ID หรือทีมที่ถนัด Clean Architecture และ shared class library
  - Database:
    - SQL:
      - [X] PostgreSQL: เหมาะเป็น primary database ของ IAM เพราะรองรับ relational data, transaction, constraint, indexing, JSONB และมีความน่าเชื่อถือสูง เหมาะกับ user, role, permission, policy, tenant และ audit metadata
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก backend ใช้ .NET Core C# และองค์กรอยู่ใน Microsoft ecosystem เพราะ integration กับ EF Core, tooling, stored procedures และ enterprise support ดีมาก
    - Log Storage:
      - [X] PostgreSQL Audit Table / Event Log: ใช้เก็บ audit log สำคัญ เช่น role changed, permission granted, login failed, password changed และ session revoked
      - [ ] Cassandra: เป็น option สำหรับ high-volume, write-heavy log/event storage กรณีระบบมี traffic สูงมาก หรือจำเป็นต้อง scale แบบ distributed/multi-region
- Integration:
  - User Service
  - Token Service
  - Session Service
  - Tenant Service
  - Notification Service
  - Audit / Logging Service
  - KMS / Secret Management

**Reason**:

- IAM เป็น service ที่ต้องการ performance ดี
- request เยอะ
- logic มักเป็น stateless-heavy เช่น permission check, policy check, service authorization
- Go เหมาะกับ service ที่ต้องเบา เร็ว deploy ง่าย
- PostgreSQL เหมาะกับ RBAC, permission, role, policy, tenant mapping

---

Service: **Privacy**

จัดการ Personally Identifiable Information (PII), Sensitive Data และข้อมูลส่วนบุคคลของผู้ใช้งานให้สอดคล้องกับ PDPA และ GDPR โดยครอบคลุม Data Encryption, Data Masking, Access Control, Audit Logging, Data Retention และ Data Subject Rights เช่น Right to Access, Right to Rectification, Right to Erasure และ Data Portability โดยไม่รวม Consent Management

- Stack:
  - Backend:
    - [X] GoLang: Privacy Service ต้องทำ encryption, masking, access validation และ data processing บ่อย จึงเหมาะกับ GoLang ที่ทำงานเร็ว ใช้ resource ต่ำ และรองรับ request จำนวนมากได้ดี
    - [ ] .NET Core C#: เหมาะเป็น option หาก Privacy Service อยู่ในองค์กรที่ใช้ Microsoft ecosystem และต้องการ integration กับ enterprise security/compliance tooling
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Privacy Service มี policy, workflow หรือ compliance rule ซับซ้อน และทีมมีมาตรฐาน Java ecosystem อยู่แล้ว
  - Database:
    - SQL:
      - [X] PostgreSQL: Privacy Service ต้องเก็บ PII metadata, data classification, retention policy และ data subject request ที่ต้องการ transaction, consistency และ query ที่ยืดหยุ่น จึงเหมาะกับ PostgreSQL
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก Privacy Service ต้องทำงานร่วมกับ Microsoft ecosystem และต้องพึ่ง enterprise security feature ของ SQL Server
    - Log Storage:
      - [X] PostgreSQL Audit Table / Event Log: Privacy Service ต้องมี audit trail ที่ตรวจสอบย้อนหลังได้สำหรับ action สำคัญ เช่น view, mask, export, delete และ key rotation จึงควรเก็บ audit log สำคัญใน PostgreSQL
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หาก Privacy Service มี privacy event log ปริมาณมาก หรือต้องค้นหา audit trail จำนวนมากแบบรวดเร็ว
- Integration:
  - User Service
  - Tenant Service
  - IAM Service
  - Audit / Logging Service
  - KMS / Secret Management
  - Data Export Service
  - Notification Service

**Reason**:

- Privacy Service ต้องควบคุมการเข้าถึง PII และ Sensitive Data ตาม PDPA/GDPR
- ระบบต้องรองรับการ mask, encrypt, export, delete และจัดการ data subject request
- ข้อมูลหลักต้องมี consistency เพราะเกี่ยวข้องกับ policy, retention rule และสถานะคำขอของเจ้าของข้อมูล
- ทุก action สำคัญต้องมี audit trail เพื่อบอกได้ว่าใครทำอะไร กับข้อมูลใด และเมื่อไหร่
- cache ต้องใช้เฉพาะ rule หรือ metadata ที่ไม่ใช่ PII เพื่อลด latency โดยไม่เพิ่มความเสี่ยง
- encryption key ต้องแยกจาก application เพื่อรองรับ key rotation และลดความเสี่ยงจาก key leakage

---

Service: **User**

จัดการ User Account, User Profile และ User-related Domain Logic เช่น ข้อมูลส่วนตัว การตั้งค่าผู้ใช้ สถานะบัญชี ความสัมพันธ์กับ Tenant และข้อมูลประกอบของผู้ใช้ โดยไม่รวม Authentication Token และ Session Lifecycle

- Stack:
  - Backend:
    - [ ] GoLang: เหมาะเป็น option หาก User Service ต้องรองรับ request ปริมาณมากและต้องการ service ที่เบา แต่ไม่ใช่ตัวเลือกหลักเมื่อระบบเน้น enterprise domain logic และ Microsoft ecosystem
    - [X] .NET Core C#: User Service มักมี business logic, validation, profile workflow และ integration กับระบบองค์กร จึงเหมาะกับ .NET Core C# ที่จัดโครงสร้างแบบ Clean Architecture, shared class library และ enterprise microservices ได้ดี
    - [ ] Java Spring Boot: เหมาะเป็น option หาก User Service อยู่ในองค์กรที่ใช้ Java ecosystem หรือมี workflow และ domain model ขนาดใหญ่
  - Database:
    - SQL:
      - [ ] PostgreSQL: เหมาะเป็น option หากต้องการ database ที่ยืดหยุ่นและรองรับ relational data ได้ดี แต่ไม่ใช่ตัวเลือกหลักเมื่อ backend และองค์กรเลือก Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option สำหรับองค์กรขนาดใหญ่ที่มี Oracle ecosystem อยู่แล้ว หรือมี requirement ด้าน legacy enterprise system
      - [X] Microsoft SQL Server: User Service ต้องเก็บข้อมูล account, profile, user setting, tenant mapping และสถานะผู้ใช้ที่ต้องการ transaction และ consistency จึงเหมาะกับ SQL Server เมื่อ backend ใช้ .NET Core C# และต้องการ integration กับ Microsoft ecosystem
    - Log Storage:
      - [X] Microsoft SQL Server Audit Table / Event Log: User Service ต้องบันทึกการเปลี่ยนแปลงข้อมูลผู้ใช้ เช่น profile updated, account disabled, email changed และ tenant assigned เพื่อใช้ตรวจสอบย้อนหลัง
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หากมี user event log ปริมาณมาก หรือต้องค้นหา activity history จำนวนมากแบบรวดเร็ว
  - Cache:
    - [X] Redis: เหมาะเป็น cache user profile summary, user setting หรือ tenant-user mapping ที่ถูกอ่านบ่อย แต่ควรหลีกเลี่ยงการ cache PII แบบ plain text
  - Object Storage:
    - [X] AWS S3: User Service อาจต้องเก็บไฟล์ของผู้ใช้ เช่น avatar, profile image หรือ attachment จึงเหมาะกับ S3 ที่รองรับ object storage แบบ scalable และเชื่อมกับ cloud ecosystem ได้ดี
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือลดการพึ่งพา cloud provider
- Integration:
  - Tenant Service
  - Notification Service
  - IAM Service
  - Privacy Service
  - Audit / Logging Service
  - Object Storage Service

**Reason**:

- User Service ต้องจัดการ account, profile, user setting และ tenant-user mapping ที่มี business rule ชัดเจน
- ระบบต้องการ consistency เพราะข้อมูลผู้ใช้เกี่ยวข้องกับสถานะบัญชี การผูก tenant และข้อมูลที่ service อื่นนำไปใช้งาน
- ระบบต้องรองรับการแก้ไขข้อมูลผู้ใช้และเก็บ audit trail ของ action สำคัญ
- ข้อมูล profile อาจมี PII จึงต้องควบคุมการเข้าถึง และไม่ควร cache ข้อมูลอ่อนไหวแบบ plain text
- ไฟล์ผู้ใช้ เช่น avatar หรือ profile image ควรเก็บใน object storage แยกจาก database
- User Service ต้อง integration กับ Tenant Service, Notification Service, IAM Service และ Privacy Service เพื่อให้ข้อมูลผู้ใช้สัมพันธ์กับสิทธิ์ การแจ้งเตือน และการจัดการข้อมูลส่วนบุคคล

---

Service: **Token**

จัดการการออก Access Token, Refresh Token และการตรวจสอบ Token รวมถึง Token Lifecycle เช่น issue, validate, refresh, revoke, expire และ rotate token เพื่อรองรับการยืนยันตัวตนและการเข้าถึง service ภายในระบบ

- Stack:
  - Backend:
    - [X] GoLang: Token Service ต้อง issue, validate, refresh และ revoke token ด้วย latency ต่ำและ request สูง จึงเหมาะกับ GoLang ที่เบา เร็ว และรองรับ concurrent request ได้ดี
    - [ ] .NET Core C#: เหมาะเป็น option หากระบบหลักใช้ Microsoft ecosystem และต้องการ integration กับ .NET security library หรือ enterprise identity tooling
    - [ ] Java Spring Boot: เหมาะเป็น option หากระบบอยู่ใน Java ecosystem หรือมี authentication policy และ enterprise workflow ซับซ้อน
  - Database:
    - SQL:
      - [X] PostgreSQL: Token Service ต้องเก็บ refresh token metadata, token family, token hash, revoked token metadata และ rotation state ที่ต้องการ transaction และ consistency โดยไม่เก็บ token แบบ plain text
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก Token Service ใช้ .NET Core C# และองค์กรอยู่ใน Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option เฉพาะองค์กรที่มี Oracle ecosystem หรือ compliance requirement ที่ผูกกับ Oracle อยู่แล้ว
    - Log Storage:
      - [X] PostgreSQL Audit Table / Event Log: Token Service ต้องเก็บ security event สำคัญ เช่น token issued, refreshed, revoked, validation failed และ refresh token reuse detected
      - [ ] OpenSearch / ClickHouse / SIEM: เหมาะเป็น option หาก token event log มีปริมาณมาก หรือต้องทำ security monitoring และ search แบบหนัก
  - Cache:
    - [X] Redis: Token Service ต้องตรวจ revoked token, blacklist, refresh state, introspection cache และ rate limit บ่อย จึงใช้ Redis เพื่อลด latency และลดภาระ database
- Integration:
  - User Service
  - Session Service
  - IAM Service
  - KMS / Secret Management
  - Audit / Logging Service

**Reason**:

- Token Service ต้องตอบสนองเร็วมาก เพราะ issue, validate และ refresh token ถูกเรียกบ่อย
- ระบบต้องรองรับ token lifecycle ที่ปลอดภัย เช่น revoke, rotate, expire และ refresh token reuse detection
- token จริงไม่ควรถูกเก็บแบบ plain text จึงควรเก็บเป็น hash หรือ metadata เท่านั้น
- refresh token state และ revoked token ต้องมี consistency เพื่อป้องกัน token reuse และ unauthorized access
- cache จำเป็นสำหรับ blacklist, introspection cache และ rate limit เพื่อลด latency
- signing key และ encryption key ต้องแยกจัดการผ่าน KMS/Vault เพื่อรองรับ key rotation และลดความเสี่ยง key leakage

---

Service: **Session**

จัดการ User Session, Session State และ Session Lifecycle เช่น login, logout, expiry, session renewal, session revocation และ device/session tracking เพื่อควบคุมสถานะการใช้งานของผู้ใช้ในระบบ

- Stack:
  - Backend:
    - [X] GoLang: Session Service ต้อง validate, renew, revoke และ expire session ด้วย latency ต่ำและ request สูง จึงเหมาะกับ GoLang ที่เบา เร็ว และรองรับ concurrent request จำนวนมากได้ดี
    - [ ] .NET Core C#: เหมาะเป็น option หาก Session Service อยู่ใน Microsoft ecosystem และต้องการ integration กับ enterprise application หรือ .NET security tooling
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Session Service อยู่ใน Java ecosystem หรือมี session policy และ workflow ซับซ้อน
  - Database:
    - Cache / Primary Session Store:
      - [X] Redis: Session Service ต้องอ่าน active session, expiry, revoked session และ device-session mapping บ่อย จึงควรใช้ Redis เป็น primary store สำหรับ session runtime ที่ต้องการ TTL และ latency ต่ำ
    - SQL:
      - [X] PostgreSQL: ใช้เก็บ session metadata ถาวร เช่น device binding, session policy snapshot, revocation record และข้อมูลที่ต้องการ transaction โดยไม่ใช้เป็นตัวหลักในการ validate active session ทุกครั้ง
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก Session Service ใช้ .NET Core C# และองค์กรอยู่ใน Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option เฉพาะองค์กรที่มี Oracle ecosystem หรือ compliance requirement ที่ผูกกับ Oracle อยู่แล้ว
    - NoSQL / Log Storage:
      - [X] Cassandra: Session Service มี session log ที่เกิดถี่ เช่น login, logout, expired, revoked, device activity และ suspicious activity จึงเหมาะกับ Cassandra สำหรับ write-heavy log/event storage ที่ต้อง scale ได้
      - [ ] OpenSearch / ClickHouse / SIEM: เหมาะเป็น option หากต้องการค้นหา วิเคราะห์ หรือทำ security monitoring จาก session log จำนวนมาก
  - Key Management:
    - [ ] KMS / Vault: ใช้เป็น option หาก Session Service ต้องเข้ารหัส session-related data หรือมี secret ที่เกี่ยวข้องกับ session
- Integration:
  - User Service
  - Token Service
  - IAM Service
  - Audit / Logging Service

**Reason**:

- Session Service ต้องตรวจ active session, expiry และ revoke state ด้วย latency ต่ำ
- active session เป็นข้อมูลอายุสั้นและมี TTL จึงควรอยู่ใน Redis มากกว่า SQL
- PostgreSQL ใช้เก็บ session metadata ที่ต้องการ consistency ไม่ใช่ใช้รับ session log จำนวนมาก
- session log เช่น login, logout, expired, revoked และ device activity อาจเกิดถี่ จึงเหมาะกับ NoSQL ที่รองรับ write-heavy workload
- Cassandra เหมาะกับ session log ที่ต้องเขียนต่อเนื่องและ scale ตามปริมาณ event ได้
- หากต้องค้นหา log เชิง security หรือ investigation บ่อย ควรต่อ OpenSearch, ClickHouse หรือ SIEM เพิ่ม

---

Service: **Tenant**

จัดการ Multi-Tenant Configuration, Tenant Isolation และ Tenant-level Settings เช่น tenant profile, tenant status, feature flag, quota, policy, region, branding, contract document และ configuration ที่ใช้แยกข้อมูลหรือพฤติกรรมของแต่ละ tenant

- Stack:
  - Backend:
    - [X] GoLang: Tenant Service ถูกเรียกอ่าน tenant config, feature flag, quota และ isolation rule จากหลาย service บ่อยมาก จึงเหมาะกับ GoLang ที่เบา เร็ว และรองรับ request จำนวนมากได้ดี
    - [ ] .NET Core C#: เหมาะเป็น option หาก Tenant Service อยู่ใน Microsoft ecosystem และต้องการใช้ shared class library หรือ Clean Architecture ร่วมกับ service อื่น
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Tenant Service มี enterprise policy, workflow หรือ integration กับระบบองค์กรที่ซับซ้อน
  - Database:
    - SQL:
      - [X] PostgreSQL: Tenant Service ต้องเก็บ tenant profile, tenant setting, isolation policy, subscription mapping, contract metadata และ feature configuration ที่ต้องการ consistency และ query ที่ยืดหยุ่น จึงเหมาะกับ PostgreSQL
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก Tenant Service ใช้ .NET Core C# และองค์กรอยู่ใน Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option เฉพาะองค์กรที่มี Oracle ecosystem หรือมี enterprise requirement ผูกกับ Oracle อยู่แล้ว
    - Log Storage:
      - [X] PostgreSQL Tenant Event Log: Tenant Service ต้องเก็บ event สำคัญ เช่น tenant created, setting changed, feature enabled, quota changed, contract uploaded และ tenant suspended เพื่อ audit และตรวจสอบย้อนหลัง
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หาก tenant event log มีปริมาณมาก หรือต้องค้นหา configuration history จำนวนมากแบบรวดเร็ว
  - Cache:
    - [X] Redis: Tenant Service ถูก service อื่นเรียกอ่าน config บ่อย จึงควร cache tenant setting, feature flag, quota และ isolation rule เพื่อลด latency
  - Object Storage:
    - [X] AWS S3: Tenant Service ต้องเก็บไฟล์สัญญาและเอกสารประกอบ tenant จึงเหมาะกับ S3 ที่รองรับไฟล์จำนวนมาก, versioning, access policy และ lifecycle management
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือเก็บไฟล์เอกสารภายใน infrastructure ขององค์กรเอง
  - Message Broker:
    - [X] Kafka / RabbitMQ: Tenant Service ต้องกระจาย event เช่น TenantCreated, TenantUpdated, TenantSuspended, FeatureEnabled, QuotaChanged และ ContractUploaded ให้ service อื่น sync config หรือ clear cache
- Integration:
  - User Service
  - Product Service
  - Order Service
  - Subscription Service
  - IAM Service
  - Audit / Logging Service
  - Object Storage Service

**Reason**:

- Tenant Service ต้องเป็นแหล่งกลางของ tenant config, tenant isolation และ tenant-level settings
- ระบบต้องถูกเรียกอ่านบ่อยจากหลาย service เพื่อเช็ค tenant status, feature flag, quota และ policy
- ข้อมูล tenant ต้องมี consistency เพราะมีผลต่อการแยกข้อมูล สิทธิ์ การใช้งาน feature และ subscription
- ไฟล์สัญญาและเอกสาร tenant ควรเก็บใน object storage และเก็บเฉพาะ metadata/path ใน database
- Redis จำเป็นสำหรับลด latency ของ config ที่ถูกอ่านบ่อย แต่ database ยังต้องเป็น source of truth
- ทุกการเปลี่ยน tenant setting, feature flag, quota หรือ contract document ต้องมี audit log เพื่อตรวจสอบย้อนหลัง
- Message Broker จำเป็นสำหรับกระจายการเปลี่ยนแปลง tenant ไปยัง service อื่นที่ต้อง sync config หรือ clear cache

---

### Core e-Commerce

Service: **Product**

จัดการ Product Catalog, Category, Product Attribute, SKU, Product Variant, Product Media และ Product Search Indexing เพื่อให้ระบบสามารถค้นหา แสดงผล จัดกลุ่ม และเชื่อมข้อมูลสินค้ากับ Inventory, Promotion และ Tenant ได้ถูกต้อง

- Stack:
  - Backend:
    - [ ] GoLang: เหมาะเป็น option หาก Product Service เน้น API ที่เบาและ request สูง แต่ไม่ใช่ตัวเลือกหลักเมื่อ catalog มี attribute, variant และ workflow ที่ซับซ้อน
    - [X] .NET Core C#: Product Service ต้องจัดการ catalog domain, category hierarchy, product attribute, variant และ validation หลายรูปแบบ จึงเหมาะกับ .NET Core C# ที่จัดโครงสร้าง domain logic และ enterprise workflow ได้ดี
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Product Service อยู่ใน Java ecosystem หรือมี catalog workflow ขนาดใหญ่ระดับ enterprise
    - [ ] NestJS: เหมาะเป็น option หากทีมต้องการพัฒนา API เร็ว ใช้ TypeScript และมี frontend/backend ecosystem ใกล้กัน
  - Database:
    - SQL:
      - [X] PostgreSQL: Product Service ต้องเก็บ product, SKU, category, attribute, variant และ tenant-specific catalog ที่มีทั้ง relational data และ flexible attribute จึงเหมาะกับ PostgreSQL ที่รองรับ JSONB, indexing และ transaction ได้ดี
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก Product Service ใช้ .NET Core C# และองค์กรอยู่ใน Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option หาก Product Service ต้องเชื่อมกับ ERP หรือ enterprise product master เดิมที่ใช้ Oracle อยู่แล้ว
    - Log Storage:
      - [X] PostgreSQL Product Event Log: Product Service ต้องเก็บ event สำคัญ เช่น product created, product updated, price changed, category changed, attribute changed และ product unpublished เพื่อ audit และ sync search index
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หาก product event log มีปริมาณมาก หรือต้องทำ analytics/report จาก catalog history
  - Search / Indexing:
    - [X] AWS OpenSearch: Product Service ต้องรองรับ product search, filter, category browsing, attribute search และ full-text search จึงควรใช้ OpenSearch เป็น search index แยกจาก primary database
  - Cache:
    - [X] Redis: Product Service ถูกอ่านบ่อยจาก catalog page, product detail และ promotion flow จึงควร cache product summary, category tree และ attribute metadata เพื่อลด latency
  - Object Storage:
    - [X] AWS S3: Product Service ต้องเก็บ product image, gallery, manual, spec sheet และ media file จึงเหมาะกับ S3 ที่รองรับไฟล์จำนวนมากและเชื่อมกับ CDN ได้ดี
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือเก็บไฟล์จำนวนมากภายใน infrastructure ขององค์กรเอง
- Integration:
  - Inventory Service
  - Campaign & Promotion Service
  - Tenant Service
  - Search / Indexing Service
  - Object Storage Service
  - Audit / Logging Service

**Reason**:

- Product Service ต้องเป็นแหล่งกลางของ catalog, SKU, category, attribute, variant และ product media
- ระบบต้องรองรับ product attribute ที่ยืดหยุ่น เพราะสินค้าแต่ละประเภทมีข้อมูลไม่เหมือนกัน
- SQL ยังควรเป็น source of truth เพราะข้อมูลสินค้า, SKU, category และ tenant catalog ต้องมี consistency
- Search ควรแยกออกจาก database หลัก เพราะ product search, filter และ full-text search มี query pattern ต่างจาก transaction data
- Product image และไฟล์ประกอบสินค้าไม่ควรเก็บใน database ควรเก็บใน object storage แล้วเก็บเฉพาะ metadata/path
- Redis ช่วยลด latency ของข้อมูลที่ถูกอ่านบ่อย เช่น product summary, category tree และ attribute metadata
- Message Broker จำเป็นสำหรับ sync product change ไปยัง Inventory, Promotion และ Search Indexing

---

Service: **Inventory**

จัดการ Stock Level, Stock Reservation, Stock Allocation และ Inventory Consistency เพื่อให้ระบบรู้จำนวนสินค้าคงเหลือ จำนวนที่ถูกจอง จำนวนที่ขายได้ และจำนวนที่ต้องกันไว้สำหรับ Order โดยต้องป้องกันปัญหา overselling และรองรับการ update stock จากหลาย transaction พร้อมกัน

- Stack:
  - Backend:
    - [X] GoLang: Inventory Service ต้องจัดการ stock movement, reservation และ allocation ที่เกิดถี่และต้องตอบสนองเร็ว จึงเหมาะกับ GoLang ที่เบา เร็ว และรองรับ concurrent request ได้ดี
    - [ ] .NET Core C#: เหมาะเป็น option หาก Inventory Service อยู่ใน Microsoft ecosystem และต้องการจัดการ business logic ผ่าน Clean Architecture และ enterprise tooling
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Inventory Service มี workflow ซับซ้อน เช่น warehouse rule, allocation strategy หรือ integration กับ ERP/WMS ขนาดใหญ่
  - Database:
    - SQL:
      - [X] PostgreSQL: Inventory Service ต้องการ transaction, row-level locking, consistency และ audit ของ stock movement เพื่อป้องกัน overselling และจัดการ reservation/allocation ได้ถูกต้อง จึงเหมาะกับ PostgreSQL
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก Inventory Service ใช้ .NET Core C# และองค์กรอยู่ใน Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option หาก Inventory Service ต้องเชื่อมกับ ERP หรือ enterprise system เดิมที่ใช้ Oracle อยู่แล้ว
    - Log Storage:
      - [X] PostgreSQL Stock Movement / Event Log: Inventory Service ต้องเก็บประวัติ stock movement เช่น stock in, stock out, reserved, released, allocated และ adjusted เพื่อ audit และ reconcile stock
      - [ ] ClickHouse / OpenSearch: เหมาะเป็น option หากมี stock event ปริมาณมาก หรือต้อง query history/report จำนวนมากแบบรวดเร็ว
  - Cache:
    - [X] Redis: Inventory Service สามารถใช้ cache stock snapshot หรือ availability ที่ถูกอ่านบ่อย เพื่อลด latency แต่ไม่ควรใช้เป็น source of truth สำหรับ stock
  - Object Storage:
    - [X] AWS S3: Inventory Service อาจต้องเก็บรูป shelf, bin location, warehouse zone, damaged stock หรือหลักฐานการปรับ stock จึงเหมาะกับ S3 ที่รองรับไฟล์จำนวนมากและขยายได้ดี
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือมีไฟล์จำนวนมากใน internal warehouse network
- Integration:
  - Order Service
  - Product Service
  - Warehouse Service
  - Audit / Logging Service
  - Notification Service
  - Object Storage Service

**Reason**:

- Inventory Service ต้องรักษาความถูกต้องของ stock และป้องกัน overselling
- ระบบต้องรองรับ transaction ที่มีการจอง ตัดคืน และปรับ stock จากหลาย order พร้อมกัน
- Stock และ reservation ต้องมี consistency สูง จึงควรใช้ SQL เป็น source of truth
- Redis ใช้ได้สำหรับ cache availability หรือ stock snapshot เท่านั้น ไม่ควรใช้เป็นตัวหลักในการตัด stock
- รูป shelf, bin location หรือหลักฐาน stock adjustment ควรเก็บใน object storage และเก็บเฉพาะ metadata/path ใน database
- Stock movement ต้องมี log เพื่อ audit, reconcile และตรวจสอบย้อนหลัง
- Message Broker จำเป็นสำหรับกระจาย stock event ไปยัง Order Service, Notification Service หรือระบบอื่นโดยไม่ผูก transaction ตรงเกินไป

---

Service: **Order**

จัดการ Order Lifecycle และ Order-related Workflow เช่น order creation, order confirmation, payment coordination, inventory reservation, shipping request, cancellation, refund flow และ order status tracking เพื่อให้คำสั่งซื้อทำงานต่อเนื่องและสอดคล้องกันระหว่างหลาย service

- Stack:
  - Backend:
    - [ ] GoLang: เหมาะเป็น option หาก Order Service เน้น throughput สูงและ workflow ไม่ซับซ้อนมาก แต่ไม่ใช่ตัวเลือกหลักเมื่อ order flow มี business rule และ integration หลาย service
    - [X] .NET Core C#: Order Service ต้องจัดการ business workflow, validation, transaction boundary และ integration กับหลาย service จึงเหมาะกับ .NET Core C# ที่จัดโครงสร้าง domain logic, Clean Architecture และ enterprise workflow ได้ดี
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Order Service อยู่ใน Java ecosystem หรือมี workflow ซับซ้อนระดับ enterprise เช่น saga, orchestration หรือ integration กับ ERP ขนาดใหญ่
  - Database:
    - SQL:
      - [ ] PostgreSQL: เหมาะเป็น option หากต้องการ relational database ที่ยืดหยุ่นและรองรับ transaction ได้ดี แต่ไม่ใช่ตัวเลือกหลักหากระบบเลือก .NET และ Microsoft ecosystem เป็นแกน
      - [ ] Oracle: เหมาะเป็น option หาก Order Service ต้องเชื่อมกับ ERP หรือ enterprise system เดิมที่ใช้ Oracle อยู่แล้ว
      - [X] Microsoft SQL Server: Order Service ต้องเก็บ order, order item, order status, payment reference, shipping reference และ workflow state ที่ต้องการ transaction และ consistency จึงเหมาะกับ SQL Server เมื่อ backend ใช้ .NET Core C#
    - Log Storage:
      - [X] Microsoft SQL Server Order Event Log: Order Service ต้องเก็บ order event สำคัญ เช่น created, confirmed, paid, reserved, shipped, cancelled, refunded และ failed เพื่อ audit และตรวจสอบย้อนหลัง
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หากมี order event ปริมาณมาก หรือต้องค้นหา order history/report จำนวนมากแบบรวดเร็ว
  - Cache:
    - [X] Redis: Order Service สามารถ cache order summary, order status หรือ temporary checkout state ที่ถูกอ่านบ่อย เพื่อลด latency แต่ไม่ควรใช้เป็น source of truth ของ order
- Integration:
  - User Service
  - Product Service
  - Inventory Service
  - Payment Service
  - Shipping Service
  - Campaign & Promotion Service
  - Notification Service
  - Audit / Logging Service

**Reason**:

- Order Service ต้องควบคุม lifecycle ของคำสั่งซื้อจากสร้าง order จนถึงจ่ายเงิน จอง stock จัดส่ง ยกเลิก หรือ refund
- ระบบต้องมี consistency สูง เพราะ order status ต้องสัมพันธ์กับ payment, inventory และ shipping
- Order workflow มี business rule หลายขั้น เช่น promotion, stock reservation, payment confirmation และ cancellation rule
- SQL ควรเป็น source of truth เพราะ order, order item และ order status ต้องมี transaction และตรวจสอบย้อนหลังได้
- Redis ใช้ได้สำหรับ order summary หรือ temporary checkout state เท่านั้น ไม่ควรใช้เป็นตัวหลักของ order
- Message Broker จำเป็นเพราะ Order Service ต้องประสานหลาย service แบบ asynchronous และลดการผูก transaction ข้าม service

---

### Payment & Financial

Service: **Subscription**

จัดการ Subscription Plan, Billing Cycle และ Subscription Lifecycle เช่น trial, active, suspended, expired, renewed, upgraded, downgraded และ cancelled เพื่อควบคุมสิทธิ์การใช้งานของ Tenant ตามแพ็กเกจและรอบการชำระเงิน

- Stack:
  - Backend:
    - [ ] GoLang: เหมาะเป็น option หาก Subscription Service เน้น check subscription status หรือ entitlement lookup ที่ต้องการ latency ต่ำและ request สูงมาก
    - [X] .NET Core C#: Subscription Service ต้องจัดการ plan, billing cycle, entitlement, payment reference และ lifecycle state ที่มี business rule หลายขั้น จึงเหมาะกับ .NET Core C# ที่จัดโครงสร้าง domain logic และ workflow ได้ดี
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Subscription Service อยู่ใน Java ecosystem หรือมี billing workflow / saga orchestration ระดับ enterprise
  - Database:
    - SQL:
      - [ ] PostgreSQL: เหมาะเป็น option หากต้องการ relational database ที่ยืดหยุ่นและรองรับ transaction ได้ดี แต่ไม่ใช่ตัวเลือกหลักเมื่อเลือก .NET Core C# และ Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option หาก Subscription Service ต้องเชื่อมกับ ERP, finance system หรือ enterprise system เดิมที่ใช้ Oracle อยู่แล้ว
      - [X] Microsoft SQL Server: Subscription Service ต้องเก็บ subscription, plan, billing cycle, entitlement, invoice reference และ payment status ที่ต้องการ transaction และ consistency จึงเหมาะกับ SQL Server เมื่อใช้ .NET Core C#
    - Log Storage:
      - [X] Microsoft SQL Server Subscription Event Log: Subscription Service ต้องเก็บ event สำคัญ เช่น trial started, activated, renewed, upgraded, downgraded, expired, suspended และ cancelled เพื่อ audit และตรวจสอบย้อนหลัง
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หาก subscription event log มีปริมาณมาก หรือต้องค้นหา lifecycle history/report จำนวนมากแบบรวดเร็ว
  - Cache:
    - [X] Redis: Subscription Service ถูก Tenant Service หรือ service อื่นเรียกเช็ค plan, entitlement และ subscription status บ่อย จึงควร cache ข้อมูลที่ไม่อ่อนไหวเพื่อลด latency
  - Message Broker:
    - [X] Kafka / RabbitMQ: Subscription Service ต้องส่ง event เช่น SubscriptionActivated, SubscriptionRenewed, SubscriptionExpired, PlanChanged และ SubscriptionCancelled ให้ Tenant, Payment และ Notification ทำงานต่อแบบ asynchronous
- Integration:
  - Payment Service
  - Tenant Service
  - Notification Service
  - IAM Service
  - Audit / Logging Service

**Reason**:

- Subscription Service ต้องควบคุม lifecycle ของแพ็กเกจ เช่น trial, active, expired, suspended และ cancelled
- ระบบต้องจัดการ plan, billing cycle, entitlement และสถานะ subscription ให้สัมพันธ์กับ Tenant
- ข้อมูล subscription ต้องมี consistency เพราะมีผลต่อสิทธิ์การใช้งาน feature, quota และการต่ออายุบริการ
- SQL ต้องเป็น source of truth เพราะ subscription, plan, billing cycle และ payment reference ต้องตรวจสอบย้อนหลังได้
- Redis ช่วยลด latency สำหรับการเช็ค subscription status, plan และ entitlement ที่ถูกอ่านบ่อย
- ทุกการเปลี่ยนสถานะ subscription ต้องมี event log เพื่อ audit และ trace ปัญหา billing/lifecycle
- Message Broker จำเป็นสำหรับกระจาย subscription event ไปยัง Tenant Service, Payment Service และ Notification Service

---

Service: **Payment**

จัดการ Payment Processing, Payment Transaction, Payment Execution และ Payment Status เช่น payment request, authorization, capture, void, refund, failed payment, payment retry และ payment callback เพื่อให้การชำระเงินของ Order ถูกต้อง ตรวจสอบได้ และเชื่อมต่อกับระบบการเงินภายในองค์กร

- Stack:
  - Backend:
    - [ ] GoLang: เหมาะเป็น option หาก Payment Service เน้น throughput สูง, callback handling และ payment status check ที่ต้องการ latency ต่ำมาก
    - [X] .NET Core C#: Payment Service ต้องจัดการ payment workflow, transaction state, validation, retry logic และ integration กับ Order/Settlement/Reconciliation จึงเหมาะกับ .NET Core C# ที่จัดโครงสร้าง business logic และ workflow ได้ดี
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Payment Service อยู่ใน Java ecosystem หรือมี payment workflow ระดับ enterprise ที่ซับซ้อนมาก
  - Database:
    - SQL:
      - [ ] PostgreSQL: เหมาะเป็น option หากต้องการ relational database ที่รองรับ transaction และ consistency ได้ดี แต่ไม่ใช่ตัวเลือกหลักเมื่อเลือก .NET Core C# และ Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option หาก Payment Service ต้องเชื่อมกับ core finance, ERP หรือ banking system เดิมที่ใช้ Oracle อยู่แล้ว
      - [X] Microsoft SQL Server: Payment Service ต้องเก็บ payment transaction, payment status, gateway reference, refund record, retry state และ callback result ที่ต้องการ transaction และ consistency จึงเหมาะกับ SQL Server เมื่อใช้ .NET Core C#
    - Log Storage:
      - [X] Microsoft SQL Server Payment Event Log: Payment Service ต้องเก็บ event สำคัญ เช่น payment requested, authorized, captured, failed, refunded, voided และ callback received เพื่อ audit และตรวจสอบย้อนหลัง
      - [ ] OpenSearch / ClickHouse / SIEM: เหมาะเป็น option หาก payment event log มีปริมาณมาก หรือต้องค้นหา transaction/security event จำนวนมากแบบรวดเร็ว
  - Cache:
    - [X] Redis: Payment Service ใช้ cache payment status ชั่วคราว, idempotency key, retry lock และ callback deduplication เพื่อลด duplicate transaction และลด latency
  - Object Storage:
    - [X] AWS S3: Payment Service อาจต้องเก็บไฟล์หลักฐานการชำระเงิน, payment slip, gateway report หรือ settlement document จึงเหมาะกับ S3 ที่รองรับไฟล์จำนวนมากและ lifecycle management
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือเก็บเอกสารการเงินภายใน infrastructure ขององค์กรเอง
  - Message Broker:
    - [X] Kafka / RabbitMQ: Payment Service ต้องส่ง event เช่น PaymentAuthorized, PaymentCaptured, PaymentFailed, PaymentRefunded และ PaymentCallbackReceived ให้ Order, Settlement, Reconciliation และ Notification ทำงานต่อแบบ asynchronous
- Integration:
  - Order Service
  - Settlement Service
  - Reconciliation Service
  - Financial Service
  - Notification Service
  - Audit / Logging Service
  - Object Storage Service

**Reason**:

- Payment Service ต้องควบคุม payment lifecycle ตั้งแต่ request, authorize, capture, failed, refund และ void
- ระบบต้องมี consistency สูง เพราะ payment status มีผลต่อ order, settlement, reconciliation และ financial record
- payment transaction ต้องรองรับ idempotency เพื่อป้องกันการตัดเงินซ้ำจาก retry หรือ callback ซ้ำ
- SQL ต้องเป็น source of truth เพราะ transaction, refund, gateway reference และ payment status ต้องตรวจสอบย้อนหลังได้
- Redis จำเป็นสำหรับ idempotency key, retry lock และ callback deduplication ที่ต้องตอบสนองเร็ว
- เอกสารหรือหลักฐานการชำระเงินควรเก็บใน object storage และเก็บเฉพาะ metadata/path ใน database
- ทุก payment event ต้องมี audit log เพื่อ trace ปัญหา transaction, gateway callback และ refund
- Message Broker จำเป็นสำหรับกระจาย payment event ไปยัง Order, Settlement, Reconciliation และ Notification โดยไม่ผูก transaction ข้าม service

---

Service: **Settlement**

รวบรวมและคำนวณ Settlement Amount จาก Payment Transaction เพื่อใช้ทำ Financial Reconciliation, Payout, Fee Calculation และ Settlement Status Tracking โดยต้องรองรับการรวมยอดตามรอบเวลา เช่น รายวัน รายสัปดาห์ หรือรายเดือน และตรวจสอบยอดกับระบบการเงินได้

- Stack:
  - Backend:
    - [ ] GoLang: เหมาะเป็น option หาก Settlement Service เน้น batch processing ที่เบาและต้องการ throughput สูง แต่ไม่ใช่ตัวเลือกหลักเมื่อมี calculation rule, payout rule และ financial workflow หลายขั้น
    - [X] .NET Core C#: Settlement Service ต้องจัดการ settlement rule, fee calculation, payout cycle, transaction grouping และ workflow ที่เกี่ยวข้องกับการเงิน จึงเหมาะกับ .NET Core C# ที่จัดโครงสร้าง domain logic และ business workflow ได้ดี
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Settlement Service อยู่ใน Java ecosystem หรือมี financial workflow ระดับ enterprise ที่ซับซ้อนมาก
  - Database:
    - SQL:
      - [ ] PostgreSQL: เหมาะเป็น option หากต้องการ relational database ที่รองรับ transaction และ consistency ได้ดี แต่ไม่ใช่ตัวเลือกหลักเมื่อเลือก .NET Core C# และ Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option หาก Settlement Service ต้องเชื่อมกับ core finance, ERP หรือ banking system เดิมที่ใช้ Oracle อยู่แล้ว
      - [X] Microsoft SQL Server: Settlement Service ต้องเก็บ settlement batch, settlement item, payout record, fee calculation, payment reference และ settlement status ที่ต้องการ transaction และ consistency จึงเหมาะกับ SQL Server เมื่อใช้ .NET Core C#
    - Log Storage:
      - [X] Microsoft SQL Server Settlement Event Log: Settlement Service ต้องเก็บ event สำคัญ เช่น settlement calculated, batch created, payout requested, payout completed, settlement failed และ settlement adjusted เพื่อ audit และตรวจสอบย้อนหลัง
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หาก settlement event log มีปริมาณมาก หรือต้องค้นหา settlement history/report จำนวนมากแบบรวดเร็ว
  - Cache:
    - [ ] Redis: เหมาะเป็น option สำหรับ cache settlement summary หรือ report snapshot ที่ถูกอ่านบ่อย แต่ไม่ควรใช้เป็น source of truth ของยอด settlement
- Integration:
  - Payment Service
  - Financial Service
  - Reconciliation Service
  - Audit / Logging Service

**Reason**:

- Settlement Service ต้องรวบรวม payment transaction เพื่อคำนวณยอดที่ต้องจ่ายหรือยอดที่ต้อง reconcile
- ระบบต้องรองรับ settlement cycle เช่น daily, weekly, monthly และต้องคำนวณ fee, refund, adjustment และ payout ให้ถูกต้อง
- ข้อมูล settlement ต้องมี consistency สูง เพราะเกี่ยวข้องกับยอดเงินจริงและ financial record
- SQL ต้องเป็น source of truth เพราะ settlement batch, payout record และ payment reference ต้องตรวจสอบย้อนหลังได้
- Redis ไม่จำเป็นเป็น core component แต่ใช้ได้หากต้อง cache summary หรือ report ที่อ่านบ่อย
- ทุกการคำนวณและการเปลี่ยนสถานะ settlement ต้องมี event log เพื่อ audit และ trace ปัญหาย้อนหลัง
- Message Broker จำเป็นสำหรับรับ payment event และส่ง settlement event ไปยัง Financial และ Reconciliation Service

---

Service: **Reconciliation**

ตรวจสอบและเปรียบเทียบ Transaction Record ระหว่างระบบภายในกับข้อมูลจากภายนอก เช่น Payment Gateway, Bank Statement, Settlement Report และ Financial Record เพื่อหา mismatch, missing transaction, duplicate transaction และยอดที่ไม่ตรงกัน

- Stack:
  - Backend:
    - [ ] GoLang: เหมาะเป็น option หาก Reconciliation Service เน้นประมวลผลไฟล์หรือ transaction ปริมาณมากแบบเร็ว แต่ไม่ใช่ตัวเลือกหลักเมื่อมี matching rule, exception workflow และ financial validation หลายขั้น
    - [X] .NET Core C#: Reconciliation Service ต้องจัดการ matching rule, validation rule, exception workflow และ financial comparison ที่มี business logic ชัดเจน จึงเหมาะกับ .NET Core C# ที่จัดโครงสร้าง domain logic และ workflow ได้ดี
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Reconciliation Service อยู่ใน Java ecosystem หรือมี batch processing / enterprise finance workflow ซับซ้อนมาก
  - Database:
    - SQL:
      - [ ] PostgreSQL: เหมาะเป็น option หากต้องการ relational database ที่รองรับ transaction และ query ที่ยืดหยุ่น แต่ไม่ใช่ตัวเลือกหลักเมื่อเลือก .NET Core C# และ Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option หาก Reconciliation Service ต้องเชื่อมกับ core finance, ERP หรือ banking system เดิมที่ใช้ Oracle อยู่แล้ว
      - [X] Microsoft SQL Server: Reconciliation Service ต้องเก็บ reconciliation batch, matching result, mismatch record, external transaction reference และ exception status ที่ต้องการ consistency และตรวจสอบย้อนหลัง จึงเหมาะกับ SQL Server เมื่อใช้ .NET Core C#
    - Log Storage:
      - [X] Microsoft SQL Server Reconciliation Event Log: Reconciliation Service ต้องเก็บ event สำคัญ เช่น file imported, transaction matched, mismatch found, duplicate detected, exception resolved และ reconciliation completed เพื่อ audit และ trace ปัญหา
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หาก reconciliation event log หรือ transaction comparison result มีปริมาณมาก และต้องค้นหา/report อย่างรวดเร็ว
  - Object Storage:
    - [X] AWS S3: Reconciliation Service ต้องเก็บไฟล์จากภายนอก เช่น gateway report, bank statement, settlement file และ reconciliation evidence จึงเหมาะกับ S3 ที่รองรับไฟล์จำนวนมากและ lifecycle management
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือเก็บไฟล์การเงินภายใน infrastructure ขององค์กรเอง
- Integration:
  - Payment Service
  - Financial Service
  - Settlement Service
  - Audit / Logging Service
  - Object Storage Service
  - Notification Service

**Reason**:

- Reconciliation Service ต้องเปรียบเทียบข้อมูลธุรกรรมจากหลายแหล่ง เช่น Payment, Settlement, Financial และข้อมูลภายนอก
- ระบบต้องตรวจจับ mismatch, missing transaction, duplicate transaction และยอดที่ไม่ตรงกัน
- ข้อมูล reconciliation ต้องมี consistency เพราะมีผลต่อ financial record และการปิดยอด
- SQL ต้องเป็น source of truth สำหรับ reconciliation batch, matching result, mismatch record และ exception status
- ไฟล์จาก payment gateway, bank หรือ settlement report ควรเก็บใน object storage และเก็บเฉพาะ metadata/path ใน database
- ทุกขั้นตอนของ reconciliation ต้องมี event log เพื่อ audit และ trace ปัญหาย้อนหลัง
- Message Broker จำเป็นสำหรับรับ transaction event และส่งผลลัพธ์ reconciliation ไปยัง Financial, Settlement หรือ Notification Service

---

Service: **Financial**

จัดการ Financial Ledger, Accounting Record และ Transaction Auditability เช่น journal entry, debit/credit record, financial posting, adjustment, refund record และ transaction trace เพื่อให้ข้อมูลทางการเงินถูกต้อง ตรวจสอบย้อนหลังได้ และใช้ปิดบัญชีหรือทำรายงานทางการเงินได้

- Stack:
  - Backend:
    - [ ] GoLang: เหมาะเป็น option หาก Financial Service เน้นรับ event ปริมาณมากและต้องการ processing ที่เร็ว แต่ไม่ใช่ตัวเลือกหลักเมื่อระบบต้องจัดการ accounting rule, ledger rule และ financial workflow ที่ซับซ้อน
    - [X] .NET Core C#: Financial Service ต้องจัดการ ledger, accounting rule, posting workflow, validation และ audit trail ที่มี business logic ชัดเจน จึงเหมาะกับ .NET Core C# ที่จัดโครงสร้าง domain logic และ workflow ได้ดี
    - [ ] Java Spring Boot: เหมาะเป็น option หาก Financial Service อยู่ใน Java ecosystem หรือมี accounting workflow ระดับ enterprise ที่ซับซ้อนมาก
  - Database:
    - SQL:
      - [ ] PostgreSQL: เหมาะเป็น option หากต้องการ relational database ที่รองรับ transaction, consistency และ audit ได้ดี แต่ไม่ใช่ตัวเลือกหลักเมื่อเลือก .NET Core C# และ Microsoft ecosystem
      - [ ] Oracle: เหมาะเป็น option หาก Financial Service ต้องเชื่อมกับ ERP, accounting system หรือ core finance เดิมที่ใช้ Oracle อยู่แล้ว
      - [X] Microsoft SQL Server: Financial Service ต้องเก็บ ledger entry, journal entry, debit/credit record, posting status, adjustment และ financial transaction reference ที่ต้องการ transaction, consistency และ auditability จึงเหมาะกับ SQL Server เมื่อใช้ .NET Core C#
    - Log Storage:
      - [X] Microsoft SQL Server Financial Event Log: Financial Service ต้องเก็บ event สำคัญ เช่น ledger posted, journal created, adjustment made, refund posted, transaction reversed และ financial period closed เพื่อ audit และตรวจสอบย้อนหลัง
      - [ ] OpenSearch / ClickHouse: เหมาะเป็น option หาก financial event log หรือ reporting query มีปริมาณมาก และต้องค้นหา/report อย่างรวดเร็ว
  - Cache:
    - [ ] Redis: เหมาะเป็น option สำหรับ cache financial summary หรือ report snapshot ที่อ่านบ่อย แต่ไม่ควรใช้เป็น source of truth ของ ledger หรือ accounting record
  - Object Storage:
    - [X] AWS S3: Financial Service อาจต้องเก็บไฟล์รายงานทางการเงิน, statement, export file, audit evidence หรือเอกสารประกอบการปิดบัญชี จึงเหมาะกับ S3 ที่รองรับไฟล์จำนวนมากและ lifecycle management
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือเก็บเอกสารการเงินภายใน infrastructure ขององค์กรเอง
- Integration:
  - Payment Service
  - Settlement Service
  - Reconciliation Service
  - Order Service
  - Audit / Logging Service
  - Object Storage Service

**Reason**:

- Financial Service ต้องเป็น source of truth ของ ledger, journal entry และ accounting record
- ระบบต้องรองรับ debit/credit, posting, adjustment, refund และ transaction reversal ให้ถูกต้อง
- ข้อมูล financial ต้องมี consistency สูง เพราะมีผลต่อบัญชี รายงานการเงิน และการตรวจสอบย้อนหลัง
- SQL ต้องเป็น source of truth เพราะ ledger และ journal entry ต้องมี transaction และ auditability ชัดเจน
- Redis ไม่ควรใช้เก็บข้อมูล ledger หลัก ใช้ได้เฉพาะ summary หรือ report snapshot ที่อ่านบ่อย
- ไฟล์รายงาน statement หรือ audit evidence ควรเก็บใน object storage และเก็บเฉพาะ metadata/path ใน database
- ทุก financial posting และ adjustment ต้องมี event log เพื่อ audit และ trace ปัญหาย้อนหลัง
- Message Broker จำเป็นสำหรับรับ financial event จาก Payment, Settlement, Reconciliation และ Order โดยไม่ผูก transaction ข้าม service

---

### Fulfillment

Service: **Shipping**

จัดการ Shipment Creation, Delivery Tracking และ Logistics Coordination เช่น shipment request, carrier assignment, tracking number, shipping label, pickup, in-transit, delivered, failed delivery และ return shipment เพื่อให้การจัดส่งสัมพันธ์กับ Order และ Inventory อย่างถูกต้อง

- Stack:
  - Backend:
    - [X] NestJS: Shipping Service ต้องเชื่อมต่อ Carrier API, รับ Webhook/Callback, sync tracking status และ trigger notification บ่อย จึงเหมาะกับ NestJS ที่รองรับ async I/O ดีและมีโครงสร้างชัดเจน
    - [ ] Express.js: เหมาะเป็น option หาก Shipping Service ต้องการ API/Webhook service ที่เบาและเรียบง่าย โดยไม่ต้องมี framework structure มาก
    - [ ] GoLang: เหมาะเป็น option หาก Shipping Service ต้องรับ tracking update หรือ carrier callback ปริมาณมากมาก และต้องการ service ที่เบาเป็นพิเศษ
  - Database:
    - SQL:
      - [X] PostgreSQL: Shipping Service ต้องเก็บ shipment, tracking number, carrier reference, delivery status, return shipment และ delivery state ที่ต้องการ consistency และ query ที่ยืดหยุ่น จึงเหมาะกับ PostgreSQL
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก Shipping Service ใช้ .NET ecosystem หรือองค์กรต้องการผูกกับ Microsoft tooling
      - [ ] Oracle: เหมาะเป็น option หาก Shipping Service ต้องเชื่อมกับ ERP, WMS หรือ logistics system เดิมที่ใช้ Oracle อยู่แล้ว
    - Log Storage:
      - [X] PostgreSQL Shipping Event Log: Shipping Service ต้องเก็บ event สำคัญ เช่น shipment created, carrier assigned, picked up, in transit, delivered, failed delivery และ returned เพื่อ audit และ trace สถานะการจัดส่ง
      - [ ] OpenSearch: เหมาะเป็น option หาก tracking event มีปริมาณมาก และต้องค้นหา delivery history หรือ tracking log แบบรวดเร็ว
      - [ ] ClickHouse: เหมาะเป็น option หากต้องทำ report หรือ analytics จาก tracking event จำนวนมาก
  - Cache:
    - [X] Redis: Shipping Service ถูกอ่าน tracking status บ่อย และต้องจัดการ webhook deduplication จึงควรใช้ Redis cache shipment status, tracking summary และ callback lock เพื่อลด latency
  - Object Storage:
    - [X] AWS S3: Shipping Service อาจต้องเก็บ shipping label, proof of delivery, invoice, return document หรือรูปหลักฐานการจัดส่ง จึงเหมาะกับ S3 ที่รองรับไฟล์จำนวนมากและ lifecycle management
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือเก็บเอกสารจัดส่งภายใน infrastructure ขององค์กรเอง
- Integration:
  - Order Service
  - Inventory Service
  - Carrier / Logistics Provider
  - Notification Service
  - Audit / Logging Service
  - Object Storage Service

**Reason**:

- Shipping Service เป็น integration-heavy service ที่ต้องคุยกับ Carrier API และ Logistics Provider หลายเจ้า
- ระบบต้องรับ Webhook/Callback และ sync tracking status บ่อย จึงเหมาะกับ backend ที่เด่นด้าน async I/O
- Shipping workflow ส่วนใหญ่เป็น status coordination มากกว่า complex financial domain logic
- ข้อมูล shipment, tracking number และ delivery status ต้องมี consistency เพราะมีผลต่อ Order และ Inventory
- Redis จำเป็นสำหรับ tracking cache, callback deduplication และ temporary lock เพื่อลดปัญหา webhook ซ้ำ
- Shipping label, proof of delivery และ return document ควรเก็บใน object storage ไม่ควรเก็บเป็นไฟล์ใน database
- Message Broker จำเป็นสำหรับกระจาย shipping event ไปยัง Order, Inventory และ Notification Service

---

### Growth / Marketing

Service: **Shipping**

จัดการ Shipment Creation, Delivery Tracking และ Logistics Coordination เช่น shipment request, carrier assignment, tracking number, shipping label, pickup, in-transit, delivered, failed delivery และ return shipment เพื่อให้การจัดส่งสัมพันธ์กับ Order และ Inventory อย่างถูกต้อง

- Stack:
  - Backend:
    - [X] NestJS: Shipping Service ต้องเชื่อมต่อ Carrier API, รับ Webhook/Callback, sync tracking status และ trigger notification บ่อย จึงเหมาะกับ NestJS ที่รองรับ async I/O ดีและมีโครงสร้างชัดเจน
    - [ ] Express.js: เหมาะเป็น option หาก Shipping Service ต้องการ API/Webhook service ที่เบาและเรียบง่าย โดยไม่ต้องมี framework structure มาก
    - [ ] GoLang: เหมาะเป็น option หาก Shipping Service ต้องรับ tracking update หรือ carrier callback ปริมาณมากมาก และต้องการ service ที่เบาเป็นพิเศษ
  - Database:
    - SQL:
      - [X] PostgreSQL: Shipping Service ต้องเก็บ shipment, tracking number, carrier reference, delivery status, return shipment และ delivery state ที่ต้องการ consistency และ query ที่ยืดหยุ่น จึงเหมาะกับ PostgreSQL
      - [ ] Microsoft SQL Server: เหมาะเป็น option หาก Shipping Service ใช้ .NET ecosystem หรือองค์กรต้องการผูกกับ Microsoft tooling
      - [ ] Oracle: เหมาะเป็น option หาก Shipping Service ต้องเชื่อมกับ ERP, WMS หรือ logistics system เดิมที่ใช้ Oracle อยู่แล้ว
    - Log Storage:
      - [X] PostgreSQL Shipping Event Log: Shipping Service ต้องเก็บ event สำคัญ เช่น shipment created, carrier assigned, picked up, in transit, delivered, failed delivery และ returned เพื่อ audit และ trace สถานะการจัดส่ง
      - [ ] OpenSearch: เหมาะเป็น option หาก tracking event มีปริมาณมาก และต้องค้นหา delivery history หรือ tracking log แบบรวดเร็ว
      - [ ] ClickHouse: เหมาะเป็น option หากต้องทำ report หรือ analytics จาก tracking event จำนวนมาก
  - Cache:
    - [X] Redis: Shipping Service ถูกอ่าน tracking status บ่อย และต้องจัดการ webhook deduplication จึงควรใช้ Redis cache shipment status, tracking summary และ callback lock เพื่อลด latency
  - Object Storage:
    - [X] AWS S3: Shipping Service อาจต้องเก็บ shipping label, proof of delivery, invoice, return document หรือรูปหลักฐานการจัดส่ง จึงเหมาะกับ S3 ที่รองรับไฟล์จำนวนมากและ lifecycle management
    - [ ] SeaweedFS: เหมาะเป็น option หากต้องการ self-hosted object storage หรือเก็บเอกสารจัดส่งภายใน infrastructure ขององค์กรเอง
- Integration:
  - Order Service
  - Inventory Service
  - Carrier / Logistics Provider
  - Notification Service
  - Audit / Logging Service
  - Object Storage Service

**Reason**:

- Shipping Service เป็น integration-heavy service ที่ต้องคุยกับ Carrier API และ Logistics Provider หลายเจ้า
- ระบบต้องรับ Webhook/Callback และ sync tracking status บ่อย จึงเหมาะกับ backend ที่เด่นด้าน async I/O
- Shipping workflow ส่วนใหญ่เป็น status coordination มากกว่า complex financial domain logic
- ข้อมูล shipment, tracking number และ delivery status ต้องมี consistency เพราะมีผลต่อ Order และ Inventory
- Redis จำเป็นสำหรับ tracking cache, callback deduplication และ temporary lock เพื่อลดปัญหา webhook ซ้ำ
- Shipping label, proof of delivery และ return document ควรเก็บใน object storage ไม่ควรเก็บเป็นไฟล์ใน database
- Message Broker จำเป็นสำหรับกระจาย shipping event ไปยัง Order, Inventory และ Notification Service

---

### Communication

Service: **Notification**

จัดการ Asynchronous Notification, Message Delivery Workflow และ Delivery Status เช่น email, SMS, push notification, template rendering, retry, scheduling, provider callback และ delivery tracking เพื่อให้ระบบสามารถส่งข้อความจาก service ต่าง ๆ ได้แบบไม่ผูกกับ transaction หลักโดยตรง

- Stack:
  - Backend:
    - [X] Python (FastAPI) + AWS Lambda: Notification Service เป็น event-driven และ asynchronous โดยธรรมชาติ จึงเหมาะกับ Python + Lambda สำหรับรับ event, render template, ส่ง email/SMS/push, retry และบันทึก delivery status โดยไม่ต้องดูแล server ตลอดเวลา
    - [ ] NestJS: เหมาะเป็น option หาก Notification Service ต้องมี API layer, dashboard, template management หรือ webhook endpoint ที่ต้องรันเป็น service ตลอดเวลา
    - [ ] GoLang: เหมาะเป็น option หาก Notification Service ต้องประมวลผล message ปริมาณมากมาก และต้องการ worker ที่เบา เร็ว และ cold start ต่ำ
  - Database:
    - SQL:
      - [X] PostgreSQL: Notification Service ต้องเก็บ notification request, template metadata, delivery status, retry state และ provider reference ที่ต้องตรวจสอบย้อนหลังได้ จึงเหมาะกับ PostgreSQL
      - [ ] DynamoDB: เหมาะเป็น option หากต้องการ serverless-native storage สำหรับ delivery log, retry state หรือ event status ที่มีปริมาณมากและ schema ไม่ซับซ้อน
      - [ ] Microsoft SQL Server: เหมาะเป็น option หากระบบหลักอยู่ใน Microsoft ecosystem และต้องการใช้ร่วมกับ .NET service อื่น
    - Log Storage:
      - [X] CloudWatch Logs: Notification Service ที่รันบน Lambda ควรใช้ CloudWatch Logs เป็น log หลักสำหรับ execution log, error log และ provider response
      - [ ] OpenSearch: เหมาะเป็น option หากต้องค้นหา delivery log, error log หรือ provider callback จำนวนมากแบบรวดเร็ว
      - [ ] ClickHouse: เหมาะเป็น option หากต้องทำ analytics/report จาก notification event จำนวนมาก เช่น delivery rate, failure rate และ provider performance
  - Cache:
    - [X] Redis: ใช้สำหรับ deduplication, rate limit หรือ provider throttle หาก logic ซับซ้อนขึ้น แต่ไม่จำเป็นต้องเป็น core component ตั้งแต่แรกเมื่อใช้ SQS/Lambda
  - Object Storage:
    - [X] AWS S3: Notification Service อาจต้องเก็บ email template asset, attachment, export delivery report หรือไฟล์ประกอบ notification จึงเหมาะกับ S3
- Integration:
  - Email Provider
  - SMS Provider
  - Push Notification Provider
  - Audit / Logging Service
  - Object Storage Service

**Reason**:

- Notification Service เป็น asynchronous workload ที่ควรทำงานตาม event ไม่ควรผูกกับ transaction หลักของ Order, Payment หรือ Shipping
- ระบบต้องรองรับ burst traffic เช่น campaign, payment failed, order status update หรือ shipment delivered โดยไม่ต้อง scale service เองตลอดเวลา
- Python + Lambda เหมาะกับงาน notification เพราะ logic ส่วนใหญ่เป็น integration, template rendering, provider call และ retry workflow
- SQS / SNS / EventBridge จำเป็นสำหรับแยก notification ออกจาก service ต้นทาง และช่วยจัดการ retry, delay, fan-out และ dead-letter queue
- PostgreSQL เหมาะสำหรับเก็บ notification request, delivery status, retry state และ provider reference ที่ต้องตรวจสอบย้อนหลัง
- CloudWatch Logs เหมาะเป็น execution log หลักของ Lambda เพราะต้อง trace error, timeout, provider response และ retry behavior
- Redis ยังไม่จำเป็นเป็น core component ถ้าใช้ SQS/Lambda ได้ดีอยู่แล้ว แต่เพิ่มได้เมื่อมี deduplication หรือ rate limit ที่ซับซ้อนขึ้น
- ไฟล์แนบ, template asset หรือ delivery report ควรเก็บใน S3 แล้วเก็บเฉพาะ metadata/path ใน database

## BACKEND FOR FRONTEND (BFF)

แยก BFF layer ออกจาก Mobile และ Web เพื่อ optimize response และ handle client-specific logic สำหรับแต่ละ client โดยไม่ให้ Web/Mobile ต้องเรียก microservices หลายตัวเองโดยตรง

- Stack:
  - [X] Node.js + Apollo GraphQL: เหมาะกับ BFF เพราะทำ GraphQL server ได้ดี, aggregate data จากหลาย microservices ได้ง่าย และเหมาะกับ use case ที่ Web/Mobile ต้องการ response ไม่เหมือนกัน
  - [ ] Node.js + NestJS GraphQL: เหมาะถ้าต้องการ structure ที่เป็นระบบมากขึ้น มี dependency injection, module pattern และเหมาะกับทีมที่ต้องการ maintain BFF ระยะยาว
  - [ ] .NET + Hot Chocolate GraphQL: เหมาะถ้าทีม backend ใช้ .NET เป็นหลัก และต้องการ GraphQL server ที่ performance ดี พร้อม ecosystem ฝั่ง enterprise

**Reason**:

- BFF ต้อง aggregate data จาก microservices หลายตัวให้เป็น response เดียวสำหรับ client
- GraphQL เหมาะกับ BFF เพราะช่วยให้ client เลือก field ที่ต้องการได้เอง ลด over-fetching/under-fetching
- Node.js + Apollo GraphQL เหมาะเป็น best practice สำหรับ BFF เพราะทำงานกับ frontend ecosystem ได้ดี และเหมาะกับการ compose response จากหลาย backend services
- NestJS GraphQL เป็น option ที่ดีถ้าต้องการโครงสร้างชัดเจนกว่า Apollo แบบ standalone
- .NET + Hot Chocolate GraphQL เป็น option ที่ดีถ้าองค์กรใช้ .NET เป็น core backend อยู่แล้ว
- แยก client-specific logic ไว้ที่ BFF ช่วยให้ microservices หลักไม่ต้องผูกกับ requirement เฉพาะของ Web หรือ Mobile

## FRONTEND / CLIENT APPLICATION

รองรับช่องทางการใช้งานหลักของระบบทั้ง Mobile Application และ Web Application โดยแยก technology ให้เหมาะกับแต่ละ platform

### Mobile Application

- Stack:
  - [X] iOS + Swift: เหมาะกับการพัฒนา native iOS application เพราะทำงานกับ Apple ecosystem ได้เต็มประสิทธิภาพ และรองรับ UX/UI ที่เป็นมาตรฐานของ iOS
  - [X] Android + Kotlin: เหมาะกับการพัฒนา native Android application เพราะเป็นภาษาหลักที่ Google แนะนำสำหรับ Android และทำงานร่วมกับ Android SDK ได้ดี
  - [ ] Flutter: เหมาะเป็น option ถ้าต้องการพัฒนา mobile app แบบ cross-platform ด้วย codebase เดียว แต่ยังต้องการ performance และ UI consistency ที่ดี

### Web Application

- Stack:
  - [X] Next.js: เหมาะกับ Web Application เพราะรองรับ React ecosystem, routing, SSR/SSG และเหมาะกับระบบที่ต้องการ performance, SEO และ developer experience ที่ดี
  - [ ] React: เหมาะเป็น option ถ้าต้องการพัฒนา web application ด้วย React ecosystem โดยสามารถเลือก architecture ได้ยืดหยุ่น เช่น CSR, SSR หรือ hybrid rendering ตาม requirement ของระบบ
  - [ ] Angular: เหมาะเป็น option ถ้าทีมต้องการ framework ที่ opinionated มากขึ้น มี structure ชัดเจน และเหมาะกับ enterprise frontend ขนาดใหญ่

**Reason**:

- Mobile Application ควรใช้ native stack เพื่อให้ได้ performance, UX และ platform integration ที่ดีที่สุด
- Swift เหมาะกับ iOS เพราะสามารถใช้ capability ของ Apple platform ได้เต็มที่ เช่น push notification, biometric, camera, wallet หรือ location
- Kotlin เหมาะกับ Android เพราะเป็น modern language สำหรับ Android development และทำงานร่วมกับ Jetpack libraries ได้ดี
- Next.js เหมาะกับ Web Application เพราะช่วยให้สร้าง web frontend ที่เร็ว, maintain ง่าย และรองรับทั้ง public-facing page กับ authenticated app page
- React เป็น option ที่ยืดหยุ่นหากต้องการควบคุม architecture เอง โดยไม่ผูกกับ framework เต็มรูปแบบตั้งแต่แรก
- แยก Mobile และ Web stack ชัดเจนช่วยให้แต่ละ platform optimize user experience ได้ตามข้อจำกัดของ device และ behavior ของผู้ใช้

## OPERATIONAL INFRASTRUCTURE

- Container & Orchestration:
  - Docker
  - Kubernetes
- Networking & Gateway:
  - Kong (API Gateway)
  - Istio (Service Mesh, Load Balancing)
- Messaging:
  - Kafka
- Caching:
  - Redis
- Observability:
  - OpenTelemetry
    - Prometheus
    - Grafana
    - Loki
    - Jaeger
- Key Management System:
  - AWS KMS
- Secrets Management:
  - AWS Secrets Manager
