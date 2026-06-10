# 1. Networking

Networking เป็นรากฐานของทุก Cloud Project ก่อนจะ deploy service ใด ๆ ต้องออกแบบ network topology ก่อนเสมอ เพราะโครงสร้าง network ส่งผลต่อความปลอดภัย การเข้าถึง performance และค่าใช้จ่าย การเข้าใจ Networking ช่วยให้ระบบมีความปลอดภัย แยก environment ได้ถูกต้อง และสื่อสารระหว่าง service ได้อย่างมีประสิทธิภาพ

---

## Virtual Private Cloud (VPC)

### คืออะไร

VPC (Virtual Private Cloud) คือเครือข่ายส่วนตัวเสมือนที่สร้างขึ้นบน Cloud โดยผู้ใช้สามารถกำหนด IP address range, Subnet, Routing และ Network Policy ได้เอง ทำให้ resource ต่าง ๆ ทำงานอยู่ใน network ที่แยกออกจาก cloud อื่น ๆ อย่างสมบูรณ์

### ใช้งานแบบไหน

ทุก project ควรสร้าง VPC เป็นลำดับแรก จากนั้นแบ่ง Subnet ออกเป็น Public Subnet (เชื่อมต่อ internet ได้) และ Private Subnet (เข้าถึงได้เฉพาะภายใน) แบ่ง environment เช่น production, staging, development ออกจากกัน และใช้ VPC Peering หรือ Transit Gateway เพื่อเชื่อมต่อหลาย VPC เข้าด้วยกันเมื่อจำเป็น

### เหมาะกับงานแบบไหน

เหมาะกับทุก project ไม่ว่าจะเล็กหรือใหญ่ เป็น Service พื้นฐานที่ทุก workload บน Cloud ต้องมี

### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ VPC เว้นแต่เป็นการทดสอบเบื้องต้นระยะสั้นใน default network ที่ Cloud Provider จัดให้

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                 |
| -------------- | ---------------------------- |
| AWS            | Amazon VPC                   |
| GCP            | Virtual Private Cloud (VPC)  |
| Azure          | Azure Virtual Network (VNet) |
| Huawei Cloud   | Virtual Private Cloud (VPC)  |

### Spec / Configuration

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

### Pricing Model & ราคาโดยประมาณ

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

### ตัวอย่างการใช้งานใน Project

```
Internet
    │
Internet Gateway
    │
Public Subnet  ── Load Balancer, Bastion Host, NAT Gateway
    │
Private Subnet ── Application Server, Database, Cache
```

### Best Practice

- แบ่ง Public / Private Subnet ทุกครั้ง อย่าวาง Database หรือ Application Server ใน Public Subnet
- สร้าง Subnet กระจายหลาย AZ เสมอเพื่อรองรับ High Availability
- ใช้ CIDR Block ที่ไม่ซ้อนทับกันระหว่าง VPC และ on-premise network เผื่อต้องทำ VPN หรือ Direct Connect ในอนาคต
- เปิด VPC Flow Logs เพื่อ audit และ troubleshoot network traffic

### Common Mistakes

- ใช้ CIDR Block แคบเกินไป (/24 หรือเล็กกว่า) ทำให้ขยาย IP ไม่ได้ในอนาคต
- วาง resource ทุกอย่างใน Public Subnet เพราะ "ง่ายกว่า"
- ลืมสร้าง Subnet ใน AZ ที่สอง ทำให้ไม่มี failover
- ใช้ default VPC ใน production โดยไม่กำหนด network policy เพิ่มเติม

---

## Security Group

### คืออะไร

Security Group คือ virtual firewall ระดับ instance ที่ควบคุม inbound และ outbound traffic ให้กับ resource เช่น Virtual Machine, Database หรือ Container Security Group ทำงานแบบ stateful หมายความว่าถ้าอนุญาต inbound traffic response จะผ่านกลับออกไปได้โดยอัตโนมัติ

### ใช้งานแบบไหน

กำหนด Security Group แยกตาม layer เช่น Security Group สำหรับ Load Balancer, Security Group สำหรับ Application Server และ Security Group สำหรับ Database แล้วอนุญาตเฉพาะ port และ source ที่จำเป็นเท่านั้น

### เหมาะกับงานแบบไหน

เหมาะกับทุก resource ที่รับ network traffic ใช้ร่วมกับ Network ACL เพื่อควบคุม traffic หลายชั้น

### ไม่เหมาะกับงานแบบไหน

Security Group ไม่ใช่ Web Application Firewall (WAF) ไม่ควรใช้แทน WAF สำหรับกรองการโจมตีระดับ application

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                  |
| -------------- | ----------------------------- |
| AWS            | Security Groups               |
| GCP            | Firewall Rules (VPC Firewall) |
| Azure          | Network Security Groups (NSG) |
| Huawei Cloud   | Security Groups               |

### Spec / Configuration

| Spec / Configuration  | ความหมาย                                               | ตัวอย่าง                                           |
| --------------------- | ------------------------------------------------------ | ------------------------------------------------ |
| Inbound Rule          | กฎควบคุม traffic ที่เข้ามา                                 | อนุญาต TCP port 443 จาก `0.0.0.0/0`               |
| Outbound Rule         | กฎควบคุม traffic ที่ออกไป                                 | อนุญาต TCP port 5432 ไปหา Database Security Group |
| Protocol              | โปรโตคอลที่กรอง                                          | TCP, UDP, ICMP                                   |
| Port Range            | ช่วง port ที่อนุญาต                                        | `443`, `3000-3999`                               |
| Source / Destination  | แหล่งที่มาหรือปลายทาง                                      | IP CIDR หรือ Security Group ID อื่น                 |
| Stateful vs Stateless | Security Group เป็น stateful, Network ACL เป็น stateless | ต่างกันในการจัดการ response traffic                 |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี — ทุก Cloud Provider ไม่คิดค่าบริการสำหรับ Security Group / Firewall Rules

|                | AWS Security Group       | GCP VPC Firewall    | Azure NSG   | Huawei Security Group |
| -------------- | ------------------------ | ------------------- | ----------- | --------------------- |
| ราคา           | ฟรี                       | ฟรี                  | ฟรี          | ฟรี                    |
| จำนวน rule สูงสุด | 60 inbound + 60 outbound | 1,000 rules/network | 1,000 rules | 100 rules             |

### ตัวอย่างการใช้งานใน Project

```
sg-loadbalancer: รับ 443 จาก 0.0.0.0/0
sg-app: รับ 8080 จาก sg-loadbalancer เท่านั้น
sg-db: รับ 5432 จาก sg-app เท่านั้น
```

### Best Practice

- ใช้หลัก Least Privilege เปิด port เฉพาะที่จำเป็น
- อ้างอิง Security Group ID แทน IP CIDR เมื่อเป็น internal traffic
- ตั้งชื่อ Security Group ให้สื่อถึง role เช่น `prod-app-sg`, `prod-db-sg`
- Review Security Group rules สม่ำเสมอ ลบ rule ที่ไม่ใช้แล้ว

### Common Mistakes

- เปิด port 22 (SSH) หรือ 3389 (RDP) จาก `0.0.0.0/0`
- ใช้ Security Group เดียวกันกับทุก resource
- อนุญาต all traffic (`0.0.0.0/0`) ในทุก port เพราะ "ทดสอบก่อน" แล้วลืมปิด

---

## Network ACL (Access Control List)

### คืออะไร

Network ACL (Access Control List) คือ firewall ระดับ Subnet ทำงานแบบ stateless โดยกำหนด rule แบบมีลำดับ (numbered rules) สำหรับกรอง traffic ที่เข้าและออกจาก Subnet ทั้งหมด แตกต่างจาก Security Group ที่ทำงานระดับ instance

### ใช้งานแบบไหน

ใช้เป็น defense layer เพิ่มเติมเหนือ Security Group โดยกำหนด rule ที่ Subnet level เพื่อ block IP range หรือ port ที่ไม่ต้องการก่อนที่ traffic จะถึง instance

### เหมาะกับงานแบบไหน

เหมาะกับการ block IP ที่ไม่ต้องการในระดับ Subnet หรือใช้ใน compliance requirement ที่ต้องการ stateless firewall

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับการกำหนด rule แบบละเอียดระดับ instance เพราะ Network ACL ครอบทุก resource ใน Subnet

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                          |
| -------------- | ------------------------------------- |
| AWS            | Network ACLs                          |
| GCP            | Hierarchical Firewall Policies        |
| Azure          | Network Security Groups (ระดับ Subnet) |
| Huawei Cloud   | Network ACL                           |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                     | ตัวอย่าง                                  |
| -------------------- | -------------------------------------------- | --------------------------------------- |
| Rule Number          | ลำดับความสำคัญของ rule (ตัวเลขน้อย = ประมวลก่อน)    | Rule 100: Allow 443, Rule 200: Deny all |
| Allow / Deny         | action ที่จะทำเมื่อ traffic match rule            | Allow หรือ Deny                          |
| Inbound / Outbound   | ทิศทาง traffic ที่ rule ใช้งาน                   | Inbound rule สำหรับ traffic เข้า Subnet    |
| Stateless            | ต้องกำหนด rule สำหรับ request และ response แยกกัน | ต้องเปิด ephemeral port ใน outbound ด้วย   |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี — Network ACL ไม่มีค่าบริการแยกต่างหาก

|      | AWS Network ACL | GCP (ใช้ Firewall Policy แทน) | Azure NSG (subnet-level) | Huawei NACL |
| ---- | --------------- | ---------------------------- | ------------------------ | ----------- |
| ราคา | ฟรี              | ฟรี                           | ฟรี                       | ฟรี          |

### ตัวอย่างการใช้งานใน Project

ใช้ Network ACL บน Private Subnet เพื่อ block traffic จาก internet โดยตรง แม้ว่า route table จะไม่มี Internet Gateway อยู่แล้ว

### Best Practice

- ใช้ร่วมกับ Security Group อย่าพึ่งอย่างใดอย่างหนึ่งอย่างเดียว
- เปิด ephemeral port range (1024-65535) ใน outbound rule เพราะ Network ACL เป็น stateless
- เริ่ม rule number ด้วยช่วงกว้าง เช่น 100, 200, 300 เพื่อแทรก rule เพิ่มได้ในอนาคต

### Common Mistakes

- ลืมเพิ่ม outbound rule สำหรับ response traffic (เพราะ stateless)
- กำหนด Deny rule ที่ rule number ต่ำ แล้ว block traffic ที่ต้องการโดยไม่ตั้งใจ

---

## VPN Gateway

### คืออะไร

VPN Gateway คือบริการที่ช่วยสร้าง encrypted tunnel ระหว่าง Cloud VPC กับ on-premise network หรือ remote user ผ่าน internet โดยใช้โปรโตคอล IPsec หรือ SSL/TLS

### ใช้งานแบบไหน

ใช้ในกรณีที่ต้องการเชื่อมต่อ Cloud network กับ office network หรือ data center ขององค์กร แบบ Site-to-Site VPN หรือสำหรับ developer เชื่อมต่อแบบ Client-to-Site VPN

### เหมาะกับงานแบบไหน

เหมาะกับองค์กรที่มี on-premise system และต้องการเข้าถึง Cloud resource อย่างปลอดภัย หรือ hybrid cloud architecture

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ workload ที่ต้องการ bandwidth สูงและ latency ต่ำมาก ควรใช้ Direct Connect / Dedicated Line แทน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                           |
| -------------- | -------------------------------------- |
| AWS            | AWS VPN (Site-to-Site VPN, Client VPN) |
| GCP            | Cloud VPN                              |
| Azure          | Azure VPN Gateway                      |
| Huawei Cloud   | VPN Gateway                            |

### Spec / Configuration

| Spec / Configuration | ความหมาย                             | ตัวอย่าง                           |
| -------------------- | ------------------------------------ | -------------------------------- |
| VPN Type             | ประเภทของ VPN connection             | Site-to-Site, Client-to-Site     |
| Tunnel Protocol      | โปรโตคอลที่ใช้เข้ารหัส                    | IKEv2/IPsec, OpenVPN, WireGuard  |
| Pre-Shared Key (PSK) | รหัสสำหรับ authenticate tunnel          | ควรสุ่มและซับซ้อน                    |
| BGP / Static Routing | วิธีการ routing ระหว่าง network         | BGP ดีกว่าสำหรับ dynamic routing     |
| Bandwidth            | ความเร็วสูงสุดของ VPN tunnel            | ขึ้นกับ Gateway tier                |
| High Availability    | การ deploy VPN Gateway แบบ redundant | Active/Active หรือ Active/Standby |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม connection/tunnel-hour และ data transfer

|                                 | AWS Site-to-Site VPN | GCP Cloud VPN (HA) | Azure VPN Gateway (VpnGw1) | Huawei VPN Gateway |
| ------------------------------- | -------------------- | ------------------ | -------------------------- | ------------------ |
| Gateway/hour                    | $0.05/connection     | $0.20/tunnel       | $0.190                     | ~$0.050            |
| Data Transfer Out (internet)    | $0.09/GB             | $0.085/GB          | $0.087/GB                  | ~$0.072/GB         |
| ค่าใช้จ่ายตัวอย่าง (2 tunnels, 24/7) | ~$72/month           | ~$288/month        | ~$137/month                | ~$72/month         |

### ตัวอย่างการใช้งานใน Project

เชื่อมต่อ on-premise data center กับ Cloud VPC เพื่อให้ application บน Cloud เข้าถึง legacy database ที่ยังไม่ได้ migrate ขึ้น Cloud

### Best Practice

- ใช้ IKEv2 แทน IKEv1 เพราะปลอดภัยและเสถียรกว่า
- deploy VPN Gateway แบบ Active/Active เพื่อ high availability
- monitor tunnel status สม่ำเสมอ

### Common Mistakes

- ใช้ VPN แทน Direct Connect กับ workload ที่ต้องการ bandwidth สูง
- ไม่ได้ทำ redundant tunnel ทำให้ connection ขาดเมื่อ gateway หนึ่งมีปัญหา

---

## Direct Connect / Dedicated Line

### คืออะไร

Direct Connect หรือ Dedicated Line คือบริการที่ช่วยสร้าง private network connection ความเร็วสูงระหว่าง on-premise infrastructure กับ Cloud โดยไม่ผ่าน internet สาธารณะ ให้ bandwidth สม่ำเสมอและ latency ต่ำกว่า VPN

### ใช้งานแบบไหน

ใช้สำหรับ workload ที่ transfer data ขนาดใหญ่ระหว่าง on-premise และ Cloud หรือ application ที่ต้องการ latency ต่ำสม่ำเสมอ

### เหมาะกับงานแบบไหน

เหมาะกับ enterprise hybrid cloud, workload ที่ transfer data ขนาดใหญ่, financial system ที่ต้องการ latency ต่ำ, หรือ compliance ที่ห้ามใช้ internet สาธารณะ

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ project ขนาดเล็กที่ไม่มี on-premise infrastructure เพราะมีต้นทุนและเวลาในการ provision สูง

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                             |
| -------------- | ---------------------------------------- |
| AWS            | AWS Direct Connect                       |
| GCP            | Cloud Interconnect (Dedicated / Partner) |
| Azure          | Azure ExpressRoute                       |
| Huawei Cloud   | Direct Connect                           |

### Spec / Configuration

| Spec / Configuration    | ความหมาย                                               | ตัวอย่าง                               |
| ----------------------- | ------------------------------------------------------ | ------------------------------------ |
| Port Speed              | ความเร็วของ physical connection                         | 1 Gbps, 10 Gbps                      |
| Virtual Interface (VIF) | logical connection บน Direct Connect circuit           | Private VIF, Public VIF, Transit VIF |
| VLAN                    | ใช้แยก traffic หลาย connection บน physical circuit เดียว | VLAN 100 สำหรับ production             |
| BGP ASN                 | Autonomous System Number สำหรับ BGP routing              | ต้องไม่ซ้ำกับ Cloud Provider              |
| Redundancy              | การ deploy circuit หลายเส้นเพื่อ failover                 | Active/Active dual circuit           |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม port-hour และ data transfer outbound

| Speed             | AWS Direct Connect | GCP Cloud Interconnect | Azure ExpressRoute | Huawei Direct Connect |
| ----------------- | ------------------ | ---------------------- | ------------------ | --------------------- |
| 50–100 Mbps       | $0.30/hour         | ~$0.10/hour            | ~$55/month         | ~$0.10/hour           |
| 1 Gbps            | $0.30/hour         | N/A                    | ~$220/month        | ~$0.25/hour           |
| 10 Gbps           | $1.60/hour         | $1.735/hour            | ~$5,000/month      | ~$1.20/hour           |
| Data Transfer Out | $0.02/GB           | $0.02/GB               | $0.025/GB          | ~$0.02/GB             |

> Direct Connect มีค่า port-hour สูงมาก — 1 Gbps AWS DX ≈ $216/month เพิ่มค่า Partner/colocation อีก ควรใช้เฉพาะเมื่อ bandwidth, latency หรือ compliance requirement จำเป็นจริง ๆ

### ตัวอย่างการใช้งานใน Project

ใช้ Direct Connect เชื่อมต่อ data center ขององค์กรกับ Cloud เพื่อรองรับ database replication และ backup ขนาดใหญ่ทุกคืน

### Best Practice

- deploy อย่างน้อย 2 circuit จาก provider ที่แตกต่างกันเพื่อ redundancy
- ใช้ BGP routing แทน static route เพื่อ failover อัตโนมัติ
- วางแผน bandwidth ให้เผื่อ peak traffic

### Common Mistakes

- deploy circuit เดียวโดยไม่มี backup ทำให้ outage เมื่อ circuit ขาด
- ไม่ได้ทดสอบ failover จาก Direct Connect ไป VPN

---
