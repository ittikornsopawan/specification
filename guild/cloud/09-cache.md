# 9. Cache

Cache คือ Service กลุ่มที่ช่วยเก็บข้อมูลที่เข้าถึงบ่อยใน memory เพื่อลด latency และลด load บน database หลัก ช่วยให้ระบบ scale ได้โดยไม่ต้อง scale database เร็ว ๆ

---

## In-Memory Cache (Redis / Memcached)

### คืออะไร

In-Memory Cache คือ managed cache service ที่เก็บข้อมูลใน RAM ทำให้เข้าถึงได้ในระดับ sub-millisecond รองรับ data structure หลากหลายเช่น string, hash, list, set, sorted set (Redis) และ simple key-value (Memcached)

### ใช้งานแบบไหน

ใช้เป็น cache layer หน้า database เก็บผลลัพธ์ของ query ที่ใช้บ่อย, เก็บ user session, เก็บ rate limit counter, หรือใช้เป็น message broker เบื้องต้น (Redis Pub/Sub)

### เหมาะกับงานแบบไหน

เหมาะกับ session storage, database query cache, rate limiting, leaderboard, real-time analytics, distributed lock

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับการเก็บข้อมูลถาวรที่ต้องการ durability สูงโดยไม่มี persistence strategy เพราะ memory อาจหายได้

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                               |
| -------------- | ---------------------------------------------------------- |
| AWS            | Amazon ElastiCache (Redis OSS, Memcached), Amazon MemoryDB |
| GCP            | Memorystore for Redis, Memorystore for Memcached           |
| Azure          | Azure Cache for Redis                                      |
| Huawei Cloud   | Distributed Cache Service (DCS)                            |

### Spec / Configuration

#### AWS ElastiCache Node Type

| Node Type           | vCPU |   Memory | เหมาะกับงานแบบไหน            | On-demand ($/hr) | RI 1yr All Upfront | RI 3yr All Upfront |
| ------------------- | ---: | -------: | --------------------------- | ---------------: | -----------------: | -----------------: |
| `cache.t3.micro`    |    2 |   0.5 GB | dev/test เท่านั้น              |            0.017 |              0.011 |              0.008 |
| `cache.t3.small`    |    2 |  1.37 GB | small application           |            0.034 |              0.021 |              0.016 |
| `cache.t3.medium`   |    2 |  3.09 GB | small to medium application |            0.068 |              0.043 |              0.032 |
| `cache.r6g.large`   |    2 | 13.07 GB | production, general purpose |            0.166 |              0.105 |              0.075 |
| `cache.r6g.xlarge`  |    4 | 26.04 GB | production, high traffic    |            0.332 |              0.210 |              0.150 |
| `cache.r6g.2xlarge` |    8 | 52.82 GB | production, heavy workload  |            0.664 |              0.419 |              0.300 |

#### GCP Memorystore Tier

| Tier     | Memory Range  | เหมาะกับงานแบบไหน               | $/GB-hr |
| -------- | ------------- | ------------------------------ | ------: |
| Basic    | 1 GB – 300 GB | dev/test, no replication       |  $0.016 |
| Standard | 1 GB – 300 GB | production, automatic failover |  $0.049 |

#### Azure Cache for Redis SKU

| SKU         | Memory | เหมาะกับงานแบบไหน               | PAYG ($/hr) |
| ----------- | -----: | ------------------------------ | ----------: |
| Basic C0    | 250 MB | dev/test เท่านั้น                 |       0.016 |
| Standard C1 |   1 GB | small application              |       0.101 |
| Standard C2 |   6 GB | medium application             |       0.202 |
| Premium P1  |   6 GB | production, clustering support |       0.544 |
| Premium P2  |  13 GB | production, high traffic       |       1.088 |

#### Huawei Cloud DCS Flavor

| Flavor                 | Memory | เหมาะกับงานแบบไหน        | On-demand ($/hr) |
| ---------------------- | -----: | ----------------------- | ---------------: |
| `redis.single.small.1` |   1 GB | dev/test                |           ~0.020 |
| `redis.ha.medium.4`    |   4 GB | small production        |           ~0.075 |
| `redis.ha.large.8`     |   8 GB | medium production       |           ~0.150 |
| `redis.ha.xlarge.16`   |  16 GB | high traffic production |           ~0.300 |

### Cache Configuration

| Spec / Configuration  | ความหมาย                             | ตัวอย่าง                                |
| --------------------- | ------------------------------------ | ------------------------------------- |
| Engine & Version      | cache engine ที่ใช้                     | Redis 7.x, Memcached 1.6              |
| Cluster Mode          | เปิด cluster สำหรับ horizontal sharding | Cluster Mode Enabled/Disabled         |
| Replication           | จำนวน replica ต่อ shard                | 1-5 replicas                          |
| Eviction Policy       | วิธีลบ key เมื่อ memory เต็ม              | allkeys-lru, volatile-lru, noeviction |
| TTL Default           | เวลาหมดอายุของ key                    | กำหนดต่อ key ตอน SET                    |
| Persistence           | การ persist data ลง disk (Redis)     | RDB snapshot, AOF log                 |
| Encryption in Transit | เข้ารหัส TLS สำหรับ connection           | เปิดเพื่อความปลอดภัย                      |
| Auth Token            | password สำหรับ authenticate           | ใช้ AUTH token                         |

### ตัวอย่างการใช้งานใน Project

```
API Server → Redis Cache → (hit) → return cached response
           ↓ (miss)
           Database → save to Redis with TTL 300s → return response
```

### Best Practice

- กำหนด TTL ทุก key เสมอ ไม่ควรเก็บ key ที่ไม่มี expiry ใน production
- ใช้ Redis Cluster สำหรับ data ขนาดใหญ่หรือ throughput สูง
- monitor memory usage และ eviction rate สม่ำเสมอ
- ใช้ read replica ใน replication group เพื่อ scale read

### Common Mistakes

- ไม่ได้ตั้ง TTL ทำให้ memory เต็มจาก stale data
- เก็บ object ขนาดใหญ่มากใน Redis ทำให้ latency สูง
- ไม่ได้ตั้ง eviction policy ทำให้ cache error เมื่อ memory เต็ม
