# 18. DevOps & CI/CD

DevOps และ CI/CD คือ Service กลุ่มที่ช่วย automate กระบวนการ build, test และ deploy code ทำให้ release cycle เร็วขึ้น ลด human error และรองรับ deployment หลาย environment

---

## CI/CD Pipeline

### คืออะไร

CI/CD Pipeline คือบริการที่ automate กระบวนการตั้งแต่ code push ไปจนถึง deploy production โดย Continuous Integration (CI) ทำ build และ test อัตโนมัติ และ Continuous Delivery/Deployment (CD) ทำ deploy อัตโนมัติ

### ใช้งานแบบไหน

กำหนด pipeline ที่ trigger เมื่อมี code push ไปยัง branch ที่กำหนด pipeline run: checkout code → build → unit test → integration test → build container image → push to registry → deploy to staging → smoke test → deploy to production

### เหมาะกับงานแบบไหน

เหมาะกับทุก project ที่ต้องการ deploy บ่อยและต้องการลด manual error ในกระบวนการ release

### ไม่เหมาะกับงานแบบไหน

ไม่มีกรณีที่ไม่ควรใช้ CI/CD แต่ซับซ้อนเกินจำเป็นสำหรับ project ที่ deploy น้อยมาก

### Service ตัวอย่างในแต่ละ Cloud Provider

| Cloud Provider | Service Name                                    |
| -------------- | ----------------------------------------------- |
| AWS            | AWS CodePipeline, AWS CodeBuild, AWS CodeDeploy |
| GCP            | Cloud Build, Cloud Deploy                       |
| Azure          | Azure DevOps (Pipelines), GitHub Actions        |
| Huawei Cloud   | CodeArts Pipeline                               |

### Spec / Configuration

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

### Pricing Model & ราคาโดยประมาณ

**Pricing Model:** On-demand (build minutes) + Subscription (active pipeline/parallel jobs)

|                     | AWS CodeBuild           | AWS CodePipeline                     | GCP Cloud Build | Azure Pipelines     | Huawei CodeArts    |
| ------------------- | ----------------------- | ------------------------------------ | --------------- | ------------------- | ------------------ |
| Build/minute        | $0.005 (general1.small) | —                                    | $0.003          | $0.008 (after free) | รวมใน plan         |
| Pipeline/month      | —                       | $1.00/active pipeline (after 1 free) | —               | ฟรี (1 parallel)     | รวมใน plan         |
| Free Tier           | 100 min/month           | 1 pipeline ฟรี                        | 120 min/day     | 1,800 min/month     | —                  |
| Extra parallel jobs | —                       | —                                    | —               | $40/extra parallel  | —                  |
| Subscription        | —                       | —                                    | —               | —                   | ~$15/month (basic) |

> ค่าใช้จ่าย CI/CD หลักมาจาก **build minutes** — optimize build time และ cache dependency ลดได้มาก GitHub Actions/GitLab CI อาจถูกกว่าสำหรับ small-medium team

### ตัวอย่างการใช้งานใน Project

```
GitHub push → CodePipeline trigger
  → Stage 1: CodeBuild - build + test
  → Stage 2: CodeBuild - build Docker image + push ECR
  → Stage 3: Deploy to EKS staging
  → Stage 4: Manual approval
  → Stage 5: Deploy to EKS production (Blue/Green)
```

### Best Practice

- ใช้ Infrastructure as Code (IaC) สำหรับ manage pipeline และ environment
- ทดสอบทุก stage ก่อน deploy production
- ใช้ Blue/Green deployment เพื่อ zero-downtime deploy
- เก็บ secret ใน Secrets Manager ไม่ใช่ใน pipeline config

### Common Mistakes

- ไม่มี rollback strategy ทำให้ revert ยากเมื่อมีปัญหา
- deploy production โดยตรงโดยไม่ผ่าน staging
- เก็บ credential ใน pipeline environment variable แบบ plain text
