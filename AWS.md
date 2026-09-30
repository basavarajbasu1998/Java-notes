# AWS

## Core services mapped to our project
| Need | AWS service | Analogy |
|---|---|---|
| Virtual servers | **EC2** | rent a computer |
| Run containers (managed) | **ECS / Fargate** (simple), **EKS** (Kubernetes) | |
| Store images | **ECR** | private Docker Hub |
| Relational DB | **RDS** (MySQL/Postgres, backups, Multi-AZ) / Aurora | managed DB |
| Cache | **ElastiCache** (Redis) | |
| Files/static React build | **S3** + **CloudFront** (CDN) | infinite hard disk + global delivery |
| Load balancer | **ALB** | traffic police |
| DNS | **Route 53** | phone book |
| Network | **VPC**, subnets (public/private), security groups, NAT | your private network |
| Identity | **IAM** (users, roles, policies) | who can do what |
| Queue / pub-sub | **SQS**, **SNS** (managed alternatives to RabbitMQ) | |
| Kafka managed | **MSK** | |
| Serverless functions | **Lambda** | run code without servers |
| Secrets | **Secrets Manager / Parameter Store** | |
| Monitoring/logs | **CloudWatch**, X-Ray | |
| IaC | **CloudFormation / Terraform** | infrastructure as code |
| CI/CD | CodePipeline/CodeBuild (or GitHub Actions/Jenkins) | |

## Production network layout
```
Internet ─► Route53 ─► CloudFront ─► S3 (React static files)
                    └► ALB (public subnet, HTTPS via ACM certificate)
                          │
             ┌────────────▼─────────────┐  PRIVATE subnets (no direct internet)
             │ EKS worker nodes / ECS    │──► RDS (Multi-AZ), ElastiCache, MSK
             └───────────────────────────┘
   Security groups = firewall per resource (ALB→app port 8081 only; app→DB port 3306 only)
   NAT gateway lets private subnets reach internet outward only. Two Availability Zones for HA.
```

## IAM essentials
- **Never** hard-code access keys. Give the pod/instance an **IAM Role** (EKS: IRSA / Pod Identity) with **least privilege**.
- Policy = JSON: Effect, Action, Resource.
```json
{ "Version": "2012-10-17", "Statement": [
  { "Effect": "Allow", "Action": ["s3:GetObject","s3:PutObject"], "Resource": "arn:aws:s3:::shop-invoices/*" } ] }
```
Use MFA, rotate credentials, separate accounts for dev/prod.

## Scaling & availability
- **Vertical** (bigger machine) vs **horizontal** (more machines) → prefer horizontal + stateless services.
- **Auto Scaling Group / HPA** based on CPU/requests; **Multi-AZ** for failover; **Read replicas** for read-heavy DB; **S3** 11-nines durability; **RPO/RTO** define disaster recovery; backups + tested restores.
- Cost: right-size, Reserved/Savings Plans, Spot for batch, S3 lifecycle rules, turn off dev at night.

## Deployment flow to AWS
```
git push → GitHub Actions → mvn test → docker build → push to ECR
        → kubectl/helm deploy to EKS (or ECS service update) → ALB health checks pass → live
        → CloudWatch alarms watch error rate/latency → auto-rollback or page on-call
```
S3 example (Spring): `S3Client.putObject(...)`; generate **pre-signed URL** so browser uploads directly to S3 without passing through your server. Lambda + API Gateway = serverless endpoint (cold start is the trade-off).
