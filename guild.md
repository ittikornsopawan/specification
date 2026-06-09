# Cloud Service Bible

เอกสารนี้ใช้เป็น guide สำหรับอ่านและทำความเข้าใจ Cloud Service ในการเริ่มต้น project หนึ่งระบบ อธิบายว่าแต่ละ Service Type คืออะไร ใช้งานแบบไหน เหมาะกับงานแบบไหน และต้องดู Spec / Configuration อะไรบ้าง

---

## 1. Networking

Networking เป็นรากฐานของทุก Cloud Project ก่อนจะ deploy service ใด ๆ ต้องออกแบบ network topology ก่อนเสมอ เพราะโครงสร้าง network ส่งผลต่อความปลอดภัย การเข้าถึง performance และค่าใช้จ่าย การเข้าใจ Networking ช่วยให้ระบบมีความปลอดภัย แยก environment ได้ถูกต้อง และสื่อสารระหว่าง service ได้อย่างมีประสิทธิภาพ

---

### Virtual Private Cloud (VPC)

#### คืออะไร

VPC (Virtual Private Cloud) คือเครือข่ายส่วนตัวเสมือนที่สร้างขึ้นบน Cloud โดยผู้ใช้สามารถกำหนด IP address range, Subnet, Routing และ Network Policy ได้เอง ทำให้ resource ต่าง ๆ ทำงานอยู่ใน network ที่แยกออกจาก cloud อื่น ๆ อย่างสมบูรณ์

#### ใช้งานแบบไหน

ทุก project ควรสร้าง VPC เป็นลำดับแรก จากนั้นแบ่ง Subnet ออกเป็น Public Subnet (เชื่อมต่อ internet ได้) และ Private Subnet (เข้าถึงได้เฉพาะภายใน) แบ่ง environment เช่น production, staging, development ออกจากกัน และใช้ VPC Peering หรือ Transit Gateway เพื่อเชื่อมต่อหลาย VPC เข้าด้วยกันเมื่อจำเป็น

#### เหมาะกับงานแบบไหน

เหมาะกับทุก project ไม่ว่าจะเล็กหรือใหญ่ เป็น Service พื้นฐานที่ทุก workload บน Cloud ต้องมี

#### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ VPC เว้นแต่เป็นการทดสอบเบื้องต้นระยะสั้นใน default network ที่ Cloud Provider จัดให้

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                 |
| -------------- | ---------------------------- |
| AWS            | Amazon VPC                   |
| GCP            | Virtual Private Cloud (VPC)  |
| Azure          | Azure Virtual Network (VNet) |
| Huawei Cloud   | Virtual Private Cloud (VPC)  |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration   | ความหมาย                                         | ตัวอย่าง                                                    |
| ---------------------- | ------------------------------------------------ | --------------------------------------------------------- |
| CIDR Block             | ช่วง IP address ที่ใช้ใน VPC                         | `10.0.0.0/16`                                             |
| Subnet                 | การแบ่งย่อย VPC ออกเป็นส่วนๆ ตาม Availability Zone   | Public Subnet `10.0.1.0/24`, Private Subnet `10.0.2.0/24` |
| Internet Gateway       | Gateway สำหรับเชื่อมต่อ VPC กับ internet               | ติดกับ Public Subnet                                        |
| NAT Gateway            | ให้ Private Subnet ออก internet ได้โดยไม่รับ inbound | deploy ใน Public Subnet                                   |
| Route Table            | ตารางกำหนดทิศทาง traffic                           | route `0.0.0.0/0` ไปหา Internet Gateway                   |
| VPC Peering            | เชื่อมต่อ VPC สองตัวเข้าด้วยกัน                         | เชื่อม production VPC กับ shared services VPC                |
| Availability Zone (AZ) | Data center zone ย่อยที่แยกออกจากกัน                 | deploy ใน 2-3 AZ เพื่อ high availability                    |
| Flow Logs              | บันทึก network traffic ใน VPC เพื่อ audit และ debug  | เปิดบนทุก Subnet                                            |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** VPC ตัวเองไม่มีค่าบริการ — ค่าใช้จ่ายเกิดจาก component ภายใน VPC ตาม On-demand

| Component                       | AWS                   | GCP                | Azure              | Huawei Cloud        | หน่วย                  |
| ------------------------------- | --------------------- | ------------------ | ------------------ | ------------------- | --------------------- |
| VPC / VNet เอง                  | ฟรี                    | ฟรี                 | ฟรี                 | ฟรี                  | —                     |
| Elastic IP (idle, ไม่ได้ใช้งาน)    | $0.005                | ฟรี (ถ้า attach)     | $0.004             | ~$0.004             | /hour                 |
| NAT Gateway                     | $0.045 + $0.045/GB    | $0.010 + $0.045/GB | $0.045 + $0.045/GB | ~$0.030 + $0.040/GB | /hour + /GB processed |
| VPN Connection                  | $0.05/connection/hour | $0.20/tunnel/hour  | $0.19/hour         | ~$0.05/hour         | —                     |
| Flow Logs (storage)             | $0.50/GB              | $0.01/GB           | $0.10/GB           | ~$0.04/GB           | ingested              |
| Data Transfer Egress (internet) | $0.09/GB              | $0.085/GB          | $0.087/GB          | ~$0.072/GB          | —                     |

> ค่าใช้จ่ายหลักที่มักถูกมองข้ามคือ **NAT Gateway** ($0.045/hr + $0.045/GB) และ **Data Transfer Egress** — ระบบที่มี outbound traffic สูงอาจมีค่า NAT Gateway หลายร้อย USD/เดือน

#### ตัวอย่างการใช้งานใน Project

```
Internet
    │
Internet Gateway
    │
Public Subnet  ── Load Balancer, Bastion Host, NAT Gateway
    │
Private Subnet ── Application Server, Database, Cache
```

#### Best Practice

- แบ่ง Public / Private Subnet ทุกครั้ง อย่าวาง Database หรือ Application Server ใน Public Subnet
- สร้าง Subnet กระจายหลาย AZ เสมอเพื่อรองรับ High Availability
- ใช้ CIDR Block ที่ไม่ซ้อนทับกันระหว่าง VPC และ on-premise network เผื่อต้องทำ VPN หรือ Direct Connect ในอนาคต
- เปิด VPC Flow Logs เพื่อ audit และ troubleshoot network traffic

#### Common Mistakes

- ใช้ CIDR Block แคบเกินไป (/24 หรือเล็กกว่า) ทำให้ขยาย IP ไม่ได้ในอนาคต
- วาง resource ทุกอย่างใน Public Subnet เพราะ "ง่ายกว่า"
- ลืมสร้าง Subnet ใน AZ ที่สอง ทำให้ไม่มี failover
- ใช้ default VPC ใน production โดยไม่กำหนด network policy เพิ่มเติม

---

### Security Group

#### คืออะไร

Security Group คือ virtual firewall ระดับ instance ที่ควบคุม inbound และ outbound traffic ให้กับ resource เช่น Virtual Machine, Database หรือ Container Security Group ทำงานแบบ stateful หมายความว่าถ้าอนุญาต inbound traffic response จะผ่านกลับออกไปได้โดยอัตโนมัติ

#### ใช้งานแบบไหน

กำหนด Security Group แยกตาม layer เช่น Security Group สำหรับ Load Balancer, Security Group สำหรับ Application Server และ Security Group สำหรับ Database แล้วอนุญาตเฉพาะ port และ source ที่จำเป็นเท่านั้น

#### เหมาะกับงานแบบไหน

เหมาะกับทุก resource ที่รับ network traffic ใช้ร่วมกับ Network ACL เพื่อควบคุม traffic หลายชั้น

#### ไม่เหมาะกับงานแบบไหน

Security Group ไม่ใช่ Web Application Firewall (WAF) ไม่ควรใช้แทน WAF สำหรับกรองการโจมตีระดับ application

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                  |
| -------------- | ----------------------------- |
| AWS            | Security Groups               |
| GCP            | Firewall Rules (VPC Firewall) |
| Azure          | Network Security Groups (NSG) |
| Huawei Cloud   | Security Groups               |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration  | ความหมาย                                               | ตัวอย่าง                                           |
| --------------------- | ------------------------------------------------------ | ------------------------------------------------ |
| Inbound Rule          | กฎควบคุม traffic ที่เข้ามา                                 | อนุญาต TCP port 443 จาก `0.0.0.0/0`               |
| Outbound Rule         | กฎควบคุม traffic ที่ออกไป                                 | อนุญาต TCP port 5432 ไปหา Database Security Group |
| Protocol              | โปรโตคอลที่กรอง                                          | TCP, UDP, ICMP                                   |
| Port Range            | ช่วง port ที่อนุญาต                                        | `443`, `3000-3999`                               |
| Source / Destination  | แหล่งที่มาหรือปลายทาง                                      | IP CIDR หรือ Security Group ID อื่น                 |
| Stateful vs Stateless | Security Group เป็น stateful, Network ACL เป็น stateless | ต่างกันในการจัดการ response traffic                 |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี — ทุก Cloud Provider ไม่คิดค่าบริการสำหรับ Security Group / Firewall Rules

|                | AWS Security Group       | GCP VPC Firewall    | Azure NSG   | Huawei Security Group |
| -------------- | ------------------------ | ------------------- | ----------- | --------------------- |
| ราคา           | ฟรี                       | ฟรี                  | ฟรี          | ฟรี                    |
| จำนวน rule สูงสุด | 60 inbound + 60 outbound | 1,000 rules/network | 1,000 rules | 100 rules             |

#### ตัวอย่างการใช้งานใน Project

```
sg-loadbalancer: รับ 443 จาก 0.0.0.0/0
sg-app: รับ 8080 จาก sg-loadbalancer เท่านั้น
sg-db: รับ 5432 จาก sg-app เท่านั้น
```

#### Best Practice

- ใช้หลัก Least Privilege เปิด port เฉพาะที่จำเป็น
- อ้างอิง Security Group ID แทน IP CIDR เมื่อเป็น internal traffic
- ตั้งชื่อ Security Group ให้สื่อถึง role เช่น `prod-app-sg`, `prod-db-sg`
- Review Security Group rules สม่ำเสมอ ลบ rule ที่ไม่ใช้แล้ว

#### Common Mistakes

- เปิด port 22 (SSH) หรือ 3389 (RDP) จาก `0.0.0.0/0`
- ใช้ Security Group เดียวกันกับทุก resource
- อนุญาต all traffic (`0.0.0.0/0`) ในทุก port เพราะ "ทดสอบก่อน" แล้วลืมปิด

---

### Network ACL (Access Control List)

#### คืออะไร

Network ACL (Access Control List) คือ firewall ระดับ Subnet ทำงานแบบ stateless โดยกำหนด rule แบบมีลำดับ (numbered rules) สำหรับกรอง traffic ที่เข้าและออกจาก Subnet ทั้งหมด แตกต่างจาก Security Group ที่ทำงานระดับ instance

#### ใช้งานแบบไหน

ใช้เป็น defense layer เพิ่มเติมเหนือ Security Group โดยกำหนด rule ที่ Subnet level เพื่อ block IP range หรือ port ที่ไม่ต้องการก่อนที่ traffic จะถึง instance

#### เหมาะกับงานแบบไหน

เหมาะกับการ block IP ที่ไม่ต้องการในระดับ Subnet หรือใช้ใน compliance requirement ที่ต้องการ stateless firewall

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับการกำหนด rule แบบละเอียดระดับ instance เพราะ Network ACL ครอบทุก resource ใน Subnet

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                          |
| -------------- | ------------------------------------- |
| AWS            | Network ACLs                          |
| GCP            | Hierarchical Firewall Policies        |
| Azure          | Network Security Groups (ระดับ Subnet) |
| Huawei Cloud   | Network ACL                           |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                     | ตัวอย่าง                                  |
| -------------------- | -------------------------------------------- | --------------------------------------- |
| Rule Number          | ลำดับความสำคัญของ rule (ตัวเลขน้อย = ประมวลก่อน)    | Rule 100: Allow 443, Rule 200: Deny all |
| Allow / Deny         | action ที่จะทำเมื่อ traffic match rule            | Allow หรือ Deny                          |
| Inbound / Outbound   | ทิศทาง traffic ที่ rule ใช้งาน                   | Inbound rule สำหรับ traffic เข้า Subnet    |
| Stateless            | ต้องกำหนด rule สำหรับ request และ response แยกกัน | ต้องเปิด ephemeral port ใน outbound ด้วย   |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี — Network ACL ไม่มีค่าบริการแยกต่างหาก

|      | AWS Network ACL | GCP (ใช้ Firewall Policy แทน) | Azure NSG (subnet-level) | Huawei NACL |
| ---- | --------------- | ---------------------------- | ------------------------ | ----------- |
| ราคา | ฟรี              | ฟรี                           | ฟรี                       | ฟรี          |

#### ตัวอย่างการใช้งานใน Project

ใช้ Network ACL บน Private Subnet เพื่อ block traffic จาก internet โดยตรง แม้ว่า route table จะไม่มี Internet Gateway อยู่แล้ว

#### Best Practice

- ใช้ร่วมกับ Security Group อย่าพึ่งอย่างใดอย่างหนึ่งอย่างเดียว
- เปิด ephemeral port range (1024-65535) ใน outbound rule เพราะ Network ACL เป็น stateless
- เริ่ม rule number ด้วยช่วงกว้าง เช่น 100, 200, 300 เพื่อแทรก rule เพิ่มได้ในอนาคต

#### Common Mistakes

- ลืมเพิ่ม outbound rule สำหรับ response traffic (เพราะ stateless)
- กำหนด Deny rule ที่ rule number ต่ำ แล้ว block traffic ที่ต้องการโดยไม่ตั้งใจ

---

### VPN Gateway

#### คืออะไร

VPN Gateway คือบริการที่ช่วยสร้าง encrypted tunnel ระหว่าง Cloud VPC กับ on-premise network หรือ remote user ผ่าน internet โดยใช้โปรโตคอล IPsec หรือ SSL/TLS

#### ใช้งานแบบไหน

ใช้ในกรณีที่ต้องการเชื่อมต่อ Cloud network กับ office network หรือ data center ขององค์กร แบบ Site-to-Site VPN หรือสำหรับ developer เชื่อมต่อแบบ Client-to-Site VPN

#### เหมาะกับงานแบบไหน

เหมาะกับองค์กรที่มี on-premise system และต้องการเข้าถึง Cloud resource อย่างปลอดภัย หรือ hybrid cloud architecture

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ bandwidth สูงและ latency ต่ำมาก ควรใช้ Direct Connect / Dedicated Line แทน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                           |
| -------------- | -------------------------------------- |
| AWS            | AWS VPN (Site-to-Site VPN, Client VPN) |
| GCP            | Cloud VPN                              |
| Azure          | Azure VPN Gateway                      |
| Huawei Cloud   | VPN Gateway                            |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                             | ตัวอย่าง                           |
| -------------------- | ------------------------------------ | -------------------------------- |
| VPN Type             | ประเภทของ VPN connection             | Site-to-Site, Client-to-Site     |
| Tunnel Protocol      | โปรโตคอลที่ใช้เข้ารหัส                    | IKEv2/IPsec, OpenVPN, WireGuard  |
| Pre-Shared Key (PSK) | รหัสสำหรับ authenticate tunnel          | ควรสุ่มและซับซ้อน                    |
| BGP / Static Routing | วิธีการ routing ระหว่าง network         | BGP ดีกว่าสำหรับ dynamic routing     |
| Bandwidth            | ความเร็วสูงสุดของ VPN tunnel            | ขึ้นกับ Gateway tier                |
| High Availability    | การ deploy VPN Gateway แบบ redundant | Active/Active หรือ Active/Standby |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม connection/tunnel-hour และ data transfer

|                                 | AWS Site-to-Site VPN | GCP Cloud VPN (HA) | Azure VPN Gateway (VpnGw1) | Huawei VPN Gateway |
| ------------------------------- | -------------------- | ------------------ | -------------------------- | ------------------ |
| Gateway/hour                    | $0.05/connection     | $0.20/tunnel       | $0.190                     | ~$0.050            |
| Data Transfer Out (internet)    | $0.09/GB             | $0.085/GB          | $0.087/GB                  | ~$0.072/GB         |
| ค่าใช้จ่ายตัวอย่าง (2 tunnels, 24/7) | ~$72/month           | ~$288/month        | ~$137/month                | ~$72/month         |

#### ตัวอย่างการใช้งานใน Project

เชื่อมต่อ on-premise data center กับ Cloud VPC เพื่อให้ application บน Cloud เข้าถึง legacy database ที่ยังไม่ได้ migrate ขึ้น Cloud

#### Best Practice

- ใช้ IKEv2 แทน IKEv1 เพราะปลอดภัยและเสถียรกว่า
- deploy VPN Gateway แบบ Active/Active เพื่อ high availability
- monitor tunnel status สม่ำเสมอ

#### Common Mistakes

- ใช้ VPN แทน Direct Connect กับ workload ที่ต้องการ bandwidth สูง
- ไม่ได้ทำ redundant tunnel ทำให้ connection ขาดเมื่อ gateway หนึ่งมีปัญหา

---

### Direct Connect / Dedicated Line

#### คืออะไร

Direct Connect หรือ Dedicated Line คือบริการที่ช่วยสร้าง private network connection ความเร็วสูงระหว่าง on-premise infrastructure กับ Cloud โดยไม่ผ่าน internet สาธารณะ ให้ bandwidth สม่ำเสมอและ latency ต่ำกว่า VPN

#### ใช้งานแบบไหน

ใช้สำหรับ workload ที่ transfer data ขนาดใหญ่ระหว่าง on-premise และ Cloud หรือ application ที่ต้องการ latency ต่ำสม่ำเสมอ

#### เหมาะกับงานแบบไหน

เหมาะกับ enterprise hybrid cloud, workload ที่ transfer data ขนาดใหญ่, financial system ที่ต้องการ latency ต่ำ, หรือ compliance ที่ห้ามใช้ internet สาธารณะ

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ project ขนาดเล็กที่ไม่มี on-premise infrastructure เพราะมีต้นทุนและเวลาในการ provision สูง

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                             |
| -------------- | ---------------------------------------- |
| AWS            | AWS Direct Connect                       |
| GCP            | Cloud Interconnect (Dedicated / Partner) |
| Azure          | Azure ExpressRoute                       |
| Huawei Cloud   | Direct Connect                           |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration    | ความหมาย                                               | ตัวอย่าง                               |
| ----------------------- | ------------------------------------------------------ | ------------------------------------ |
| Port Speed              | ความเร็วของ physical connection                         | 1 Gbps, 10 Gbps                      |
| Virtual Interface (VIF) | logical connection บน Direct Connect circuit           | Private VIF, Public VIF, Transit VIF |
| VLAN                    | ใช้แยก traffic หลาย connection บน physical circuit เดียว | VLAN 100 สำหรับ production             |
| BGP ASN                 | Autonomous System Number สำหรับ BGP routing              | ต้องไม่ซ้ำกับ Cloud Provider              |
| Redundancy              | การ deploy circuit หลายเส้นเพื่อ failover                 | Active/Active dual circuit           |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม port-hour และ data transfer outbound

| Speed             | AWS Direct Connect | GCP Cloud Interconnect | Azure ExpressRoute | Huawei Direct Connect |
| ----------------- | ------------------ | ---------------------- | ------------------ | --------------------- |
| 50–100 Mbps       | $0.30/hour         | ~$0.10/hour            | ~$55/month         | ~$0.10/hour           |
| 1 Gbps            | $0.30/hour         | N/A                    | ~$220/month        | ~$0.25/hour           |
| 10 Gbps           | $1.60/hour         | $1.735/hour            | ~$5,000/month      | ~$1.20/hour           |
| Data Transfer Out | $0.02/GB           | $0.02/GB               | $0.025/GB          | ~$0.02/GB             |

> Direct Connect มีค่า port-hour สูงมาก — 1 Gbps AWS DX ≈ $216/month เพิ่มค่า Partner/colocation อีก ควรใช้เฉพาะเมื่อ bandwidth, latency หรือ compliance requirement จำเป็นจริง ๆ

#### ตัวอย่างการใช้งานใน Project

ใช้ Direct Connect เชื่อมต่อ data center ขององค์กรกับ Cloud เพื่อรองรับ database replication และ backup ขนาดใหญ่ทุกคืน

#### Best Practice

- deploy อย่างน้อย 2 circuit จาก provider ที่แตกต่างกันเพื่อ redundancy
- ใช้ BGP routing แทน static route เพื่อ failover อัตโนมัติ
- วางแผน bandwidth ให้เผื่อ peak traffic

#### Common Mistakes

- deploy circuit เดียวโดยไม่มี backup ทำให้ outage เมื่อ circuit ขาด
- ไม่ได้ทดสอบ failover จาก Direct Connect ไป VPN

---

## 2. Compute

Compute เป็น Service กลุ่มที่ให้ computational power สำหรับ run application ตั้งแต่ Virtual Machine ไปจนถึง bare metal server การเลือก Compute ที่เหมาะสมส่งผลต่อ performance, cost และการบริหารจัดการ

---

### Virtual Machine (VM) / Compute Instance

#### คืออะไร

Virtual Machine (VM) หรือ Compute Instance คือเครื่อง server เสมือนที่ run บน Cloud Infrastructure ผู้ใช้สามารถกำหนด OS, CPU, Memory, Storage และ Network ได้เอง เป็น Infrastructure as a Service (IaaS) ที่ให้ control ระดับสูงสุด

#### ใช้งานแบบไหน

ใช้ deploy application ที่ต้องการ full OS control เช่น legacy application, application ที่ต้องการ custom kernel หรือ driver หรือ application ที่ยังไม่ได้ containerize นอกจากนี้ยังใช้เป็น worker node ใน Kubernetes cluster

#### เหมาะกับงานแบบไหน

เหมาะกับ legacy application ที่ไม่สามารถ containerize ได้, workload ที่ต้องการ persistent OS state, database server, หรือ workload ที่ต้องการ hardware access พิเศษ

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ scale เร็วมาก หรือ stateless microservice ที่ควรใช้ Container แทน เพราะ VM มี boot time นานกว่าและ overhead ในการจัดการสูงกว่า

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name               |
| -------------- | -------------------------- |
| AWS            | Amazon EC2                 |
| GCP            | Compute Engine             |
| Azure          | Azure Virtual Machines     |
| Huawei Cloud   | Elastic Cloud Server (ECS) |

#### Spec / Configuration ที่ควรรู้

##### AWS Instance Type

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

##### GCP Machine Type

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

##### Azure VM Size

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

##### Huawei Cloud Flavor

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

#### Spec / Configuration อื่น ๆ ที่ควรรู้

| Spec / Configuration        | ความหมาย                                                          | ตัวอย่าง                               |
| --------------------------- | ----------------------------------------------------------------- | ------------------------------------ |
| OS Image / AMI              | Image ของ OS ที่ใช้ boot VM                                          | Amazon Linux 2, Ubuntu 22.04 LTS     |
| Storage Type                | ประเภทของ disk ที่ attach กับ VM                                     | gp3 SSD, io1 SSD, HDD                |
| Key Pair                    | SSH key สำหรับเข้าถึง VM                                              | `prod-bastion-key.pem`               |
| Auto Scaling                | ปรับจำนวน VM อัตโนมัติตาม load                                         | min 2, max 10 instances              |
| Spot / Preemptible Instance | VM ราคาถูกที่ Cloud อาจ terminate ได้                                 | เหมาะกับ batch job ที่ interruptible ได้ |
| Placement Group             | การจัดวาง VM ให้อยู่ใกล้กัน (low latency) หรือกระจาย (high availability) | Cluster, Spread, Partition           |

#### ตัวอย่างการใช้งานใน Project

Deploy Backend API Server บน EC2 / Compute Engine โดยอยู่ใน Private Subnet, รับ traffic ผ่าน Load Balancer, ใช้ Auto Scaling Group เพื่อขยายตาม request volume

#### Best Practice

- ใช้ Auto Scaling Group ร่วมกับ Load Balancer ทุกครั้งเพื่อ high availability
- เลือก Instance Type ให้ตรงกับ workload pattern เช่น compute-heavy ใช้ c-family, memory-heavy ใช้ r-family
- ใช้ Spot Instance สำหรับ batch processing หรือ non-critical workload เพื่อลดต้นทุน
- ไม่ควร SSH เข้า production instance โดยตรง ใช้ Bastion Host หรือ Systems Manager Session Manager แทน

#### Common Mistakes

- เลือก Instance Type ที่ใหญ่เกินจำเป็น
- ไม่ได้ตั้ง Auto Scaling ทำให้ระบบล่มเมื่อ traffic พุ่ง
- เก็บ application code และ data ไว้บน root disk ทำให้ข้อมูลหายเมื่อ instance terminate

---

### Auto Scaling

#### คืออะไร

Auto Scaling คือ Service ที่ปรับจำนวน compute instance โดยอัตโนมัติตาม policy ที่กำหนด เช่น CPU utilization, request count หรือ custom metric ช่วยให้ระบบรองรับ traffic ที่เปลี่ยนแปลงได้โดยไม่ต้องปรับ capacity เอง

#### ใช้งานแบบไหน

กำหนด minimum, maximum และ desired capacity จากนั้นตั้ง scaling policy เช่น เพิ่ม instance เมื่อ CPU > 70% หรือ ลด instance เมื่อ CPU < 30% นาน 10 นาที

#### เหมาะกับงานแบบไหน

เหมาะกับ stateless application ที่รับ web traffic, batch processing worker, หรือ microservice ที่ traffic ไม่สม่ำเสมอ

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ stateful application ที่ instance แต่ละตัวเก็บ state ต่างกัน เช่น database cluster ที่มีการ coordinate กันซับซ้อน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                      |
| -------------- | ------------------------------------------------- |
| AWS            | Amazon EC2 Auto Scaling, Application Auto Scaling |
| GCP            | Managed Instance Groups (MIG) Autoscaler          |
| Azure          | Azure Virtual Machine Scale Sets (VMSS)           |
| Huawei Cloud   | Auto Scaling (AS)                                 |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration            | ความหมาย                                       | ตัวอย่าง                                   |
| ------------------------------- | ---------------------------------------------- | ---------------------------------------- |
| Min / Max / Desired Capacity    | จำนวน instance ต่ำสุด สูงสุด และเป้าหมาย              | min=2, max=10, desired=3                 |
| Scaling Policy Type             | วิธีการ trigger scaling                          | Target Tracking, Step Scaling, Scheduled |
| Cooldown Period                 | ช่วงเวลาหยุดพักหลัง scale ครั้งหนึ่ง ก่อน scale ครั้งต่อไป | 300 seconds                              |
| Launch Template / Configuration | template สำหรับสร้าง instance ใหม่                 | AMI, Instance Type, Security Group       |
| Health Check                    | วิธีตรวจสอบว่า instance ยังทำงานปกติ                 | EC2 health check, ELB health check       |
| Warm-up Period                  | เวลาให้ instance ใหม่ warm up ก่อนรับ traffic      | 120 seconds                              |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี — Auto Scaling service ไม่คิดค่าบริการ จ่ายเฉพาะ VM instance ที่ถูก launch

|                      | AWS Auto Scaling  | GCP Managed Instance Group | Azure VMSS  | Huawei AS      |
| -------------------- | ----------------- | -------------------------- | ----------- | -------------- |
| Auto Scaling Service | ฟรี                | ฟรี                         | ฟรี          | ฟรี             |
| EC2/VM ที่ถูก launch    | ตาม instance type | ตาม machine type           | ตาม VM size | ตาม ECS flavor |

#### ตัวอย่างการใช้งานใน Project

Application server ที่รับ API request ใช้ Auto Scaling โดยตั้ง Target Tracking Scaling Policy ให้รักษา CPU utilization เฉลี่ยที่ 60% ระบบจะ scale out เมื่อ traffic สูงและ scale in เมื่อ traffic ลด

#### Best Practice

- ใช้ Target Tracking Policy เป็นค่าเริ่มต้น เพราะ manage ง่ายที่สุด
- ตั้ง minimum capacity ≥ 2 เสมอ เพื่อ high availability
- test scaling behavior ใน staging environment ก่อน production

#### Common Mistakes

- ตั้ง cooldown period สั้นเกินไป ทำให้ scale เข้าออกถี่เกินจำเป็น (thrashing)
- ลืม update Launch Template เมื่อ deploy version ใหม่ทำให้ instance ใหม่ run code เก่า

---

## 3. Container & Kubernetes

Container และ Kubernetes เป็น Service กลุ่มที่ช่วยให้ deploy, manage และ scale application แบบ containerized ได้อย่างมีประสิทธิภาพ ลด overhead ในการจัดการ infrastructure และรองรับ microservice architecture ได้ดี

---

### Kubernetes (Managed Kubernetes Service)

#### คืออะไร

Managed Kubernetes Service คือบริการ Kubernetes cluster ที่ Cloud Provider จัดการ control plane ให้ ผู้ใช้ต้องดูแลเฉพาะ worker node และ workload ที่ deploy บน cluster ลด operational overhead จากการจัดการ etcd, API server และ scheduler เอง

#### ใช้งานแบบไหน

สร้าง cluster พร้อม Node Pool แล้ว deploy workload ผ่าน Kubernetes manifest (YAML) หรือ Helm Chart จัดการ scaling ผ่าน Horizontal Pod Autoscaler (HPA) และ Cluster Autoscaler

#### เหมาะกับงานแบบไหน

เหมาะกับ microservice architecture, application ที่ต้องการ scale อย่างอิสระ, CI/CD pipeline ที่ deploy บ่อย, หรือ workload ที่ต้องการ high availability

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ project เล็กที่มี service เดียว เพราะ Kubernetes มี overhead ด้าน complexity สูง อาจใช้ Container service แบบ serverless หรือ VM แทน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                            |
| -------------- | --------------------------------------- |
| AWS            | Amazon EKS (Elastic Kubernetes Service) |
| GCP            | Google Kubernetes Engine (GKE)          |
| Azure          | Azure Kubernetes Service (AKS)          |
| Huawei Cloud   | Cloud Container Engine (CCE)            |

#### Spec / Configuration ที่ควรรู้

##### Cluster Configuration

| Spec / Configuration    | ความหมาย                           | ตัวอย่าง                      |
| ----------------------- | ---------------------------------- | --------------------------- |
| Kubernetes Version      | version ของ cluster                | 1.29, 1.30                  |
| Control Plane Mode      | แบบ Managed หรือ Self-managed       | Managed (ค่า default)        |
| Network Plugin (CNI)    | plugin สำหรับ pod networking         | Calico, Cilium, AWS VPC CNI |
| Cluster Endpoint Access | การเข้าถึง API server                | Public, Private, หรือ Both   |
| OIDC Provider           | สำหรับ IAM Role for Service Accounts | เปิดใช้เพื่อ pod-level IAM      |

##### Node Pool Configuration (AWS EKS)

| Instance Type | vCPU | Memory | เหมาะกับงานแบบไหน                 | On-demand ($/hr) | RI 1yr No Upfront | RI 1yr All Upfront | RI 3yr All Upfront | Spot avg |
| ------------- | ---: | -----: | -------------------------------- | ---------------: | ----------------: | -----------------: | -----------------: | -------: |
| `t3.medium`   |    2 |   4 GB | dev/test cluster, small workload |           0.0416 |            0.0280 |             0.0267 |             0.0188 |  ~0.0125 |
| `m6i.large`   |    2 |   8 GB | general purpose worker node      |           0.0960 |            0.0629 |             0.0604 |             0.0430 |  ~0.0288 |
| `m6i.xlarge`  |    4 |  16 GB | medium traffic workload          |           0.1920 |            0.1258 |             0.1208 |             0.0860 |  ~0.0576 |
| `m6i.2xlarge` |    8 |  32 GB | high traffic workload            |           0.3840 |            0.2516 |             0.2416 |             0.1720 |  ~0.1152 |
| `c6i.xlarge`  |    4 |   8 GB | compute-heavy workload           |           0.1700 |            0.1098 |             0.1054 |             0.0748 |  ~0.0510 |
| `r6i.xlarge`  |    4 |  32 GB | memory-heavy workload            |           0.2520 |            0.1650 |             0.1584 |             0.1122 |  ~0.0756 |

##### Node Pool Configuration (GKE)

| Machine Type    | vCPU | Memory | เหมาะกับงานแบบไหน            | On-demand ($/hr) | CUD 1yr | CUD 3yr | Spot avg |
| --------------- | ---: | -----: | --------------------------- | ---------------: | ------: | ------: | -------: |
| `e2-medium`     |    2 |   4 GB | dev/test cluster            |           0.0335 |       — |       — |  ~0.0101 |
| `n2-standard-2` |    2 |   8 GB | general purpose worker node |           0.0971 |  0.0613 |  0.0476 |  ~0.0243 |
| `n2-standard-4` |    4 |  16 GB | medium traffic workload     |           0.1942 |  0.1226 |  0.0952 |  ~0.0486 |
| `n2-standard-8` |    8 |  32 GB | high traffic workload       |           0.3884 |  0.2452 |  0.1904 |  ~0.0972 |
| `c2-standard-4` |    4 |  16 GB | compute-heavy workload      |           0.2088 |  0.1318 |  0.1024 |  ~0.0522 |

##### Node Pool Configuration (AKS)

| VM Size  | vCPU | Memory | เหมาะกับงานแบบไหน            | PAYG ($/hr) | Reserved 1yr | Reserved 3yr | Spot avg |
| -------- | ---: | -----: | --------------------------- | ----------: | -----------: | -----------: | -------: |
| `B2s`    |    2 |   4 GB | dev/test cluster            |      0.0416 |       0.0267 |       0.0188 |  ~0.0083 |
| `D2s v5` |    2 |   8 GB | general purpose worker node |      0.0960 |       0.0620 |       0.0432 |  ~0.0192 |
| `D4s v5` |    4 |  16 GB | medium traffic workload     |      0.1920 |       0.1240 |       0.0864 |  ~0.0384 |
| `D8s v5` |    8 |  32 GB | high traffic workload       |      0.3840 |       0.2480 |       0.1728 |  ~0.0768 |
| `F4s v2` |    4 |   8 GB | compute-heavy workload      |      0.1690 |       0.1090 |       0.0768 |  ~0.0254 |

##### Node Pool Configuration (Huawei Cloud CCE)

| Flavor         | vCPU | Memory | เหมาะกับงานแบบไหน            | On-demand ($/hr) | Reserved 1yr | Spot avg |
| -------------- | ---: | -----: | --------------------------- | ---------------: | -----------: | -------: |
| `c6.large.4`   |    2 |   8 GB | general purpose worker node |           ~0.085 |       ~0.055 |   ~0.026 |
| `c6.xlarge.4`  |    4 |  16 GB | medium traffic workload     |           ~0.170 |       ~0.111 |   ~0.051 |
| `c6.2xlarge.4` |    8 |  32 GB | high traffic workload       |           ~0.340 |       ~0.221 |   ~0.102 |
| `c6.xlarge.2`  |    4 |   8 GB | compute-heavy workload      |           ~0.160 |       ~0.104 |   ~0.048 |

#### ตัวอย่างการใช้งานใน Project

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

#### Best Practice

- แยก Node Pool ตาม workload type (general, compute, memory)
- ใช้ Horizontal Pod Autoscaler (HPA) และ Cluster Autoscaler ร่วมกัน
- กำหนด Resource Requests และ Limits ทุก Pod เสมอ
- ใช้ Namespace แยก environment หรือ team
- update Kubernetes version สม่ำเสมอก่อน version หมด support

#### Common Mistakes

- ไม่ได้กำหนด Resource Requests/Limits ทำให้ Pod หนึ่ง consume resource จน Pod อื่น evict
- deploy workload ใน `default` namespace ทั้งหมด
- ไม่ได้ตั้ง Pod Disruption Budget (PDB) ทำให้ maintenance ทำให้ downtime

---

### Container Instance / Serverless Container

#### คืออะไร

Container Instance หรือ Serverless Container คือบริการ run container โดยไม่ต้องจัดการ cluster หรือ node เองเลย ผู้ใช้ระบุแค่ container image, CPU, Memory และ environment variable แล้ว Cloud จะ run ให้ทันที

#### ใช้งานแบบไหน

ใช้ deploy container ที่ไม่ต้องการ orchestration ซับซ้อน เช่น API service เดี่ยว, background worker, task ที่ run แบบ one-off, หรือ microservice ขนาดเล็กที่ต้องการ deploy เร็ว

#### เหมาะกับงานแบบไหน

เหมาะกับ stateless microservice, background job, event-driven worker, หรือ project ที่ต้องการ deploy container โดยไม่อยาก manage Kubernetes

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ inter-container communication ซับซ้อน หรือ workload ที่ต้องการ custom node configuration

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                    |
| -------------- | ----------------------------------------------- |
| AWS            | AWS Fargate (บน ECS หรือ EKS), AWS App Runner    |
| GCP            | Cloud Run                                       |
| Azure          | Azure Container Apps, Azure Container Instances |
| Huawei Cloud   | Cloud Container Instance (CCI)                  |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                                    | ตัวอย่าง                       |
| -------------------- | ----------------------------------------------------------- | ---------------------------- |
| vCPU                 | จำนวน virtual CPU ที่จัดสรรให้ container                         | 0.5, 1, 2, 4 vCPU            |
| Memory               | ขนาด memory ที่จัดสรรให้ container                              | 1 GB, 2 GB, 4 GB             |
| Container Image      | Docker image ที่ใช้ run                                        | `my-registry/api:v1.2.3`     |
| Min / Max Instances  | จำนวน container ต่ำสุดและสูงสุดสำหรับ scaling                       | min=1, max=10                |
| Concurrency          | จำนวน request ที่ container instance หนึ่งรับพร้อมกันได้ (Cloud Run) | 80 concurrent requests       |
| Timeout              | เวลาสูงสุดที่ request สามารถ process ได้                         | 300 seconds                  |
| Environment Variable | ค่าที่ inject เข้า container ตอน runtime                        | `DATABASE_URL`, `SECRET_KEY` |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม vCPU-second และ GB-second ที่ใช้จริง ไม่มีค่า idle

|                                     | AWS Fargate           | GCP Cloud Run | Azure Container Apps | Huawei CCI |
| ----------------------------------- | --------------------- | ------------- | -------------------- | ---------- |
| vCPU/hour                           | $0.04048              | $0.0864       | $0.0864              | ~$0.035    |
| GB Memory/hour                      | $0.004445             | $0.009        | $0.0108              | ~$0.004    |
| Request (per 1M)                    | —                     | $0.40         | $0.40                | —          |
| Scale-to-zero                       | ไม่รองรับ (ECS Fargate) | รองรับ         | รองรับ                | รองรับ      |
| ตัวอย่าง 1 vCPU + 2 GB RAM, 24/7/30วัน | ~$35/month            | ~$69/month    | ~$69/month           | ~$28/month |

> Cloud Run / Container Apps scale to zero — ช่วง idle ไม่มีค่าใช้จ่าย เหมาะสำหรับ workload ที่ traffic ไม่สม่ำเสมอ

#### ตัวอย่างการใช้งานใน Project

Deploy REST API บน Cloud Run โดยไม่ต้องจัดการ cluster โดย Cloud Run จะ scale จาก 0 ถึง N instance ตาม traffic อัตโนมัติ ลดค่าใช้จ่ายเมื่อไม่มี request

#### Best Practice

- เก็บ secret ใน Secret Manager ไม่ใช่ environment variable โดยตรง
- ตั้ง health check endpoint ให้ container ทุกตัว
- ใช้ min instances ≥ 1 เพื่อหลีกเลี่ยง cold start ถ้า latency สำคัญ

#### Common Mistakes

- เก็บ database credential ใน environment variable แทน Secret Manager
- ไม่ได้กำหนด resource limit ทำให้ container ใช้ resource เกินที่คาด
- ไม่ทดสอบ cold start behavior


---

## 4. Serverless

Serverless คือรูปแบบการ run code โดยไม่ต้องจัดการ server เลย ผู้ใช้เขียนแค่ function หรือ logic แล้ว Cloud จะ provision, scale และ manage infrastructure ให้ทั้งหมด จ่ายเฉพาะเวลาที่ code ทำงานจริง

---

### Function as a Service (FaaS)

#### คืออะไร

FaaS (Function as a Service) คือบริการ run code เป็น function ขนาดเล็ก ถูก trigger โดย event ต่าง ๆ เช่น HTTP request, message queue, file upload หรือ scheduled timer โดยไม่ต้องจัดการ server หรือ OS เลย

#### ใช้งานแบบไหน

เขียน function สำหรับ handle event เฉพาะเจาะจง เช่น process image เมื่อมี upload ขึ้น S3, handle webhook, หรือ run scheduled job ทุกคืน

#### เหมาะกับงานแบบไหน

เหมาะกับ event-driven workload, scheduled task, webhook handler, data transformation pipeline, หรืองานที่มี traffic ไม่สม่ำเสมอและต้องการ cost efficiency สูง

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ execution นาน (มักมี timeout 15-60 นาที), workload ที่ต้องการ persistent connection หรือ in-memory state ระหว่าง invocation

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                         |
| -------------- | ------------------------------------ |
| AWS            | AWS Lambda                           |
| GCP            | Cloud Functions, Cloud Run Functions |
| Azure          | Azure Functions                      |
| Huawei Cloud   | FunctionGraph                        |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                 | ตัวอย่าง                                        |
| -------------------- | ---------------------------------------- | --------------------------------------------- |
| Runtime              | ภาษาและ version ที่ function ใช้            | Node.js 20, Python 3.12, Go 1.21              |
| Memory               | ขนาด memory ที่จัดสรร (ส่งผลต่อ CPU ด้วย)      | 128 MB, 512 MB, 1024 MB                       |
| Timeout              | เวลา execution สูงสุด                      | 30 seconds, 15 minutes                        |
| Concurrency          | จำนวน function instance ที่ run พร้อมกันได้    | Reserved Concurrency, Provisioned Concurrency |
| Trigger Type         | สิ่งที่ trigger function ให้ทำงาน              | HTTP, S3 Event, SQS, EventBridge, Schedule    |
| Environment Variable | ค่า configuration ที่ inject เข้า function   | `DB_HOST`, `API_KEY`                          |
| Layer / Dependency   | code หรือ library ที่ share ระหว่าง function | Lambda Layers                                 |
| VPC Integration      | การให้ function เข้าถึง VPC resource ได้     | เพื่อเชื่อมต่อ database ใน Private Subnet          |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม invocation และ compute time (GB-second)

|                               | AWS Lambda           | GCP Cloud Functions (2nd gen) | Azure Functions (Consumption) | Huawei FunctionGraph |
| ----------------------------- | -------------------- | ----------------------------- | ----------------------------- | -------------------- |
| Invocations                   | $0.20/1M             | $0.40/1M                      | $0.20/1M                      | $0.20/1M             |
| Compute (GB-sec)              | $0.0000166667        | $0.0000100                    | $0.000016                     | $0.00001667          |
| Free Tier/month               | 1M req + 400K GB-sec | 2M req + 400K GB-sec          | 1M req + 400K GB-sec          | 1M req + 400K GB-sec |
| ตัวอย่าง 1M req × 512MB × 200ms | ~$0.21               | ~$0.41                        | ~$0.21                        | ~$0.21               |

> หลัง free tier, FaaS ถูกที่สุดสำหรับ workload < 10M invocation/เดือน เมื่อเทียบกับการ run container 24/7

#### ตัวอย่างการใช้งานใน Project

เมื่อผู้ใช้ upload รูปภาพขึ้น S3 ระบบ trigger Lambda function เพื่อ resize, compress และสร้าง thumbnail แล้วบันทึกกลับ S3 โดยที่ไม่ต้องมี server รอรับ event ตลอดเวลา

#### Best Practice

- ออกแบบ function ให้ idempotent รับ event เดียวกันหลายครั้งแล้วได้ผลลัพธ์เหมือนกัน
- เก็บ secret ใน Secrets Manager ไม่ใช่ environment variable
- ใช้ Provisioned Concurrency สำหรับ function ที่ latency สำคัญ เพื่อหลีกเลี่ยง cold start
- แยก function ให้เล็กและทำสิ่งเดียว (single responsibility)

#### Common Mistakes

- เขียน function ที่ทำงานนาน แล้ว hit timeout
- ไม่ได้จัดการ error และ retry ทำให้ event ถูก process ซ้ำหรือหาย
- เปิด VPC integration โดยไม่จำเป็น ทำให้ cold start นานขึ้น

---

## 5. Load Balancing

Load Balancing คือ Service ที่กระจาย network traffic ไปยัง backend server หลายตัว เพื่อให้ระบบรองรับ load ได้มากขึ้น ไม่มี single point of failure และ scale ได้อย่างมีประสิทธิภาพ

---

### Application Load Balancer (ALB)

#### คืออะไร

Application Load Balancer (ALB) คือ Load Balancer ที่ทำงานในระดับ Layer 7 (HTTP/HTTPS) สามารถ route traffic ตาม URL path, hostname, HTTP header หรือ query string ได้ รองรับ WebSocket, HTTP/2 และ SSL termination

#### ใช้งานแบบไหน

ใช้เป็น entry point ของ web application โดย terminate SSL/TLS แล้ว forward request ไปยัง backend service ตาม routing rule เช่น `/api/*` ไปหา API service, `/static/*` ไปหา CDN origin

#### เหมาะกับงานแบบไหน

เหมาะกับ web application, REST API, microservice ที่ต้องการ path-based routing, หรือ blue/green deployment

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ non-HTTP protocol เช่น TCP/UDP game server หรือ custom binary protocol ควรใช้ Network Load Balancer แทน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                 |
| -------------- | -------------------------------------------- |
| AWS            | Application Load Balancer (ALB)              |
| GCP            | Cloud Load Balancing (HTTP(S) Load Balancer) |
| Azure          | Azure Application Gateway                    |
| Huawei Cloud   | Elastic Load Balance (ELB) - Dedicated       |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                              |
| -------------------- | ----------------------------------------- | ----------------------------------- |
| Listener             | port และ protocol ที่ ALB รับ traffic        | HTTPS:443                           |
| SSL/TLS Certificate  | certificate สำหรับ terminate HTTPS          | ACM Certificate, Let's Encrypt      |
| Target Group         | กลุ่ม backend ที่รับ traffic                   | EC2 instances, IP addresses, Lambda |
| Health Check         | วิธีตรวจสอบ backend ที่ยังทำงานปกติ              | GET /health → 200 OK                |
| Routing Rule         | เงื่อนไขการ route request                   | path `/api/*` → target-group-api    |
| Sticky Session       | ส่ง request จาก client เดิมไปยัง backend เดิม | Duration-based cookie               |
| WAF Integration      | เชื่อมต่อ WAF เพื่อกรอง malicious request      | AWS WAF, Azure WAF                  |
| Access Log           | บันทึก request ทุกรายการ                     | บันทึกไปยัง S3                         |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม hour + usage unit (LCU / CU / GB)

|                          | AWS ALB         | GCP HTTPS LB | Azure App Gateway WAF v2 | Huawei ELB Dedicated |
| ------------------------ | --------------- | ------------ | ------------------------ | -------------------- |
| Gateway/hour             | $0.008          | $0.008       | $0.246                   | ~$0.007              |
| Usage unit               | $0.008/LCU-hour | $0.006/GB    | $0.008/CU-hour           | ~$0.003/GB           |
| Free Tier                | —               | —            | —                        | —                    |
| ตัวอย่าง (ปานกลาง, 1 เดือน) | ~$20–40         | ~$20–40      | ~$180–250                | ~$15–25              |

> Azure App Gateway WAF v2 มีค่า gateway สูงกว่า AWS/GCP ~30× เหมาะกับ enterprise ที่ต้องการ WAF รวมอยู่ด้วย

#### ตัวอย่างการใช้งานใน Project

```
Internet → Route 53 → ALB (HTTPS:443)
    ├── Rule: /api/* → Target Group: api-servers (ECS Fargate)
    ├── Rule: /admin/* → Target Group: admin-servers (EC2)
    └── Rule: /* → Target Group: frontend-servers (EC2)
```

#### Best Practice

- ใช้ ALB เป็น SSL termination point เสมอ ไม่ส่ง HTTPS ผ่านต่อถึง backend
- ตั้ง Health Check ที่เหมาะสม ไม่ใช้ `/` แต่ใช้ endpoint เฉพาะเช่น `/health`
- เปิด Access Log เพื่อ troubleshoot และ audit
- ตั้ง Idle Timeout ให้เหมาะสมกับ application

#### Common Mistakes

- ไม่ได้ตั้ง Health Check ทำให้ traffic ถูกส่งไปยัง backend ที่ dead
- ใช้ self-signed certificate ใน production
- ลืม redirect HTTP → HTTPS

---

### Network Load Balancer (NLB)

#### คืออะไร

Network Load Balancer (NLB) คือ Load Balancer ที่ทำงานในระดับ Layer 4 (TCP/UDP) มี latency ต่ำมาก สามารถรับ traffic ได้หลายล้าน request ต่อวินาที เหมาะกับ protocol ที่ไม่ใช่ HTTP

#### ใช้งานแบบไหน

ใช้สำหรับ distribute TCP/UDP traffic ไปยัง backend โดยไม่ inspect HTTP content ใช้กับ game server, IoT protocol, database proxy หรือ application ที่ต้องการ static IP

#### เหมาะกับงานแบบไหน

เหมาะกับ TCP/UDP workload, game server, real-time communication, application ที่ต้องการ static IP สำหรับ whitelist, หรือ throughput สูงมาก

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ HTTP routing ที่ต้องการ path-based rule เพราะ NLB ไม่ inspect HTTP header

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                 |
| -------------- | -------------------------------------------- |
| AWS            | Network Load Balancer (NLB)                  |
| GCP            | Cloud Load Balancing (TCP/UDP Load Balancer) |
| Azure          | Azure Load Balancer (Standard)               |
| Huawei Cloud   | Elastic Load Balance (ELB) - Network         |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration      | ความหมาย                         | ตัวอย่าง                                  |
| ------------------------- | -------------------------------- | --------------------------------------- |
| Protocol                  | โปรโตคอลที่ Listener รับ            | TCP, UDP, TLS                           |
| Static IP / Elastic IP    | IP address คงที่สำหรับ whitelist     | สำคัญสำหรับ partner integration             |
| Target Type               | ประเภทของ backend                | Instance, IP, Application Load Balancer |
| Cross-Zone Load Balancing | กระจาย traffic ข้าม AZ            | เปิดเพื่อ even distribution                |
| Health Check              | ตรวจสอบ backend ด้วย TCP หรือ HTTP | TCP connect check                       |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม hour + NLCU/GB

|                          | AWS NLB          | GCP Network LB | Azure Standard Load Balancer | Huawei ELB Network |
| ------------------------ | ---------------- | -------------- | ---------------------------- | ------------------ |
| Gateway/hour             | $0.008           | $0.008         | $0.025                       | ~$0.006            |
| Usage unit               | $0.006/NLCU-hour | $0.006/GB      | $0.005/GB processed          | ~$0.003/GB         |
| ตัวอย่าง (ปานกลาง, 1 เดือน) | ~$20–30          | ~$15–25        | ~$18–25                      | ~$10–15            |

#### ตัวอย่างการใช้งานใน Project

Game server ที่ใช้ UDP protocol ใช้ NLB เพื่อ distribute player connection ไปยัง game server instance หลายตัว โดยที่ static IP ช่วยให้ player ไม่ต้อง update IP ใน client

#### Best Practice

- ใช้ NLB เมื่อต้องการ static IP หรือ protocol ที่ไม่ใช่ HTTP
- เปิด Cross-Zone Load Balancing เพื่อกระจาย traffic อย่างสม่ำเสมอ

#### Common Mistakes

- ใช้ NLB กับ HTTP workload ทั้งที่ ALB เหมาะกว่า เพราะ ALB มี feature มากกว่า

---

## 6. API Management

API Management คือ Service กลุ่มที่ช่วย manage, secure, monitor และ publish API ให้กับ consumer ภายในและภายนอกองค์กร ช่วยลด boilerplate ที่ต้องเขียนซ้ำในทุก service เช่น authentication, rate limiting และ logging

---

### API Gateway

#### คืออะไร

API Gateway คือ service ที่ทำหน้าที่เป็น single entry point สำหรับ API ทั้งหมด รับ request จาก client แล้ว route ไปยัง backend service ที่เหมาะสม พร้อมจัดการ authentication, rate limiting, caching, request/response transformation และ logging

#### ใช้งานแบบไหน

ใช้เป็น front door ของ microservice architecture กำหนด API route แต่ละ endpoint ว่า forward ไปยัง backend service ใด ตั้ง authentication policy, rate limit และ quota ต่อ API key หรือ consumer

#### เหมาะกับงานแบบไหน

เหมาะกับ microservice architecture, public API ที่ต้องการ security, mobile backend, หรือ system ที่ต้องการ centralize API policy

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ internal service-to-service communication ที่ไม่ต้องการ overhead ของ API Gateway อาจใช้ service mesh แทน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                 |
| -------------- | -------------------------------------------- |
| AWS            | Amazon API Gateway (REST / HTTP / WebSocket) |
| GCP            | Apigee, Cloud Endpoints                      |
| Azure          | Azure API Management                         |
| Huawei Cloud   | API Gateway (APIG)                           |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration              | ความหมาย                           | ตัวอย่าง                          |
| --------------------------------- | ---------------------------------- | ------------------------------- |
| API Type                          | ประเภทของ API                      | REST, HTTP, WebSocket, GraphQL  |
| Stage / Environment               | environment สำหรับ deploy API        | dev, staging, prod              |
| Authorization / Authentication    | วิธี auth ที่ใช้                        | API Key, JWT, OAuth 2.0, IAM    |
| Rate Limiting / Throttling        | จำกัดจำนวน request ต่อหน่วยเวลา         | 1000 req/min ต่อ API key         |
| Usage Plan / Quota                | กำหนด limit การใช้งานต่อ consumer     | 10,000 req/day                  |
| Request / Response Transformation | แปลง request หรือ response          | เพิ่ม header, แปลง JSON format    |
| Caching                           | cache response เพื่อลด backend load  | TTL 60 seconds                  |
| CORS Policy                       | กำหนด Cross-Origin Resource Sharing | อนุญาต `https://app.example.com` |
| Integration Type                  | วิธีเชื่อมต่อกับ backend                 | Lambda, HTTP endpoint, VPC Link |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม request count + optional cache fee

|                   | AWS API GW (REST)        | GCP API Gateway | Azure API Mgmt (Consumption) | Huawei APIG  |
| ----------------- | ------------------------ | --------------- | ---------------------------- | ------------ |
| REST/HTTP Request | $3.50/1M                 | $3.50/1M        | $3.50/1M                     | ~$3.00/1M    |
| WebSocket msg     | $1.00/1M msgs            | —               | included                     | ~$1.00/1M    |
| Cache (0.5 GB)    | $0.020/hour              | N/A             | included                     | ~$0.015/hour |
| Free Tier         | 1M req/month (12 months) | 2M calls/month  | 1M calls/month               | —            |

> GCP Apigee X สำหรับ enterprise API management มีค่าใช้จ่ายตามแพลน ($600–$6,000+/month) แตกต่างจาก API Gateway ทั่วไปมาก

#### ตัวอย่างการใช้งานใน Project

```
Mobile App → API Gateway → JWT Validation → Rate Limiter
    ├── GET /users → Lambda: user-service
    ├── POST /orders → HTTP: order-service (ECS)
    └── WebSocket /chat → Lambda: chat-handler
```

#### Best Practice

- ใช้ JWT หรือ OAuth 2.0 แทน API Key สำหรับ user authentication
- ตั้ง rate limit ทุก endpoint เพื่อป้องกัน abuse
- เปิด access log และ execution log เพื่อ troubleshoot
- ใช้ stage variables แยก configuration ระหว่าง environment

#### Common Mistakes

- ไม่ตั้ง rate limit ทำให้ backend ถูก flood จาก API call
- expose internal error message ออกสู่ client โดยไม่ mask
- ไม่ได้ version API ทำให้ยาก maintain เมื่อ breaking change

---

## 7. Storage

Storage คือ Service กลุ่มที่ให้บริการเก็บข้อมูลรูปแบบต่าง ๆ ตั้งแต่ file, object, block และ archive แต่ละประเภทเหมาะกับ use case ที่ต่างกัน

---

### Object Storage

#### คืออะไร

Object Storage คือบริการเก็บไฟล์แบบ unstructured data ในรูปแบบ object ซึ่งแต่ละ object ประกอบด้วย data, metadata และ unique key เหมาะกับการเก็บ static file, media, backup, log, dataset ขนาดใหญ่ โดย scale ได้ไม่จำกัดและทนทานสูง (typically 99.999999999% durability)

#### ใช้งานแบบไหน

ใช้เก็บ static file ของ web application เช่น รูปภาพ, video, document, เก็บ log file จาก application, เก็บ backup ของ database, หรือใช้เป็น data lake สำหรับ analytics

#### เหมาะกับงานแบบไหน

เหมาะกับ static website hosting, media storage, backup storage, data lake, log archive, artifact storage สำหรับ CI/CD pipeline

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ read/write แบบ random access เร็ว เช่น database หรือ application ที่ต้องการ POSIX file system

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                 |
| -------------- | ---------------------------- |
| AWS            | Amazon S3                    |
| GCP            | Google Cloud Storage         |
| Azure          | Azure Blob Storage           |
| Huawei Cloud   | Object Storage Service (OBS) |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                            | ตัวอย่าง                                       |
| -------------------- | --------------------------------------------------- | -------------------------------------------- |
| Storage Class        | ระดับ tier ของ storage ที่ส่งผลต่อ access speed และ cost | Standard, Infrequent Access, Archive/Glacier |
| Bucket Policy / ACL  | กำหนดสิทธิ์การเข้าถึง bucket                              | Public read สำหรับ static website              |
| Versioning           | เก็บหลาย version ของ object เดียวกัน                   | เปิดเพื่อป้องกันการลบโดยผิดพลาด                    |
| Lifecycle Policy     | กำหนด rule เปลี่ยน storage class หรือลบ object อัตโนมัติ   | ย้าย log เก่ากว่า 30 วันไป Infrequent Access     |
| Encryption           | การเข้ารหัส object ที่เก็บ                               | SSE-S3, SSE-KMS                              |
| Replication          | copy object ไปยัง bucket อื่นหรือ Region อื่น             | Cross-Region Replication                     |
| Access Logging       | บันทึก request ที่เข้าถึง bucket                          | เพื่อ audit และ security                       |
| Pre-signed URL       | URL ชั่วคราวสำหรับให้ access object โดยไม่ต้องมี credential | Download link หมดอายุใน 1 ชั่วโมง               |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB-month ที่เก็บ + request count + egress

**Storage Cost**

| Storage Class     | AWS S3   | GCP Cloud Storage | Azure Blob | Huawei OBS | หน่วย      |
| ----------------- | -------- | ----------------- | ---------- | ---------- | --------- |
| Standard          | $0.023   | $0.020            | $0.018     | $0.020     | /GB-month |
| Infrequent Access | $0.0125  | $0.010            | $0.010     | $0.010     | /GB-month |
| Archive (Instant) | $0.004   | $0.004            | $0.002     | $0.002     | /GB-month |
| Deep Archive      | $0.00099 | $0.0012           | $0.00099   | $0.001     | /GB-month |

**Request Cost**

|                        | AWS S3   | GCP Cloud Storage | Azure Blob | Huawei OBS | หน่วย         |
| ---------------------- | -------- | ----------------- | ---------- | ---------- | ------------ |
| PUT/COPY/POST/LIST     | $0.005   | $0.005            | $0.055     | $0.004     | /1K requests |
| GET/SELECT/other       | $0.0004  | $0.0004           | $0.0044    | $0.0004    | /1K requests |
| Data Egress (internet) | $0.09/GB | $0.085/GB         | $0.087/GB  | ~$0.072/GB | first 10 TB  |

> ค่า **Data Egress** คือกับดักหลัก — serve object โดยตรงจาก S3 อาจแพงกว่า serve ผ่าน CloudFront CDN เกือบ 10 เท่า ควรใช้ CDN นำหน้า S3 เสมอสำหรับ public content

#### ตัวอย่างการใช้งานใน Project

Application ให้ผู้ใช้ upload รูปภาพ → API server สร้าง Pre-signed URL → Client upload โดยตรงไปยัง S3 (ไม่ผ่าน server) → Lambda process thumbnail → รูปพร้อม serve ผ่าน CloudFront CDN

#### Best Practice

- ปิด public access โดย default เปิดเฉพาะที่จำเป็น
- เปิด Versioning สำหรับ bucket ที่เก็บข้อมูลสำคัญ
- ตั้ง Lifecycle Policy ลบหรือย้าย tier data เก่าที่ไม่ใช้แล้ว
- ใช้ Pre-signed URL แทนการทำ bucket public

#### Common Mistakes

- ทำ bucket เป็น public access โดยไม่ตั้งใจ
- ไม่ได้เปิด Versioning ทำให้ข้อมูลหายเมื่อถูกเขียนทับหรือลบ
- เก็บ secret หรือ credential ใน bucket ที่ไม่ได้ encrypt

---

### Block Storage

#### คืออะไร

Block Storage คือบริการ disk เสมือนที่ attach กับ VM หรือ container ทำงานเหมือน hard disk ทั่วไป รองรับ random read/write access มี latency ต่ำ เหมาะกับ database, OS disk, และ application ที่ต้องการ POSIX file system

#### ใช้งานแบบไหน

ใช้เป็น disk ของ VM สำหรับ OS, application code หรือ database data directory สามารถ attach/detach จาก instance ได้และ take snapshot สำหรับ backup

#### เหมาะกับงานแบบไหน

เหมาะกับ database server, application server ที่ต้องการ fast local disk, หรือ workload ที่ต้องการ POSIX-compliant file system

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับการ share file ระหว่างหลาย instance พร้อมกัน ควรใช้ Shared File Storage แทน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                     |
| -------------- | -------------------------------- |
| AWS            | Amazon EBS (Elastic Block Store) |
| GCP            | Persistent Disk, Hyperdisk       |
| Azure          | Azure Managed Disks              |
| Huawei Cloud   | Elastic Volume Service (EVS)     |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                                                |
| -------------------- | ----------------------------------------- | ----------------------------------------------------- |
| Volume Type          | ประเภทของ disk ส่งผลต่อ IOPS และ throughput | gp3 (General Purpose SSD), io2 (High IOPS), st1 (HDD) |
| Size                 | ขนาดของ volume                            | 100 GB, 500 GB, 2 TB                                  |
| IOPS                 | จำนวน I/O operations ต่อวินาที                | 3000 IOPS (gp3), 64000 IOPS (io2)                     |
| Throughput           | ความเร็วในการ transfer data                | 125 MB/s, 1000 MB/s                                   |
| Snapshot             | การ backup disk ณ จุดเวลาหนึ่ง               | Snapshot ทุกคืนก่อน maintenance                          |
| Encryption           | การเข้ารหัส disk                            | AES-256 ด้วย KMS key                                   |
| Multi-Attach         | attach volume เดียวกันกับหลาย instance       | ใช้กับ cluster database บางประเภท                       |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB ที่ **provision** (ไม่ใช่ที่ใช้จริง)

| Volume Type                       | AWS EBS              | GCP Persistent Disk | Azure Managed Disk | Huawei EVS | หน่วย      |
| --------------------------------- | -------------------- | ------------------- | ------------------ | ---------- | --------- |
| SSD General (gp3 / Balanced)      | $0.080               | $0.170              | $0.084             | ~$0.075    | /GB-month |
| SSD High IOPS (io2 / Extreme)     | $0.125 + $0.065/IOPS | $0.187              | $0.127             | ~$0.120    | /GB-month |
| HDD Throughput (st1 / Throughput) | $0.045               | $0.040              | $0.045             | ~$0.035    | /GB-month |
| Snapshot                          | $0.050               | $0.026              | $0.050             | ~$0.040    | /GB-month |

> Provision ต้องจ่ายเต็มแม้ใช้บางส่วน — ตั้งขนาดให้เหมาะสมและเปิด auto-extend แทนการ over-provision ตั้งแต่ต้น

#### ตัวอย่างการใช้งานใน Project

PostgreSQL database server ใช้ io2 volume ขนาด 500 GB เพื่อให้ได้ IOPS สูงพอสำหรับ production workload พร้อมตั้ง automated snapshot ทุกคืน

#### Best Practice

- เลือก volume type ให้ตรงกับ workload pattern เช่น database ควรใช้ SSD
- เปิด encryption โดย default
- ตั้ง automated snapshot policy

#### Common Mistakes

- ใช้ gp2 (เก่า) แทน gp3 ทั้งที่ gp3 ให้ IOPS มากกว่าในราคาเท่ากัน
- ไม่ได้ตั้ง snapshot ทำให้ไม่มี backup เมื่อ disk เสีย

---

### Shared File Storage (NFS/SMB)

#### คืออะไร

Shared File Storage คือบริการ file system ที่ share ระหว่างหลาย instance พร้อมกันได้ ใช้โปรโตคอล NFS หรือ SMB เหมาะกับ application ที่ต้องการ shared storage เช่น content management system, shared media library

#### ใช้งานแบบไหน

mount file system บน instance หลายตัวพร้อมกัน ทุก instance เห็น file เดียวกัน เหมาะกับ application ที่ยังไม่ได้ออกแบบให้ store file ใน Object Storage

#### เหมาะกับงานแบบไหน

เหมาะกับ legacy application ที่ต้องการ shared disk, CMS เช่น WordPress ที่ต้องการ shared media directory, หรือ Kubernetes persistent volume ที่ต้องการ ReadWriteMany

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ high-throughput workload เพราะ latency สูงกว่า Block Storage หรือ workload ที่ทำได้ด้วย Object Storage

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                     |
| -------------- | -------------------------------- |
| AWS            | Amazon EFS (Elastic File System) |
| GCP            | Cloud Filestore                  |
| Azure          | Azure Files                      |
| Huawei Cloud   | Scalable File Service (SFS)      |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                            | ตัวอย่าง                              |
| -------------------- | ----------------------------------- | ----------------------------------- |
| Performance Mode     | ระดับ performance ของ file system    | General Purpose, Max I/O            |
| Throughput Mode      | วิธีกำหนด throughput                   | Bursting, Provisioned               |
| Storage Class        | tier ของ storage                    | Standard, Infrequent Access         |
| Mount Target         | endpoint ที่ใช้ mount จาก instance     | สร้างใน Subnet ที่ต้องการ mount         |
| Access Point         | entry point สำหรับ application แต่ละตัว | กำหนด root path และ permission แยกกัน |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — EFS จ่ายตาม GB ที่ใช้จริง, Filestore/Azure จ่ายตาม capacity ที่ provision

|                   | AWS EFS           | GCP Filestore        | Azure Files (Premium) | Huawei SFS | หน่วย      |
| ----------------- | ----------------- | -------------------- | --------------------- | ---------- | --------- |
| Standard Tier     | $0.300            | $0.200               | $0.060                | ~$0.060    | /GB-month |
| Infrequent Access | $0.025            | N/A                  | N/A                   | N/A        | /GB-month |
| Minimum Size      | ไม่มี (pay per use) | 1 TB (Basic HDD)     | 100 GB                | ไม่มี        | —         |
| Billing basis     | ใช้จริง (GB)        | Provisioned capacity | Provisioned capacity  | ใช้จริง      | —         |

> EFS แพงกว่า S3 Standard ~13 เท่า และแพงกว่า EBS gp3 ~3.75 เท่า ใช้เฉพาะเมื่อต้องการ shared POSIX filesystem จริง ๆ เช่น legacy app หรือ shared config

#### ตัวอย่างการใช้งานใน Project

WordPress cluster ที่ run บน EC2 หลาย instance ใช้ EFS เป็น shared `/var/www/html/wp-content/uploads` ทำให้ทุก instance เห็น media file เดียวกัน

#### Best Practice

- ใช้ Lifecycle Policy ย้าย file ที่ไม่ได้ access ไป Infrequent Access เพื่อลดค่าใช้จ่าย
- ใช้ Access Point เพื่อแยก root directory ต่อ application

#### Common Mistakes

- ใช้ Shared File Storage กับ workload ที่ควรใช้ Object Storage เพราะ Shared File Storage แพงกว่า
- ไม่ได้ตั้ง Security Group ควบคุมการ mount

---

## 8. Database

Database คือ Service กลุ่มที่ให้บริการ managed database ประเภทต่าง ๆ Cloud Provider จัดการ provisioning, patching, backup และ high availability ให้ ทำให้ทีมโฟกัสที่ data และ schema แทนการจัดการ infrastructure

---

### Relational Database (RDS / Managed SQL)

#### คืออะไร

Relational Database Service คือ managed database สำหรับ SQL engine ต่าง ๆ เช่น MySQL, PostgreSQL, MariaDB, SQL Server, Oracle Cloud Provider จัดการ installation, patching, backup, failover และ monitoring ให้

#### ใช้งานแบบไหน

ใช้เป็น primary database ของ application ที่ต้องการ ACID transaction, structured data, หรือ complex query ด้วย SQL

#### เหมาะกับงานแบบไหน

เหมาะกับ transactional workload, e-commerce, ERP, CRM, application ที่ต้องการ relational data model และ ACID compliance

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ document data ที่ structure เปลี่ยนบ่อย, time-series data ปริมาณมาก, หรือ graph data ที่มี complex relationship

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                            |
| -------------- | ------------------------------------------------------- |
| AWS            | Amazon RDS, Amazon Aurora                               |
| GCP            | Cloud SQL, AlloyDB                                      |
| Azure          | Azure Database for PostgreSQL/MySQL, Azure SQL Database |
| Huawei Cloud   | Relational Database Service (RDS)                       |

#### Spec / Configuration ที่ควรรู้

##### AWS RDS Instance Class

| Instance Class   | vCPU | Memory | เหมาะกับงานแบบไหน                           | On-demand ($/hr) | RI 1yr No Upfront | RI 1yr All Upfront | RI 3yr All Upfront |
| ---------------- | ---: | -----: | ------------------------------------------ | ---------------: | ----------------: | -----------------: | -----------------: |
| `db.t3.medium`   |    2 |   4 GB | dev/test, small application                |            0.068 |             0.044 |              0.042 |              0.030 |
| `db.t3.large`    |    2 |   8 GB | dev/test, medium application               |            0.136 |             0.088 |              0.084 |              0.060 |
| `db.m6g.large`   |    2 |   8 GB | general purpose production                 |            0.150 |             0.098 |              0.094 |              0.067 |
| `db.m6g.xlarge`  |    4 |  16 GB | general purpose production, medium traffic |            0.300 |             0.196 |              0.188 |              0.134 |
| `db.m6g.2xlarge` |    8 |  32 GB | general purpose production, high traffic   |            0.600 |             0.392 |              0.376 |              0.268 |
| `db.r6g.large`   |    2 |  16 GB | memory-heavy workload                      |            0.240 |             0.157 |              0.150 |              0.107 |
| `db.r6g.xlarge`  |    4 |  32 GB | memory-heavy workload, large database      |            0.480 |             0.314 |              0.300 |              0.214 |

##### GCP Cloud SQL Tier

| Machine Type       |   vCPU | Memory | เหมาะกับงานแบบไหน                           | On-demand ($/hr) | CUD 1yr |
| ------------------ | -----: | -----: | ------------------------------------------ | ---------------: | ------: |
| `db-f1-micro`      | shared | 0.6 GB | dev/test เท่านั้น                             |            0.013 |       — |
| `db-g1-small`      | shared | 1.7 GB | small dev/test                             |            0.025 |       — |
| `db-n1-standard-2` |      2 | 7.5 GB | general purpose production                 |            0.096 |   0.061 |
| `db-n1-standard-4` |      4 |  15 GB | general purpose production, medium traffic |            0.192 |   0.122 |
| `db-n1-highmem-4`  |      4 |  26 GB | memory-heavy workload                      |            0.256 |   0.162 |

##### Azure Database Instance (PostgreSQL Flexible)

| SKU Name           | vCPU | Memory | เหมาะกับงานแบบไหน                      | PAYG ($/hr) | Reserved 1yr |
| ------------------ | ---: | -----: | ------------------------------------- | ----------: | -----------: |
| `Standard_B2s`     |    2 |   4 GB | dev/test                              |       0.068 |        0.044 |
| `Standard_D2ds_v4` |    2 |   8 GB | general purpose production            |       0.150 |        0.095 |
| `Standard_D4ds_v4` |    4 |  16 GB | medium traffic                        |       0.300 |        0.190 |
| `Standard_E2ds_v4` |    2 |  16 GB | memory-heavy workload                 |       0.300 |        0.190 |
| `Standard_E4ds_v4` |    4 |  32 GB | memory-heavy workload, large database |       0.600 |        0.380 |

##### Huawei Cloud RDS Flavor

| Flavor                | vCPU | Memory | เหมาะกับงานแบบไหน           | On-demand ($/hr) | Reserved 1yr |
| --------------------- | ---: | -----: | -------------------------- | ---------------: | -----------: |
| `rds.mysql.c2.medium` |    2 |   4 GB | dev/test                   |           ~0.060 |       ~0.039 |
| `rds.mysql.c2.large`  |    2 |   8 GB | general purpose production |           ~0.130 |       ~0.085 |
| `rds.mysql.c2.xlarge` |    4 |  16 GB | medium traffic             |           ~0.260 |       ~0.169 |
| `rds.mysql.m2.xlarge` |    4 |  32 GB | memory-heavy workload      |           ~0.400 |       ~0.260 |

#### Database Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                           | ตัวอย่าง                          |
| -------------------- | -------------------------------------------------- | ------------------------------- |
| Engine & Version     | database engine และ version                        | PostgreSQL 16, MySQL 8.0        |
| Multi-AZ             | deploy database แบบ synchronous replication ข้าม AZ | เปิดสำหรับ production              |
| Read Replica         | copy ที่ read-only สำหรับ scale read workload          | สร้าง 1-2 read replica           |
| Storage Type         | ประเภทของ storage                                  | gp3, io1                        |
| Storage Size         | ขนาด disk                                          | 100 GB เริ่มต้น                    |
| Automated Backup     | backup อัตโนมัติพร้อม retention                        | 7 วัน                            |
| Parameter Group      | ค่า database configuration                          | max_connections, shared_buffers |
| Maintenance Window   | ช่วงเวลาสำหรับ patch อัตโนมัติ                           | อาทิตย์ 02:00-03:00 UTC           |
| Deletion Protection  | ป้องกันการลบ database โดยบังเอิญ                       | เปิดสำหรับ production              |

#### ตัวอย่างการใช้งานใน Project

```
Application Server → RDS PostgreSQL Primary (Multi-AZ)
                   → RDS Read Replica (สำหรับ analytics query)
```

#### Best Practice

- เปิด Multi-AZ ทุก production database เสมอ
- สร้าง Read Replica สำหรับ analytics หรือ reporting query แยกออกจาก primary
- ตั้ง Deletion Protection เพื่อป้องกันการลบโดยบังเอิญ
- monitor connection count เพราะ database มี connection limit

#### Common Mistakes

- ใช้ database เดียวกันสำหรับทั้ง application และ analytics
- ไม่ได้เปิด Multi-AZ ทำให้ downtime นานเมื่อ AZ มีปัญหา
- ไม่ได้ตั้ง Parameter Group ที่เหมาะสม ใช้ค่า default ทั้งหมด

---

### NoSQL Database

#### คืออะไร

NoSQL Database คือ database ที่ไม่ใช้ relational model มีหลายประเภทเช่น Document Store, Key-Value Store, Wide Column, Graph database แต่ละประเภทเหมาะกับ data model และ access pattern ที่ต่างกัน

#### ใช้งานแบบไหน

เลือกประเภท NoSQL ตาม data model เช่น ใช้ Document Store สำหรับ user profile หรือ product catalog, ใช้ Key-Value Store สำหรับ session หรือ feature flag, ใช้ Wide Column สำหรับ time-series data

#### เหมาะกับงานแบบไหน

เหมาะกับ workload ที่ต้องการ horizontal scaling สูง, schema ยืดหยุ่น, document data, time-series, หรือ simple key-value lookup

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ complex JOIN หรือ multi-table transaction แบบ ACID อย่างเข้มงวด

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                                                 |
| -------------- | ---------------------------------------------------------------------------- |
| AWS            | Amazon DynamoDB (Key-Value/Document), Amazon DocumentDB (MongoDB-compatible) |
| GCP            | Cloud Firestore (Document), Cloud Bigtable (Wide Column)                     |
| Azure          | Azure Cosmos DB (Multi-model)                                                |
| Huawei Cloud   | Document Database Service (DDS), GaussDB NoSQL                               |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration         | ความหมาย                                | ตัวอย่าง                                   |
| ---------------------------- | --------------------------------------- | ---------------------------------------- |
| Read / Write Capacity        | ความสามารถในการรับ read/write            | On-demand หรือ Provisioned (RCU/WCU)      |
| Partition Key / Sort Key     | primary key structure ของ table         | `userId` (partition), `timestamp` (sort) |
| Global Secondary Index (GSI) | index เพิ่มเติมสำหรับ query pattern ต่างออกไป | query by email แทน userId                |
| Consistency Level            | ระดับความ consistent ของ read            | Eventual Consistency, Strong Consistency |
| TTL (Time to Live)           | กำหนดวันหมดอายุของ item อัตโนมัติ             | session หมดอายุใน 24 ชั่วโมง                |
| Replication                  | จำนวน region ที่ replicate ข้อมูล            | Global Tables ใน 3 Region                |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม read/write operation และ storage

**AWS DynamoDB**

| Billing Mode | Write           | Read             | Storage        |
| ------------ | --------------- | ---------------- | -------------- |
| On-demand    | $1.25/1M WRU    | $0.25/1M RRU     | $0.25/GB-month |
| Provisioned  | $0.47/WCU-month | $0.047/RCU-month | $0.25/GB-month |

**GCP Firestore**

| Operation | ราคา                  |
| --------- | --------------------- |
| Write     | $0.18/100K operations |
| Read      | $0.06/100K operations |
| Storage   | $0.18/GB-month        |

**Azure Cosmos DB**

| Billing Mode          | ราคา                    | Storage        |
| --------------------- | ----------------------- | -------------- |
| Serverless            | $0.25/1M RU             | $0.25/GB-month |
| Autoscale Provisioned | $0.012/100 RU/sec-month | $0.25/GB-month |

**Huawei DDS (MongoDB-compatible)**

| Instance       | ราคา/hour |
| -------------- | --------- |
| 2 vCPU / 8 GB  | ~$0.100   |
| 4 vCPU / 16 GB | ~$0.200   |

#### ตัวอย่างการใช้งานใน Project

E-commerce ใช้ DynamoDB เก็บ shopping cart session โดยใช้ `userId` เป็น partition key และตั้ง TTL 7 วัน ให้ item ลบตัวเองอัตโนมัติเมื่อหมดอายุ

#### Best Practice

- ออกแบบ data model จาก access pattern ก่อน ไม่ใช่จาก entity model
- เลือก partition key ที่กระจาย load ได้ดี ไม่ควรเป็น key ที่มีค่าซ้ำกันมาก
- ใช้ TTL สำหรับ data ที่มีอายุ เช่น session, cache

#### Common Mistakes

- ออกแบบ schema โดยคิดแบบ relational ทำให้ query ไม่ efficient
- เลือก partition key ที่ไม่กระจาย เช่นใช้ date เป็น partition key ทำให้ hot partition

---

## 9. Cache

Cache คือ Service กลุ่มที่ช่วยเก็บข้อมูลที่เข้าถึงบ่อยใน memory เพื่อลด latency และลด load บน database หลัก ช่วยให้ระบบ scale ได้โดยไม่ต้อง scale database เร็ว ๆ

---

### In-Memory Cache (Redis / Memcached)

#### คืออะไร

In-Memory Cache คือ managed cache service ที่เก็บข้อมูลใน RAM ทำให้เข้าถึงได้ในระดับ sub-millisecond รองรับ data structure หลากหลายเช่น string, hash, list, set, sorted set (Redis) และ simple key-value (Memcached)

#### ใช้งานแบบไหน

ใช้เป็น cache layer หน้า database เก็บผลลัพธ์ของ query ที่ใช้บ่อย, เก็บ user session, เก็บ rate limit counter, หรือใช้เป็น message broker เบื้องต้น (Redis Pub/Sub)

#### เหมาะกับงานแบบไหน

เหมาะกับ session storage, database query cache, rate limiting, leaderboard, real-time analytics, distributed lock

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับการเก็บข้อมูลถาวรที่ต้องการ durability สูงโดยไม่มี persistence strategy เพราะ memory อาจหายได้

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                               |
| -------------- | ---------------------------------------------------------- |
| AWS            | Amazon ElastiCache (Redis OSS, Memcached), Amazon MemoryDB |
| GCP            | Memorystore for Redis, Memorystore for Memcached           |
| Azure          | Azure Cache for Redis                                      |
| Huawei Cloud   | Distributed Cache Service (DCS)                            |

#### Spec / Configuration ที่ควรรู้

##### AWS ElastiCache Node Type

| Node Type           | vCPU |   Memory | เหมาะกับงานแบบไหน            | On-demand ($/hr) | RI 1yr All Upfront | RI 3yr All Upfront |
| ------------------- | ---: | -------: | --------------------------- | ---------------: | -----------------: | -----------------: |
| `cache.t3.micro`    |    2 |   0.5 GB | dev/test เท่านั้น              |            0.017 |              0.011 |              0.008 |
| `cache.t3.small`    |    2 |  1.37 GB | small application           |            0.034 |              0.021 |              0.016 |
| `cache.t3.medium`   |    2 |  3.09 GB | small to medium application |            0.068 |              0.043 |              0.032 |
| `cache.r6g.large`   |    2 | 13.07 GB | production, general purpose |            0.166 |              0.105 |              0.075 |
| `cache.r6g.xlarge`  |    4 | 26.04 GB | production, high traffic    |            0.332 |              0.210 |              0.150 |
| `cache.r6g.2xlarge` |    8 | 52.82 GB | production, heavy workload  |            0.664 |              0.419 |              0.300 |

##### GCP Memorystore Tier

| Tier     | Memory Range  | เหมาะกับงานแบบไหน               | $/GB-hr |
| -------- | ------------- | ------------------------------ | ------: |
| Basic    | 1 GB – 300 GB | dev/test, no replication       |  $0.016 |
| Standard | 1 GB – 300 GB | production, automatic failover |  $0.049 |

##### Azure Cache for Redis SKU

| SKU         | Memory | เหมาะกับงานแบบไหน               | PAYG ($/hr) |
| ----------- | -----: | ------------------------------ | ----------: |
| Basic C0    | 250 MB | dev/test เท่านั้น                 |       0.016 |
| Standard C1 |   1 GB | small application              |       0.101 |
| Standard C2 |   6 GB | medium application             |       0.202 |
| Premium P1  |   6 GB | production, clustering support |       0.544 |
| Premium P2  |  13 GB | production, high traffic       |       1.088 |

##### Huawei Cloud DCS Flavor

| Flavor                 | Memory | เหมาะกับงานแบบไหน        | On-demand ($/hr) |
| ---------------------- | -----: | ----------------------- | ---------------: |
| `redis.single.small.1` |   1 GB | dev/test                |           ~0.020 |
| `redis.ha.medium.4`    |   4 GB | small production        |           ~0.075 |
| `redis.ha.large.8`     |   8 GB | medium production       |           ~0.150 |
| `redis.ha.xlarge.16`   |  16 GB | high traffic production |           ~0.300 |

#### Cache Configuration ที่ควรรู้

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

#### ตัวอย่างการใช้งานใน Project

```
API Server → Redis Cache → (hit) → return cached response
           ↓ (miss)
           Database → save to Redis with TTL 300s → return response
```

#### Best Practice

- กำหนด TTL ทุก key เสมอ ไม่ควรเก็บ key ที่ไม่มี expiry ใน production
- ใช้ Redis Cluster สำหรับ data ขนาดใหญ่หรือ throughput สูง
- monitor memory usage และ eviction rate สม่ำเสมอ
- ใช้ read replica ใน replication group เพื่อ scale read

#### Common Mistakes

- ไม่ได้ตั้ง TTL ทำให้ memory เต็มจาก stale data
- เก็บ object ขนาดใหญ่มากใน Redis ทำให้ latency สูง
- ไม่ได้ตั้ง eviction policy ทำให้ cache error เมื่อ memory เต็ม

---

## 10. Messaging & Queue

Messaging และ Queue คือ Service กลุ่มที่ช่วยให้ component ต่าง ๆ ของระบบสื่อสารกันแบบ asynchronous ลดการ coupling ระหว่าง service และช่วยรองรับ load ที่ spike กะทันหัน

---

### Message Queue

#### คืออะไร

Message Queue คือบริการ queue ที่เก็บ message ไว้รอให้ consumer มาดึงไป (pull model) ใช้สำหรับ decouple producer และ consumer ให้ทำงานในอัตราที่ต่างกันได้ รองรับ retry และ dead-letter queue

#### ใช้งานแบบไหน

Producer ส่ง message เข้า queue, Consumer ดึง message ออกมา process ทีละรายการ ใช้กับ background job, task offloading, หรือ pipeline ที่ต้องการ guaranteed delivery

#### เหมาะกับงานแบบไหน

เหมาะกับ background job processing, email/notification sending, order processing, file conversion, หรืองานที่ต้องการ retry mechanism

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ use case ที่ต้องการ publish message ให้หลาย consumer รับพร้อมกัน (fan-out) ควรใช้ Pub/Sub แทน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                   |
| -------------- | ---------------------------------------------- |
| AWS            | Amazon SQS                                     |
| GCP            | Cloud Tasks                                    |
| Azure          | Azure Queue Storage, Azure Service Bus         |
| Huawei Cloud   | Distributed Message Service (DMS) for RocketMQ |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration      | ความหมาย                                               | ตัวอย่าง                                                 |
| ------------------------- | ------------------------------------------------------ | ------------------------------------------------------ |
| Queue Type                | ประเภทของ queue                                        | Standard (at-least-once), FIFO (exactly-once, ordered) |
| Visibility Timeout        | ช่วงเวลาที่ message ถูก "lock" ระหว่าง consumer กำลัง process | 30 seconds                                             |
| Message Retention Period  | ระยะเวลาที่เก็บ message ไว้ใน queue                        | 4 วัน (default), สูงสุด 14 วัน                             |
| Max Message Size          | ขนาดสูงสุดของ message                                    | 256 KB (SQS), 1 MB (Service Bus)                       |
| Dead-Letter Queue (DLQ)   | queue สำหรับเก็บ message ที่ process ไม่สำเร็จหลาย retry       | แยก queue สำหรับ alert ทีม                                |
| Receive Message Wait Time | การทำ long polling เพื่อลด empty receive                  | 20 seconds                                             |
| Max Receive Count         | จำนวน retry ก่อนส่งไป DLQ                                 | 3-5 ครั้ง                                                |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม message/request count

|           | AWS SQS Standard  | AWS SQS FIFO      | GCP Cloud Tasks | Azure Service Bus Standard | Huawei DMS RocketMQ   |
| --------- | ----------------- | ----------------- | --------------- | -------------------------- | --------------------- |
| ราคาหลัก   | $0.40/1M requests | $0.50/1M requests | $0.40/1M ops    | $10/month + $0.013/1M      | ~$0.006/hour/instance |
| Free Tier | 1M/month          | —                 | 1M/month        | 10M/month                  | —                     |

> SQS request = 1 API call (Send, Receive, Delete); ข้อความขนาดใหญ่กว่า 64 KB นับเป็นหลาย request

#### ตัวอย่างการใช้งานใน Project

```
API Server → SQS Queue → Lambda Worker → send email via SES
                       ↓ (fail 3x)
                       Dead-Letter Queue → Alert ทีม
```

#### Best Practice

- ตั้ง Dead-Letter Queue ทุก queue ใน production
- ออกแบบ consumer ให้ idempotent รับ message เดิมซ้ำแล้วได้ผลเหมือนกัน
- ใช้ FIFO Queue เมื่อต้องการ ordering และ exactly-once processing
- monitor queue depth และ DLQ message count

#### Common Mistakes

- ไม่ได้ตั้ง DLQ ทำให้ message หายเมื่อ process ล้มเหลว
- Visibility Timeout สั้นกว่าเวลา process จริง ทำให้ message ถูก process ซ้ำ
- ไม่ได้ทำ consumer ให้ idempotent

---

### Pub/Sub (Publish/Subscribe)

#### คืออะไร

Pub/Sub คือ messaging pattern ที่ publisher ส่ง message ไปยัง topic และ subscriber หลายคนรับ message จาก topic นั้นพร้อมกัน (fan-out) ใช้สำหรับ event notification, broadcasting และ event-driven architecture

#### ใช้งานแบบไหน

Publisher ส่ง event ไปยัง topic เมื่อเกิด event บางอย่าง เช่น user สร้าง order, subscriber หลายตัว เช่น notification service, inventory service, analytics service รับ event เดียวกันและ process แบบ parallel และอิสระต่อกัน

#### เหมาะกับงานแบบไหน

เหมาะกับ event broadcasting, microservice ที่ต้องการ decoupling, notification system, หรือ event-driven architecture ที่มีหลาย consumer

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ message ordering เข้มงวด หรือ consumer เดียวที่ต้องการ guaranteed processing

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                 |
| -------------- | -------------------------------------------- |
| AWS            | Amazon SNS, Amazon EventBridge               |
| GCP            | Cloud Pub/Sub                                |
| Azure          | Azure Service Bus (Topics), Azure Event Grid |
| Huawei Cloud   | Distributed Message Service (DMS) for Kafka  |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration    | ความหมาย                                | ตัวอย่าง                               |
| ----------------------- | --------------------------------------- | ------------------------------------ |
| Topic                   | channel ที่ publisher ส่ง message เข้า      | `user-events`, `order-created`       |
| Subscription            | การ subscribe ของ consumer ไปยัง topic   | push subscription, pull subscription |
| Message Retention       | ระยะเวลาเก็บ message ใน topic            | 7 วัน (GCP Pub/Sub)                   |
| Acknowledgment Deadline | เวลาที่ subscriber ต้อง ack ก่อน re-deliver | 600 seconds                          |
| Filter                  | กรอง message ตาม attribute              | รับเฉพาะ event ที่ `type=ORDER_CREATED` |
| Dead-Letter Topic       | topic สำหรับ message ที่ deliver ไม่สำเร็จ     | เพื่อ monitor และ debug                |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม message/event count หรือ data volume

|                 | AWS SNS                | AWS EventBridge | GCP Cloud Pub/Sub  | Azure Event Grid    | Huawei DMS |
| --------------- | ---------------------- | --------------- | ------------------ | ------------------- | ---------- |
| ราคาหลัก         | $0.50/1M notifications | $1.00/1M events | $0.04/GB processed | $0.60/1M operations | ~$0.020/GB |
| HTTP/S delivery | $0.60/1M deliveries    | —               | รวมใน data GB      | —                   | —          |
| Free Tier       | 1M/month               | —               | 10 GB/month        | 100K ops/month      | —          |

#### ตัวอย่างการใช้งานใน Project

```
Order Service → SNS Topic: order-created
    ├── SQS: notification-queue → send push notification
    ├── SQS: inventory-queue → update stock
    └── SQS: analytics-queue → update dashboard
```

#### Best Practice

- ออกแบบ event schema ให้ชัดเจน มี versioning
- ใช้ filter subscription เพื่อให้ consumer รับเฉพาะ event ที่ต้องการ
- ตั้ง Dead-Letter Topic เพื่อจับ message ที่ deliver ไม่สำเร็จ

#### Common Mistakes

- ส่ง message ขนาดใหญ่ใน Pub/Sub แทนที่จะส่งแค่ reference (S3 key)
- ไม่ได้ออกแบบ consumer ให้ idempotent

---

## 11. Event Streaming

Event Streaming คือ Service กลุ่มที่ออกแบบมาสำหรับ process data stream แบบ real-time ต่างจาก Message Queue ตรงที่เก็บ event ไว้เป็นเวลานาน consumer หลายตัวอ่าน event stream เดิมได้อิสระต่อกัน และรองรับ throughput สูงมาก

---

### Managed Kafka / Event Streaming

#### คืออะไร

Managed Kafka หรือ Event Streaming Service คือบริการ Apache Kafka หรือ Kafka-compatible ที่ Cloud Provider จัดการ broker, replication และ scaling ให้ ใช้สำหรับ ingest และ process event ปริมาณมากแบบ real-time

#### ใช้งานแบบไหน

Producer ส่ง event เข้า topic (แบ่งเป็น partition) Consumer Group ดึง event จาก partition ตาม offset ของตัวเอง ทำให้หลาย consumer group อ่าน stream เดิมได้อิสระ

#### เหมาะกับงานแบบไหน

เหมาะกับ real-time data pipeline, log aggregation, change data capture (CDC), event sourcing, activity tracking, หรือ workload ที่มี throughput หลายล้าน event ต่อวินาที

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ simple task queue ที่ต้องการ per-message retry หรือ workload ที่ event volume ต่ำ เพราะ Kafka มี operational complexity สูง

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                                    |
| -------------- | --------------------------------------------------------------- |
| AWS            | Amazon MSK (Managed Streaming for Apache Kafka), Amazon Kinesis |
| GCP            | Cloud Pub/Sub (Kafka-compatible), Managed Kafka                 |
| Azure          | Azure Event Hubs (Kafka-compatible)                             |
| Huawei Cloud   | Distributed Message Service (DMS) for Kafka                     |

#### Spec / Configuration ที่ควรรู้

##### AWS MSK Broker Instance Type

| Instance Type      | เหมาะกับงานแบบไหน           | On-demand ($/broker-hr) | Reserved 1yr |
| ------------------ | -------------------------- | ----------------------: | ------------ |
| `kafka.t3.small`   | dev/test เท่านั้น             |                   0.054 | ~0.034       |
| `kafka.m5.large`   | small to medium production |                   0.210 | ~0.133       |
| `kafka.m5.xlarge`  | medium to high throughput  |                   0.420 | ~0.265       |
| `kafka.m5.2xlarge` | high throughput workload   |                   0.840 | ~0.530       |

#### Kafka Configuration ที่ควรรู้

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

#### ตัวอย่างการใช้งานใน Project

```
Web App → Kafka Topic: page-views (24 partitions)
    ├── Consumer Group: real-time-dashboard → update live counter
    ├── Consumer Group: ml-pipeline → feed recommendation model
    └── Consumer Group: data-warehouse → batch load ไป BigQuery
```

#### Best Practice

- กำหนด Replication Factor ≥ 3 สำหรับ production topic
- เลือกจำนวน partition ให้เหมาะสมตั้งแต่ต้น การเพิ่ม partition ภายหลังส่งผลต่อ ordering
- ใช้ Schema Registry เพื่อจัดการ schema evolution
- monitor consumer lag สม่ำเสมอ

#### Common Mistakes

- ตั้ง partition น้อยเกินไป ทำให้ scale consumer ได้จำกัด
- Replication Factor = 1 ทำให้ data สูญเมื่อ broker crash
- ไม่ monitor consumer lag ทำให้ไม่รู้ว่า consumer ตาม producer ไม่ทัน

---

## 12. Security

Security คือ Service กลุ่มที่ช่วยปกป้องระบบจากการโจมตีและการเข้าถึงที่ไม่ได้รับอนุญาต ครอบคลุมตั้งแต่ firewall ระดับ application ไปจนถึงการตรวจจับ intrusion และ DDoS protection

---

### Web Application Firewall (WAF)

#### คืออะไร

WAF (Web Application Firewall) คือ firewall ระดับ Layer 7 ที่ตรวจสอบ HTTP/HTTPS traffic เพื่อกรองการโจมตีประเภทต่าง ๆ เช่น SQL Injection, Cross-Site Scripting (XSS), Remote Code Execution และ OWASP Top 10

#### ใช้งานแบบไหน

ติดตั้งหน้า Load Balancer หรือ API Gateway กำหนด rule set เพื่อ block หรือ monitor request ที่ pattern เข้าข่ายการโจมตี

#### เหมาะกับงานแบบไหน

เหมาะกับทุก web application ที่ expose สู่ internet โดยเฉพาะ application ที่รับ user input หรืออยู่ภายใต้ compliance เช่น PCI DSS, HIPAA

#### ไม่เหมาะกับงานแบบไหน

WAF ไม่ใช่ substitute สำหรับ secure coding practice ควรใช้ร่วมกับ application security ไม่ใช่แทนกัน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                         |
| -------------- | ------------------------------------ |
| AWS            | AWS WAF                              |
| GCP            | Cloud Armor                          |
| Azure          | Azure Web Application Firewall (WAF) |
| Huawei Cloud   | Web Application Firewall (WAF)       |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                                        |
| -------------------- | ----------------------------------------- | --------------------------------------------- |
| Rule Group           | ชุด rule สำเร็จรูปสำหรับการโจมตีประเภทต่าง ๆ      | AWS Managed Rules: CommonRuleSet, SQLiRuleSet |
| Custom Rule          | rule ที่กำหนดเองตาม business logic           | block IP range ของคู่แข่ง                        |
| Action               | สิ่งที่ทำเมื่อ request match rule                | Allow, Block, Count, CAPTCHA                  |
| Rate-Based Rule      | จำกัด request จาก IP เดียวตามเวลา            | block IP ที่ส่ง > 1000 req ใน 5 นาที              |
| Geo-Restriction      | block หรือ allow traffic จาก country ที่กำหนด | อนุญาตเฉพาะ Thailand, Singapore                |
| Bot Control          | จัดการ bot traffic                         | block bad bot, allow Google bot               |
| IP Set               | กลุ่ม IP ที่ whitelist หรือ blacklist          | whitelist office IP                           |
| Logging              | บันทึก request ที่ match rule                 | ส่ง log ไป S3 หรือ CloudWatch                   |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** Subscription (per ACL/policy) + On-demand (per request)

|                                  | AWS WAF          | GCP Cloud Armor    | Azure WAF on App GW v2    | Huawei WAF         |
| -------------------------------- | ---------------- | ------------------ | ------------------------- | ------------------ |
| Web ACL / Policy                 | $5.00/ACL/month  | $5.00/policy/month | รวมใน App GW (~$0.246/hr) | ~$30/month (basic) |
| Rule                             | $1.00/rule/month | —                  | รวมใน Gateway             | รวมใน plan         |
| Request                          | $0.60/1M         | $0.75/1M evaluated | รวมใน CU charge           | ~$0.30/1M          |
| Managed Rule Group               | $20/group/month  | —                  | —                         | —                  |
| ตัวอย่าง (10 rules + 5M req/month) | ~$43/month       | ~$8/month          | ~$180/month               | ~$30/month         |

#### ตัวอย่างการใช้งานใน Project

```
Internet → CloudFront → WAF → ALB → Application
WAF Rules:
- AWS Managed: CommonRuleSet (block OWASP Top 10)
- Rate-Based: block IP > 2000 req/5min
- Geo Block: block traffic จาก high-risk countries
```

#### Best Practice

- เริ่มต้นด้วย Count mode ก่อน Block mode เพื่อดูว่า rule block request ที่ถูกต้องหรือไม่
- ใช้ managed rule group ของ Cloud Provider เป็นพื้นฐาน
- เปิด logging เพื่อ analyze attack pattern

#### Common Mistakes

- เปิด Block mode ทันทีโดยไม่ได้ test ทำให้ block legitimate user
- ไม่ได้ update managed rule group หลังจาก subscribe

---

### DDoS Protection

#### คืออะไร

DDoS Protection คือบริการที่ตรวจจับและบรรเทาการโจมตีแบบ Distributed Denial of Service (DDoS) ที่พยายาม flood ระบบด้วย traffic ปริมาณมากจนทำให้ service ไม่สามารถทำงานได้ตามปกติ

#### ใช้งานแบบไหน

เปิดใช้งานระดับ network และ application layer ป้องกันทั้ง volumetric attack (bandwidth flooding) และ application layer attack (HTTP flood)

#### เหมาะกับงานแบบไหน

เหมาะกับทุก public-facing service โดยเฉพาะ e-commerce, gaming, financial service, หรือ service ที่เป็น target ของ DDoS

#### ไม่เหมาะกับงานแบบไหน

DDoS Protection ระดับสูงมี cost สูง อาจไม่คุ้มค่าสำหรับ internal service ที่ไม่ expose สู่ internet

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                        |
| -------------- | --------------------------------------------------- |
| AWS            | AWS Shield Standard (ฟรี), AWS Shield Advanced       |
| GCP            | Cloud Armor (รวม DDoS protection)                   |
| Azure          | Azure DDoS Protection (Basic ฟรี, Standard มีค่าใช้จ่าย) |
| Huawei Cloud   | Anti-DDoS                                           |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                          | ตัวอย่าง                                   |
| -------------------- | ------------------------------------------------- | ---------------------------------------- |
| Protection Tier      | ระดับการป้องกัน                                      | Standard (ฟรี, auto), Advanced (มีค่าใช้จ่าย) |
| Mitigation Capacity  | bandwidth ที่รับมือได้ก่อน mitigate                     | Tbps level                               |
| Attack Visibility    | dashboard แสดง attack metric                      | CloudWatch metrics, attack report        |
| Response Team        | ทีม support ระหว่างการโจมตี                          | AWS Shield Advanced มี DRT                |
| Rate Limiting        | จำกัด request rate เพื่อรับมือ application layer attack | ร่วมกับ WAF rate-based rule                |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี (basic auto-protection) / Subscription (advanced)

| Tier                  | AWS Shield               | GCP Cloud Armor DDoS     | Azure DDoS Protection    | Huawei Anti-DDoS |
| --------------------- | ------------------------ | ------------------------ | ------------------------ | ---------------- |
| Basic                 | ฟรี (auto L3/L4)          | ฟรี (auto L3/L4)          | ฟรี (auto basic)          | ฟรี (basic)       |
| Advanced / Standard   | $3,000/month (org-level) | $5/policy + request fees | $2,944/month/VIP         | ~$1,500/month    |
| DRT / Expert Response | Shield Advanced only     | N/A                      | Azure DDoS Standard only | Advanced only    |
| SLA / refund          | มี                        | มี                        | มี                        | มี                |

> Basic tier ปกป้อง L3/L4 ได้โดยอัตโนมัติโดยไม่ต้องเปิด Advanced — Advanced เหมาะสำหรับ enterprise ที่ต้องการ 24/7 expert response และ cost protection guarantee

#### ตัวอย่างการใช้งานใน Project

เปิด AWS Shield Standard บน CloudFront และ ALB โดยอัตโนมัติ ถ้า service มีความเสี่ยง DDoS สูงอาจ upgrade เป็น Shield Advanced เพื่อ SLA protection และ cost protection

#### Best Practice

- ใช้ CDN เป็น DDoS absorption layer แรก
- ร่วมกับ WAF rate limiting เพื่อรับมือ application layer attack
- มี runbook สำหรับขั้นตอนรับมือเมื่อเกิด DDoS

#### Common Mistakes

- ไม่ได้เปิด protection จนกว่าจะถูก attack แล้ว
- พึ่ง DDoS Protection อย่างเดียวโดยไม่มี WAF และ rate limiting

---

## 13. Identity & Access Management (IAM)

IAM คือ Service กลุ่มที่จัดการ identity ของ user, service และ application และกำหนดว่าแต่ละ identity มีสิทธิ์ทำอะไรได้บ้างบน Cloud Resource เป็น first line of defense สำหรับ cloud security

---

### IAM (User, Role, Policy)

#### คืออะไร

IAM (Identity and Access Management) คือบริการที่จัดการ identity และ permission บน Cloud ประกอบด้วย User (บุคคลหรือ service), Group (กลุ่ม user), Role (permission ชั่วคราวที่ assume ได้) และ Policy (เอกสารกำหนดสิทธิ์)

#### ใช้งานแบบไหน

สร้าง Role สำหรับ application แต่ละตัว กำหนด Policy ที่อนุญาตเฉพาะ action และ resource ที่จำเป็น แทนการใช้ long-term access key

#### เหมาะกับงานแบบไหน

ใช้กับทุก Cloud resource การออกแบบ IAM ที่ดีเป็นพื้นฐานของ security

#### ไม่เหมาะกับงานแบบไหน

IAM ไม่ใช่ application-level authentication ไม่ควรใช้ IAM จัดการ end user ของ application ควรใช้ Cognito / Identity Platform แทน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                  |
| -------------- | --------------------------------------------- |
| AWS            | AWS IAM                                       |
| GCP            | Cloud IAM                                     |
| Azure          | Azure Active Directory (Entra ID), Azure RBAC |
| Huawei Cloud   | Identity and Access Management (IAM)          |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                              | ตัวอย่าง                                                     |
| -------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| IAM User             | identity สำหรับ human ที่ต้องการ long-term access          | developer, ops team                                        |
| IAM Role             | identity ชั่วคราวที่ service หรือ user assume ได้           | EC2 instance role, Lambda execution role                   |
| IAM Policy           | JSON document กำหนด Allow/Deny สำหรับ action บน resource | `s3:GetObject` บน bucket เฉพาะ                             |
| Policy Type          | ประเภทของ policy                                      | AWS Managed Policy, Customer Managed Policy, Inline Policy |
| Permissions Boundary | จำกัดสิทธิ์สูงสุดที่ entity สามารถมีได้                          | ป้องกัน privilege escalation                                 |
| Service Account      | identity สำหรับ application/service                     | GCP Service Account, AWS IAM Role for service              |
| Condition            | เงื่อนไขเพิ่มเติมใน policy                                 | อนุญาตเฉพาะจาก IP office หรือ MFA                            |
| Principal            | ผู้ที่ policy บังคับใช้กับ                                    | AWS account, IAM user, IAM role                            |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี — IAM ไม่มีค่าบริการ

| Feature                  | AWS IAM    | GCP Cloud IAM | Azure RBAC | Huawei IAM |
| ------------------------ | ---------- | ------------- | ---------- | ---------- |
| Users / Service Accounts | ฟรี         | ฟรี            | ฟรี         | ฟรี         |
| Roles / Policies         | ฟรี         | ฟรี            | ฟรี         | ฟรี         |
| MFA                      | ฟรี         | ฟรี            | ฟรี         | ฟรี         |
| Access Analyzer / Audit  | ฟรี (basic) | ฟรี            | ฟรี         | ฟรี         |

#### ตัวอย่างการใช้งานใน Project

```
Lambda Function → IAM Role: lambda-api-role
    Policy: lambda-api-policy
    - Allow: s3:GetObject บน bucket: my-bucket
    - Allow: dynamodb:Query บน table: users
    - Allow: logs:CreateLogStream, logs:PutLogEvents
```

#### Best Practice

- ใช้หลัก Least Privilege ให้สิทธิ์เฉพาะที่จำเป็นเท่านั้น
- ใช้ IAM Role แทน Access Key เสมอสำหรับ service บน Cloud
- ไม่ใช้ root account สำหรับ operation ปกติ
- เปิด MFA สำหรับ IAM user ที่มีสิทธิ์สูง
- review permission สม่ำเสมอและลบ unused access

#### Common Mistakes

- ให้ AdministratorAccess กับ application โดยไม่จำเป็น
- เก็บ Access Key ใน source code หรือ environment variable แบบ plain text
- ไม่ rotate Access Key ที่เก่า

---

### Single Sign-On (SSO) / Identity Provider (IdP)

#### คืออะไร

SSO (Single Sign-On) คือบริการที่ให้ user login ครั้งเดียวแล้วเข้าถึงได้หลาย application หรือ Cloud Console โดยใช้ Identity Provider กลาง รองรับ SAML 2.0, OIDC และ OAuth 2.0

#### ใช้งานแบบไหน

ผูก Cloud account กับ Corporate Identity Provider เช่น Active Directory, Okta หรือ Azure AD ทำให้ developer login ด้วย corporate credential เดียวกับที่ใช้ login email ทำงาน

#### เหมาะกับงานแบบไหน

เหมาะกับ enterprise ที่มีทีมขนาดใหญ่ ต้องการ centralize identity management และ audit trail

#### ไม่เหมาะกับงานแบบไหน

อาจ overkill สำหรับ team เล็กมากที่ใช้ IAM user ปกติก็เพียงพอ

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                     |
| -------------- | -------------------------------- |
| AWS            | AWS IAM Identity Center (SSO)    |
| GCP            | Cloud Identity, Google Workspace |
| Azure          | Microsoft Entra ID (Azure AD)    |
| Huawei Cloud   | IAM Identity Provider            |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration    | ความหมาย                                      | ตัวอย่าง                                       |
| ----------------------- | --------------------------------------------- | -------------------------------------------- |
| Identity Provider (IdP) | ระบบที่ manage identity                         | Okta, Azure AD, Active Directory             |
| Protocol                | โปรโตคอลที่ใช้ federate identity                 | SAML 2.0, OIDC                               |
| Permission Set          | ชุด permission ที่ assign ให้ user ใน account หนึ่ง | ReadOnly, PowerUser, Admin                   |
| Attribute Mapping       | การ map attribute จาก IdP ไปยัง Cloud role     | group `cloud-admin` → Permission Set `Admin` |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี (basic SSO) / Subscription per user (advanced features)

| Feature                   | AWS IAM Identity Center | GCP Cloud Identity          | Azure Entra ID     | Huawei IAM IdP |
| ------------------------- | ----------------------- | --------------------------- | ------------------ | -------------- |
| Basic SSO + MFA           | ฟรี                      | ฟรี (Free edition)           | ฟรี (Free tier)     | ฟรี             |
| Conditional Access        | ฟรี                      | $6/user/month (Premium)     | $6/user/month (P1) | —              |
| Identity Protection + PIM | ฟรี                      | $12/user/month (Enterprise) | $9/user/month (P2) | —              |
| ตัวอย่าง 100 users, P1      | ฟรี                      | $600/month                  | $600/month         | —              |

#### ตัวอย่างการใช้งานใน Project

Developer login ผ่าน AWS IAM Identity Center ด้วย corporate Okta credential โดยไม่ต้องมี IAM user ใน AWS account แต่ละตัว

#### Best Practice

- ใช้ SSO แทน IAM User สำหรับ human access ทุกกรณีที่เป็นไปได้
- กำหนด Permission Set ตาม role ไม่ใช่ตามบุคคล
- เปิด MFA ที่ Identity Provider

#### Common Mistakes

- สร้าง IAM User แยกในทุก account แทนที่จะใช้ SSO
- ให้ permission มากกว่าที่จำเป็นเพราะ "สะดวกกว่า"

---

## 14. Key Management & Secrets Management

Key Management และ Secrets Management คือ Service กลุ่มที่ช่วยจัดการ cryptographic key และ secret (เช่น database password, API key, certificate) อย่างปลอดภัย แยกออกจาก application code

---

### Key Management Service (KMS)

#### คืออะไร

KMS (Key Management Service) คือบริการจัดการ cryptographic key สำหรับ encrypt และ decrypt data บน Cloud Cloud Provider เก็บ key อย่างปลอดภัยใน hardware security module (HSM) และจัดการ key rotation ให้

#### ใช้งานแบบไหน

ใช้ encrypt data ที่เก็บใน storage เช่น S3, EBS, RDS หรือ encrypt/decrypt data ใน application โดยตรง ผ่าน API call แทนการ manage key เอง

#### เหมาะกับงานแบบไหน

เหมาะกับทุก workload ที่ต้องการ encrypt data at rest โดยเฉพาะ data ที่มีความอ่อนไหว เช่น PII, financial data, health data

#### ไม่เหมาะกับงานแบบไหน

KMS ไม่ใช่ Secrets Manager ไม่ควรใช้ KMS เก็บ password หรือ connection string โดยตรง

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                   |
| -------------- | ------------------------------ |
| AWS            | AWS KMS                        |
| GCP            | Cloud KMS                      |
| Azure          | Azure Key Vault (Keys)         |
| Huawei Cloud   | Data Encryption Workshop (DEW) |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                    | ตัวอย่าง                                       |
| -------------------- | ------------------------------------------- | -------------------------------------------- |
| Key Type             | ประเภทของ key                               | Symmetric (AES-256), Asymmetric (RSA, ECC)   |
| Key Origin           | ว่า key material มาจากไหน                    | AWS KMS generated, External (BYOK), CloudHSM |
| Key Usage            | วัตถุประสงค์การใช้ key                          | ENCRYPT_DECRYPT, SIGN_VERIFY                 |
| Key Rotation         | การ rotate key อัตโนมัติ                       | ทุก 1 ปี                                       |
| Key Policy           | กำหนดว่า identity ใดใช้ key ได้                 | อนุญาตเฉพาะ Lambda role                       |
| Envelope Encryption  | pattern การ encrypt data key ด้วย master key | ใช้ใน S3, EBS, RDS                            |
| Multi-Region Key     | key ที่ replicate ข้าม Region                  | สำหรับ multi-region application                |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม key/month และ API call count

|                           | AWS KMS                | GCP Cloud KMS           | Azure Key Vault        | Huawei DEW       |
| ------------------------- | ---------------------- | ----------------------- | ---------------------- | ---------------- |
| Customer Managed Key      | $1.00/key/month        | $0.06/key version/month | $0.03/10K transactions | ~$1.00/key/month |
| AWS/Cloud Managed Key     | ฟรี                     | N/A                     | N/A                    | N/A              |
| Symmetric Encrypt/Decrypt | $0.03/10K requests     | $0.03/10K               | $0.03/10K              | ~$0.03/10K       |
| Asymmetric Sign/Verify    | $0.15/10K requests     | $0.06/10K               | $0.15/10K              | ~$0.10/10K       |
| Automatic Key Rotation    | ฟรี (เพิ่ม 1 key version) | $0.06/version           | ฟรี                     | ฟรี               |

#### ตัวอย่างการใช้งานใน Project

RDS database ใช้ KMS key ในการ encrypt storage, S3 bucket ใช้ SSE-KMS โดยกำหนด key policy ว่าเฉพาะ application role และ ops team เท่านั้นที่ decrypt ได้

#### Best Practice

- ใช้ Customer Managed Key (CMK) แทน AWS Managed Key เพื่อ control key rotation และ access
- กำหนด key policy แบบ least privilege
- monitor key usage ผ่าน CloudTrail
- เปิด automatic key rotation

#### Common Mistakes

- ใช้ key เดียวกันสำหรับทุก service (single key for everything)
- ไม่ได้กำหนด key policy ที่ชัดเจน ทำให้ทุก IAM admin เข้าถึงได้
- ลืม enable key rotation

---

### Secrets Manager

#### คืออะไร

Secrets Manager คือบริการเก็บ secret เช่น database password, API key, OAuth token, SSH key อย่างปลอดภัย โดยมีความสามารถ rotate secret อัตโนมัติและ audit การเข้าถึง

#### ใช้งานแบบไหน

Application ดึง secret จาก Secrets Manager ผ่าน API call ตอน runtime แทนการเก็บใน environment variable หรือ config file บน server

#### เหมาะกับงานแบบไหน

เหมาะกับทุก application ที่ต้องการ database credential, third-party API key, หรือ token ที่ต้องเปลี่ยนตามรอบ

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ configuration ทั่วไปที่ไม่ sensitive ควรใช้ Parameter Store หรือ environment variable แทน

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                       |
| -------------- | -------------------------------------------------- |
| AWS            | AWS Secrets Manager                                |
| GCP            | Secret Manager                                     |
| Azure          | Azure Key Vault (Secrets)                          |
| Huawei Cloud   | Data Encryption Workshop (DEW) - Secret Management |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                               | ตัวอย่าง                                        |
| -------------------- | -------------------------------------- | --------------------------------------------- |
| Secret Type          | ประเภทของ secret                       | Database credentials, API key, SSH key, Other |
| Secret Value         | ค่าของ secret (เข้ารหัสด้วย KMS)           | `{"username":"admin","password":"xxx"}`       |
| Automatic Rotation   | การ rotate secret อัตโนมัติ               | ทุก 30 วัน                                      |
| Rotation Lambda      | Lambda function ที่ทำ rotation            | สร้าง password ใหม่ใน DB แล้วอัปเดต secret        |
| Versioning           | เก็บหลาย version ของ secret             | AWSCURRENT, AWSPREVIOUS                       |
| Resource Policy      | กำหนด access จาก account หรือ service อื่น | cross-account access                          |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม secret/month และ API call

|                          | AWS Secrets Manager       | GCP Secret Manager                | Azure Key Vault (Secrets) | Huawei DEW Secrets  |
| ------------------------ | ------------------------- | --------------------------------- | ------------------------- | ------------------- |
| Secret Storage           | $0.40/secret/month        | $0.06/secret version/month        | $0.03/10K transactions    | ~$0.10/secret/month |
| API Call                 | $0.05/10K calls           | $0.03/10K                         | รวมใน transactions        | ~$0.03/10K          |
| Cross-Region Replication | $0.40/secret/region/month | N/A                               | N/A                       | N/A                 |
| Free Tier                | —                         | 6 active versions + 10K ops/month | —                         | —                   |

> 100 secrets × AWS Secrets Manager ≈ $40/month (ไม่รวม API calls) — พิจารณาใช้ SSM Parameter Store (Standard = ฟรี) สำหรับ config ที่ไม่ sensitive

#### ตัวอย่างการใช้งานใน Project

```python
# Application code - ดึง secret จาก Secrets Manager ตอน startup
import boto3
client = boto3.client('secretsmanager')
secret = client.get_secret_value(SecretId='prod/myapp/db-credentials')
```

#### Best Practice

- เปิด automatic rotation สำหรับ database credential
- ให้ application permission เฉพาะ secret ที่จำเป็น
- monitor access log ผ่าน CloudTrail

#### Common Mistakes

- เก็บ secret ใน environment variable หรือ source code แทน Secrets Manager
- ไม่ได้เปิด rotation ทำให้ secret ไม่เคยเปลี่ยน
- ให้ application access ทุก secret แทนที่จะ restrict เฉพาะที่จำเป็น

---

## 15. Monitoring & Observability

Monitoring และ Observability คือ Service กลุ่มที่ช่วยให้เห็น health และ performance ของระบบแบบ real-time ช่วย detect ปัญหาก่อนที่จะส่งผลกระทบต่อ user และช่วย debug เมื่อเกิดปัญหา

---

### Cloud Monitoring (Metrics & Alerting)

#### คืออะไร

Cloud Monitoring คือบริการเก็บ metric จาก Cloud resource ต่าง ๆ เช่น CPU, memory, disk, network และ custom metric จาก application แล้วแสดงผลใน dashboard และ alert เมื่อ metric เกิน threshold

#### ใช้งานแบบไหน

ตั้ง alarm บน metric ที่สำคัญเช่น CPU > 80%, error rate > 1%, p99 latency > 500ms เชื่อมต่อกับ notification เช่น email, Slack, PagerDuty เมื่อ alarm trigger

#### เหมาะกับงานแบบไหน

เหมาะกับทุก production system ที่ต้องการ observability

#### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ monitoring ใน production

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                     |
| -------------- | ------------------------------------------------ |
| AWS            | Amazon CloudWatch                                |
| GCP            | Cloud Monitoring (Google Cloud Operations Suite) |
| Azure          | Azure Monitor                                    |
| Huawei Cloud   | Cloud Eye                                        |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                | ตัวอย่าง                                  |
| -------------------- | --------------------------------------- | --------------------------------------- |
| Metric               | ข้อมูลตัวเลขที่วัดตามเวลา                     | CPUUtilization, RequestCount, ErrorRate |
| Namespace            | กลุ่มของ metric                           | `AWS/EC2`, `MyApp/API`                  |
| Dimension            | attribute ที่ใช้ filter metric             | InstanceId, Environment                 |
| Period               | ช่วงเวลาของแต่ละ data point               | 60 seconds                              |
| Statistics           | วิธีคำนวณ aggregate                        | Average, Sum, Maximum, p99              |
| Alarm                | rule ที่ trigger เมื่อ metric เกิน threshold | CPU > 80% นาน 5 นาที                     |
| Alarm Action         | สิ่งที่เกิดขึ้นเมื่อ alarm trigger               | SNS notification, Auto Scaling          |
| Dashboard            | หน้าแสดงผล metric หลายตัวในที่เดียว          | Production Overview Dashboard           |
| Retention            | ระยะเวลาเก็บ metric data                 | 15 months (CloudWatch)                  |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี (platform metrics) + On-demand (custom metrics, dashboards, alerts)

| Feature                         | AWS CloudWatch                       | GCP Cloud Monitoring                | Azure Monitor      | Huawei Cloud Eye    |
| ------------------------------- | ------------------------------------ | ----------------------------------- | ------------------ | ------------------- |
| Platform Metrics (CPU/Disk ฯลฯ) | ฟรี                                   | ฟรี                                  | ฟรี                 | ฟรี                  |
| Custom Metric                   | $0.30/metric/month (first 10K)       | $0.18/metric/month (after 150 free) | $0.25/metric/month | ~$0.20/metric/month |
| Dashboard                       | $3.00/dashboard/month (after 3 free) | ฟรี                                  | ฟรี                 | ฟรี                  |
| Alert Rule                      | $0.10/alarm/month                    | ฟรี (first 5)                        | $0.10/rule/month   | ~$0.05/alarm/month  |
| GetMetricData API               | $0.01/1K metrics requested           | ฟรี                                  | ฟรี                 | ฟรี                  |

#### ตัวอย่างการใช้งานใน Project

กำหนด alarm บน API Gateway: `5xx error rate > 1%` ส่ง SNS notification ไปยัง Slack channel ทีมพร้อม PagerDuty สำหรับ on-call engineer

#### Best Practice

- monitor ทั้ง infrastructure metric และ application metric (custom metric)
- ตั้ง alarm ในระดับ warning ก่อน critical เพื่อให้มีเวลาตอบสนอง
- สร้าง dashboard สำหรับ daily review
- ทดสอบ alarm ว่า fire ไปถึง on-call จริง

#### Common Mistakes

- ตั้ง alarm เยอะเกินไปทำให้ alert fatigue ทีมไม่สนใจ alarm
- ไม่ได้ monitor application metric เช่น error rate, latency เฉพาะ infra metric
- ลืม test ว่า notification ส่งถึงคนที่ใช่จริง

---

## 16. Logging

Logging คือ Service กลุ่มที่รวบรวม เก็บ และ query log จาก application และ infrastructure เพื่อ troubleshoot ปัญหา audit การเข้าถึง และ detect security incident

---

### Centralized Log Management

#### คืออะไร

Centralized Log Management คือบริการที่รวบรวม log จาก source หลาย ๆ ตัวเช่น application server, Load Balancer, database, Cloud service เข้ามาเก็บไว้ที่เดียว พร้อมความสามารถ search, filter และ analyze log

#### ใช้งานแบบไหน

ติดตั้ง log agent บน server หรือ configure service ให้ส่ง log ไปยัง centralized log service จากนั้น query ด้วย query language เช่น CloudWatch Logs Insights หรือ Kibana

#### เหมาะกับงานแบบไหน

เหมาะกับทุก production system ที่มีมากกว่า 1 service เพราะ distributed log ทำให้ troubleshoot ยากมาก

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับการเก็บ log ขนาดเล็กมากใน managed service ที่แพงเกินจำเป็น

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                       |
| -------------- | ---------------------------------- |
| AWS            | Amazon CloudWatch Logs             |
| GCP            | Cloud Logging                      |
| Azure          | Azure Monitor Logs (Log Analytics) |
| Huawei Cloud   | Log Tank Service (LTS)             |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration       | ความหมาย                                     | ตัวอย่าง                                              |
| -------------------------- | -------------------------------------------- | --------------------------------------------------- |
| Log Group / Bucket         | กลุ่มของ log stream จาก source เดียวกัน          | `/aws/lambda/my-function`, `/app/api`               |
| Log Stream                 | ลำดับของ log event จาก instance/container เดียว | `i-0123456789/application.log`                      |
| Retention Policy           | ระยะเวลาเก็บ log ก่อนลบอัตโนมัติ                  | 30 วัน, 90 วัน, 1 ปี                                   |
| Log Filter / Metric Filter | แปลง log pattern ให้เป็น metric                | นับ error log สร้าง metric ErrorCount                 |
| Subscription Filter        | ส่ง log ไปยัง destination อื่น                   | ส่งไป Kinesis Data Firehose หรือ S3                   |
| Log Agent                  | agent ที่ collect log จาก server               | CloudWatch Agent, Fluentd, Fluent Bit               |
| Structured Logging         | log แบบ JSON format เพื่อ query ง่าย            | `{"level":"error","message":"...","traceId":"..."}` |
| Log Query Language         | ภาษา query สำหรับ analyze log                  | CloudWatch Logs Insights, KQL                       |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB ที่ ingest + store + query

|                   | AWS CloudWatch Logs | GCP Cloud Logging           | Azure Monitor Logs              | Huawei LTS      |
| ----------------- | ------------------- | --------------------------- | ------------------------------- | --------------- |
| Log Ingestion     | $0.50/GB            | $0.01/GB (after 50 GB free) | $2.30/GB                        | ~$0.04/GB       |
| Log Storage       | $0.03/GB-month      | $0.01/GB-month              | รวมใน ingestion (31 วัน default) | ~$0.02/GB-month |
| Log Query         | $0.005/GB scanned   | $0.01/GB scanned            | รวมใน workspace                 | ~$0.003/GB      |
| Default Retention | กำหนดเอง             | 30 วัน (_Default bucket)     | 31 วัน                           | 7 วัน            |

> AWS CloudWatch Logs ingestion ($0.50/GB) แพงที่สุด — ระบบที่มี log volume สูง ควร export ไป S3 ผ่าน Kinesis Firehose (S3 storage $0.023/GB-month) แทน

#### ตัวอย่างการใช้งานใน Project

```
Application (JSON log) → Fluent Bit → CloudWatch Logs
CloudWatch Logs → Metric Filter → CloudWatch Alarm (error spike)
CloudWatch Logs → S3 (archive) → Athena (long-term analysis)
```

#### Best Practice

- ใช้ structured logging (JSON) เสมอเพื่อ query ได้ง่าย
- กำหนด retention policy ทุก log group ไม่ปล่อยให้เก็บนิรันดร์
- รวม trace ID ทุก log เพื่อ correlate ระหว่าง service
- ส่ง log ไป long-term storage (S3) สำหรับ compliance และ analysis

#### Common Mistakes

- log แบบ plain text ทำให้ query ยาก
- ไม่กำหนด retention ทำให้ค่าใช้จ่ายสะสม
- เก็บ sensitive data เช่น password, PII ใน log

---

## 17. Tracing

Tracing คือ Service กลุ่มที่ช่วย track request ตลอดเส้นทางที่ผ่าน microservice หลาย ๆ ตัว ทำให้เห็นว่า bottleneck อยู่ที่ service ใด และ latency เกิดขึ้นที่ไหน

---

### Distributed Tracing

#### คืออะไร

Distributed Tracing คือบริการที่ track request journey ข้าม service หลายตัวโดยแสดงผลเป็น trace ที่ประกอบด้วย span แต่ละ span แสดง latency ของ operation หนึ่ง ๆ เช่น database query, API call, หรือ service call

#### ใช้งานแบบไหน

Instrument application ด้วย tracing SDK (OpenTelemetry) แล้วส่ง trace ไปยัง tracing backend จากนั้น query trace เพื่อดู waterfall view ของ request และระบุ bottleneck

#### เหมาะกับงานแบบไหน

เหมาะกับ microservice architecture ที่มีหลาย service request ข้ามหลาย service ทำให้ยาก debug ด้วย log อย่างเดียว

#### ไม่เหมาะกับงานแบบไหน

อาจมี overhead เกินจำเป็นสำหรับ monolithic application ที่ง่ายมาก

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                             |
| -------------- | ---------------------------------------- |
| AWS            | AWS X-Ray                                |
| GCP            | Cloud Trace                              |
| Azure          | Azure Monitor (Application Insights)     |
| Huawei Cloud   | Application Performance Management (APM) |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                     | ตัวอย่าง                                    |
| -------------------- | -------------------------------------------- | ----------------------------------------- |
| Trace                | การ track request หนึ่งตลอดเส้นทาง              | request ID: abc-123 จาก API จนถึง database |
| Span                 | unit ย่อยของ trace แสดง latency ของ operation | span: DynamoDB Query ใช้เวลา 12ms          |
| Trace ID             | ID ที่ผูก span ทุกตัวใน trace เดียวกัน              | propagate ผ่าน HTTP header `X-Trace-Id`    |
| Sampling Rate        | สัดส่วนของ request ที่ trace                     | 1% (high traffic), 100% (debug)           |
| Instrumentation      | การ inject tracing code เข้า application      | OpenTelemetry SDK, AWS X-Ray SDK          |
| Service Map          | diagram แสดง dependency ระหว่าง service       | auto-generated จาก trace data             |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม trace/span ที่ record และ retrieve

|                          | AWS X-Ray         | GCP Cloud Trace  | Azure Application Insights | Huawei APM                |
| ------------------------ | ----------------- | ---------------- | -------------------------- | ------------------------- |
| Trace Recording          | $5.00/1M traces   | $0.20/1M spans   | $2.30/GB ingested          | ~$2.00/month (basic plan) |
| Trace Retrieval/Scan     | $0.50/1M traces   | ฟรี               | รวมใน ingestion            | รวมใน plan                |
| Free Tier                | 100K traces/month | 2.5M spans/month | 5 GB/month                 | —                         |
| ตัวอย่าง (1M traces/month) | ~$5.50            | ~$0.20           | ~$2.30/GB                  | ~$2.00+                   |

#### ตัวอย่างการใช้งานใน Project

API Gateway → Lambda → DynamoDB trace แสดงว่า latency 450ms มาจาก DynamoDB query 400ms ทำให้รู้ว่าต้องเพิ่ม index

#### Best Practice

- ใช้ OpenTelemetry เป็น standard เพื่อ vendor-neutral instrumentation
- propagate trace context ทุก service call ผ่าน HTTP header
- ตั้ง sampling rate ให้เหมาะสมกับ traffic volume
- ผูก trace ID กับ log และ metric เพื่อ full observability

#### Common Mistakes

- instrument เฉพาะ service บางตัว ทำให้ trace ขาดกลางทาง
- sampling rate สูงเกินใน high-traffic production ทำให้ค่าใช้จ่ายสูง

---

## 18. DevOps & CI/CD

DevOps และ CI/CD คือ Service กลุ่มที่ช่วย automate กระบวนการ build, test และ deploy code ทำให้ release cycle เร็วขึ้น ลด human error และรองรับ deployment หลาย environment

---

### CI/CD Pipeline

#### คืออะไร

CI/CD Pipeline คือบริการที่ automate กระบวนการตั้งแต่ code push ไปจนถึง deploy production โดย Continuous Integration (CI) ทำ build และ test อัตโนมัติ และ Continuous Delivery/Deployment (CD) ทำ deploy อัตโนมัติ

#### ใช้งานแบบไหน

กำหนด pipeline ที่ trigger เมื่อมี code push ไปยัง branch ที่กำหนด pipeline run: checkout code → build → unit test → integration test → build container image → push to registry → deploy to staging → smoke test → deploy to production

#### เหมาะกับงานแบบไหน

เหมาะกับทุก project ที่ต้องการ deploy บ่อยและต้องการลด manual error ในกระบวนการ release

#### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ CI/CD แต่ซับซ้อนเกินจำเป็นสำหรับ project ที่ deploy น้อยมาก

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                    |
| -------------- | ----------------------------------------------- |
| AWS            | AWS CodePipeline, AWS CodeBuild, AWS CodeDeploy |
| GCP            | Cloud Build, Cloud Deploy                       |
| Azure          | Azure DevOps (Pipelines), GitHub Actions        |
| Huawei Cloud   | CodeArts Pipeline                               |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                 | ตัวอย่าง                                     |
| -------------------- | ---------------------------------------- | ------------------------------------------ |
| Pipeline Stage       | ขั้นตอนใน pipeline                         | Source, Build, Test, Deploy                |
| Trigger              | สิ่งที่ start pipeline                       | push to main branch, PR merge, schedule    |
| Build Environment    | environment ที่ run build                  | Docker image, managed runtime              |
| Artifact             | output ของ build stage                   | Docker image, zip file, binary             |
| Approval Gate        | การขอ approval ก่อน proceed ไป stage ต่อไป | manual approval ก่อน deploy production      |
| Deployment Strategy  | วิธีการ deploy                             | Rolling, Blue/Green, Canary                |
| Environment          | ปลายทางที่ deploy                          | dev, staging, production                   |
| Rollback             | การ revert กลับ version ก่อนหน้า            | automatic rollback เมื่อ health check ล้มเหลว |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand (build minutes) + Subscription (active pipeline/parallel jobs)

|                     | AWS CodeBuild           | AWS CodePipeline                     | GCP Cloud Build | Azure Pipelines     | Huawei CodeArts    |
| ------------------- | ----------------------- | ------------------------------------ | --------------- | ------------------- | ------------------ |
| Build/minute        | $0.005 (general1.small) | —                                    | $0.003          | $0.008 (after free) | รวมใน plan         |
| Pipeline/month      | —                       | $1.00/active pipeline (after 1 free) | —               | ฟรี (1 parallel)     | รวมใน plan         |
| Free Tier           | 100 min/month           | 1 pipeline ฟรี                        | 120 min/day     | 1,800 min/month     | —                  |
| Extra parallel jobs | —                       | —                                    | —               | $40/extra parallel  | —                  |
| Subscription        | —                       | —                                    | —               | —                   | ~$15/month (basic) |

> ค่าใช้จ่าย CI/CD หลักมาจาก **build minutes** — optimize build time และ cache dependency ลดได้มาก GitHub Actions/GitLab CI อาจถูกกว่าสำหรับ small-medium team

#### ตัวอย่างการใช้งานใน Project

```
GitHub push → CodePipeline trigger
  → Stage 1: CodeBuild - build + test
  → Stage 2: CodeBuild - build Docker image + push ECR
  → Stage 3: Deploy to EKS staging
  → Stage 4: Manual approval
  → Stage 5: Deploy to EKS production (Blue/Green)
```

#### Best Practice

- ใช้ Infrastructure as Code (IaC) สำหรับ manage pipeline และ environment
- ทดสอบทุก stage ก่อน deploy production
- ใช้ Blue/Green deployment เพื่อ zero-downtime deploy
- เก็บ secret ใน Secrets Manager ไม่ใช่ใน pipeline config

#### Common Mistakes

- ไม่มี rollback strategy ทำให้ revert ยากเมื่อมีปัญหา
- deploy production โดยตรงโดยไม่ผ่าน staging
- เก็บ credential ใน pipeline environment variable แบบ plain text

---

## 19. Container Registry

Container Registry คือ Service สำหรับ store, manage และ distribute container image ก่อน deploy image ขึ้น Kubernetes หรือ container service ต้องมีที่เก็บ image ที่ accessible

---

### Private Container Registry

#### คืออะไร

Private Container Registry คือบริการ repository สำหรับ Docker/OCI container image ที่ private เข้าถึงได้เฉพาะ authorized identity รองรับ image versioning, vulnerability scanning และ access control

#### ใช้งานแบบไหน

CI/CD pipeline build Docker image แล้ว push ไปยัง registry เมื่อ deploy ระบบ pull image จาก registry เพื่อ run container

#### เหมาะกับงานแบบไหน

เหมาะกับทุก project ที่ใช้ container ไม่ควรใช้ public registry เช่น Docker Hub สำหรับ production image

#### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ container registry สำหรับ production container workload

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                            |
| -------------- | --------------------------------------- |
| AWS            | Amazon ECR (Elastic Container Registry) |
| GCP            | Artifact Registry, Container Registry   |
| Azure          | Azure Container Registry (ACR)          |
| Huawei Cloud   | Software Repository for Container (SWR) |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration   | ความหมาย                                   | ตัวอย่าง                                    |
| ---------------------- | ------------------------------------------ | ----------------------------------------- |
| Repository             | ที่เก็บ image ของ application หนึ่ง             | `my-company/api-service`                  |
| Image Tag              | version ของ image                          | `v1.2.3`, `latest`, `git-sha-abc123`      |
| Vulnerability Scanning | ตรวจหา security vulnerability ใน image     | scan ทุก push, block image ที่มี critical CVE |
| Image Lifecycle Policy | ลบ image เก่าอัตโนมัติ                         | เก็บไว้แค่ 10 image ล่าสุด                     |
| Repository Policy      | access control ว่า identity ใด pull/push ได้ | เฉพาะ CI/CD role push, EKS node pull      |
| Replication            | copy image ไปยัง Registry ใน Region อื่น      | multi-region deployment                   |
| Image Signing          | เซ็น image เพื่อ verify ว่าไม่ถูกแก้ไข            | Sigstore, Notary                          |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม storage GB และ data transfer

|                              | AWS ECR        | GCP Artifact Registry  | Azure Container Registry (Basic) | Huawei SWR      |
| ---------------------------- | -------------- | ---------------------- | -------------------------------- | --------------- |
| Storage                      | $0.10/GB-month | $0.10/GB-month         | ~$3/GB-month                     | ~$0.05/GB-month |
| Registry Fee                 | ฟรี             | ฟรี                     | $0.167/day (~$5/month)           | ฟรี              |
| Pull (same region)           | ฟรี             | ฟรี                     | ฟรี                               | ฟรี              |
| Pull (cross-region/internet) | $0.09/GB       | $0.08/GB               | $0.087/GB                        | ~$0.07/GB       |
| Free Tier                    | 500 MB/month   | ฟรี (Artifact Registry) | —                                | —               |

#### ตัวอย่างการใช้งานใน Project

```
GitHub Actions → build image → push to ECR
EKS Node → pull image from ECR (ผ่าน IAM Role) → run container
```

#### Best Practice

- tag image ด้วย git commit SHA ไม่ใช่แค่ `latest`
- เปิด vulnerability scanning ทุก push และ block deploy เมื่อมี critical CVE
- ตั้ง lifecycle policy ลบ image เก่าที่ไม่ใช้
- ใช้ IAM Role สำหรับ pull/push ไม่ใช้ long-term credential

#### Common Mistakes

- ใช้ tag `latest` ใน production ทำให้ไม่รู้ว่า version ไหน deploy อยู่
- ไม่ได้ตั้ง lifecycle policy ทำให้ storage เต็มจาก image เก่า

---

## 20. Backup & Disaster Recovery

Backup และ Disaster Recovery คือ Service กลุ่มที่ช่วยปกป้องข้อมูลและให้ระบบสามารถ recover ได้เมื่อเกิด failure ตั้งแต่ human error ไปจนถึง region-wide outage

---

### Backup Service

#### คืออะไร

Backup Service คือบริการที่ automate การ backup resource ต่าง ๆ เช่น database, volume, file system ตามตารางเวลาที่กำหนด พร้อม retention policy และ restore capability

#### ใช้งานแบบไหน

สร้าง backup plan กำหนดว่า resource ใดต้อง backup บ่อยแค่ไหน เก็บ backup ไว้นานแค่ไหน และต้องการ cross-region หรือ cross-account copy หรือไม่

#### เหมาะกับงานแบบไหน

เหมาะกับทุก production data ที่ไม่ยอมสูญเสียได้ โดยเฉพาะ database, critical file และ configuration

#### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรมี backup ใน production

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                    |
| -------------- | ------------------------------- |
| AWS            | AWS Backup                      |
| GCP            | Cloud Backup and DR             |
| Azure          | Azure Backup                    |
| Huawei Cloud   | Cloud Backup and Recovery (CBR) |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration           | ความหมาย                                    | ตัวอย่าง                                      |
| ------------------------------ | ------------------------------------------- | ------------------------------------------- |
| Backup Plan                    | กำหนด rule ว่า backup อะไร เมื่อไหร่ เก็บนานแค่ไหน | daily backup, retention 30 days             |
| Recovery Point Objective (RPO) | ข้อมูลที่ยอมสูญเสียได้มากที่สุด                       | 1 ชั่วโมง = backup ทุกชั่วโมง                    |
| Recovery Time Objective (RTO)  | เวลาสูงสุดที่ยอมรับในการ restore                 | 4 ชั่วโมง                                     |
| Backup Frequency               | ความถี่ในการ backup                           | hourly, daily, weekly                       |
| Retention Period               | ระยะเวลาเก็บ backup                          | 7 วัน daily, 4 สัปดาห์ weekly, 12 เดือน monthly |
| Cross-Region Copy              | copy backup ไปยัง Region อื่น                  | เพื่อ DR ใน Region ที่สอง                       |
| Backup Vault                   | ที่เก็บ backup พร้อม access control             | encrypted, immutable backup                 |
| Restore Test                   | การทดสอบ restore backup จริง                 | ทดสอบทุกไตรมาส                               |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB ที่ backup จริง (incremental หลังแรก)

| Backup Type             | AWS Backup           | GCP Cloud Backup | Azure Backup    | Huawei CBR      |
| ----------------------- | -------------------- | ---------------- | --------------- | --------------- |
| VM Disk / EBS Snapshot  | $0.05/GB-month       | $0.026/GB-month  | $0.05/GB-month  | ~$0.04/GB-month |
| RDS / Database Backup   | $0.095/GB-month      | $0.08/GB-month   | $0.095/GB-month | ~$0.04/GB-month |
| File System / S3 Backup | $0.05/GB-month       | included         | $0.02/GB-month  | ~$0.03/GB-month |
| Cross-Region Copy       | $0.02/GB transferred | $0.02/GB         | $0.02/GB        | ~$0.015/GB      |

> RDS automated backup ใน AWS ฟรีสำหรับ 0–7 วัน (ขนาดเท่า storage instance) — ค่าใช้จ่าย Backup ส่วนใหญ่เกิดจาก long-term retention และ cross-region copy

#### ตัวอย่างการใช้งานใน Project

RDS กำหนด daily backup เวลา 02:00 UTC retention 30 วัน และ cross-region copy ไปยัง secondary region เพื่อ DR พร้อมทดสอบ restore ทุก quarter

#### Best Practice

- กำหนด RPO และ RTO ก่อนออกแบบ backup strategy
- ทดสอบ restore จริงสม่ำเสมอ ไม่ใช่แค่มี backup
- เก็บ backup ใน account หรือ region ที่แยกจาก production
- ใช้ immutable backup เพื่อป้องกัน ransomware ลบ backup

#### Common Mistakes

- มี backup แต่ไม่เคยทดสอบ restore ทำให้ไม่รู้ว่า restore ได้จริงหรือไม่
- เก็บ backup ใน account เดียวกัน ทำให้ ransomware โจมตีได้ทั้ง production และ backup
- ตั้ง retention สั้นเกินไป ทำให้ไม่มี backup เก่าพอสำหรับ compliance

---

## 21. CDN & Edge

CDN (Content Delivery Network) คือ Service กลุ่มที่กระจาย content ไปยัง edge server ทั่วโลก ทำให้ user ได้รับ content จาก server ที่ใกล้ที่สุด ลด latency และลด load บน origin server

---

### Content Delivery Network (CDN)

#### คืออะไร

CDN คือเครือข่ายของ edge server ที่กระจายอยู่ทั่วโลก cache content เช่น static file, image, video, API response เพื่อให้ user ได้รับ response จาก edge ที่ใกล้ที่สุดแทนการดึงจาก origin ทุกครั้ง

#### ใช้งานแบบไหน

ชี้ DNS ไปยัง CDN endpoint แล้วกำหนด origin ว่าเป็น S3, ALB หรือ web server กำหนด cache behavior ว่า path ไหน cache นานแค่ไหน

#### เหมาะกับงานแบบไหน

เหมาะกับ static website, media streaming, large file download, API ที่มี response cacheable, หรือ global audience ที่ต้องการ low latency

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ API response ที่ personalized ต่อ user แต่ละคนและ cache ไม่ได้ หรือ real-time data ที่ต้องการ fresh ทุกครั้ง

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                   |
| -------------- | ------------------------------ |
| AWS            | Amazon CloudFront              |
| GCP            | Cloud CDN                      |
| Azure          | Azure Front Door, Azure CDN    |
| Huawei Cloud   | Content Delivery Network (CDN) |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                                   |
| -------------------- | ----------------------------------------- | ---------------------------------------- |
| Origin               | server ต้นทางที่ CDN ดึง content มา           | S3 bucket, ALB, custom domain            |
| Cache Behavior       | กำหนด cache policy ต่อ path pattern         | `/static/*` cache 1 ปี, `/api/*` ไม่ cache |
| TTL (Time to Live)   | ระยะเวลา cache อยู่ที่ edge                   | 86400 seconds (1 วัน)                     |
| Cache Invalidation   | การลบ cache ก่อน TTL หมด                   | invalidate `/index.html` หลัง deploy      |
| HTTPS / SSL          | การ force HTTPS                           | redirect HTTP → HTTPS                    |
| Custom Domain        | domain name ที่ใช้กับ CDN                     | `cdn.example.com`                        |
| Geo Restriction      | block หรือ allow access จาก country ที่กำหนด  | เฉพาะ APAC countries                     |
| Origin Shield        | cache layer กลางเพื่อลด request ไปยัง origin | ลด origin load                           |
| WAF Integration      | เชื่อมต่อ WAF บน CDN layer                   | ป้องกันก่อนถึง origin                        |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม GB egress + request count

|                         | AWS CloudFront              | GCP Cloud CDN | Azure Front Door Standard | Huawei CDN  |
| ----------------------- | --------------------------- | ------------- | ------------------------- | ----------- |
| Egress (0–10 TB/month)  | $0.0085/GB                  | $0.0080/GB    | $0.0075/GB                | $0.0070/GB  |
| Egress (10–50 TB/month) | $0.0080/GB                  | $0.0060/GB    | $0.0060/GB                | $0.0060/GB  |
| HTTP/S Request          | $0.0100/10K                 | $0.0075/10K   | $0.0090/10K               | $0.0080/10K |
| Cache Invalidation      | $0.005/path (after 1K free) | ฟรี            | ฟรี                        | ฟรี          |
| Origin Shield           | $0.0075/10K requests        | —             | รวมใน tier                | —           |

> CDN egress ($0.0085/GB) ถูกกว่า serve จาก origin S3/EC2 โดยตรง ($0.09/GB) ~10 เท่า — ใช้ CDN นำหน้า origin เสมอสำหรับ static/public content

#### ตัวอย่างการใช้งานใน Project

```
User (Bangkok) → CloudFront Edge (Singapore)
    → /static/* : cache 1 ปี จาก S3 bucket
    → /api/* : forward ไป ALB ใน ap-southeast-1
    → /* : serve index.html จาก S3 (SPA)
```

#### Best Practice

- ใช้ Cache-Control header ที่ถูกต้องใน origin response
- ตั้งชื่อ file ด้วย content hash เพื่อ long cache สำหรับ versioned asset
- ใช้ CDN เป็น WAF integration point
- invalidate cache เฉพาะ path ที่จำเป็นไม่ invalidate ทั้งหมด (แพงและช้า)

#### Common Mistakes

- ไม่ได้ set Cache-Control header ทำให้ CDN cache ตาม default ที่ไม่เหมาะสม
- invalidate `/*` ทุก deploy ทำให้ cache miss ทั้งหมด
- เปิด caching สำหรับ endpoint ที่มี user-specific response

---

## 22. DNS

DNS (Domain Name System) คือ Service ที่แปลง domain name เป็น IP address ช่วยให้ user เข้าถึง application ด้วย domain ที่จำง่าย และช่วย manage traffic routing ระหว่าง region หรือ environment

---

### Managed DNS

#### คืออะไร

Managed DNS คือบริการ DNS ที่ Cloud Provider จัดการ name server ให้มีความพร้อมใช้งานสูง รองรับ routing policy ต่าง ๆ เช่น latency-based, geolocation, failover และ weighted routing

#### ใช้งานแบบไหน

สร้าง hosted zone สำหรับ domain แล้วสร้าง DNS record เพื่อชี้ domain หรือ subdomain ไปยัง IP address, Load Balancer หรือ CloudFront distribution

#### เหมาะกับงานแบบไหน

เหมาะกับทุก project ที่ต้องการ domain name และ routing ขั้นสูง

#### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ managed DNS สำหรับ production

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name              |
| -------------- | ------------------------- |
| AWS            | Amazon Route 53           |
| GCP            | Cloud DNS                 |
| Azure          | Azure DNS                 |
| Huawei Cloud   | Domain Name Service (DNS) |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                  | ตัวอย่าง                                           |
| -------------------- | ----------------------------------------- | ------------------------------------------------ |
| Record Type          | ประเภทของ DNS record                      | A, AAAA, CNAME, MX, TXT, NS, Alias               |
| TTL                  | เวลาที่ resolver cache record ก่อน query ใหม่ | 300 seconds                                      |
| Routing Policy       | วิธีการตัดสินใจว่าจะ return IP ใด              | Simple, Weighted, Latency, Geolocation, Failover |
| Health Check         | ตรวจสอบ endpoint ก่อน DNS ส่ง traffic ให้    | HTTP health check ทุก 30 วินาที                     |
| Hosted Zone          | container ของ DNS record ทั้งหมดของ domain  | `example.com`                                    |
| Alias Record         | record ที่ชี้ไปยัง AWS resource โดยตรง         | Alias ไป ALB หรือ CloudFront                      |
| Private Hosted Zone  | DNS ที่ใช้งานได้เฉพาะใน VPC                   | internal service discovery                       |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม hosted zone/month และ query count

|                           | AWS Route 53      | GCP Cloud DNS    | Azure DNS        | Huawei DNS        |
| ------------------------- | ----------------- | ---------------- | ---------------- | ----------------- |
| Hosted Zone               | $0.50/zone/month  | $0.20/zone/month | $0.90/zone/month | ~$0.60/zone/month |
| Standard Query            | $0.400/1M         | $0.400/1M        | $0.400/1M        | ~$0.300/1M        |
| Latency/Geo Routing Query | $0.700/1M         | $0.700/1M        | $0.700/1M        | —                 |
| Health Check              | $0.50/check/month | —                | —                | —                 |

#### ตัวอย่างการใช้งานใน Project

```
example.com → Alias → CloudFront Distribution
api.example.com → Latency-based routing
    → ap-southeast-1 ALB (Thailand, Singapore users)
    → us-east-1 ALB (US users)
    Health Check → failover อัตโนมัติเมื่อ region มีปัญหา
```

#### Best Practice

- ตั้ง Health Check กับ DNS record เพื่อ automatic failover
- ใช้ low TTL (60-300s) สำหรับ record ที่อาจเปลี่ยนบ่อย
- ใช้ Alias record แทน CNAME สำหรับ AWS resource (ไม่มีค่าใช้จ่ายและ query เร็วกว่า)
- ใช้ Private Hosted Zone สำหรับ internal service communication

#### Common Mistakes

- TTL สูงเกินไปทำให้ DNS change ช้าเมื่อต้องการ failover
- ไม่มี Health Check ทำให้ DNS ยังชี้ไปยัง endpoint ที่ dead

---

## 23. Search

Search คือ Service กลุ่มที่ให้ความสามารถ full-text search, relevance ranking และ faceted search ที่ database ทั่วไปทำได้ไม่ดีนัก

---

### Managed Search Service

#### คืออะไร

Managed Search Service คือบริการ search engine ที่ Cloud Provider จัดการ cluster, scaling, patching ให้ ส่วนใหญ่ใช้ Elasticsearch หรือ OpenSearch เป็น engine รองรับ full-text search, relevance scoring, aggregation และ real-time indexing

#### ใช้งานแบบไหน

index document เข้า search engine แล้ว query ด้วย full-text search พร้อม filter, aggregation และ highlight ใช้กับ product catalog search, document search, log analysis

#### เหมาะกับงานแบบไหน

เหมาะกับ product search ที่ต้องการ relevance, document management, log analysis, e-commerce search with facets

#### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ primary transactional database เพราะ search engine ไม่ใช่ ACID database

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                           |
| -------------- | -------------------------------------- |
| AWS            | Amazon OpenSearch Service              |
| GCP            | Vertex AI Search, Elastic Cloud on GCP |
| Azure          | Azure AI Search                        |
| Huawei Cloud   | Cloud Search Service (CSS)             |

#### Spec / Configuration ที่ควรรู้

##### AWS OpenSearch Instance Type

| Instance Type       | vCPU | Memory | เหมาะกับงานแบบไหน               | On-demand ($/hr) | RI 1yr All Upfront | RI 3yr All Upfront |
| ------------------- | ---: | -----: | ------------------------------ | ---------------: | -----------------: | -----------------: |
| `t3.small.search`   |    2 |   2 GB | dev/test เท่านั้น                 |            0.036 |              0.023 |              0.016 |
| `m6g.large.search`  |    2 |   8 GB | small production               |            0.148 |              0.093 |              0.066 |
| `m6g.xlarge.search` |    4 |  16 GB | medium production              |            0.296 |              0.186 |              0.132 |
| `r6g.large.search`  |    2 |  16 GB | memory-heavy, large index      |            0.219 |              0.138 |              0.098 |
| `r6g.xlarge.search` |    4 |  32 GB | large index, high query volume |            0.438 |              0.276 |              0.196 |

#### Search Configuration ที่ควรรู้

| Spec / Configuration | ความหมาย                                      | ตัวอย่าง                               |
| -------------------- | --------------------------------------------- | ------------------------------------ |
| Index                | ที่เก็บ document collection                      | `products`, `articles`               |
| Shard                | การแบ่ง index เป็นส่วน ๆ เพื่อ parallel processing | 3 primary shards                     |
| Replica              | copy ของ shard เพื่อ high availability          | 1 replica                            |
| Mapping              | กำหนด data type ของแต่ละ field                  | `title: text`, `price: float`        |
| Analyzer             | วิธีการ tokenize และ process text               | thai language analyzer               |
| Query Type           | ประเภทของ query                               | full-text, term, range, bool, vector |

#### ตัวอย่างการใช้งานใน Project

E-commerce ใช้ OpenSearch สำหรับ product search โดย sync ข้อมูลจาก RDS เมื่อมี product update ผ่าน event และ serve search query โดยตรงจาก OpenSearch

#### Best Practice

- แยก read node และ write operation เพื่อ performance
- ตั้ง replica ≥ 1 สำหรับ production
- กำหนด index mapping ก่อน ingest data ไม่ใช่ dynamic mapping
- monitor index size และ query latency

#### Common Mistakes

- ใช้ dynamic mapping ทำให้ type inference ผิดพลาด
- ไม่ได้ตั้ง replica ทำให้ data หายเมื่อ node crash
- ใช้ search engine เป็น primary database

---

## 24. Cost Management

Cost Management คือ Service กลุ่มที่ช่วย monitor, analyze และ optimize ค่าใช้จ่ายบน Cloud ช่วยให้ทีมเข้าใจว่าเงินถูกใช้ไปกับ resource อะไรและมีโอกาส optimize ที่ไหนบ้าง

---

### Cloud Cost Management & Billing

#### คืออะไร

Cloud Cost Management คือบริการที่แสดงรายละเอียดค่าใช้จ่ายตาม service, account, tag, region และ time period พร้อม budget alert และ recommendation สำหรับ cost optimization

#### ใช้งานแบบไหน

ตั้ง budget สำหรับแต่ละ account หรือ project กำหนด alert เมื่อค่าใช้จ่ายเกิน threshold review cost report รายสัปดาห์ ใช้ tag กำกับ resource เพื่อ track cost ต่อ team หรือ feature

#### เหมาะกับงานแบบไหน

เหมาะกับทุก Cloud environment การไม่ monitor cost มักทำให้ค่าใช้จ่ายบานปลาย

#### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ cost management

#### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                              |
| -------------- | --------------------------------------------------------- |
| AWS            | AWS Cost Explorer, AWS Budgets, AWS Cost and Usage Report |
| GCP            | Cloud Billing, Cost Management                            |
| Azure          | Azure Cost Management + Billing                           |
| Huawei Cloud   | Cost Center                                               |

#### Spec / Configuration ที่ควรรู้

| Spec / Configuration               | ความหมาย                                 | ตัวอย่าง                                     |
| ---------------------------------- | ---------------------------------------- | ------------------------------------------ |
| Budget                             | วงเงินที่กำหนดต่อ period                      | $500/เดือน ต่อ account                       |
| Budget Alert                       | alert เมื่อค่าใช้จ่ายเกิน % ของ budget         | alert เมื่อถึง 80% ของ budget                 |
| Cost Allocation Tag                | tag สำหรับ group cost ตาม dimension        | `Environment: production`, `Team: backend` |
| Savings Plans / Reserved Instances | commitment ล่วงหน้าเพื่อลดค่าใช้จ่าย            | 1 ปี Savings Plans ลด 30-40%                |
| Spot / Preemptible Instance        | instance ราคาถูกที่ interruptible ได้        | ลดค่าใช้จ่าย batch workload 60-90%            |
| Right Sizing                       | การ resize resource ให้เหมาะกับการใช้งานจริง | ลด instance ที่ CPU < 5% ตลอด                |
| Idle Resource                      | resource ที่ไม่ได้ใช้งานแต่ยังถูก charge         | ลบ unattached EBS volume                   |
| Cost Anomaly Detection             | ตรวจหาค่าใช้จ่ายที่ผิดปกติ                      | alert เมื่อค่าใช้จ่ายเพิ่มขึ้น 50% จากสัปดาห์ก่อน      |

#### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี (core features) + On-demand (advanced API)

| Feature                            | AWS                   | GCP | Azure | Huawei Cloud |
| ---------------------------------- | --------------------- | --- | ----- | ------------ |
| Cost Dashboard / Bills             | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| Budget Alert                       | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| Cost Explorer UI                   | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| Cost Explorer API                  | $0.01/paginated query | ฟรี  | ฟรี    | ฟรี           |
| Anomaly Detection                  | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| Savings Recommendations            | ฟรี                    | ฟรี  | ฟรี    | ฟรี           |
| 3rd-party Cost Tools (Spot.io ฯลฯ) | ค่าใช้จ่ายแยกต่างหาก      | —   | —     | —            |

#### ตัวอย่างการใช้งานใน Project

กำหนด Budget $1000/เดือน ต่อ account ตั้ง alert ที่ 70%, 90%, 100% ใช้ tag `Environment` และ `Team` ทุก resource review Cost Explorer ทุกสัปดาห์เพื่อดู trend

#### Best Practice

- ใช้ tag ทุก resource ตั้งแต่เริ่มต้น ไม่ใช่ย้อนหลัง
- ตั้ง budget alert ทุก account ทุก environment
- review unused resource สม่ำเสมอ เช่น idle instance, unattached volume, old snapshot
- ใช้ Savings Plans หรือ Reserved Instance สำหรับ steady-state workload
- เปิด Cost Anomaly Detection เพื่อจับ cost spike กะทันหัน

#### Common Mistakes

- ไม่ได้ tag resource ทำให้ไม่รู้ว่า cost ไปกับ team หรือ service ใด
- ไม่ได้ตั้ง budget alert ทำให้รู้ค่าใช้จ่ายหลังจาก bill มาแล้ว
- ลืมลบ resource ที่ไม่ใช้แล้วเช่น dev environment ที่ไม่มีใครใช้
- ใช้ On-Demand pricing สำหรับ resource ที่ run 24/7 โดยไม่ได้ซื้อ Savings Plans


---

## 25. Pricing Model / Billing Model

การเข้าใจ Pricing Model ช่วยให้ตัดสินใจได้ว่าจะใช้รูปแบบการจ่ายเงินแบบไหนให้เหมาะกับ workload และงบประมาณของ project

> **หมายเหตุ:** ราคาทั้งหมดในเอกสารนี้เป็นราคาโดยประมาณ ณ ปี 2025 อ้างอิงจาก Region: **AWS us-east-1 / GCP us-central1 / Azure East US / Huawei Cloud AP-Southeast** ราคาจริงอาจแตกต่างตาม Region, configuration และข้อตกลงพิเศษ ควรตรวจสอบจาก pricing calculator ของแต่ละ Cloud Provider เสมอ

---

### Pay-as-you-go / On-demand

#### คืออะไร

รูปแบบจ่ายตามการใช้งานจริง ไม่มีการ commit ล่วงหน้า ใช้เท่าไรจ่ายเท่านั้น

#### เหมาะกับงานแบบไหน

dev/test environment, workload ที่มี traffic spike ไม่สม่ำเสมอ, project ใหม่ที่ยังไม่รู้ usage pattern

#### ไม่เหมาะกับงานแบบไหน

steady-state production workload ที่ run 24/7 เป็นเดือน ๆ เพราะแพงกว่า Reserved อย่างมีนัยสำคัญ

#### ตัวอย่างราคาต่อหน่วย (On-demand)

---

##### Networking

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

##### Compute / Virtual Machine

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

##### Container Instance / Serverless Container

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

##### Serverless / FaaS

|                            | AWS Lambda              | GCP Cloud Functions  | Azure Functions (Consumption) | Huawei FunctionGraph  |
| -------------------------- | ----------------------- | -------------------- | ----------------------------- | --------------------- |
| Invocation                 | $0.20/1M requests       | $0.40/1M requests    | $0.20/1M requests             | $0.20/1M requests     |
| Compute                    | $0.0000166667/GB-second | $0.0000100/GB-second | $0.000016/GB-second           | $0.00001667/GB-second |
| Free Tier (monthly)        | 1M req + 400K GB-sec    | 2M req + 400K GB-sec | 1M req + 400K GB-sec          | 1M req + 400K GB-sec  |
| ตัวอย่าง 1M req × 512MB × 1s | ~$0.208                 | ~$0.405              | ~$0.208                       | ~$0.208               |

---

##### Load Balancing

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

##### API Management

|           | AWS API Gateway (REST) | GCP Apigee (Eval)  | Azure API Management (Consumption) | Huawei APIG  |
| --------- | ---------------------- | ------------------ | ---------------------------------- | ------------ |
| Request   | $3.50/1M               | $0.03/1K API calls | $3.50/1M                           | ~$3.00/1M    |
| Cache     | $0.020/hour (0.5 GB)   | N/A                | included                           | ~$0.015/hour |
| WebSocket | $1.00/1M messages      | N/A                | included                           | ~$1.00/1M    |

---

##### Object Storage

| Storage Class              | AWS S3            | GCP Cloud Storage | Azure Blob Storage | Huawei OBS      |
| -------------------------- | ----------------- | ----------------- | ------------------ | --------------- |
| Standard                   | $0.023/GB-month   | $0.020/GB-month   | $0.018/GB-month    | $0.020/GB-month |
| Infrequent Access          | $0.0125/GB-month  | $0.010/GB-month   | $0.010/GB-month    | $0.010/GB-month |
| Archive/Glacier Instant    | $0.004/GB-month   | $0.004/GB-month   | $0.002/GB-month    | $0.002/GB-month |
| Deep Archive               | $0.00099/GB-month | $0.0012/GB-month  | $0.00099/GB-month  | $0.001/GB-month |
| PUT/POST (per 1K requests) | $0.005            | $0.005            | $0.055             | $0.004          |
| GET (per 1K requests)      | $0.0004           | $0.0004           | $0.0044            | $0.0004         |

---

##### Block Storage

| Volume Type                   | AWS EBS                 | GCP Persistent Disk | Azure Managed Disk    | Huawei EVS       |
| ----------------------------- | ----------------------- | ------------------- | --------------------- | ---------------- |
| SSD General Purpose (gp3/SSD) | $0.08/GB-month          | $0.170/GB-month     | $0.084/GB-month (P10) | ~$0.075/GB-month |
| SSD High IOPS (io2/Extreme)   | $0.125/GB + $0.065/IOPS | $0.187/GB-month     | $0.127/GB-month (P20) | ~$0.120/GB-month |
| HDD (st1/Standard)            | $0.045/GB-month         | $0.040/GB-month     | $0.045/GB-month       | ~$0.035/GB-month |
| Snapshot                      | $0.05/GB-month          | $0.026/GB-month     | $0.05/GB-month        | ~$0.04/GB-month  |

---

##### Shared File Storage (NFS)

|                   | AWS EFS             | GCP Filestore  | Azure Files (Premium) | Huawei SFS       |
| ----------------- | ------------------- | -------------- | --------------------- | ---------------- |
| Standard Tier     | $0.30/GB-month      | $0.20/GB-month | $0.06/GB-month        | ~$0.060/GB-month |
| Infrequent Access | $0.025/GB-month     | N/A            | N/A                   | N/A              |
| ค่า Read/Write     | $0.01/GB (IA reads) | รวมอยู่ใน tier   | รวมอยู่ใน tier          | รวมอยู่ใน tier     |

---

##### Relational Database (RDS / Managed SQL)

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

##### NoSQL Database

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

##### Cache (Redis / Memcached)

**On-demand Node Price (per hour)**

| Node Type          | AWS ElastiCache      | GCP Memorystore | Azure Cache for Redis | Huawei DCS |
| ------------------ | -------------------- | --------------- | --------------------- | ---------- |
| Small dev (~1 GB)  | $0.034 (t3.small)    | $0.049/GB × 1   | $0.055 (C1 Basic)     | ~$0.040    |
| Medium (6–8 GB)    | $0.068 (t3.large)    | $0.049/GB × 8   | $0.101 (C1 Standard)  | ~$0.082    |
| Production (13 GB) | $0.166 (r6g.large)   | $0.049/GB × 13  | $0.202 (C2 Standard)  | ~$0.120    |
| Large (26 GB)      | $0.332 (r6g.xlarge)  | $0.049/GB × 26  | $0.403 (C3 Standard)  | ~$0.240    |
| XLarge (52 GB)     | $0.664 (r6g.2xlarge) | $0.049/GB × 52  | $0.806 (C4 Standard)  | ~$0.480    |

---

##### Message Queue

|                     | AWS SQS           | GCP Cloud Tasks     | Azure Service Bus (Standard) | Huawei DMS RocketMQ      |
| ------------------- | ----------------- | ------------------- | ---------------------------- | ------------------------ |
| Standard Queue      | $0.40/1M requests | $0.40/1M operations | $10/month + $0.013/1M        | ~$0.006/hour + $0.020/GB |
| FIFO Queue          | $0.50/1M requests | N/A                 | $10/month + $0.013/1M        | ~$0.008/hour             |
| Free Tier (monthly) | 1M requests ฟรี    | 1M operations ฟรี    | 10M operations ฟรี            | —                        |
| Message Size (max)  | 256 KB            | 1 MB                | 256 KB                       | 4 MB                     |
| Retention (max)     | 14 วัน             | 30 วัน               | 14 วัน                        | 3 วัน                     |

---

##### Pub/Sub

|                     | AWS SNS                | AWS EventBridge | GCP Cloud Pub/Sub  | Azure Event Grid    | Huawei DMS |
| ------------------- | ---------------------- | --------------- | ------------------ | ------------------- | ---------- |
| ราคา                | $0.50/1M notifications | $1.00/1M events | $0.04/GB processed | $0.60/1M operations | ~$0.020/GB |
| HTTP/HTTPS delivery | $0.60/1M               | —               | รวมใน data         | —                   | —          |
| Email delivery      | $2.00/1M               | —               | N/A                | —                   | —          |
| Free Tier (monthly) | 1M ฟรี                  | —               | 10 GB ฟรี           | 100K ฟรี             | —          |

---

##### Event Streaming (Kafka / Kinesis)

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

##### Security — WAF

|                    | AWS WAF                       | GCP Cloud Armor             | Azure WAF (App Gateway WAF v2) | Huawei WAF         |
| ------------------ | ----------------------------- | --------------------------- | ------------------------------ | ------------------ |
| Web ACL / Policy   | $5.00/month                   | $5.00/policy/month          | รวมใน Gateway fee              | ~$30/month (basic) |
| Rule               | $1.00/rule/month              | —                           | รวมใน Gateway fee              | รวมใน plan         |
| Request            | $0.60/1M requests             | $0.75/1M requests evaluated | รวมใน CU charge                | ~$0.30/1M          |
| Managed Rule Group | $20/group/month               | —                           | —                              | —                  |
| Bot Control        | $10/month + $1.00/1M requests | $1.00/1M requests           | $0.004/1K requests             | —                  |

---

##### Security — DDoS Protection

|          | AWS Shield Standard | AWS Shield Advanced | GCP Cloud Armor   | Azure DDoS Basic | Azure DDoS Standard        | Huawei Anti-DDoS |
| -------- | ------------------- | ------------------- | ----------------- | ---------------- | -------------------------- | ---------------- |
| ราคา     | ฟรี (auto)           | $3,000/month        | รวมใน Cloud Armor | ฟรี (auto)        | $2,944/month/protected VIP | ~$1,500/month    |
| Coverage | L3/L4 auto          | L3/L4/L7 + DRT      | L3/L4/L7          | L3/L4 basic      | L3/L4/L7 + SLA             | L3/L4            |

---

##### Identity & Access Management (IAM)

| Service                                      | AWS | GCP                 | Azure              | Huawei Cloud |
| -------------------------------------------- | --- | ------------------- | ------------------ | ------------ |
| IAM Users/Roles/Policies                     | ฟรี  | ฟรี                  | ฟรี                 | ฟรี           |
| IAM Identity Center (SSO)                    | ฟรี  | ฟรี (Cloud Identity) | ฟรี (Entra ID Free) | ฟรี           |
| MFA                                          | ฟรี  | ฟรี                  | ฟรี                 | ฟรี           |
| Azure Entra ID P1 (Conditional Access, MFA)  | N/A | N/A                 | $6/user/month      | N/A          |
| Azure Entra ID P2 (PIM, Identity Protection) | N/A | N/A                 | $9/user/month      | N/A          |

---

##### Key Management Service (KMS)

|                           | AWS KMS            | GCP Cloud KMS           | Azure Key Vault (Standard) | Huawei DEW/KMS   |
| ------------------------- | ------------------ | ----------------------- | -------------------------- | ---------------- |
| Customer Managed Key      | $1.00/key/month    | $0.06/key version/month | $0.03/10K transactions     | ~$1.00/key/month |
| AWS Managed Key           | ฟรี                 | N/A                     | N/A                        | N/A              |
| Cryptographic Operations  | $0.03/10K requests | $0.03/10K operations    | $0.03/10K transactions     | ~$0.03/10K       |
| Asymmetric Key Operations | $0.15/10K requests | $0.06/10K               | $0.15/10K                  | ~$0.10/10K       |

---

##### Secrets Manager

|                           | AWS Secrets Manager       | GCP Secret Manager          | Azure Key Vault (Secrets) | Huawei DEW Secrets  |
| ------------------------- | ------------------------- | --------------------------- | ------------------------- | ------------------- |
| Secret Storage            | $0.40/secret/month        | $0.06/secret version/month  | $0.03/10K transactions    | ~$0.10/secret/month |
| API Call                  | $0.05/10K API calls       | $0.03/10K access operations | รวมใน transactions        | ~$0.03/10K          |
| Cross-Account Replication | $0.40/secret/month/Region | N/A                         | N/A                       | N/A                 |

---

##### Monitoring & Observability

|                   | AWS CloudWatch                       | GCP Cloud Monitoring                | Azure Monitor          | Huawei Cloud Eye    |
| ----------------- | ------------------------------------ | ----------------------------------- | ---------------------- | ------------------- |
| Custom Metric     | $0.30/metric/month (first 10K)       | $0.18/metric/month (after 150 free) | $0.25/metric/month     | ~$0.20/metric/month |
| Dashboard         | $3.00/dashboard/month (after 3 free) | ฟรี                                  | ฟรี                     | ฟรี                  |
| Alarm             | $0.10/alarm/month                    | ฟรี (first 5)                        | $0.10/alert rule/month | ~$0.05/alarm/month  |
| GetMetricData API | $0.01/1K metrics requested           | ฟรี                                  | ฟรี                     | ฟรี                  |

---

##### Logging

|                  | AWS CloudWatch Logs | GCP Cloud Logging           | Azure Monitor Logs      | Huawei LTS      |
| ---------------- | ------------------- | --------------------------- | ----------------------- | --------------- |
| Log Ingestion    | $0.50/GB            | $0.01/GB (after 50 GB free) | $2.30/GB                | ~$0.04/GB       |
| Log Storage      | $0.03/GB-month      | $0.01/GB-month              | รวมใน ingestion (90 วัน) | ~$0.02/GB-month |
| Log Query        | $0.005/GB scanned   | $0.01/GB scanned            | รวมใน workspace fee     | ~$0.003/GB      |
| Retention (free) | N/A                 | 30 วันสำหรับ _Default bucket   | 31 วัน                   | 7 วัน            |

---

##### Tracing

|                     | AWS X-Ray                 | GCP Cloud Trace                  | Azure Application Insights | Huawei APM           |
| ------------------- | ------------------------- | -------------------------------- | -------------------------- | -------------------- |
| Trace Recording     | $5.00/1M traces           | $0.20/1M spans (after 2.5M free) | $2.30/GB ingested          | ~$2.00/month (basic) |
| Trace Retrieval     | $0.50/1M traces retrieved | ฟรี                               | รวมใน ingestion            | รวมใน plan           |
| Free Tier (monthly) | 100K traces ฟรี            | 2.5M spans ฟรี                    | 5 GB ฟรี                    | —                    |

---

##### DevOps & CI/CD

|                     | AWS CodeBuild           | AWS CodePipeline            | GCP Cloud Build          | Azure Pipelines                 | Huawei CodeArts    |
| ------------------- | ----------------------- | --------------------------- | ------------------------ | ------------------------------- | ------------------ |
| Build/minute        | $0.005 (general1.small) | —                           | $0.003/build-minute      | $0.008/minute (after free tier) | รวมใน plan         |
| Pipeline            | —                       | $1.00/active pipeline/month | —                        | ฟรี (1 parallel job)             | รวมใน plan         |
| Free Tier (monthly) | 100 build-minutes ฟรี    | 1 pipeline ฟรี               | 120 build-minutes/day ฟรี | 1,800 minutes ฟรี                | —                  |
| Subscription Plan   | —                       | —                           | —                        | —                               | ~$15/month (basic) |

---

##### Container Registry

|                                    | AWS ECR        | GCP Artifact Registry | Azure Container Registry (Basic) | Huawei SWR      |
| ---------------------------------- | -------------- | --------------------- | -------------------------------- | --------------- |
| Storage                            | $0.10/GB-month | $0.10/GB-month        | $0.10/GB-day (~$3/GB-month)      | ~$0.05/GB-month |
| Data Transfer (pull, same region)  | ฟรี             | ฟรี                    | ฟรี                               | ฟรี              |
| Data Transfer (pull, cross-region) | $0.09/GB       | $0.08/GB              | $0.087/GB                        | ~$0.07/GB       |
| Registry Fee                       | ฟรี             | ฟรี                    | $0.167/day (~$5/month)           | ฟรี              |

---

##### Backup & Disaster Recovery

|                                    | AWS Backup           | GCP Cloud Backup | Azure Backup    | Huawei CBR      |
| ---------------------------------- | -------------------- | ---------------- | --------------- | --------------- |
| EBS / Disk Snapshot (warm)         | $0.05/GB-month       | $0.026/GB-month  | $0.05/GB-month  | ~$0.04/GB-month |
| RDS Backup (beyond free retention) | $0.095/GB-month      | $0.08/GB-month   | $0.095/GB-month | ~$0.04/GB-month |
| S3 / Object Backup                 | $0.05/GB-month       | included         | $0.02/GB-month  | ~$0.03/GB-month |
| Cross-Region Copy                  | $0.02/GB transferred | $0.02/GB         | $0.02/GB        | ~$0.015/GB      |

---

##### CDN & Edge

|                            | AWS CloudFront                    | GCP Cloud CDN     | Azure Front Door (Standard) | Huawei CDN |
| -------------------------- | --------------------------------- | ----------------- | --------------------------- | ---------- |
| Egress (first 10 TB/month) | $0.0085/GB                        | $0.0080/GB        | $0.0075/GB                  | $0.0070/GB |
| Egress (next 40 TB/month)  | $0.0080/GB                        | $0.0060/GB        | $0.0060/GB                  | $0.0060/GB |
| HTTP Request (per 10K)     | $0.0100                           | $0.0075           | $0.0090                     | $0.0080    |
| HTTPS Request (per 10K)    | $0.0100                           | $0.0090           | รวมใน request               | $0.0090    |
| Cache Invalidation         | $0.005/path (after 1K free/month) | ฟรี                | ฟรี                          | ฟรี         |
| WAF add-on                 | $5/policy + $0.60/1M req          | รวมใน Cloud Armor | รวมใน WAF rule              | แยกซื้อ      |

---

##### DNS

|                         | AWS Route 53                | GCP Cloud DNS    | Azure DNS        | Huawei DNS        |
| ----------------------- | --------------------------- | ---------------- | ---------------- | ----------------- |
| Hosted Zone             | $0.50/zone/month (first 25) | $0.20/zone/month | $0.90/zone/month | ~$0.60/zone/month |
| DNS Query (Standard)    | $0.40/1M queries            | $0.40/1M queries | $0.40/1M queries | ~$0.30/1M queries |
| DNS Query (Latency/Geo) | $0.70/1M queries            | $0.70/1M queries | $0.70/1M queries | —                 |
| Health Check            | $0.50/check/month           | —                | —                | —                 |

---

##### Search

|                                     | AWS OpenSearch  | GCP (Elastic on Marketplace) | Azure AI Search          | Huawei CSS       |
| ----------------------------------- | --------------- | ---------------------------- | ------------------------ | ---------------- |
| Small dev (t3.small.search)         | $0.036/hour     | varies                       | —                        | —                |
| General (m6g.large.search / 8 GB)   | $0.148/hour     | ~$0.150/hour                 | $245/month (Standard S1) | ~$0.120/hour     |
| General (m6g.xlarge.search / 16 GB) | $0.296/hour     | ~$0.300/hour                 | $245/month (S1, 25 GB)   | ~$0.240/hour     |
| Memory (r6g.large.search / 16 GB)   | $0.219/hour     | ~$0.250/hour                 | $981/month (Standard S2) | ~$0.200/hour     |
| Storage                             | $0.135/GB-month | ~$0.100/GB-month             | รวมใน tier               | ~$0.100/GB-month |
| Free Tier                           | —               | —                            | Basic ~$73/month         | —                |

---

##### Cost Management

|                                    | AWS Cost Explorer     | GCP Cost Management | Azure Cost Management | Huawei Cost Center |
| ---------------------------------- | --------------------- | ------------------- | --------------------- | ------------------ |
| Cost Visibility                    | ฟรี                    | ฟรี                  | ฟรี                    | ฟรี                 |
| Budget Alert                       | ฟรี                    | ฟรี                  | ฟรี                    | ฟรี                 |
| Advanced Query (Cost Explorer API) | $0.01/paginated query | ฟรี                  | ฟรี                    | ฟรี                 |
| Savings Plans Recommendations      | ฟรี                    | ฟรี                  | ฟรี                    | ฟรี                 |
| Anomaly Detection                  | ฟรี                    | ฟรี                  | ฟรี                    | ฟรี                 |

---

#### Best Practice

- ใช้ On-demand ใน dev/test เพราะ environment เหล่านี้ไม่ได้ run ตลอด
- ติดตาม usage ด้วย Cost Explorer และ Budget Alert ตั้งแต่วันแรก
- มองข้าม data transfer cost ไม่ได้ โดยเฉพาะระบบที่มี cross-region หรือ egress สูง
- หลังจาก run production 2–3 เดือน evaluate ว่าคุ้มค่าไหมที่จะเปลี่ยนเป็น Reserved

#### Common Mistakes

- ใช้ On-demand กับ production instance ที่ run 24/7 เป็นปี โดยไม่ได้ evaluate Reserved
- มองข้าม data transfer cost จนเกิด bill shock
- ไม่ได้ตั้ง Budget Alert ทำให้ไม่รู้ว่า usage พุ่งขึ้น

---

### Reserved / Committed Use

#### คืออะไร

รูปแบบที่ commit ว่าจะใช้ resource ต่อเนื่อง 1 ปี หรือ 3 ปี แลกกับส่วนลด 35–72% เมื่อเทียบกับ On-demand

#### เหมาะกับงานแบบไหน

production workload ที่ run ตลอด 24/7 อย่างน้อย 1 ปี

#### ไม่เหมาะกับงานแบบไหน

dev/test environment, workload ที่ยังไม่แน่ใจ pattern หรือ project ที่อาจ shutdown ก่อน commitment หมด

#### ตัวอย่างราคาต่อหน่วย (Reserved vs On-demand)

---

##### AWS EC2 Reserved Instance (us-east-1)

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

##### AWS Savings Plans (us-east-1)

| Type                       | Flexibility                                     | ส่วนลดสูงสุด (1yr) | ส่วนลดสูงสุด (3yr) |
| -------------------------- | ----------------------------------------------- | --------------: | --------------: |
| Compute Savings Plans      | EC2 ทุก family + Lambda + Fargate                |            ~66% |            ~66% |
| EC2 Instance Savings Plans | EC2 ใน instance family ที่ commit (ยืดหยุ่น size/OS) |            ~72% |            ~72% |

##### AWS RDS Reserved (us-east-1, Single-AZ)

| Instance Class (MySQL/PostgreSQL) | On-demand/hr | 1yr All Upfront/hr | 3yr All Upfront/hr | ประหยัด (3yr) |
| --------------------------------- | -----------: | -----------------: | -----------------: | -----------: |
| `db.t3.medium` (2/4 GB)           |       $0.068 |             $0.046 |             $0.033 |         ~51% |
| `db.m6g.large` (2/8 GB)           |       $0.150 |             $0.100 |             $0.072 |         ~52% |
| `db.m6g.xlarge` (4/16 GB)         |       $0.300 |             $0.200 |             $0.144 |         ~52% |
| `db.m6g.2xlarge` (8/32 GB)        |       $0.600 |             $0.400 |             $0.288 |         ~52% |
| `db.r6g.large` (2/16 GB)          |       $0.240 |             $0.160 |             $0.115 |         ~52% |
| `db.r6g.xlarge` (4/32 GB)         |       $0.480 |             $0.320 |             $0.230 |         ~52% |

> Multi-AZ แพงกว่า Single-AZ ประมาณ 2 เท่า

##### AWS ElastiCache Reserved (us-east-1)

| Node Type                   | On-demand/hr | 1yr All Upfront/hr | 3yr All Upfront/hr | ประหยัด (3yr) |
| --------------------------- | -----------: | -----------------: | -----------------: | -----------: |
| `cache.t3.medium` (3 GB)    |       $0.068 |             $0.046 |             $0.033 |         ~51% |
| `cache.r6g.large` (13 GB)   |       $0.166 |             $0.112 |             $0.080 |         ~52% |
| `cache.r6g.xlarge` (26 GB)  |       $0.332 |             $0.224 |             $0.160 |         ~52% |
| `cache.r6g.2xlarge` (52 GB) |       $0.664 |             $0.448 |             $0.320 |         ~52% |

##### AWS OpenSearch Reserved (us-east-1)

| Instance Type                 | On-demand/hr | 1yr All Upfront/hr | 3yr All Upfront/hr | ประหยัด (3yr) |
| ----------------------------- | -----------: | -----------------: | -----------------: | -----------: |
| `m6g.large.search` (2/8 GB)   |       $0.148 |             $0.100 |             $0.071 |         ~52% |
| `m6g.xlarge.search` (4/16 GB) |       $0.296 |             $0.200 |             $0.143 |         ~52% |
| `r6g.large.search` (2/16 GB)  |       $0.219 |             $0.148 |             $0.106 |         ~52% |

##### GCP Committed Use Discount (us-central1)

| Machine Type            | On-demand/hr | 1yr CUD/hr | 3yr CUD/hr | ประหยัด (3yr) |
| ----------------------- | -----------: | ---------: | ---------: | -----------: |
| `n2-standard-2`         |       $0.097 |     $0.067 |     $0.048 |         ~51% |
| `n2-standard-4`         |       $0.194 |     $0.134 |     $0.096 |         ~51% |
| `n2-standard-8`         |       $0.388 |     $0.268 |     $0.192 |         ~51% |
| `c2-standard-4`         |       $0.209 |     $0.149 |     $0.107 |         ~49% |
| `n2-highmem-4`          |       $0.262 |     $0.181 |     $0.130 |         ~50% |
| Cloud SQL n1-standard-2 |       $0.096 |     $0.067 |     $0.048 |         ~50% |
| Cloud SQL n1-standard-4 |       $0.192 |     $0.134 |     $0.096 |         ~50% |

##### Azure Reserved VM Instance (East US)

| VM Size  | Pay-as-you-go/hr | 1yr Reserved/hr | 3yr Reserved/hr | ประหยัด (3yr) |
| -------- | ---------------: | --------------: | --------------: | -----------: |
| `B2s`    |          $0.0416 |          $0.028 |          $0.018 |         ~57% |
| `D2s v5` |           $0.096 |          $0.061 |          $0.043 |         ~55% |
| `D4s v5` |           $0.192 |          $0.122 |          $0.086 |         ~55% |
| `D8s v5` |           $0.384 |          $0.244 |          $0.172 |         ~55% |
| `E2s v5` |           $0.126 |          $0.080 |          $0.056 |         ~56% |
| `E4s v5` |           $0.252 |          $0.160 |          $0.112 |         ~56% |

##### Huawei Cloud Reserved Instance (AP-Southeast)

| Flavor                  | Pay-per-use/hr | 1yr Reserved/hr | ประหยัด |
| ----------------------- | -------------: | --------------: | -----: |
| `s6.large.2` (2/4 GB)   |        ~$0.040 |         ~$0.026 |   ~35% |
| `c6.large.4` (2/8 GB)   |        ~$0.085 |         ~$0.055 |   ~35% |
| `c6.xlarge.4` (4/16 GB) |        ~$0.170 |         ~$0.110 |   ~35% |
| `m6.large.8` (2/16 GB)  |        ~$0.120 |         ~$0.078 |   ~35% |
| RDS MySQL c6.large.4    |        ~$0.135 |         ~$0.088 |   ~35% |

#### Best Practice

- ซื้อ Savings Plans แทน Reserved Instance เมื่อเป็นไปได้เพราะยืดหยุ่นกว่า
- commit เฉพาะ baseline capacity อย่า commit รวม peak
- review utilization ทุกไตรมาสว่า Reserved ที่ซื้อถูกใช้งานอยู่ไหม (target ≥ 80%)
- ใช้ 1yr term ก่อนถ้าไม่แน่ใจ อย่า commit 3yr ตั้งแต่ต้น

#### Common Mistakes

- commit เป็น specific instance type แล้ว type นั้น end-of-life
- commit เกินกว่า baseline จริง ทำให้จ่ายค่า Reserved ที่ไม่ได้ใช้
- ลืม renew ก่อน expiry ทำให้ราคาเด้งกลับเป็น On-demand

---

### Subscription

#### คืออะไร

รูปแบบจ่ายแบบรายเดือนหรือรายปีในแบบ package หรือ plan ที่กำหนด feature และขีดจำกัดชัดเจน ไม่ผันตาม usage มากนัก

#### เหมาะกับงานแบบไหน

Support plan, security service, managed SLA, compliance tool, managed Kubernetes cluster fee

#### ไม่เหมาะกับงานแบบไหน

workload ที่ usage ผันผวนมาก เพราะ subscription จ่ายแม้จะ idle

#### ตัวอย่างราคาต่อหน่วย (Subscription)

##### Support Plan

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

##### Managed Kubernetes Cluster Fee

| Provider | Service       | ราคา                                    |
| -------- | ------------- | --------------------------------------- |
| AWS      | EKS Cluster   | $0.10/hour (~$73/month) per cluster     |
| GCP      | GKE Standard  | $0.10/hour (ฟรีสำหรับ 1 zonal cluster แรก) |
| GCP      | GKE Autopilot | ฟรี — จ่ายเฉพาะ pod resource              |
| Azure    | AKS           | ฟรี — จ่ายเฉพาะ VM/resource               |
| Huawei   | CCE           | ~$0.05/hour (~$36/month) per cluster    |

##### DDoS Protection (Advanced Tier)

| Provider | Service                  | ราคา                              |
| -------- | ------------------------ | --------------------------------- |
| AWS      | Shield Advanced          | $3,000/month + data transfer fees |
| GCP      | Cloud Armor WAF+DDoS     | $5/policy/month + request fees    |
| Azure    | DDoS Protection Standard | $2,944/month per protected VIP    |
| Huawei   | Advanced Anti-DDoS       | ~$1,500/month                     |

#### Best Practice

- เปรียบเทียบ feature ระหว่าง plan tier ก่อน subscribe
- ประเมินว่า annual plan คุ้มกว่า monthly หรือไม่
- ติดตาม renewal date เพื่อ review ก่อนต่ออายุ

#### Common Mistakes

- subscribe plan ใหญ่กว่าที่จำเป็น
- ลืม cancel subscription ของ service ที่ไม่ได้ใช้แล้ว

---

### Flat Rate

#### คืออะไร

รูปแบบจ่ายเหมาราคาเดียวโดยไม่ผันตาม usage มากนัก พบใน dedicated capacity หรือ Enterprise Agreement (EA) ที่เจรจากับ Cloud Provider โดยตรง

#### เหมาะกับงานแบบไหน

enterprise ที่ต้องการ dedicated capacity, compliance ที่ห้ามแชร์ hardware (PCI DSS, HIPAA), หรือ Cloud spend สูงพอจะเจรจา EA

#### ไม่เหมาะกับงานแบบไหน

startup หรือ project ขนาดเล็ก เพราะ Flat Rate มักต้องการ minimum commitment สูง

#### ตัวอย่างราคาต่อหน่วย (Flat Rate)

##### AWS Dedicated Host (us-east-1)

| Instance Family | ราคา Dedicated Host | instance สูงสุด  | เทียบกับ On-demand |
| --------------- | ------------------: | -------------- | ---------------- |
| `m6i`           |         $4.992/hour | 16× m6i.xlarge | คุ้มเมื่อ fill >70%  |
| `c6i`           |         $4.250/hour | 16× c6i.xlarge | คุ้มเมื่อ fill >70%  |
| `r6i`           |         $6.624/hour | 8× r6i.xlarge  | คุ้มเมื่อ fill >70%  |

> Dedicated Host เหมาะกับ BYOL license (Windows Server, Oracle) ที่ผูกกับ physical core

##### Enterprise Agreement — ประมาณการส่วนลดตาม Annual Spend

| Annual Cloud Spend | ส่วนลดโดยประมาณ | รูปแบบ                             |
| ------------------ | -------------- | --------------------------------- |
| < $100K            | 0%             | On-demand / Savings Plans         |
| $100K – $500K      | 5–10%          | Private Pricing Agreement         |
| $500K – $1M        | 10–15%         | Enterprise Discount Program (EDP) |
| $1M – $5M          | 15–25%         | EDP / Enterprise Agreement        |
| > $5M              | 25–40%+        | Custom Enterprise Agreement       |

> ส่วนลด EA จริงต้องเจรจากับ Cloud Provider โดยตรง

#### Best Practice

- เจรจา EA เมื่อ Cloud spend มีนัยสำคัญและมีแผนระยะยาวชัดเจน
- ให้ฝ่าย finance และ legal ร่วม review term ก่อน sign

#### Common Mistakes

- commit spend สูงเกินกว่าที่ใช้จริง
- ไม่ได้ review termination clause อย่างละเอียด

---

### Spot / Preemptible

#### คืออะไร

Instance ที่ใช้ spare capacity ของ Cloud Provider ในราคาถูกกว่า On-demand 60–90% แต่สามารถถูกดึงคืนได้ตลอดเวลา (2 นาที notice บน AWS, 30 วินาทีบน GCP)

#### เหมาะกับงานแบบไหน

batch processing, ML training, CI/CD build worker, rendering, Kubernetes worker node สำหรับ non-critical pod

#### ไม่เหมาะกับงานแบบไหน

stateful service ที่ไม่มี checkpoint เช่น primary database หรือ workload ที่ต้องการ uptime สูง

#### ตัวอย่างราคาต่อหน่วย (Spot vs On-demand)

##### AWS EC2 Spot (us-east-1, ราคาผันผวน)

| Instance Type       | On-demand/hr | Spot (ประมาณ)/hr |  ประหยัด | เหมาะกับ Spot Use Case  |
| ------------------- | -----------: | ---------------: | ------: | ---------------------- |
| `m6i.large`         |       $0.096 |    $0.029–$0.050 | ~48–70% | general batch worker   |
| `m6i.xlarge`        |       $0.192 |    $0.058–$0.100 | ~48–70% | medium batch, K8s node |
| `m6i.2xlarge`       |       $0.384 |    $0.115–$0.200 | ~48–70% | large batch workload   |
| `c6i.xlarge`        |       $0.170 |    $0.051–$0.090 | ~47–70% | compute-heavy batch    |
| `c6i.2xlarge`       |       $0.340 |    $0.102–$0.180 | ~47–70% | large compute batch    |
| `r6i.large`         |       $0.126 |    $0.038–$0.065 | ~48–70% | memory-heavy batch     |
| `g4dn.xlarge` (GPU) |       $0.526 |    $0.158–$0.250 | ~52–70% | ML training            |

##### GCP Spot VM (us-central1)

| Machine Type    | On-demand/hr | Spot/hr | ประหยัด |
| --------------- | -----------: | ------: | -----: |
| `n2-standard-2` |       $0.097 |  $0.024 |   ~75% |
| `n2-standard-4` |       $0.194 |  $0.048 |   ~75% |
| `n2-standard-8` |       $0.388 |  $0.097 |   ~75% |
| `c2-standard-4` |       $0.209 |  $0.049 |   ~77% |
| `n2-highmem-4`  |       $0.262 |  $0.065 |   ~75% |

##### Azure Spot VM (East US)

| VM Size  | Pay-as-you-go/hr | Spot (ประมาณ)/hr |  ประหยัด |
| -------- | ---------------: | ---------------: | ------: |
| `D2s v5` |           $0.096 |    $0.015–$0.034 | ~65–84% |
| `D4s v5` |           $0.192 |    $0.030–$0.068 | ~65–84% |
| `D8s v5` |           $0.384 |    $0.058–$0.136 | ~65–85% |
| `F4s v2` |           $0.169 |    $0.025–$0.060 | ~65–85% |

##### Huawei Cloud Spot ECS (AP-Southeast)

| Flavor         | Pay-per-use/hr | Spot (ประมาณ)/hr |  ประหยัด |
| -------------- | -------------: | ---------------: | ------: |
| `c6.large.4`   |        ~$0.085 |   ~$0.025–$0.040 | ~53–71% |
| `c6.xlarge.4`  |        ~$0.170 |   ~$0.050–$0.080 | ~53–71% |
| `c6.2xlarge.4` |        ~$0.340 |   ~$0.100–$0.160 | ~53–71% |

##### Interruption Rate vs Savings Strategy

| Interruption Rate | ส่วนลดโดยประมาณ | กลยุทธ์ที่แนะนำ                                      |
| ----------------- | -------------- | ----------------------------------------------- |
| Low (<5%/month)   | 60–75%         | long-running batch, ML training ยาวหลายชั่วโมง    |
| Medium (5–20%)    | 50–65%         | checkpoint บ่อย ทุก 10–30 นาที                     |
| High (>20%)       | 40–60%         | เฉพาะ short job (<1 ชั่วโมง) หรือ stateless worker |

#### Best Practice

- ใช้ instance type หลายแบบใน Spot Fleet/Pool เพื่อเพิ่มโอกาสได้ capacity
- กระจาย Spot request หลาย AZ เสมอ
- ออกแบบ workload ให้ resume จาก checkpoint ได้เมื่อถูก interrupt

#### Common Mistakes

- ใช้ Spot กับ single instance type และ AZ เดียว ทำให้ไม่มี fallback
- run stateful service บน Spot โดยไม่มี mechanism ย้าย state ก่อน interrupt

---

### สรุปเปรียบเทียบ Pricing Model

| Pricing Model             | ราคาเทียบ On-demand | ความยืดหยุ่น             | เหมาะกับ                                      | ไม่เหมาะกับ                    |
| ------------------------- | ------------------ | --------------------- | -------------------------------------------- | ---------------------------- |
| On-demand / Pay-as-you-go | 100% (baseline)    | สูงสุด                  | dev/test, spike traffic, project ใหม่         | steady-state 24/7 production |
| Reserved 1yr All Upfront  | ~40–55%            | ต่ำ (commit 1 ปี)        | production workload ที่ run ตลอด               | workload ที่ pattern ยังไม่นิ่ง    |
| Reserved 3yr All Upfront  | ~30–45%            | ต่ำมาก (commit 3 ปี)     | long-term stable workload                    | project ที่อาจ re-architect    |
| Savings Plans (AWS)       | ~35–55%            | ปานกลาง               | production ที่ต้องการยืดหยุ่น                      | N/A                          |
| Subscription              | Fixed monthly      | ปานกลาง               | support plan, security service, SLA          | workload usage ผันผวนมาก      |
| Flat Rate / Enterprise    | ~60–75%            | ต่ำมาก                  | enterprise large-scale spend, Dedicated Host | startup, project ขนาดเล็ก     |
| Spot / Preemptible        | ~10–40%            | สูง (แต่ interruptible) | batch, ML training, CI/CD worker             | stateful production service  |
