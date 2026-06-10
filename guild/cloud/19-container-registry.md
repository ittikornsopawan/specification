# 19. Container Registry

Container Registry คือ Service สำหรับ store, manage และ distribute container image ก่อน deploy image ขึ้น Kubernetes หรือ container service ต้องมีที่เก็บ image ที่ accessible

---

## Private Container Registry

### คืออะไร

Private Container Registry คือบริการ repository สำหรับ Docker/OCI container image ที่ private เข้าถึงได้เฉพาะ authorized identity รองรับ image versioning, vulnerability scanning และ access control

### ใช้งานแบบไหน

CI/CD pipeline build Docker image แล้ว push ไปยัง registry เมื่อ deploy ระบบ pull image จาก registry เพื่อ run container

### เหมาะกับงานแบบไหน

เหมาะกับทุก project ที่ใช้ container ไม่ควรใช้ public registry เช่น Docker Hub สำหรับ production image

### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ container registry สำหรับ production container workload

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                            |
| -------------- | --------------------------------------- |
| AWS            | Amazon ECR (Elastic Container Registry) |
| GCP            | Artifact Registry, Container Registry   |
| Azure          | Azure Container Registry (ACR)          |
| Huawei Cloud   | Software Repository for Container (SWR) |

### Spec / Configuration

| Spec / Configuration   | ความหมาย                                   | ตัวอย่าง                                    |
| ---------------------- | ------------------------------------------ | ----------------------------------------- |
| Repository             | ที่เก็บ image ของ application หนึ่ง             | `my-company/api-service`                  |
| Image Tag              | version ของ image                          | `v1.2.3`, `latest`, `git-sha-abc123`      |
| Vulnerability Scanning | ตรวจหา security vulnerability ใน image     | scan ทุก push, block image ที่มี critical CVE |
| Image Lifecycle Policy | ลบ image เก่าอัตโนมัติ                         | เก็บไว้แค่ 10 image ล่าสุด                     |
| Repository Policy      | access control ว่า identity ใด pull/push ได้ | เฉพาะ CI/CD role push, EKS node pull      |
| Replication            | copy image ไปยัง Registry ใน Region อื่น      | multi-region deployment                   |
| Image Signing          | เซ็น image เพื่อ verify ว่าไม่ถูกแก้ไข            | Sigstore, Notary                          |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม storage GB และ data transfer

|                              | AWS ECR        | GCP Artifact Registry  | Azure Container Registry (Basic) | Huawei SWR      |
| ---------------------------- | -------------- | ---------------------- | -------------------------------- | --------------- |
| Storage                      | $0.10/GB-month | $0.10/GB-month         | ~$3/GB-month                     | ~$0.05/GB-month |
| Registry Fee                 | ฟรี             | ฟรี                     | $0.167/day (~$5/month)           | ฟรี              |
| Pull (same region)           | ฟรี             | ฟรี                     | ฟรี                               | ฟรี              |
| Pull (cross-region/internet) | $0.09/GB       | $0.08/GB               | $0.087/GB                        | ~$0.07/GB       |
| Free Tier                    | 500 MB/month   | ฟรี (Artifact Registry) | —                                | —               |

### ตัวอย่างการใช้งานใน Project

```
GitHub Actions → build image → push to ECR
EKS Node → pull image from ECR (ผ่าน IAM Role) → run container
```

### Best Practice

- tag image ด้วย git commit SHA ไม่ใช่แค่ `latest`
- เปิด vulnerability scanning ทุก push และ block deploy เมื่อมี critical CVE
- ตั้ง lifecycle policy ลบ image เก่าที่ไม่ใช้
- ใช้ IAM Role สำหรับ pull/push ไม่ใช้ long-term credential

### Common Mistakes

- ใช้ tag `latest` ใน production ทำให้ไม่รู้ว่า version ไหน deploy อยู่
- ไม่ได้ตั้ง lifecycle policy ทำให้ storage เต็มจาก image เก่า
