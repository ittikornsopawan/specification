---
type: System Architecture
status: proposed
updated: 2026-04-24
date: 2026-04-24
---

# TECHNICAL DESIGN: e-Commerce Platform

## TERMS AND DEFINITIONS

| Term                                            | Definition                                                                                                                                                                                                     |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Domain-Driven Design (DDD)                      | แนวทางการออกแบบซอฟต์แวร์โดยยึด Domain หรือบริบทธุรกิจเป็นหลัก เพื่อให้โครงสร้างระบบสะท้อนกฎทางธุรกิจ คำศัพท์ทางธุรกิจ และกระบวนการทำงานจริง                                                                                           |
| Event-Driven Design (EDD)                       | แนวทางการออกแบบระบบที่ให้แต่ละส่วนสื่อสารกันผ่าน Event หรือ Event ที่เกิดขึ้นในระบบ เช่น Order Created, Payment Paid หรือ Stock Reserved                                                                                        |
| Bounded Context                                 | ขอบเขตของ Domain ที่มีความหมาย business rule, business model และ business language เพื่อป้องกันความสับสนระหว่างส่วนต่าง ๆ ของระบบที่ใช้คำเหมือนกันแต่ความหมายไม่เหมือนกัน                                                           |
| Clean Architecture                              | แนวทางการออกแบบโครงสร้างระบบโดยแยกความรับผิดชอบเป็นชั้น ๆ (Layers) เพื่อให้ business logic ไม่ผูกติดกับ framework, database, UI หรือ external service ทำให้ระบบดูแล แก้ไข และทดสอบได้ง่ายขึ้น                                        |
| Coupling                                        | ระดับการพึ่งพากันระหว่างส่วนต่าง ๆ ของระบบ ถ้า coupling สูง การแก้ส่วนหนึ่งอาจกระทบอีกหลายส่วน แต่ถ้า coupling ต่ำ ระบบจะดูแล แก้ไข และขยายต่อได้ง่ายกว่า                                                                               |
| Scalability                                     | ความสามารถของระบบในการรองรับการเติบโต เช่น ผู้ใช้มากขึ้น ข้อมูลมากขึ้น หรือ request มากขึ้น โดยยังสามารถเพิ่ม resource หรือปรับโครงสร้างเพื่อให้ระบบทำงานต่อได้อย่างเหมาะสม                                                                |
| Command Query Responsibility Segregation (CQRS) | แนวทางการออกแบบระบบที่แยกการทำงานฝั่ง Command สำหรับการเปลี่ยนแปลงข้อมูล เช่น create, update, delete ออกจากฝั่ง Query สำหรับการอ่านข้อมูล เพื่อให้แต่ละฝั่งออกแบบและ scale ได้เหมาะกับหน้าที่ของตัวเอง                                        |
| Presentation Layer                              | Layer ที่รับผิดชอบการติดต่อกับผู้ใช้หรือระบบภายนอก เช่น API Controller, Web UI หรือ Mobile UI โดยทำหน้าที่รับ request, ตรวจรูปแบบข้อมูลเบื้องต้น และส่งต่อไปยัง Application Layer                                                         |
| Application Layer                               | Layer ที่ควบคุม flow การทำงานของ use case เช่น สร้าง order, สมัครสมาชิก หรือชำระเงิน โดยประสานงานระหว่าง Domain Layer กับ Infrastructure Layer แต่ไม่ควรเก็บ business rule หลักไว้เอง                                            |
| Domain Layer                                    | Layer ที่เก็บ business logic, business rule, entity, value object และ domain behavior ที่สำคัญของระบบ เป็นหัวใจของระบบและไม่ควรผูกติดกับ framework, database หรือ external service                                           |
| Domain Layer: Entity                            | object ใน Domain ที่มี identity เป็นของตัวเอง เช่น Order, Customer หรือ Product ถึงข้อมูลบางอย่างจะเปลี่ยนไป แต่ยังถือว่าเป็นสิ่งเดิมเพราะอ้างอิงจาก identity เดิม                                                                     |
| Domain Layer: Value Object                      | object ใน Domain ที่ไม่มี identity เป็นของตัวเอง และใช้เปรียบเทียบจากค่าข้างใน เช่น Money, Address หรือ DateRange ถ้าค่าทุกอย่างเหมือนกันก็ถือว่าเป็นสิ่งเดียวกัน                                                                         |
| Domain Layer: Aggregate                         | กลุ่มของ Entity และ Value Object ที่ถูกจัดให้อยู่ภายใต้ขอบเขตเดียวกัน โดยมี Aggregate Root เป็นตัวหลักในการควบคุมการเปลี่ยนแปลง เพื่อรักษา business rule และความถูกต้องของข้อมูล                                                         |
| Domain Layer: Domain Service                    | service ที่เก็บ business logic ที่ไม่เหมาะจะอยู่ใน Entity หรือ Value Object ใดตัวหนึ่งโดยตรง มักใช้กับ logic ที่เกี่ยวข้องกับหลาย object ใน Domain                                                                                  |
| Domain Event                                    | เหตุการณ์สำคัญที่เกิดขึ้นใน Domain และมีความหมายทางธุรกิจ เช่น Order Created, Payment Completed หรือ Stock Reserved ใช้เพื่อบอกว่าสิ่งหนึ่งเกิดขึ้นแล้ว และส่วนอื่นของระบบสามารถนำ event นี้ไปทำงานต่อได้                                         |
| Infrastructure Layer                            | Layer ที่รับผิดชอบรายละเอียดทางเทคนิค เช่น database, repository implementation, external API, file storage, message queue, email service หรือ cloud service เพื่อให้ชั้นอื่นเรียกใช้งานผ่าน interface ได้โดยไม่ต้องรู้รายละเอียดภายใน  |
| Decouple Service                                | การออกแบบให้ service แต่ละตัวลดการพึ่งพากันโดยตรง เพื่อให้แต่ละ service สามารถพัฒนา แก้ไข deploy และ scale ได้อิสระมากขึ้น โดยไม่ทำให้การเปลี่ยนแปลงของ service หนึ่งกระทบ service อื่นมากเกินไป                                         |
| Eventual Consistency                            | แนวคิดที่ยอมให้ข้อมูลในแต่ละ service หรือแต่ละ database ไม่ตรงกันทันทีในช่วงเวลาสั้น ๆ แต่สุดท้ายระบบจะค่อย ๆ sync จนข้อมูลกลับมาถูกต้องตรงกัน เหมาะกับระบบแบบ distributed system หรือ microservices ที่ต้องการลดการผูกติดกันและเพิ่ม scalability  |
| Google Remote Procedure Calls (gRPC)            | รูปแบบการสื่อสารระหว่างระบบหรือ service โดยใช้ contract ที่ชัดเจนผ่านไฟล์ `.proto` เหมาะกับการสื่อสารแบบ service-to-service ที่ต้องการ performance ดีและโครงสร้างข้อมูลแน่นอน                                                        |
| RESTful                                         | รูปแบบการออกแบบ API ที่ใช้ HTTP method เช่น GET, POST, PUT, DELETE เพื่อให้ client เรียกใช้งาน resource ของระบบได้อย่างเข้าใจง่ายและเป็นมาตรฐาน                                                                               |
| Service-to-Service                              | การสื่อสารระหว่าง service ภายในระบบ เช่น Order Service เรียก Payment Service เพื่อทำงานต่อ เหมาะกับระบบที่แยกเป็นหลาย service หรือ microservices                                                                             |
| Client-to-System                                | การสื่อสารจาก client เช่น Web, Mobile App หรือ External Partner เข้ามายังระบบผ่าน API หรือช่องทางที่ระบบเปิดไว้                                                                                                             |
| Async Communication                             | การสื่อสารแบบไม่ต้องรอผลลัพธ์ทันที ผู้ส่งสามารถส่ง message หรือ event ออกไป แล้วให้ปลายทางประมวลผลภายหลัง เหมาะกับงานที่ต้องการ decouple service และรองรับ scalability                                                              |
| Sync Communication                              | การสื่อสารแบบรอผลลัพธ์ทันที ผู้เรียกต้องรอ response จากปลายทางก่อนจึงจะทำงานต่อได้ เหมาะกับงานที่ต้องการคำตอบทันที เช่น ตรวจสอบข้อมูลหรือคำนวณผลลัพธ์แบบ real-time                                                                          |
| Fault Tolerance                                 | ความสามารถของระบบในการทำงานต่อได้ แม้บางส่วนของระบบจะเกิดปัญหา เช่น service ล่ม, network ขาด, database ช้า หรือ dependency ภายนอกใช้งานไม่ได้ชั่วคราว                                                                          |
| Transient Failure                               | ความผิดพลาดแบบชั่วคราวที่มักหายได้เองหรือสำเร็จเมื่อ retry ใหม่ เช่น network timeout, service ตอบช้า, database connection หลุด หรือ external API มีปัญหาชั่วครู่                                                                      |
| Circuit Breaker                                 | pattern ที่ใช้ตัดการเรียกไปยัง service ที่มีปัญหาชั่วคราว เพื่อป้องกันไม่ให้ระบบเรียกซ้ำจนล้มเป็นลูกโซ่ และจะค่อย ๆ เปิดให้ลองเรียกใหม่เมื่อ service ปลายทางเริ่มกลับมาปกติ                                                                        |
| Idempotency                                     | คุณสมบัติของ operation ที่เรียกซ้ำหลายครั้งแล้วผลลัพธ์สุดท้ายยังเหมือนเดิม เหมาะกับงานที่เสี่ยงถูก retry เช่น create order, payment หรือ transaction ต่าง ๆ                                                                               |
| Idempotency Key                                 | key ที่ใช้ระบุว่า request นี้เคยถูกประมวลผลแล้วหรือยัง เพื่อป้องกันการทำงานซ้ำ เช่น กดจ่ายเงินซ้ำ หรือ retry request แล้วระบบสร้าง order/payment ซ้ำ                                                                                       |
| Compensating Action                             | action ที่ใช้ชดเชยหรือย้อนผลกระทบจาก step ก่อนหน้าที่ทำสำเร็จไปแล้ว เมื่อ process หลักทำงานต่อไม่สำเร็จ เช่น ถ้าตัดเงินสำเร็จแล้วแต่สร้าง order ไม่สำเร็จ ระบบอาจต้องทำ refund เพื่อชดเชยผลลัพธ์                                                      |
| Saga Pattern                                    | pattern สำหรับจัดการ transaction ที่เกี่ยวข้องกับหลาย service โดยแบ่งงานใหญ่ออกเป็นหลาย step ย่อย และถ้า step ใดล้มเหลว ระบบจะใช้ compensating action เพื่อย้อนหรือชดเชยผลกระทบที่เกิดขึ้นแทนการ rollback แบบ database transaction เดียว |
| Choreography-based Saga                         | รูปแบบของ Saga Pattern ที่แต่ละ service ทำงานต่อกันผ่าน event โดยไม่มีตัวกลางคอยสั่งงานหลัก เมื่อ service หนึ่งทำงานเสร็จจะ publish event ออกไป แล้ว service อื่นที่สนใจ event นั้นจะทำงานต่อเอง                                            |
| Outbox Pattern                                  | pattern ที่ใช้บันทึก event หรือ message ลงใน database เดียวกับ business transaction ก่อน แล้วค่อยมี process แยกไปส่ง message ออกไปภายหลัง เพื่อป้องกันปัญหา business data ถูกบันทึกสำเร็จแต่ event ส่งไม่สำเร็จ                             |
| Idempotent Consumer                             | consumer ที่ถูกออกแบบให้รับ message เดิมซ้ำได้โดยไม่ทำให้ผลลัพธ์ผิดพลาด เช่น message ถูก retry หรือส่งซ้ำ แต่ระบบไม่สร้างข้อมูลซ้ำ ไม่ตัดเงินซ้ำ และไม่เปลี่ยนสถานะผิด                                                                              |
| At-least-once Delivery                          | รูปแบบการส่ง message ที่รับประกันว่า message จะถูกส่งถึงปลายทางอย่างน้อย 1 ครั้ง แต่อาจถูกส่งซ้ำได้ ดังนั้นฝั่ง consumer ควรออกแบบให้เป็น idempotent เพื่อป้องกันการประมวลผลซ้ำแล้วเกิดผลลัพธ์ผิดพลาด                                                |

## ARCHITECTURE SNAPSHOT

### MICROSERVICES ARCHITECTURE

#### CODE STRUCTURE

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

#### PROTOCOL

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

### ARCHITECTURE PATTERN

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

### BACKEND API

#### Identity & Access Layer

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

#### Core e-Commerce

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

#### Payment & Financial

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

#### Fulfillment

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

#### Growth / Marketing

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

#### Communication

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

### BACKEND FOR FRONTEND (BFF)

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

### FRONTEND / CLIENT APPLICATION

รองรับช่องทางการใช้งานหลักของระบบทั้ง Mobile Application และ Web Application โดยแยก technology ให้เหมาะกับแต่ละ platform

#### Mobile Application

- Stack:
  - [X] iOS + Swift: เหมาะกับการพัฒนา native iOS application เพราะทำงานกับ Apple ecosystem ได้เต็มประสิทธิภาพ และรองรับ UX/UI ที่เป็นมาตรฐานของ iOS
  - [X] Android + Kotlin: เหมาะกับการพัฒนา native Android application เพราะเป็นภาษาหลักที่ Google แนะนำสำหรับ Android และทำงานร่วมกับ Android SDK ได้ดี
  - [ ] Flutter: เหมาะเป็น option ถ้าต้องการพัฒนา mobile app แบบ cross-platform ด้วย codebase เดียว แต่ยังต้องการ performance และ UI consistency ที่ดี

#### Web Application

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

### OPERATIONAL INFRASTRUCTURE

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

---

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
