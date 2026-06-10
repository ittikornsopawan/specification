# 13. Identity & Access Management (IAM)

IAM คือ Service กลุ่มที่จัดการ identity ของ user, service และ application และกำหนดว่าแต่ละ identity มีสิทธิ์ทำอะไรได้บ้างบน Cloud Resource เป็น first line of defense สำหรับ cloud security

---

## IAM (User, Role, Policy)

### คืออะไร

IAM (Identity and Access Management) คือบริการที่จัดการ identity และ permission บน Cloud ประกอบด้วย User (บุคคลหรือ service), Group (กลุ่ม user), Role (permission ชั่วคราวที่ assume ได้) และ Policy (เอกสารกำหนดสิทธิ์)

### ใช้งานแบบไหน

สร้าง Role สำหรับ application แต่ละตัว กำหนด Policy ที่อนุญาตเฉพาะ action และ resource ที่จำเป็น แทนการใช้ long-term access key

### เหมาะกับงานแบบไหน

ใช้กับทุก Cloud resource การออกแบบ IAM ที่ดีเป็นพื้นฐานของ security

### ไม่เหมาะกับงานแบบไหน

IAM ไม่ใช่ application-level authentication ไม่ควรใช้ IAM จัดการ end user ของ application ควรใช้ Cognito / Identity Platform แทน

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                  |
| -------------- | --------------------------------------------- |
| AWS            | AWS IAM                                       |
| GCP            | Cloud IAM                                     |
| Azure          | Azure Active Directory (Entra ID), Azure RBAC |
| Huawei Cloud   | Identity and Access Management (IAM)          |

### Spec / Configuration

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

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี — IAM ไม่มีค่าบริการ

| Feature                  | AWS IAM    | GCP Cloud IAM | Azure RBAC | Huawei IAM |
| ------------------------ | ---------- | ------------- | ---------- | ---------- |
| Users / Service Accounts | ฟรี         | ฟรี            | ฟรี         | ฟรี         |
| Roles / Policies         | ฟรี         | ฟรี            | ฟรี         | ฟรี         |
| MFA                      | ฟรี         | ฟรี            | ฟรี         | ฟรี         |
| Access Analyzer / Audit  | ฟรี (basic) | ฟรี            | ฟรี         | ฟรี         |

### ตัวอย่างการใช้งานใน Project

```
Lambda Function → IAM Role: lambda-api-role
    Policy: lambda-api-policy
    - Allow: s3:GetObject บน bucket: my-bucket
    - Allow: dynamodb:Query บน table: users
    - Allow: logs:CreateLogStream, logs:PutLogEvents
```

### Best Practice

- ใช้หลัก Least Privilege ให้สิทธิ์เฉพาะที่จำเป็นเท่านั้น
- ใช้ IAM Role แทน Access Key เสมอสำหรับ service บน Cloud
- ไม่ใช้ root account สำหรับ operation ปกติ
- เปิด MFA สำหรับ IAM user ที่มีสิทธิ์สูง
- review permission สม่ำเสมอและลบ unused access

### Common Mistakes

- ให้ AdministratorAccess กับ application โดยไม่จำเป็น
- เก็บ Access Key ใน source code หรือ environment variable แบบ plain text
- ไม่ rotate Access Key ที่เก่า

---

## Single Sign-On (SSO) / Identity Provider (IdP)

### คืออะไร

SSO (Single Sign-On) คือบริการที่ให้ user login ครั้งเดียวแล้วเข้าถึงได้หลาย application หรือ Cloud Console โดยใช้ Identity Provider กลาง รองรับ SAML 2.0, OIDC และ OAuth 2.0

### ใช้งานแบบไหน

ผูก Cloud account กับ Corporate Identity Provider เช่น Active Directory, Okta หรือ Azure AD ทำให้ developer login ด้วย corporate credential เดียวกับที่ใช้ login email ทำงาน

### เหมาะกับงานแบบไหน

เหมาะกับ enterprise ที่มีทีมขนาดใหญ่ ต้องการ centralize identity management และ audit trail

### ไม่เหมาะกับงานแบบไหน

อาจ overkill สำหรับ team เล็กมากที่ใช้ IAM user ปกติก็เพียงพอ

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                     |
| -------------- | -------------------------------- |
| AWS            | AWS IAM Identity Center (SSO)    |
| GCP            | Cloud Identity, Google Workspace |
| Azure          | Microsoft Entra ID (Azure AD)    |
| Huawei Cloud   | IAM Identity Provider            |

### Spec / Configuration

| Spec / Configuration    | ความหมาย                                      | ตัวอย่าง                                       |
| ----------------------- | --------------------------------------------- | -------------------------------------------- |
| Identity Provider (IdP) | ระบบที่ manage identity                         | Okta, Azure AD, Active Directory             |
| Protocol                | โปรโตคอลที่ใช้ federate identity                 | SAML 2.0, OIDC                               |
| Permission Set          | ชุด permission ที่ assign ให้ user ใน account หนึ่ง | ReadOnly, PowerUser, Admin                   |
| Attribute Mapping       | การ map attribute จาก IdP ไปยัง Cloud role     | group `cloud-admin` → Permission Set `Admin` |

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** ฟรี (basic SSO) / Subscription per user (advanced features)

| Feature                   | AWS IAM Identity Center | GCP Cloud Identity          | Azure Entra ID     | Huawei IAM IdP |
| ------------------------- | ----------------------- | --------------------------- | ------------------ | -------------- |
| Basic SSO + MFA           | ฟรี                      | ฟรี (Free edition)           | ฟรี (Free tier)     | ฟรี             |
| Conditional Access        | ฟรี                      | $6/user/month (Premium)     | $6/user/month (P1) | —              |
| Identity Protection + PIM | ฟรี                      | $12/user/month (Enterprise) | $9/user/month (P2) | —              |
| ตัวอย่าง 100 users, P1      | ฟรี                      | $600/month                  | $600/month         | —              |

### ตัวอย่างการใช้งานใน Project

Developer login ผ่าน AWS IAM Identity Center ด้วย corporate Okta credential โดยไม่ต้องมี IAM user ใน AWS account แต่ละตัว

### Best Practice

- ใช้ SSO แทน IAM User สำหรับ human access ทุกกรณีที่เป็นไปได้
- กำหนด Permission Set ตาม role ไม่ใช่ตามบุคคล
- เปิด MFA ที่ Identity Provider

### Common Mistakes

- สร้าง IAM User แยกในทุก account แทนที่จะใช้ SSO
- ให้ permission มากกว่าที่จำเป็นเพราะ "สะดวกกว่า"
