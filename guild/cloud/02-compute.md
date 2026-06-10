# 2. Compute

Compute เป็น Service กลุ่มที่ให้ computational power สำหรับ run application ตั้งแต่ Virtual Machine ไปจนถึง bare metal server การเลือก Compute ที่เหมาะสมส่งผลต่อ performance, cost และการบริหารจัดการ

---

## Virtual Machine (VM) / Compute Instance

### คืออะไร

Virtual Machine (VM) หรือ Compute Instance คือเครื่อง server เสมือนที่ run บน Cloud Infrastructure ผู้ใช้สามารถกำหนด OS, CPU, Memory, Storage และ Network ได้เอง เป็น Infrastructure as a Service (IaaS) ที่ให้ control ระดับสูงสุด

### ใช้งานแบบไหน

ใช้ deploy application ที่ต้องการ full OS control เช่น legacy application, application ที่ต้องการ custom kernel หรือ driver หรือ application ที่ยังไม่ได้ containerize นอกจากนี้ยังใช้เป็น worker node ใน Kubernetes cluster

### เหมาะกับงานแบบไหน

เหมาะกับ legacy application ที่ไม่สามารถ containerize ได้, workload ที่ต้องการ persistent OS state, database server, หรือ workload ที่ต้องการ hardware access พิเศษ

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ scale เร็วมาก หรือ stateless microservice ที่ควรใช้ Container แทน เพราะ VM มี boot time นานกว่าและ overhead ในการจัดการสูงกว่า

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name               |
| -------------- | -------------------------- |
| AWS            | Amazon EC2                 |
| GCP            | Compute Engine             |
| Azure          | Azure Virtual Machines     |
| Huawei Cloud   | Elastic Cloud Server (ECS) |

### Spec / Configuration

#### AWS Instance Type

| Instance Type | vCPU | Memory | เหมาะกับงานแบบไหน                                | On-demand ($/hr) | RI 1yr No Upfront | RI 1yr All Upfront | RI 3yr All Upfront | Spot avg |
| ------------- | ---: | -----: | ----------------------------------------------- | ---------------: | ----------------: | -----------------: | -----------------: | -------: |
| `t3.medium`   |    2 |   4 GB | small application, dev/test workload            |           0.0416 |            0.0280 |             0.0267 |             0.0188 |  ~0.0125 |
| `t3.large`    |    2 |   8 GB | small to medium application, dev/test           |           0.0832 |            0.0560 |             0.0534 |             0.0376 |  ~0.0250 |
| `m6i.large`   |    2 |   8 GB | general purpose backend, Kubernetes worker node |           0.0960 |            0.0629 |             0.0604 |             0.0430 |  ~0.0288 |
| `m6i.xlarge`  |    4 |  16 GB | general purpose backend, medium traffic         |           0.1920 |            0.1258 |             0.1208 |             0.0860 |  ~0.0576 |
| `m6i.2xlarge` |    8 |  32 GB | general purpose backend, high traffic           |           0.3840 |            0.2516 |             0.2416 |             0.1720 |  ~0.1152 |
| `c6i.large`   |    2 |   4 GB | compute-heavy workload, batch processing        |           0.0850 |            0.0549 |             0.0527 |             0.0374 |  ~0.0255 |
| `c6i.xlarge`  |    4 |   8 GB | compute-heavy workload, video transcoding       |           0.1700 |            0.1098 |             0.1054 |             0.0748 |  ~0.0510 |
| `r6i.large`   |    2 |  16 GB | memory-heavy workload, in-memory cache          |           0.1260 |            0.0825 |             0.0792 |             0.0561 |  ~0.0378 |
| `r6i.xlarge`  |    4 |  32 GB | memory-heavy workload, large database           |           0.2520 |            0.1650 |             0.1584 |             0.1122 |  ~0.0756 |

#### GCP Machine Type

| Machine Type    | vCPU | Memory | เหมาะกับงานแบบไหน                                | On-demand ($/hr) | CUD 1yr | CUD 3yr | Spot avg |
| --------------- | ---: | -----: | ----------------------------------------------- | ---------------: | ------: | ------: | -------: |
| `e2-medium`     |    2 |   4 GB | small application, dev/test workload            |           0.0335 |       — |       — |  ~0.0101 |
| `e2-standard-2` |    2 |   8 GB | small to medium application                     |           0.0672 |  0.0424 |  0.0329 |  ~0.0168 |
| `n2-standard-2` |    2 |   8 GB | general purpose backend, Kubernetes worker node |           0.0971 |  0.0613 |  0.0476 |  ~0.0243 |
| `n2-standard-4` |    4 |  16 GB | general purpose backend, medium traffic         |           0.1942 |  0.1226 |  0.0952 |  ~0.0486 |
| `n2-standard-8` |    8 |  32 GB | general purpose backend, high traffic           |           0.3884 |  0.2452 |  0.1904 |  ~0.0972 |
| `c2-standard-4` |    4 |  16 GB | compute-heavy workload                          |           0.2088 |  0.1318 |  0.1024 |  ~0.0522 |
| `c2-standard-8` |    8 |  32 GB | compute-heavy workload                          |           0.4176 |  0.2636 |  0.2048 |  ~0.1044 |
| `n2-highmem-2`  |    2 |  16 GB | memory-heavy workload                           |           0.1310 |  0.0827 |  0.0643 |  ~0.0328 |
| `n2-highmem-4`  |    4 |  32 GB | memory-heavy workload, large database           |           0.2620 |  0.1654 |  0.1286 |  ~0.0655 |

#### Azure VM Size

| VM Size  | vCPU | Memory | เหมาะกับงานแบบไหน                                | PAYG ($/hr) | Reserved 1yr | Reserved 3yr | Spot avg |
| -------- | ---: | -----: | ----------------------------------------------- | ----------: | -----------: | -----------: | -------: |
| `B2s`    |    2 |   4 GB | small application, dev/test workload            |      0.0416 |       0.0267 |       0.0188 |  ~0.0083 |
| `B2ms`   |    2 |   8 GB | small to medium application                     |      0.0832 |       0.0534 |       0.0375 |  ~0.0125 |
| `D2s v5` |    2 |   8 GB | general purpose backend, Kubernetes worker node |      0.0960 |       0.0620 |       0.0432 |  ~0.0192 |
| `D4s v5` |    4 |  16 GB | general purpose backend, medium traffic         |      0.1920 |       0.1240 |       0.0864 |  ~0.0384 |
| `D8s v5` |    8 |  32 GB | general purpose backend, high traffic           |      0.3840 |       0.2480 |       0.1728 |  ~0.0768 |
| `F2s v2` |    2 |   4 GB | compute-heavy workload                          |      0.0845 |       0.0545 |       0.0384 |  ~0.0127 |
| `F4s v2` |    4 |   8 GB | compute-heavy workload                          |      0.1690 |       0.1090 |       0.0768 |  ~0.0254 |
| `E2s v5` |    2 |  16 GB | memory-heavy workload                           |      0.1260 |       0.0830 |       0.0594 |  ~0.0189 |
| `E4s v5` |    4 |  32 GB | memory-heavy workload, large database           |      0.2520 |       0.1660 |       0.1188 |  ~0.0378 |

#### Huawei Cloud Flavor

| Flavor         | vCPU | Memory | เหมาะกับงานแบบไหน                                | On-demand ($/hr) | Reserved 1yr | Spot avg |
| -------------- | ---: | -----: | ----------------------------------------------- | ---------------: | -----------: | -------: |
| `s6.large.2`   |    2 |   4 GB | small application, dev/test workload            |           ~0.040 |       ~0.026 |   ~0.012 |
| `s6.large.4`   |    2 |   8 GB | small to medium application                     |           ~0.085 |       ~0.055 |   ~0.026 |
| `c6.large.4`   |    2 |   8 GB | general purpose backend, Kubernetes worker node |           ~0.085 |       ~0.055 |   ~0.026 |
| `c6.xlarge.4`  |    4 |  16 GB | general purpose backend, medium traffic         |           ~0.170 |       ~0.111 |   ~0.051 |
| `c6.2xlarge.4` |    8 |  32 GB | general purpose backend, high traffic           |           ~0.340 |       ~0.221 |   ~0.102 |
| `c6.xlarge.2`  |    4 |   8 GB | compute-heavy workload                          |           ~0.160 |       ~0.104 |   ~0.048 |
| `m6.large.8`   |    2 |  16 GB | memory-heavy workload                           |           ~0.120 |       ~0.078 |   ~0.036 |
| `m6.xlarge.8`  |    4 |  32 GB | memory-heavy workload, large database           |           ~0.240 |       ~0.156 |   ~0.072 |

### Spec / Configuration อื่น ๆ ที่ควรรู้

| Spec / Configuration        | ความหมาย                                                          | ตัวอย่าง                               |
| --------------------------- | ----------------------------------------------------------------- | ------------------------------------ |
| OS Image / AMI              | Image ของ OS ที่ใช้ boot VM                                          | Amazon Linux 2, Ubuntu 22.04 LTS     |
| Storage Type                | ประเภทของ disk ที่ attach กับ VM                                     | gp3 SSD, io1 SSD, HDD                |
| Key Pair                    | SSH key สำหรับเข้าถึง VM                                              | `prod-bastion-key.pem`               |
| Auto Scaling                | ปรับจำนวน VM อัตโนมัติตาม load                                         | min 2, max 10 instances              |
| Spot / Preemptible Instance | VM ราคาถูกที่ Cloud อาจ terminate ได้                                 | เหมาะกับ batch job ที่ interruptible ได้ |
| Placement Group             | การจัดวาง VM ให้อยู่ใกล้กัน (low latency) หรือกระจาย (high availability) | Cluster, Spread, Partition           |

### ตัวอย่างการใช้งานใน Project

Deploy Backend API Server บน EC2 / Compute Engine โดยอยู่ใน Private Subnet, รับ traffic ผ่าน Load Balancer, ใช้ Auto Scaling Group เพื่อขยายตาม request volume

### Best Practice

- ใช้ Auto Scaling Group ร่วมกับ Load Balancer ทุกครั้งเพื่อ high availability
- เลือก Instance Type ให้ตรงกับ workload pattern เช่น compute-heavy ใช้ c-family, memory-heavy ใช้ r-family
- ใช้ Spot Instance สำหรับ batch processing หรือ non-critical workload เพื่อลดต้นทุน
- ไม่ควร SSH เข้า production instance โดยตรง ใช้ Bastion Host หรือ Systems Manager Session Manager แทน

### Common Mistakes

- เลือก Instance Type ที่ใหญ่เกินจำเป็น
- ไม่ได้ตั้ง Auto Scaling ทำให้ระบบล่มเมื่อ traffic พุ่ง
- เก็บ application code และ data ไว้บน root disk ทำให้ข้อมูลหายเมื่อ instance terminate

---

## Auto Scaling

### คืออะไร

Auto Scaling คือ Service ที่ปรับจำนวน compute instance โดยอัตโนมัติตาม policy ที่กำหนด เช่น CPU utilization, request count หรือ custom metric ช่วยให้ระบบรองรับ traffic ที่เปลี่ยนแปลงได้โดยไม่ต้องปรับ capacity เอง

### ใช้งานแบบไหน

กำหนด minimum, maximum และ desired capacity จากนั้นตั้ง scaling policy เช่น เพิ่ม instance เมื่อ CPU > 70% หรือ ลด instance เมื่อ CPU < 30% นาน 10 นาที

### เหมาะกับงานแบบไหน

เหมาะกับ stateless application ที่รับ web traffic, batch processing worker, หรือ microservice ที่ traffic ไม่สม่ำเสมอ

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ stateful application ที่ instance แต่ละตัวเก็บ state ต่างกัน เช่น database cluster ที่มีการ coordinate กันซับซ้อน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                      |
| -------------- | ------------------------------------------------- |
| AWS            | Amazon EC2 Auto Scaling, Application Auto Scaling |
| GCP            | Managed Instance Groups (MIG) Autoscaler          |
| Azure          | Azure Virtual Machine Scale Sets (VMSS)           |
| Huawei Cloud   | Auto Scaling (AS)                                 |

### Spec / Configuration

| Spec / Configuration            | ความหมาย                                       | ตัวอย่าง                                   |
| ------------------------------- | ---------------------------------------------- | ---------------------------------------- |
| Min / Max / Desired Capacity    | จำนวน instance ต่ำสุด สูงสุด และเป้าหมาย              | min=2, max=10, desired=3                 |
| Scaling Policy Type             | วิธีการ trigger scaling                          | Target Tracking, Step Scaling, Scheduled |
| Cooldown Period                 | ช่วงเวลาหยุดพักหลัง scale ครั้งหนึ่ง ก่อน scale ครั้งต่อไป | 300 seconds                              |
| Launch Template / Configuration | template สำหรับสร้าง instance ใหม่                 | AMI, Instance Type, Security Group       |
| Health Check                    | วิธีตรวจสอบว่า instance ยังทำงานปกติ                 | EC2 health check, ELB health check       |
| Warm-up Period                  | เวลาให้ instance ใหม่ warm up ก่อนรับ traffic      | 120 seconds                              |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี — Auto Scaling service ไม่คิดค่าบริการ จ่ายเฉพาะ VM instance ที่ถูก launch

|                      | AWS Auto Scaling  | GCP Managed Instance Group | Azure VMSS  | Huawei AS      |
| -------------------- | ----------------- | -------------------------- | ----------- | -------------- |
| Auto Scaling Service | ฟรี                | ฟรี                         | ฟรี          | ฟรี             |
| EC2/VM ที่ถูก launch    | ตาม instance type | ตาม machine type           | ตาม VM size | ตาม ECS flavor |

### ตัวอย่างการใช้งานใน Project

Application server ที่รับ API request ใช้ Auto Scaling โดยตั้ง Target Tracking Scaling Policy ให้รักษา CPU utilization เฉลี่ยที่ 60% ระบบจะ scale out เมื่อ traffic สูงและ scale in เมื่อ traffic ลด

### Best Practice

- ใช้ Target Tracking Policy เป็นค่าเริ่มต้น เพราะ manage ง่ายที่สุด
- ตั้ง minimum capacity ≥ 2 เสมอ เพื่อ high availability
- test scaling behavior ใน staging environment ก่อน production

### Common Mistakes

- ตั้ง cooldown period สั้นเกินไป ทำให้ scale เข้าออกถี่เกินจำเป็น (thrashing)
- ลืม update Launch Template เมื่อ deploy version ใหม่ทำให้ instance ใหม่ run code เก่า
