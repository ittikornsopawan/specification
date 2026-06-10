# 14. Key Management & Secrets Management

Key Management และ Secrets Management คือ Service กลุ่มที่ช่วยจัดการ cryptographic key และ secret (เช่น database password, API key, certificate) อย่างปลอดภัย แยกออกจาก application code

---

## Key Management Service (KMS)

### คืออะไร

KMS (Key Management Service) คือบริการจัดการ cryptographic key สำหรับ encrypt และ decrypt data บน Cloud Cloud Provider เก็บ key อย่างปลอดภัยใน hardware security module (HSM) และจัดการ key rotation ให้

### ใช้งานแบบไหน

ใช้ encrypt data ที่เก็บใน storage เช่น S3, EBS, RDS หรือ encrypt/decrypt data ใน application โดยตรง ผ่าน API call แทนการ manage key เอง

### เหมาะกับงานแบบไหน

เหมาะกับทุก workload ที่ต้องการ encrypt data at rest โดยเฉพาะ data ที่มีความอ่อนไหว เช่น PII, financial data, health data

### ไม่เหมาะกับงานแบบไหน

KMS ไม่ใช่ Secrets Manager ไม่ควรใช้ KMS เก็บ password หรือ connection string โดยตรง

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                   |
| -------------- | ------------------------------ |
| AWS            | AWS KMS                        |
| GCP            | Cloud KMS                      |
| Azure          | Azure Key Vault (Keys)         |
| Huawei Cloud   | Data Encryption Workshop (DEW) |

### Spec / Configuration

| Spec / Configuration | ความหมาย                                    | ตัวอย่าง                                       |
| -------------------- | ------------------------------------------- | -------------------------------------------- |
| Key Type             | ประเภทของ key                               | Symmetric (AES-256), Asymmetric (RSA, ECC)   |
| Key Origin           | ว่า key material มาจากไหน                    | AWS KMS generated, External (BYOK), CloudHSM |
| Key Usage            | วัตถุประสงค์การใช้ key                          | ENCRYPT_DECRYPT, SIGN_VERIFY                 |
| Key Rotation         | การ rotate key อัตโนมัติ                       | ทุก 1 ปี                                       |
| Key Policy           | กำหนดว่า identity ใดใช้ key ได้                 | อนุญาตเฉพาะ Lambda role                       |
| Envelope Encryption  | pattern การ encrypt data key ด้วย master key | ใช้ใน S3, EBS, RDS                            |
| Multi-Region Key     | key ที่ replicate ข้าม Region                  | สำหรับ multi-region application                |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม key/month และ API call count

|                           | AWS KMS                | GCP Cloud KMS           | Azure Key Vault        | Huawei DEW       |
| ------------------------- | ---------------------- | ----------------------- | ---------------------- | ---------------- |
| Customer Managed Key      | $1.00/key/month        | $0.06/key version/month | $0.03/10K transactions | ~$1.00/key/month |
| AWS/Cloud Managed Key     | ฟรี                     | N/A                     | N/A                    | N/A              |
| Symmetric Encrypt/Decrypt | $0.03/10K requests     | $0.03/10K               | $0.03/10K              | ~$0.03/10K       |
| Asymmetric Sign/Verify    | $0.15/10K requests     | $0.06/10K               | $0.15/10K              | ~$0.10/10K       |
| Automatic Key Rotation    | ฟรี (เพิ่ม 1 key version) | $0.06/version           | ฟรี                     | ฟรี               |

### ตัวอย่างการใช้งานใน Project

RDS database ใช้ KMS key ในการ encrypt storage, S3 bucket ใช้ SSE-KMS โดยกำหนด key policy ว่าเฉพาะ application role และ ops team เท่านั้นที่ decrypt ได้

### Best Practice

- ใช้ Customer Managed Key (CMK) แทน AWS Managed Key เพื่อ control key rotation และ access
- กำหนด key policy แบบ least privilege
- monitor key usage ผ่าน CloudTrail
- เปิด automatic key rotation

### Common Mistakes

- ใช้ key เดียวกันสำหรับทุก service (single key for everything)
- ไม่ได้กำหนด key policy ที่ชัดเจน ทำให้ทุก IAM admin เข้าถึงได้
- ลืม enable key rotation

---

## Secrets Manager

### คืออะไร

Secrets Manager คือบริการเก็บ secret เช่น database password, API key, OAuth token, SSH key อย่างปลอดภัย โดยมีความสามารถ rotate secret อัตโนมัติและ audit การเข้าถึง

### ใช้งานแบบไหน

Application ดึง secret จาก Secrets Manager ผ่าน API call ตอน runtime แทนการเก็บใน environment variable หรือ config file บน server

### เหมาะกับงานแบบไหน

เหมาะกับทุก application ที่ต้องการ database credential, third-party API key, หรือ token ที่ต้องเปลี่ยนตามรอบ

### ไม่เหมาะกับงานแบบไหน

ไม่เหมาะกับ configuration ทั่วไปที่ไม่ sensitive ควรใช้ Parameter Store หรือ environment variable แทน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                       |
| -------------- | -------------------------------------------------- |
| AWS            | AWS Secrets Manager                                |
| GCP            | Secret Manager                                     |
| Azure          | Azure Key Vault (Secrets)                          |
| Huawei Cloud   | Data Encryption Workshop (DEW) - Secret Management |

### Spec / Configuration

| Spec / Configuration | ความหมาย                               | ตัวอย่าง                                        |
| -------------------- | -------------------------------------- | --------------------------------------------- |
| Secret Type          | ประเภทของ secret                       | Database credentials, API key, SSH key, Other |
| Secret Value         | ค่าของ secret (เข้ารหัสด้วย KMS)           | `{"username":"admin","password":"xxx"}`       |
| Automatic Rotation   | การ rotate secret อัตโนมัติ               | ทุก 30 วัน                                      |
| Rotation Lambda      | Lambda function ที่ทำ rotation            | สร้าง password ใหม่ใน DB แล้วอัปเดต secret        |
| Versioning           | เก็บหลาย version ของ secret             | AWSCURRENT, AWSPREVIOUS                       |
| Resource Policy      | กำหนด access จาก account หรือ service อื่น | cross-account access                          |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand — จ่ายตาม secret/month และ API call

|                          | AWS Secrets Manager       | GCP Secret Manager                | Azure Key Vault (Secrets) | Huawei DEW Secrets  |
| ------------------------ | ------------------------- | --------------------------------- | ------------------------- | ------------------- |
| Secret Storage           | $0.40/secret/month        | $0.06/secret version/month        | $0.03/10K transactions    | ~$0.10/secret/month |
| API Call                 | $0.05/10K calls           | $0.03/10K                         | รวมใน transactions        | ~$0.03/10K          |
| Cross-Region Replication | $0.40/secret/region/month | N/A                               | N/A                       | N/A                 |
| Free Tier                | —                         | 6 active versions + 10K ops/month | —                         | —                   |

> 100 secrets × AWS Secrets Manager ≈ $40/month (ไม่รวม API calls) — พิจารณาใช้ SSM Parameter Store (Standard = ฟรี) สำหรับ config ที่ไม่ sensitive

### ตัวอย่างการใช้งานใน Project

```python
# Application code - ดึง secret จาก Secrets Manager ตอน startup
import boto3
client = boto3.client('secretsmanager')
secret = client.get_secret_value(SecretId='prod/myapp/db-credentials')
```

### Best Practice

- เปิด automatic rotation สำหรับ database credential
- ให้ application permission เฉพาะ secret ที่จำเป็น
- monitor access log ผ่าน CloudTrail

### Common Mistakes

- เก็บ secret ใน environment variable หรือ source code แทน Secrets Manager
- ไม่ได้เปิด rotation ทำให้ secret ไม่เคยเปลี่ยน
- ให้ application access ทุก secret แทนที่จะ restrict เฉพาะที่จำเป็น
