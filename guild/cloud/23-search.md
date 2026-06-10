# 23. Search

Search คือ Service กลุ่มที่ให้ความสามารถ full-text search, relevance ranking และ faceted search ที่ database ทั่วไปทำได้ไม่ดีนัก

---

## Managed Search Service

### คืออะไร

Managed Search Service คือบริการ search engine ที่ Cloud Provider จัดการ cluster, scaling, patching ให้ ส่วนใหญ่ใช้ Elasticsearch หรือ OpenSearch เป็น engine รองรับ full-text search, relevance scoring, aggregation และ real-time indexing

### ใช้งานแบบไหน

index document เข้า search engine แล้ว query ด้วย full-text search พร้อม filter, aggregation และ highlight ใช้กับ product catalog search, document search, log analysis

### เหมาะกับงานแบบไหน

เหมาะกับ product search ที่ต้องการ relevance, document management, log analysis, e-commerce search with facets

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ primary transactional database เพราะ search engine ไม่ใช่ ACID database

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                           |
| -------------- | -------------------------------------- |
| AWS            | Amazon OpenSearch Service              |
| GCP            | Vertex AI Search, Elastic Cloud on GCP |
| Azure          | Azure AI Search                        |
| Huawei Cloud   | Cloud Search Service (CSS)             |

### Spec / Configuration

#### AWS OpenSearch Instance Type

| Instance Type       | vCPU | Memory | เหมาะกับงานแบบไหน               | On-demand ($/hr) | RI 1yr All Upfront | RI 3yr All Upfront |
| ------------------- | ---: | -----: | ------------------------------ | ---------------: | -----------------: | -----------------: |
| `t3.small.search`   |    2 |   2 GB | dev/test เท่านั้น                 |            0.036 |              0.023 |              0.016 |
| `m6g.large.search`  |    2 |   8 GB | small production               |            0.148 |              0.093 |              0.066 |
| `m6g.xlarge.search` |    4 |  16 GB | medium production              |            0.296 |              0.186 |              0.132 |
| `r6g.large.search`  |    2 |  16 GB | memory-heavy, large index      |            0.219 |              0.138 |              0.098 |
| `r6g.xlarge.search` |    4 |  32 GB | large index, high query volume |            0.438 |              0.276 |              0.196 |

### Search Configuration

| Spec / Configuration | ความหมาย                                      | ตัวอย่าง                               |
| -------------------- | --------------------------------------------- | ------------------------------------ |
| Index                | ที่เก็บ document collection                      | `products`, `articles`               |
| Shard                | การแบ่ง index เป็นส่วน ๆ เพื่อ parallel processing | 3 primary shards                     |
| Replica              | copy ของ shard เพื่อ high availability          | 1 replica                            |
| Mapping              | กำหนด data type ของแต่ละ field                  | `title: text`, `price: float`        |
| Analyzer             | วิธีการ tokenize และ process text               | thai language analyzer               |
| Query Type           | ประเภทของ query                               | full-text, term, range, bool, vector |

### ตัวอย่างการใช้งานใน Project

E-commerce ใช้ OpenSearch สำหรับ product search โดย sync ข้อมูลจาก RDS เมื่อมี product update ผ่าน event และ serve search query โดยตรงจาก OpenSearch

### Best Practice

- แยก read node และ write operation เพื่อ performance
- ตั้ง replica ≥ 1 สำหรับ production
- กำหนด index mapping ก่อน ingest data ไม่ใช่ dynamic mapping
- monitor index size และ query latency

### Common Mistakes

- ใช้ dynamic mapping ทำให้ type inference ผิดพลาด
- ไม่ได้ตั้ง replica ทำให้ data หายเมื่อ node crash
- ใช้ search engine เป็น primary database
