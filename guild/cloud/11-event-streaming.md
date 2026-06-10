# 11. Event Streaming

Event Streaming คือ Service กลุ่มที่ออกแบบมาสำหรับ process data stream แบบ real-time ต่างจาก Message Queue ตรงที่เก็บ event ไว้เป็นเวลานาน consumer หลายตัวอ่าน event stream เดิมได้อิสระต่อกัน และรองรับ throughput สูงมาก

---

## Managed Kafka / Event Streaming

### คืออะไร

Managed Kafka หรือ Event Streaming Service คือบริการ Apache Kafka หรือ Kafka-compatible ที่ Cloud Provider จัดการ broker, replication และ scaling ให้ ใช้สำหรับ ingest และ process event ปริมาณมากแบบ real-time

### ใช้งานแบบไหน

Producer ส่ง event เข้า topic (แบ่งเป็น partition) Consumer Group ดึง event จาก partition ตาม offset ของตัวเอง ทำให้หลาย consumer group อ่าน stream เดิมได้อิสระ

### เหมาะกับงานแบบไหน

เหมาะกับ real-time data pipeline, log aggregation, change data capture (CDC), event sourcing, activity tracking, หรือ workload ที่มี throughput หลายล้าน event ต่อวินาที

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ simple task queue ที่ต้องการ per-message retry หรือ workload ที่ event volume ต่ำ เพราะ Kafka มี operational complexity สูง

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                                    |
| -------------- | --------------------------------------------------------------- |
| AWS            | Amazon MSK (Managed Streaming for Apache Kafka), Amazon Kinesis |
| GCP            | Cloud Pub/Sub (Kafka-compatible), Managed Kafka                 |
| Azure          | Azure Event Hubs (Kafka-compatible)                             |
| Huawei Cloud   | Distributed Message Service (DMS) for Kafka                     |

### Spec / Configuration

#### AWS MSK Broker Instance Type

| Instance Type      | เหมาะกับงานแบบไหน           | On-demand ($/broker-hr) | Reserved 1yr |
| ------------------ | -------------------------- | ----------------------: | ------------ |
| `kafka.t3.small`   | dev/test เท่านั้น             |                   0.054 | ~0.034       |
| `kafka.m5.large`   | small to medium production |                   0.210 | ~0.133       |
| `kafka.m5.xlarge`  | medium to high throughput  |                   0.420 | ~0.265       |
| `kafka.m5.2xlarge` | high throughput workload   |                   0.840 | ~0.530       |

### Kafka Configuration

| Spec / Configuration | ความหมาย                                    | ตัวอย่าง                                  |
| -------------------- | ------------------------------------------- | --------------------------------------- |
| Topic                | channel หลักสำหรับ event stream                | `user-activity`, `transactions`         |
| Partition            | การแบ่ง topic เพื่อ parallel processing        | 6, 12, 24 partitions                    |
| Replication Factor   | จำนวน copy ของแต่ละ partition                 | 3 (production minimum)                  |
| Retention Period     | ระยะเวลาเก็บ event ใน topic                  | 7 วัน                                    |
| Consumer Group       | กลุ่ม consumer ที่อ่าน stream ร่วมกัน              | `analytics-group`, `notification-group` |
| Offset               | ตำแหน่งที่ consumer อ่านถึง                       | earliest, latest, specific offset       |
| Compression          | การ compress message เพื่อลด network overhead | gzip, snappy, lz4                       |
| Schema Registry      | จัดการ schema ของ event (Avro, Protobuf)     | Confluent Schema Registry               |

### ตัวอย่างการใช้งานใน Project

```
Web App → Kafka Topic: page-views (24 partitions)
    ├── Consumer Group: real-time-dashboard → update live counter
    ├── Consumer Group: ml-pipeline → feed recommendation model
    └── Consumer Group: data-warehouse → batch load ไป BigQuery
```

### Best Practice

- กำหนด Replication Factor ≥ 3 สำหรับ production topic
- เลือกจำนวน partition ให้เหมาะสมตั้งแต่ต้น การเพิ่ม partition ภายหลังส่งผลต่อ ordering
- ใช้ Schema Registry เพื่อจัดการ schema evolution
- monitor consumer lag สม่ำเสมอ

### Common Mistakes

- ตั้ง partition น้อยเกินไป ทำให้ scale consumer ได้จำกัด
- Replication Factor = 1 ทำให้ data สูญเมื่อ broker crash
- ไม่ monitor consumer lag ทำให้ไม่รู้ว่า consumer ตาม producer ไม่ทัน

---
