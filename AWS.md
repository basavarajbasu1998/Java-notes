# AWS - Deep Interview Notes for a Java Backend / Full-Stack Developer

> Scope: what a 5-year Java developer must know to design, deploy, debug and defend a Spring Boot + React system on AWS.
> Prices and limits are **approximate** and change; always "check current docs / pricing page" before quoting in a real decision.
> Related notes: `Kubernetes.md` (EKS maps onto it), `Docker.md`, `Collections.md`, Spring/Kafka/RabbitMQ notes in this folder.

---

## Table of contents
1. 60-second mental model + analogy
2. Global infrastructure and shared responsibility
3. IAM in depth
4. VPC in depth
5. Compute: EC2, ASG, ELB
6. Containers: ECR, ECS, EKS, choosing
7. Serverless: Lambda, API Gateway, Step Functions
8. Storage: S3, EBS, EFS
9. Databases: RDS, Aurora, DynamoDB, ElastiCache
10. Messaging and integration
11. Edge networking: Route 53, CloudFront, WAF, ACM
12. Security services, KMS, secrets
13. Observability
14. IaC and delivery
15. Well-Architected, HA/DR, cost
16. Spring Boot on AWS
17. Reference architectures (a, b, c) with failure analysis and cost
18. Design answers: file upload, image pipeline, monolith migration
19. Production war stories
20. Interview questions (55) with follow-ups and wrong answers
21. One-page cheat sheet

---

## 1. 60-second mental model + analogy

AWS = a **city you rent**, where every building is an API call.

| City thing | AWS thing |
|---|---|
| Country / city / districts | Region / Availability Zone (AZ) |
| Your fenced estate | **VPC** |
| Gates, roads, one-way exit door | Internet Gateway, route tables, NAT Gateway |
| Building door lock per building | Security Group |
| Estate boundary checkpoint (stateless, numbered rules) | NACL |
| ID cards + job permissions | **IAM** (users, roles, policies) |
| Rented flats / rented offices with your logo on door | EC2 / Fargate |
| Automatic hiring/firing of staff by workload | Auto Scaling |
| Receptionist that routes visitors | ALB |
| Warehouse with unlimited shelves | S3 |
| Filing cabinet with an accountant | RDS (managed) |
| Post office (letters wait until picked up) | SQS |
| Notice board that copies a notice to many mailboxes | SNS |
| Delivery vans near customers | CloudFront |
| Phone book | Route 53 |
| CCTV + audit ledger | CloudWatch + CloudTrail |
| Safe deposit boxes | KMS / Secrets Manager |

The three questions behind every AWS design decision:
1. **Who is allowed?** (IAM: roles, not keys)
2. **Who can reach whom?** (VPC: subnets, SGs, endpoints)
3. **What happens when one piece dies?** (AZ spread, health checks, retries, idempotency, backups)

Golden defaults for a typical Spring Boot + React system:
- Static React on **S3 + CloudFront (OAC)**, API on **ECS Fargate (or EKS) behind ALB**, DB on **RDS/Aurora Multi-AZ**, cache on **ElastiCache**, async work through **SQS**, secrets in **Secrets Manager**, all in **private subnets across 2+ AZs**, provisioned via **Terraform/CloudFormation/CDK**, deployed by **GitHub Actions with OIDC (no static keys)**.

---

## 2. Global infrastructure and shared responsibility

### 2.1 Regions, AZs, edge
- **Region**: geographic area (e.g. `ap-south-1` Mumbai, `us-east-1` N. Virginia). Fully independent: separate IAM-agnostic resources (IAM itself is global), separate service endpoints, most services regional. Data does not leave a region unless you move it (compliance, data residency).
- **Availability Zone (AZ)**: one or more physically separate data centres inside a region with independent power/cooling/networking, connected by low-latency links. Region typically has 3+ AZs (check docs per region). AZ names (`ap-south-1a`) are **mapped per account**; use **AZ IDs** (`aps1-az1`) when coordinating across accounts.
- **Edge locations / Regional edge caches / Points of Presence**: used by CloudFront, Route 53, WAF, Global Accelerator, Shield. Hundreds of PoPs; not places where you run EC2.
- **Local Zones / Wavelength / Outposts**: extensions for ultra-low latency or on-prem; rarely asked beyond a definition.
- Choosing a Region: (1) latency to users, (2) compliance/data residency, (3) service availability, (4) price (varies by region), (5) DR pair.
- Global vs regional services: **Global**: IAM, Route 53, CloudFront, WAF (for CloudFront scope), Organizations. **Regional**: nearly everything else (EC2, S3 buckets live in a region though names are global, RDS, Lambda, SQS...).

### 2.2 Shared responsibility model
```
 AWS: security OF the cloud            YOU: security IN the cloud
 -------------------------------       -----------------------------------------
 Data centres, hardware, hypervisor    Your data, classification, encryption choices
 Network infrastructure                IAM: users, roles, policies, MFA
 Managed service internals             Security groups, NACLs, VPC design
 (RDS engine patching windows*)        OS patching on EC2, app code, dependencies
                                       Client-side & server-side encryption config
                                       Public/private decisions (S3 bucket policy!)
```
- Responsibility shifts with the service abstraction: **EC2** (you patch OS) > **RDS** (AWS patches engine in maintenance window, you manage schema/users/params/access) > **Lambda/S3** (AWS manages the platform, you manage code, IAM, data config).
- Interview line: "AWS secures the cloud; I secure what I put in it. Every S3 leak is a customer-side misconfiguration."

---

## 3. IAM in depth

### 3.1 Building blocks
| Concept | Meaning | Note |
|---|---|---|
| **Root user** | The account owner (email login) | Enable MFA, no access keys, use only for the few root-only tasks |
| **IAM user** | Long-lived identity with password and/or access keys | Avoid for humans (use IAM Identity Center / SSO) and for workloads (use roles) |
| **Group** | Collection of users; attach policies to group | Groups cannot be principals in policies and cannot be nested |
| **Role** | Identity with **no credentials of its own**; assumed by a trusted principal, giving **temporary credentials** via STS | Used by EC2, ECS tasks, Lambda, pods, cross-account, CI/CD |
| **Policy** | JSON document of permissions | Identity-based, resource-based, boundary, SCP, session, VPC endpoint policy |

### 3.2 Two kinds of policy on a role: trust vs permission
A role has **two independent questions**:
1. **Trust policy (resource-based policy on the role)**: *Who may assume me?* (`Principal`, action `sts:AssumeRole`).
2. **Permission policy (identity-based)**: *What can I do once assumed?*

Trust policy for an ECS task role:
```json
{ "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ecs-tasks.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }] }
```
Common mistake: fixing a permissions problem by editing the trust policy (or vice versa). `AccessDenied ... not authorized to perform sts:AssumeRole` => trust policy (or caller lacks `sts:AssumeRole`). `AccessDenied ... s3:GetObject` => permission policy.

### 3.3 Policy types
| Type | Attached to | Purpose |
|---|---|---|
| Identity-based | user/group/role | "This identity can do X on Y" (managed AWS / customer / inline) |
| Resource-based | S3 bucket, SQS queue, KMS key, SNS topic, Lambda, role trust... | "This resource allows principal P to do X"; has `Principal` element |
| Permissions boundary | user/role | **Maximum** an identity-based policy can grant; grants nothing by itself |
| SCP (Organizations) | OU/account | **Maximum** for all principals in the account (except the management account); grants nothing by itself |
| Session policy | passed at AssumeRole | Further narrows one session |
| VPC endpoint policy | VPC endpoint | Limits what can pass through that endpoint |
| RCP (resource control policies) | Organizations | Newer org-level guardrail on resources; check docs |

### 3.4 Policy evaluation logic (memorise)
```
Request
  1. DEFAULT = implicit deny
  2. Any explicit Deny anywhere (identity, resource, boundary, SCP, session, endpoint)?  ---> DENY (final)
  3. Organizations SCPs: is it allowed by SCP (if account is in an org)?  no ---> DENY
  4. Resource-based policy allows?
        SAME account: allow here can be enough by itself (for user/role ARN principals)
  5. Identity-based policy allows? AND permissions boundary allows? AND session policy allows?
  6. Otherwise ---> implicit DENY
CROSS-ACCOUNT: needs allow in BOTH the identity policy of the caller AND the resource policy of the target
              (or the caller assumes a role in the target account, then it is a same-account call)
```
Short form: **explicit deny beats everything; then every guardrail layer (SCP, boundary, session) must permit; then something must actually Allow.**

Gotchas:
- SCP / boundary never grant: `SCP allows s3:*` but no identity policy => still denied.
- Same-account resource policy naming a specific role/user ARN can allow without an identity policy. If the principal is `"AWS": "arn:aws:iam::111122223333:root"` the policy only delegates to the account's IAM (identity policy still needed).
- S3 bucket policy with `Principal: "*"` is what makes a bucket public (unless Block Public Access is on).

### 3.5 Anatomy of a policy, line by line
```json
{
  "Version": "2012-10-17",                                  // policy language version; always this value
  "Statement": [
    {
      "Sid": "ReadWriteInvoices",                           // optional label
      "Effect": "Allow",                                    // Allow | Deny
      "Action": ["s3:GetObject", "s3:PutObject"],           // API operations (wildcards allowed, avoid s3:*)
      "Resource": "arn:aws:s3:::shop-invoices/tenant-*/*",  // OBJECT ARN: bucket + /key pattern
      "Condition": {
        "StringEquals": { "s3:x-amz-server-side-encryption": "aws:kms" },   // must upload with SSE-KMS
        "Bool": { "aws:SecureTransport": "true" },                           // HTTPS only
        "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }                    // only from office range
      }
    },
    {
      "Sid": "ListOnlyOwnPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::shop-invoices",             // BUCKET ARN (no /*) - ListBucket is a bucket-level action
      "Condition": { "StringLike": { "s3:prefix": ["tenant-42/*"] } }
    },
    {
      "Sid": "DenyUnencryptedTransport",
      "Effect": "Deny",                                     // explicit deny wins over any Allow elsewhere
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::shop-invoices", "arn:aws:s3:::shop-invoices/*"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    }
  ]
}
```
(Real JSON has no comments; comments are for explanation.) Key learning: **`s3:ListBucket` needs the bucket ARN, `s3:GetObject` needs the `/*` object ARN** - classic cause of AccessDenied.

Useful condition keys:
| Key | Use |
|---|---|
| `aws:SourceIp`, `aws:SourceVpce`, `aws:SourceVpc` | restrict to IP / VPC endpoint |
| `aws:PrincipalOrgID` | any principal from my Organization (better than listing accounts) |
| `aws:SecureTransport` | force TLS |
| `aws:MultiFactorAuthPresent` | require MFA |
| `aws:RequestedRegion` | region lock in an SCP |
| `aws:PrincipalTag` / `aws:ResourceTag` | attribute-based access control (ABAC) |
| `aws:SourceArn`, `aws:SourceAccount` | prevent the **confused deputy** problem (e.g. SNS -> SQS, CloudTrail -> S3) |
| `sts:ExternalId` | third-party cross-account role protection |
| `s3:x-amz-server-side-encryption` | enforce encryption on upload |

### 3.6 STS and cross-account role assumption
```
Account A (dev, 111111111111)                     Account B (prod, 222222222222)
  Alice (identity policy: sts:AssumeRole on B-role)   Role ReadOnlyProd
        |                                              trust policy: Principal = arn:aws:iam::111111111111:root
        | 1. sts:AssumeRole(RoleArn=ReadOnlyProd)         (+ optional MFA / ExternalId condition)
        |------------------------------------------>   permission policy: read-only
        | 2. STS checks trust policy AND Alice's allow
        |<------------------------------------------   3. temp creds (AccessKeyId, SecretAccessKey, SessionToken, expiry)
        | 4. call AWS APIs in B using temp creds; CloudTrail logs the role session name
```
Both sides needed: **A's identity policy must allow `sts:AssumeRole`; B's trust policy must trust A.** Default session 1 hour (role max session duration configurable; role chaining limited to 1 hour).

Trust policy with MFA and org condition:
```json
{ "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::111111111111:root" },
  "Action": "sts:AssumeRole",
  "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } } }
```

### 3.7 How workloads get credentials
| Workload | Mechanism |
|---|---|
| EC2 | **Instance profile** (container for one role) -> creds served by IMDS (`169.254.169.254`); enforce **IMDSv2** (token based) to blunt SSRF theft |
| ECS task | **Task role** (app's AWS calls) via container credentials endpoint; **execution role** is separate (see 6.3) |
| Lambda | **Execution role** injected as env vars |
| EKS pod | **IRSA** (OIDC federation) or **EKS Pod Identity** (newer, simpler) |
| CI/CD (GitHub Actions) | **OIDC federation** -> `AssumeRoleWithWebIdentity`, no stored keys (see 14.5) |
| Laptop | IAM Identity Center (SSO) short-lived creds; avoid long-lived access keys |

**IRSA flow**
```
1. Cluster has an OIDC issuer URL; IAM has an OIDC identity provider for it.
2. ServiceAccount annotated: eks.amazonaws.com/role-arn: arn:aws:iam::ACCT:role/orders-role
3. Pod webhook injects a projected service-account JWT + env AWS_WEB_IDENTITY_TOKEN_FILE, AWS_ROLE_ARN
4. SDK calls sts:AssumeRoleWithWebIdentity(JWT) -> temp creds
5. Role trust policy: Principal Federated=OIDC provider, Condition sub == system:serviceaccount:<ns>:<sa-name>
```
```json
{ "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLE" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": { "StringEquals": {
     "oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:shop:orders-sa",
     "oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLE:aud": "sts.amazonaws.com" } } }
```
**EKS Pod Identity**: install the `eks-pod-identity-agent` add-on, create a *pod identity association* (cluster + namespace + service account -> role). Role trust principal is `pods.eks.amazonaws.com` with actions `sts:AssumeRole` and `sts:TagSession`. No per-cluster OIDC provider in the trust policy, so **the same role is reusable across clusters**; credentials delivered through the container credentials endpoint (this is why "container creds" appear in the SDK chain). If the sub condition in IRSA is missing/wrong, any service account in the cluster may assume the role - a classic finding.

### 3.8 Access keys vs roles; least privilege; MFA
- Access keys (AKIA...) are long-lived secrets: leak via Git, logs, AMIs. Roles give **temporary** creds (ASIA... + session token) that rotate automatically. Rule: **workloads use roles; humans use SSO + MFA; keys only when unavoidable, rotated, and scoped**.
- Least privilege practice: start from need, use **IAM Access Analyzer** policy generation from CloudTrail, review "last accessed" data, use conditions and resource ARNs, avoid `*:*`, separate roles per service.
- MFA: root and all humans; enforce with `aws:MultiFactorAuthPresent` on sensitive actions; prefer phishing-resistant (passkey/FIDO).

### 3.9 Organizations and SCPs
- **AWS Organizations**: hierarchy of accounts in OUs. Multi-account strategy (prod/non-prod/security/log-archive/shared-services) is the strongest blast-radius control.
- **SCP** = guardrail. Example: deny leaving allowed regions and deny disabling CloudTrail.
```json
{ "Version": "2012-10-17", "Statement": [
  { "Sid": "RegionLock", "Effect": "Deny",
    "NotAction": ["iam:*","organizations:*","route53:*","cloudfront:*","support:*","sts:*"],
    "Resource": "*",
    "Condition": { "StringNotEquals": { "aws:RequestedRegion": ["ap-south-1","ap-southeast-1"] } } },
  { "Sid": "ProtectCloudTrail", "Effect": "Deny",
    "Action": ["cloudtrail:StopLogging","cloudtrail:DeleteTrail"], "Resource": "*" } ] }
```
- SCPs do not apply to the management account; they do apply to the root user of member accounts; they do not grant anything.

### 3.10 Debugging a 403 AccessDenied (checklist)
1. **Read the error**: which principal ARN, which action, which resource. Newer errors say "explicit deny in a service control policy / identity-based policy / resource-based policy" vs "no identity-based policy allows" - that alone tells you which layer.
2. **Who am I really?** `aws sts get-caller-identity` (locally) or the app's assumed-role ARN. Wrong role / wrong profile / expired session is the #1 cause.
3. **Identity policy**: does the action + **exact resource ARN** match (`/*` for objects vs bucket ARN; region/account in ARN)?
4. **Explicit Deny hunt**: SCP, permissions boundary, session policy, resource policy Deny, VPC endpoint policy.
5. **Resource policy**: bucket policy, KMS key policy (KMS is special: key policy must allow, IAM alone is not enough unless key policy delegates to IAM), SQS/SNS policy.
6. **Conditions**: IP, VPC endpoint, MFA, tags, encryption header, `aws:SourceArn`.
7. **KMS**: S3/SQS/Secrets with a CMK need `kms:Decrypt`/`GenerateDataKey` too - S3 403 that "looks like S3" may be KMS.
8. **Cross-account**: both sides allow? Object ownership (ACL) issues on uploaded objects? Role trust policy?
9. **Tools**: **CloudTrail** event (`errorCode`, `errorMessage`), **IAM Policy Simulator**, **IAM Access Analyzer**, `aws iam ... get-policy` review, decode authorization message with `aws sts decode-authorization-message`.
10. Remember eventual consistency of IAM changes: seconds, occasionally longer.
11. S3 nuance: 403 vs 404: without `s3:ListBucket` a missing object returns **403** not 404.

---

## 4. VPC in depth

### 4.1 Concepts
- **VPC**: your isolated virtual network in one Region, defined by an IPv4 CIDR (allowed size /16 to /28) plus optional IPv6.
- **Subnet**: a slice of the VPC CIDR **living in exactly one AZ**. AWS reserves **5 IPs per subnet** (network, router, DNS, future, broadcast).
- **Public subnet** = subnet whose route table has `0.0.0.0/0 -> Internet Gateway` (and instances have a public/Elastic IP). **Private subnet** = no route to IGW; outbound via NAT Gateway if needed. "Public/private" is only a routing property.
- **Route table**: per subnet association; most specific prefix wins; `local` route always exists for intra-VPC.
- **Internet Gateway (IGW)**: horizontally scaled, HA, free; two-way internet for resources with public IPs.
- **NAT Gateway**: managed, lives **in a public subnet**, gives private subnets **outbound-only** internet. Zonal: one per AZ for HA.
  - Cost: hourly charge per gateway **plus per-GB processed** (approx. a few cents each; check pricing) - this is the classic bill shock: S3/ECR/DynamoDB traffic through NAT is paid per GB. Fix with **VPC endpoints**.
  - If AZ-a NAT dies and private subnet in AZ-b routes to it (cross-AZ NAT), AZ-b loses egress. Design: **one NAT per AZ, each private subnet routes to its own AZ's NAT** (also avoids cross-AZ data charge).
- **Egress-only IGW**: NAT equivalent for IPv6.
- **DNS**: enable `enableDnsSupport` and `enableDnsHostnames` (needed for interface endpoints private DNS).

### 4.2 CIDR planning
- Plan for growth and **no overlaps** with other VPCs, on-prem, partners (peering/TGW/VPN cannot route overlapping CIDRs).
- Give each environment a distinct /16: prod `10.0.0.0/16`, staging `10.1.0.0/16`, dev `10.2.0.0/16`.
- EKS with the VPC CNI uses **a VPC IP per pod**: /24 subnets exhaust quickly. Use larger private subnets (/19-/20) or prefix delegation/custom networking.
- Lambda in VPC, RDS Proxy, interface endpoints also consume IPs (ENIs).

### 4.3 Security groups vs NACLs
| | Security Group | NACL |
|---|---|---|
| Level | ENI / resource | Subnet |
| State | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow ephemeral return ports 1024-65535) |
| Rules | **Allow only**; all rules evaluated together | Allow and Deny; **numbered, evaluated in order, first match wins**; final `*` deny |
| Source | CIDR, prefix list, **another security group** | CIDR only |
| Default | inbound denied, outbound all allowed | default NACL allows all |
| Use | primary firewall | coarse guardrail, block a bad CIDR |

Best practice: reference SGs, not IPs: `alb-sg` -> `app-sg:8080` -> `db-sg:5432`. The DB SG allows inbound only from `app-sg`.
Trap: NACL - forgetting ephemeral ports for return traffic. SG: removing a rule does not necessarily kill already-tracked connections immediately in all cases; do not rely on that.

### 4.4 VPC endpoints (keep traffic off the internet/NAT)
| Type | Services | How | Cost |
|---|---|---|---|
| **Gateway endpoint** | **S3, DynamoDB** only | A **route table entry** (prefix list `pl-xxxx`) to the endpoint; no ENI | **Free** |
| **Interface endpoint (PrivateLink)** | Most services (SQS, SNS, Secrets Manager, ECR, CloudWatch Logs, STS, KMS...) and your own/partner services | ENIs with private IPs in your subnets + private DNS name | Hourly per AZ + per GB |
| Gateway Load Balancer endpoint | Inspection appliances | | |

Use endpoint policies to restrict (e.g. only my bucket). For Fargate in private subnets pulling from ECR you need: `ecr.api`, `ecr.dkr` interface endpoints + `s3` gateway endpoint (image layers live in S3) + `logs` endpoint if using awslogs - or a NAT.

### 4.5 Connecting VPCs
| Option | Notes |
|---|---|
| **VPC Peering** | 1:1, non-transitive, no overlapping CIDR, cheap, no bandwidth bottleneck; n VPCs => n(n-1)/2 links; need route entries on both sides |
| **Transit Gateway (TGW)** | Hub-and-spoke, transitive routing, attach VPCs/VPN/Direct Connect, route tables, per-attachment + per-GB fee; use beyond a handful of VPCs |
| **PrivateLink** | Expose **one service** (not the whole network) privately to consumers; overlapping CIDRs OK; one-directional |
| Site-to-Site VPN / Direct Connect | On-prem connectivity (VPN quick, encrypted over internet; DX dedicated, consistent latency) |

### 4.6 Bastion vs SSM Session Manager
- Bastion host: public EC2 with SSH open; keys to manage, port 22 exposure, patching, audit gaps.
- **SSM Session Manager**: no inbound ports, no keys; IAM-authorised, sessions logged to CloudTrail/S3/CloudWatch; instance needs SSM Agent + instance profile with `AmazonSSMManagedInstanceCore` and a path to SSM (NAT or 3 interface endpoints: `ssm`, `ssmmessages`, `ec2messages`). Port forwarding to RDS uses document `AWS-StartPortForwardingSessionToRemoteHost`. Prefer SSM.

### 4.7 Standard 3-tier, 2-AZ layout
```
Region ap-south-1        VPC 10.0.0.0/16  (DNS hostnames on)
+-----------------------------------------------------------------------------------------+
|        AZ-a                                            AZ-b                             |
|  PUBLIC   10.0.0.0/24  (IGW route)                PUBLIC   10.0.1.0/24 (IGW route)      |
|     [ALB node]   [NAT-GW-a + EIP]                    [ALB node]   [NAT-GW-b + EIP]      |
|        |                |                                |               |              |
|  PRIVATE-APP 10.0.10.0/24 (0.0.0.0/0 -> NAT-a)    PRIVATE-APP 10.0.11.0/24 (-> NAT-b)   |
|     [ECS tasks / EKS nodes / EC2 ASG]                [ECS tasks / EKS nodes / EC2 ASG]  |
|        |                                                     |                          |
|  PRIVATE-DATA 10.0.20.0/24 (no internet route)    PRIVATE-DATA 10.0.21.0/24 (no route)  |
|     [RDS primary] <==== sync replication ====>       [RDS standby]   [ElastiCache node] |
+-----------------------------------------------------------------------------------------+
Gateway endpoints: S3, DynamoDB (in app+data route tables)   Interface endpoints: ecr, logs, secretsmanager, sqs, ssm
SGs:  alb-sg (443 from 0.0.0.0/0) -> app-sg (8080 from alb-sg) -> db-sg (5432 from app-sg), cache-sg (6379 from app-sg)
Route tables: public-rt (0/0 -> IGW) | private-app-rt-a (0/0 -> NAT-a) | private-app-rt-b (0/0 -> NAT-b) | data-rt (local only)
```
Cost-saving variation for dev: one NAT total (accept AZ SPOF), or endpoints only.

---

## 5. Compute: EC2, Auto Scaling, ELB

### 5.1 EC2 basics
- **Instance families** (letter = purpose): `t` burstable (CPU credits; exhausted => throttled), `m` general, `c` compute-optimised, `r`/`x` memory-optimised (Java heaps, caches), `i`/`d` storage (local NVMe), `g`/`p` GPU (note: suffix `g` on `m7g` means Graviton/ARM; better price/performance often; need arm64 images and arm builds of native libs, pure Java is portable), `a` = AMD, `n` = network, `d` = local disk.
- **AMI**: image (OS + config) per region; copyable. Golden AMI via Packer/EC2 Image Builder vs bootstrap via **user data** (script run at first boot as root; keep it idempotent and short; logs at `/var/log/cloud-init-output.log`).
- **Storage**: EBS (network block, persists), instance store (local, ephemeral, lost on stop/terminate/hardware failure). See section 8.
- **Metadata**: require IMDSv2; hop limit 1 blocks containers from reaching it unless raised.

### 5.2 Pricing models
| Model | Commitment | Discount (approx.) | Use |
|---|---|---|---|
| **On-Demand** | none | baseline | spiky/unknown, new workloads |
| **Savings Plans** (Compute / EC2 Instance) | $/hour for 1 or 3 yrs | up to ~60-70% depending | steady baseline; Compute SP also covers Fargate and Lambda |
| **Reserved Instances** | 1/3 yr specific config | similar to SP | legacy for EC2; still used for RDS/ElastiCache reserved nodes |
| **Spot** | none; can be reclaimed with **2-minute notice** | up to ~90% | stateless, fault-tolerant, batch, CI runners, workers; diversify instance types/AZs |
| Dedicated Host/Instance | | | licensing/compliance |

Strategy: **baseline with Savings Plan, burst on-demand, batch on Spot**.

### 5.3 Auto Scaling Group (ASG)
```
Launch template (AMI, type, SGs, IAM profile, user data)  ->  ASG (min/desired/max, subnets in >=2 AZs, health checks)
       ^                                                          |
       +-- scaling policies (target tracking / step / scheduled) -+--> registers instances in ALB target group
```
- **Launch template** (versioned; replaces legacy launch configurations, which are deprecated).
- **Scaling policies**:
  - **Target tracking**: "keep average CPU at 50%" or "ALBRequestCountPerTarget = 1000". AWS creates/manages alarms. Default choice.
  - **Step scaling**: different adjustments per alarm breach size.
  - **Scheduled**: known peaks (9 am sale).
  - **Predictive scaling**: forecast from history.
  - **Warm pools** shorten boot time. **Instance refresh** for rolling AMI updates.
  - Instance warm-up / cooldown prevents over-scaling while new instances boot.
- **Health checks**: EC2 (instance status) vs **ELB** (target group health). Enable ELB health checks on the ASG so app-unhealthy instances are replaced. **Health check grace period** = time to ignore failures after launch; must exceed app boot time or the ASG kills instances before they are ready => flapping (war story 19.7).
- **Lifecycle hooks**: pause at `Pending:Wait` or `Terminating:Wait` to run custom actions (drain, ship logs, register with config mgmt), continue via API or timeout.
- **Termination policy**: default balances AZs first; scale-in protection available.
- On AZ failure ASG launches replacements in remaining AZs (capacity headroom needed).

### 5.4 Elastic Load Balancing
| | **ALB** | **NLB** | **GLB** |
|---|---|---|---|
| Layer | 7 (HTTP/HTTPS/gRPC/WebSocket) | 4 (TCP/UDP/TLS) | 3 (GENEVE) |
| Features | host/path/header/query routing, redirects, fixed responses, OIDC/Cognito auth, WAF, sticky sessions, Lambda targets | ultra-low latency, very high throughput, **static/Elastic IP per AZ**, preserves client IP, PrivateLink provider | insert third-party firewalls/IDS |
| Target types | instance, **ip**, lambda | instance, ip, ALB | instance, ip |
| Cross-zone | **On by default** (no inter-AZ charge for ALB) | **Off by default**; enabling incurs inter-AZ data cost | off |
| Security groups | yes | supported (added 2023) | no |
(Classic LB is legacy; do not choose it.)

**ALB concepts**
- **Listener** (port/protocol + ACM certificate) -> **rules** (priority, conditions) -> action (forward to **target group**, redirect, fixed-response).
- **Target group**: set of targets + protocol/port + **health check** (path e.g. `/actuator/health/readiness`, interval, healthy/unhealthy thresholds, success codes, timeout).
- Routing algorithms: round robin (default), **least outstanding requests** (uneven request cost).
- **Sticky sessions**: cookie-based; use only when state is unavoidable - better: stateless service + Redis session store.
- **Deregistration delay** (default 300 s): on scale-in/deploy the target goes `draining`: no new requests, in-flight finish. Set slightly above your p99 request time (e.g. 30-60 s) or deployments are needlessly slow; too low => cut requests (5xx).
- **Idle timeout** default 60 s. **The app's keep-alive timeout must be greater than the ALB idle timeout**, otherwise the app closes a connection the ALB believes is open -> intermittent **502**. Spring Boot Tomcat: `server.tomcat.keep-alive-timeout=75s`. **504** = ALB waited longer than idle timeout (or could not get an answer): fix slow endpoint or raise timeouts consistently.
- Headers `X-Forwarded-For`, `X-Forwarded-Proto` (Spring: `server.forward-headers-strategy=framework` or `native`), access logs to S3.
- ALB in **public subnets in >=2 AZs**; targets in private subnets; target SG allows only ALB SG.

**Routing example**: `api.shop.com/orders/*` -> `orders-tg`; `api.shop.com/payments/*` -> `payments-tg`; default -> fixed 404.

---

## 6. Containers

### 6.1 ECR
Private registry (regional). Image scanning (basic / enhanced with Inspector), lifecycle policies (expire untagged/old images - saves cost), cross-region/account replication, **immutable tags** (so `:1.4.2` cannot be silently overwritten), IAM-based pull. Push: `aws ecr get-login-password | docker login ...`, `docker push`. In CI use the OIDC role, not keys.

### 6.2 ECS
```
Cluster
  Service (desired count, deployment config, LB target group, autoscaling)
     Task (1..n containers scheduled together; own ENI in awsvpc mode)
        <- Task Definition (versioned blueprint: image, cpu/mem, ports, env, secrets, log config, roles)
Capacity: FARGATE (serverless) | EC2 (you manage ASG via capacity provider)
```
- **Task definition**: image, `cpu`/`memory`, port mappings, environment, `secrets` (pulled from Secrets Manager/SSM at task start), `logConfiguration` (awslogs), container `healthCheck`, `taskRoleArn`, `executionRoleArn`, `networkMode: awsvpc` (own ENI + SG; required for Fargate).
- **Service**: keeps N tasks running; ALB integration (target type `ip`); rolling deployment via `minimumHealthyPercent`/`maximumPercent`; **deployment circuit breaker** (auto-rollback on failing deployment); **health check grace period** (ignore ALB health checks while Spring boots); service auto scaling. Blue/green via CodeDeploy (and native ECS blue/green - check current docs).
- **Fargate vs EC2**:

  | | Fargate | ECS on EC2 |
  |---|---|---|
  | Servers | none | you manage instances, AMI, patching, capacity |
  | Pricing | per vCPU-second + GB-second; approx. higher per unit | instance price; better bin-packing at scale; Spot/SP savings |
  | Limits | no privileged containers, no GPU (check docs), limited host access | full control |
  | Startup | tens of seconds (image pull) | faster on warm hosts |

  Default: **Fargate unless there is a concrete reason** (GPU, big steady scale cost, host-level agents).

### 6.3 Task role vs execution role (favourite question)
| | **Task role** | **Task execution role** |
|---|---|---|
| Used by | **Your application code** (SDK calls) | **ECS agent/Fargate** on your behalf, before/around your app |
| Typical permissions | s3:PutObject, sqs:SendMessage, dynamodb:*, secretsmanager:GetSecretValue (if the app fetches at runtime) | `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `logs:CreateLogStream/PutLogEvents`, `secretsmanager:GetSecretValue` / `ssm:GetParameters` for `secrets:` injected as env vars, `kms:Decrypt` |
| Trust principal | `ecs-tasks.amazonaws.com` | `ecs-tasks.amazonaws.com` |

Symptom mix-up: "CannotPullContainerError ... unauthorized" or "ResourceInitializationError: unable to pull secrets" = **execution role** (or missing network path to ECR/Secrets Manager). "AccessDenied from S3 inside the app" = **task role**.

### 6.4 EKS overview and mapping to Kubernetes.md
- EKS = **managed Kubernetes control plane** (API server, etcd across multiple AZs; hourly fee per cluster - check pricing). Data plane options: **managed node groups** (EC2 ASG managed by EKS), **Karpenter** (fast right-sized node provisioning, Spot friendly), **Fargate profiles** (pod per microVM; no DaemonSets).
- Mapping: `Deployment/Pod/Service/Ingress/HPA/ConfigMap/Secret` (see `Kubernetes.md`) are unchanged. AWS glue:

  | K8s concept | AWS glue |
  |---|---|
  | Ingress / `Service type=LoadBalancer` | **AWS Load Balancer Controller** creates ALB (Ingress) / NLB (Service); target type `ip` sends traffic straight to pod IPs |
  | Pod networking | **VPC CNI**: pods get VPC IPs |
  | PersistentVolume | EBS CSI (RWO), EFS CSI (RWX) |
  | Node scaling | Cluster Autoscaler or **Karpenter**; pods via HPA/KEDA |
  | Secrets | External Secrets Operator / Secrets Store CSI driver (Secrets Manager) |
  | Pod AWS permissions | IRSA / Pod Identity |
  | Logs/metrics | Fluent Bit -> CloudWatch, Container Insights, ADOT, managed Prometheus |
  | RBAC | IAM -> Kubernetes via EKS access entries (legacy: `aws-auth` ConfigMap) |
- You own upgrades of add-ons and nodes; minor versions come roughly every few months and older versions leave standard support (extended support costs extra - check docs).

### 6.5 Choosing ECS vs EKS vs Lambda vs Beanstalk
| Need | Pick | Why |
|---|---|---|
| Small team, containers, AWS-only, want simplicity | **ECS Fargate** | least ops, deep AWS integration |
| Existing K8s skills, portability, ecosystem (Helm, operators, mesh), many teams/services | **EKS** | standard API; higher ops and cost |
| Spiky/event-driven, short tasks, pay per request | **Lambda** | scale to zero |
| Lift-and-shift Spring Boot jar, want managed ALB+ASG | **Elastic Beanstalk** (or App Runner for simple container web apps) | PaaS convenience; less flexible |
| Long-running batch | **AWS Batch** / ECS tasks / Step Functions | |

Interview stance: pick EKS for a stated need, not prestige. "K8s tax" = control plane fee + node management + upgrades + learning curve.

---

## 7. Serverless

### 7.1 Lambda execution model
```
Event (API GW / SQS / S3 / EventBridge / schedule / SNS / Kinesis / DynamoDB stream)
   -> Lambda service finds a warm execution environment ("sandbox")
       - none free? COLD START: create microVM -> download code -> start JVM/runtime -> run static init -> handler
       - warm? just invoke handler
   -> one request at a time per environment (concurrency = number of live environments)
   -> environment frozen after response; reused later (static fields persist!)
```
- **Memory 128 MB - 10,240 MB**; **CPU scales with memory** (about 1 vCPU at ~1,769 MB). More memory often makes CPU-bound Java *cheaper* because it finishes faster. Tune with AWS Lambda Power Tuning.
- **Timeout up to 15 minutes**. API Gateway REST integration timeout defaults to ~29 s (adjustable for some configurations - check docs); ALB idle timeout 60 s. Set the Lambda timeout below the caller's.
- **Concurrency**: default account limit approx. 1,000 concurrent executions per region (quota raisable); **reserved concurrency** (guarantee and cap per function), **provisioned concurrency** (pre-initialised, no cold start, billed while provisioned). Per-function scale-out rate is limited (check docs). Throttling => 429 (sync) / retry (async, event-source specific).
- Payload limits approx.: 6 MB sync request/response; async payload smaller (recently raised - check docs). `/tmp` 512 MB default, configurable up to 10 GB. Zip package 50 MB zipped / 250 MB unzipped, or **container image up to 10 GB**.
- Invocation types: **sync** (API GW, ALB), **async** (S3, SNS, EventBridge; 2 automatic retries, then DLQ/on-failure destination), **poll-based event source mapping** (SQS, Kinesis, DynamoDB streams: Lambda polls and batches; use `ReportBatchItemFailures`).
- **Idempotency**: at-least-once (SQS, async retries) => make handlers idempotent (idempotency key, conditional write, Powertools for AWS Lambda (Java) idempotency module backed by DynamoDB).
- **DLQ**: for an SQS event source the DLQ belongs to the **queue** (redrive policy), not the function; for async invokes use the function's DLQ/on-failure destination.

### 7.2 Java cold starts and mitigations
Java cold start = JVM boot + class loading + Spring context + SDK client creation (seconds for full Spring Boot).

| Mitigation | How | Trade-off |
|---|---|---|
| **SnapStart** (Java 11+ managed runtimes) | Snapshot after init, restore on cold start | Big reduction (often sub-second); beware uniqueness (random seeds, open connections, cached time) - use **CRaC hooks** (`beforeCheckpoint/afterRestore`); works on published versions/aliases, not `$LATEST`; check regions/pricing |
| **Provisioned concurrency** | Keep N warm | Pay always; for latency SLAs; can schedule/autoscale |
| **GraalVM native image / custom runtime (`provided.al2023`)** | AOT, ms startup | Build complexity, reflection config |
| Lighter framework | Micronaut/Quarkus/Spring Cloud Function; avoid classpath scanning | Rewrite |
| Trim init | Lazy init, fewer deps, URLConnection/CRT HTTP client rather than Apache, `-XX:+TieredCompilation -XX:TieredStopAtLevel=1` via `JAVA_TOOL_OPTIONS` | Lower peak perf |
| More memory | more CPU during init | Cost |

Initialise SDK clients and DB pools **outside the handler** (static/init) to reuse across invocations.

### 7.3 Lambda in a VPC
- Needed only to reach private resources (RDS, ElastiCache). Uses shared Hyperplane ENIs (created at config time), so VPC cold-start penalty is now small.
- **No internet from a private subnet without NAT or VPC endpoints.** Putting a Lambda in a *public* subnet does **not** give internet access (no public IP on Lambda ENIs).
- **Connection storms to RDS**: 1,000 concurrent Lambdas x 1 connection exhausts `max_connections` => use **RDS Proxy** (pools and multiplexes; also speeds failover), keep the pool size 1 per environment.

### 7.4 API Gateway
| | **REST API** | **HTTP API** |
|---|---|---|
| Cost | higher | roughly 70% cheaper (approx.) |
| Features | API keys, usage plans, request validation/transformation, WAF, caching, resource policies, private APIs, canary | JWT/OIDC authorizers, CORS built in, lower latency; fewer features |
| Auth | IAM, Cognito, Lambda authorizer | IAM, JWT, Lambda authorizer |

Choose HTTP API by default; REST API when you need usage plans/API keys/WAF/caching/transformation. **WebSocket API** for push. Alternatives: **Lambda Function URL**, or ALB -> Lambda.

### 7.5 Step Functions (sagas and orchestration)
- State machine (Amazon States Language): Task, Choice, Parallel, Map, Wait, Retry/Catch, **`.waitForTaskToken`** callback pattern. **Standard** workflows (up to 1 year, exactly-once execution semantics, priced per state transition) vs **Express** (up to 5 min, at-least-once, high volume, cheap).
- Saga (orchestration) for an order:
```
Start -> ReserveStock -> ChargePayment -> CreateShipment -> Success
             |               | (Catch)          | (Catch)
             v               v                  v
        (fail) <-- RefundPayment <-- ReleaseStock  (compensations in reverse order)
```
- Each task retries with backoff; failure routes to compensating tasks. Orchestration = visible and central; choreography (events via SNS/EventBridge/Kafka) = decoupled but harder to trace.

---

## 8. Storage

### 8.1 S3 fundamentals
- **Bucket** (globally unique name, lives in one region) holds **objects** (key + value up to 5 TB + metadata + optional version id). Flat namespace; "folders" are just key prefixes.
- **Durability** 99.999999999% (11 nines) across >=3 AZs for most classes; availability differs by class.
- **Consistency**: **strong read-after-write and strong list consistency for all operations since Dec 2020** (no more eventual-consistency caveats for overwrite PUT/DELETE). Answer "S3 is eventually consistent" is outdated.
- Request rate: scales per prefix (approx. thousands of PUT/GET per second per prefix); no need to randomise prefixes anymore.

Storage classes
| Class | Use | Notes |
|---|---|---|
| Standard | hot data | |
| **Intelligent-Tiering** | unknown/changing access | auto-moves between tiers; small monitoring fee per object |
| Standard-IA / One Zone-IA | infrequent, rapid access | retrieval fee; min duration (30 d) and min object size charges; One Zone = single AZ (less resilient) |
| Glacier Instant Retrieval | archive with ms access | |
| Glacier Flexible Retrieval | archive, minutes-hours | |
| Glacier Deep Archive | cheapest, 12h+ | |
| Express One Zone | single-AZ, very low latency | newer; check docs |
Check current docs for minimum durations and retrieval prices.

Features
- **Versioning**: keeps all versions; delete adds a *delete marker*; protects from overwrite/delete; combine with **MFA Delete/Object Lock** (WORM) for ransomware protection. Costs storage for old versions - use lifecycle to expire noncurrent versions.
- **Lifecycle rules**: transition (Standard -> IA -> Glacier), expire, expire noncurrent versions, **abort incomplete multipart uploads** (hidden cost source).
- **Multipart upload**: recommended for objects > ~100 MB, required > 5 GB (single PUT max 5 GB); parts 5 MiB-5 GiB, up to 10,000 parts; parallel parts, retry a part only; TransferManager in SDK does this automatically.
- **Encryption**: all new objects are encrypted at rest by default with **SSE-S3** (since Jan 2023). Options: **SSE-S3** (AWS-owned keys), **SSE-KMS** (your KMS key: audit trail in CloudTrail, key policy control, per-request KMS cost/throttling - use **S3 Bucket Keys** to cut KMS calls), SSE-C (you supply key), client-side. Enforce TLS with `aws:SecureTransport`.
- **Access control**: (1) IAM identity policies, (2) **bucket policies** (resource-based, cross-account, conditions), (3) **ACLs = legacy; new buckets default to "Bucket owner enforced" (ACLs disabled)**. Prefer policies. (4) **Block Public Access** (account and bucket level; **on by default for new buckets**) - overrides public policies/ACLs. (5) Access Points, VPC endpoint policies.
- **Event notifications**: to SQS, SNS, Lambda, EventBridge on ObjectCreated/Removed etc. At-least-once, occasionally duplicated; not ordered - design idempotent consumers. EventBridge integration gives filtering and more targets.
- **Replication**: CRR (cross-region) / SRR, requires versioning.
- Other: Transfer Acceleration, Requester Pays, S3 Select (deprecating for new customers - check docs), Storage Lens, Inventory.

### 8.2 Presigned URLs (browser uploads directly to S3)
```
React app          Spring Boot API                 S3
   | 1. POST /files/upload-url {name,type,size}       |
   |------------------->| 2. authn/authz, validate ext/size, choose key "uploads/{userId}/{uuid}"
   |                    | 3. presigner.presignPutObject(bucket,key,expiry 5 min, contentType)
   |                    |    (signed with the task role's temp creds - no network call to S3)
   |<-------------------| 4. { url, key }
   | 5. PUT file bytes to url  (Content-Type must match what was signed)           |
   |---------------------------------------------------------------------------->|
   |<---------------------------------------------------------------- 200 OK ----|
   | 6. POST /files/confirm {key}  (or S3 event -> SQS -> worker marks it processed)
```
Facts: URL carries the signer's authority (permissions of the signing identity at use time), expires: max 7 days with long-term IAM user creds; when signed with role/STS creds it dies when those credentials expire (so short-lived on ECS/Lambda). Browser needs **CORS** on the bucket (`PUT`, allowed origin = your React domain, expose `ETag` for multipart). Limit blast radius: short expiry, fixed key, content-type, content-length-range (use **presigned POST** with policy conditions for size limits), then virus-scan/validate after upload.

Spring (SDK v2):
```java
@Service
class UploadService {
  private final S3Presigner presigner;                    // bean; region + default credentials
  String presignPut(String key, String contentType) {
    PutObjectRequest put = PutObjectRequest.builder()
        .bucket("shop-uploads").key(key).contentType(contentType).build();
    PresignedPutObjectRequest p = presigner.presignPutObject(r -> r
        .signatureDuration(Duration.ofMinutes(5)).putObjectRequest(put));
    return p.url().toString();
  }
}
```

### 8.3 Static site: React on S3 + CloudFront with OAC
```
Browser -> Route53 (alias) -> CloudFront (TLS cert from ACM in us-east-1) ->(OAC, SigV4)-> private S3 bucket
                                        \-> /api/* behavior -> ALB origin (no cache, forward headers/cookies)
```
- Bucket stays **private** (Block Public Access ON). Do not use S3 static website endpoint (HTTP only, needs public bucket). Use **Origin Access Control (OAC)**; OAI is the legacy mechanism.
- Bucket policy grants `s3:GetObject` to principal `cloudfront.amazonaws.com` with condition `AWS:SourceArn` = the distribution ARN.
- SPA routing: CloudFront **custom error responses** map 403/404 -> `/index.html` with 200 (or a CloudFront Function rewriting non-file paths).
- Caching: hashed assets (`main.abc123.js`) `Cache-Control: max-age=31536000, immutable`; `index.html` short/no-cache. On deploy: sync to S3, then **invalidate `/index.html`** (or `/*`; invalidations beyond a monthly free allowance cost - prefer versioned filenames).
- Add response headers policy (HSTS, CSP), WAF, geo restrictions as needed.

### 8.4 EBS vs EFS vs instance store
| | EBS | EFS | Instance store |
|---|---|---|---|
| Type | network block volume | managed NFS file system | local disk on host |
| Scope | **one AZ**, attached to one instance (Multi-Attach for io1/io2 special) | **multi-AZ**, mounted by many | tied to instance life |
| Persistence | persists (snapshots to S3) | persistent | **lost** on stop/terminate/failure |
| Use | boot disks, DBs on EC2 | shared content, CMS, ML shared data | cache, scratch, buffers |
| Notes | **gp3** default (baseline 3,000 IOPS, 125 MiB/s independent of size), io2 for high IOPS, st1/sc1 HDD | elastic size, pay per GB used, throughput modes; higher latency | very fast NVMe |
Snapshots are incremental; cross-AZ move = snapshot -> restore. Encrypt with KMS (default-encryption setting per region).

---

## 9. Databases

### 9.1 RDS
- Managed relational: engines **MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2**; **Aurora** (MySQL/PostgreSQL compatible, separate architecture). AWS handles provisioning, patching windows, backups, failover; you handle schema, queries, indexes, parameters, security groups.
- **Multi-AZ instance deployment**: **synchronous** replication to a standby in another AZ; **standby cannot serve reads**; automatic failover (typically ~60-120 s; check docs) when primary/AZ/storage/network fails or during maintenance; **same DNS endpoint flips** to the standby - clients must reconnect and **respect DNS TTL** (JVM caches DNS forever if a security manager is present, otherwise ~30 s default; set `networkaddress.cache.ttl=10`-ish).
  - **Multi-AZ DB cluster** (MySQL/PostgreSQL): one writer + two *readable* standbys across 3 AZs, faster failover (often ~35 s or less), semisynchronous - check docs.
- **Read replicas**: **asynchronous**, for read scaling, reporting, cross-region DR; **replica lag** (stale reads; monitor `ReplicaLag`); own endpoint; can be promoted (manual, breaks replication). Multi-AZ = availability; replicas = scale. Not the same thing (top interview question).
  ```
  Multi-AZ:        App -> [Primary AZ-a] == sync ==> [Standby AZ-b]   (standby idle; DNS flips on failure)
  Read replicas:   App(write) -> [Primary] -- async --> [Replica 1] [Replica 2]   App(read) -> replica endpoints
  ```
- **Backups**: automated daily snapshot + transaction logs => **point-in-time recovery (PITR)** to any second within retention (1-35 days); restore always creates a **new instance** (new endpoint). Manual snapshots persist beyond retention; copy cross-region/account. Test restores.
- **Parameter groups** (engine config like `max_connections`, `innodb_buffer_pool_size`, `log_min_duration_statement`); static params need reboot. **Option groups** (Oracle/SQL Server/MySQL options). **Subnet group** (which subnets), **security group** (5432/3306 from app-sg only), **not publicly accessible**.
- Storage autoscaling, gp3/io1, encryption at rest with KMS (**must be chosen at creation**; to encrypt later, snapshot -> copy encrypted -> restore), IAM database authentication (15-min tokens), Performance Insights, Enhanced Monitoring, **deletion protection**, blue/green deployments for upgrades.
- **RDS Proxy**: managed connection pool; helps **Lambda** (many short-lived connections) and **failover** (keeps app connections, reconnects to new primary faster); IAM auth/Secrets Manager integration; adds latency + cost; watch connection "pinning".

### 9.2 Aurora
```
             Cluster writer endpoint  ---->  [Writer instance AZ-a]
             Cluster reader endpoint  ---->  [Reader AZ-b] [Reader AZ-c]   (up to 15 readers, load-balanced)
                                   \____ all share ____/
        Distributed storage volume: 6 copies across 3 AZs, 10 GB segments, auto-grows (up to 128 TiB)
        Write quorum 4/6, read quorum 3/6; self-healing; replicas share storage => low replica lag (ms)
```
- Compute separate from storage; instead of shipping data pages, the writer ships **redo log records** to storage nodes.
- **Failover**: promote a reader (typically < ~30 s; no data copy since storage is shared); with no readers, must create a new instance (slower). Custom endpoints; global database for cross-region (typical RPO ~1 s, RTO ~minutes).
- **Serverless v2**: scales in fine-grained **ACUs** (min/max), including instances in a normal cluster; can be combined with provisioned; newer versions can scale down to 0 ACU with auto-pause (check docs). Good for variable/unpredictable/dev workloads; steady load is usually cheaper provisioned.
- Fast clones (copy-on-write), backtrack (MySQL), Data API. Costs more per instance-hour than RDS but often better throughput; I/O-Optimized vs Standard pricing - compare with your I/O bill.

### 9.3 DynamoDB fundamentals
- Fully managed key-value/document NoSQL; single-digit ms; scales horizontally; no servers/connections, **HTTP API**, IAM-controlled.
- **Primary key**: partition key (PK) or PK + sort key (SK). Items <= 400 KB. Data is spread across **partitions by hash(PK)**.
- **Partition key design**: high cardinality, uniform access. **Hot partition** = one key gets disproportionate traffic (e.g. `status=ACTIVE`, celebrity user, date as PK). A partition serves approx. 3,000 RCU / 1,000 WCU max (check docs); adaptive capacity helps but does not fix a single hot key. Fixes: write sharding (`PK#0..N`), composite keys, caching (DAX/ElastiCache), spreading by tenant+time bucket.
- **Capacity modes**: **On-demand** (pay per request, instant scale, no planning, pricier per request at steady load) vs **Provisioned** (RCU/WCU + auto scaling, cheaper for predictable). Throttling => `ProvisionedThroughputExceededException`; SDK retries with backoff.
- **Indexes**: **GSI** - different PK/SK, own throughput, **eventually consistent only**, can be added anytime, default quota ~20 per table; **LSI** - same PK, different SK, must be created **at table creation**, strongly consistent option, shares partition, item-collection size limit 10 GB, ~5 per table.
- **Consistency**: reads are eventually consistent by default (half RCU cost); **strongly consistent** optional on table/LSI (not GSI, not across global-table regions); **transactions** (TransactGetItems/TransactWriteItems, 2x cost) for ACID across items.
- **Query vs Scan**: Query by key (efficient); Scan reads all (avoid). Conditional writes for optimistic locking (`attribute_not_exists`, version attribute) - also the idempotency primitive.
- **Single-table design**: put multiple entity types in one table using generic `PK`/`SK` (e.g. `PK=CUSTOMER#42`, `SK=ORDER#2026-01-05#987`) so one Query fetches a customer and orders. Pros: fewer round trips, one capacity pool. Cons: steep learning curve, rigid to new access patterns. **Design from access patterns first**.
- **TTL** (free expiry), **Streams** (24 h ordered change log per item -> Lambda / Kinesis for CDC, outbox, cache invalidation), **PITR**, on-demand backup, **Global Tables** (multi-region active-active, last-writer-wins), DAX (in-memory cache for microsecond reads).
- **DynamoDB vs RDS**:
  | Choose DynamoDB when | Choose RDS/Aurora when |
  |---|---|
  | Known key-based access patterns, huge scale, spiky traffic, serverless stack, session/cart/IoT/leaderboards | Complex joins/ad-hoc queries/reporting, strong relational integrity, transactions across many rows, existing SQL skills/ORMs (JPA) |
  | Predictable single-digit ms at any scale | Moderate scale, evolving access patterns |
  For an e-commerce order module with reporting a Java team knows: **RDS/Aurora**. Use DynamoDB for carts, sessions, idempotency keys, event dedupe.
- Java: AWS SDK v2 **DynamoDB Enhanced Client** (`@DynamoDbBean`), Spring Data DynamoDB is community; Spring Cloud AWS has a DynamoDB starter (v3).

### 9.4 ElastiCache (Redis/Valkey, Memcached)
- **Redis/Valkey** (ElastiCache supports both; check docs): data structures, persistence (AOF/snapshots), replication with Multi-AZ auto failover, cluster mode (sharding), pub/sub, Lua, TTL. **Memcached**: simple, multi-threaded, no persistence/replication, sharding by client.
- Patterns:
  - **Cache-aside (lazy loading)**: read cache -> on miss read DB -> populate with TTL. Most common; risk stale data - invalidate on write (`@CacheEvict`) + TTL.
  - **Write-through**: write DB and cache together; **write-behind** riskier.
  - Sessions (Spring Session Redis), rate limiting (`INCR` + expiry), distributed lock (Redisson/`SET NX PX`), leaderboards (sorted sets), idempotency keys.
- Problems: **cache stampede** (many misses at once => request coalescing, jittered TTL, locks), **hot keys**, **eviction policy** (`allkeys-lru`), **thundering herd after failover** (cold cache; DB must survive).
- Network: private subnet, SG only from app-sg, in-transit encryption (TLS) + AUTH/RBAC, cluster failover ~tens of seconds. **Serverless** option exists.
- Spring: `spring-boot-starter-data-redis` + `spring.cache.type=redis`, Lettuce with topology refresh for cluster.

---

## 10. Messaging and integration

### 10.1 SQS
| | Standard | FIFO |
|---|---|---|
| Throughput | nearly unlimited | limited (approx. 300 msg/s per API action, 3,000 with batching; **high throughput mode** far higher - check docs) |
| Ordering | best-effort | strict **per MessageGroupId** |
| Delivery | **at-least-once**, duplicates possible | **exactly-once processing** within 5-min dedup window (`MessageDeduplicationId` or content-based) |
| Name | any | must end `.fifo` |
- **Visibility timeout** (default 30 s, max 12 h): after a consumer receives a message it's hidden; if not deleted in time it **reappears** (=> duplicate processing). Set > max processing time; extend via `ChangeMessageVisibility` heartbeat. For Lambda event source: set queue visibility timeout >= ~6x function timeout (AWS guidance).
- **Long polling** (`WaitTimeSeconds` up to 20 s): fewer empty receives, lower cost and latency. Prefer over short polling.
- **DLQ**: redrive policy `maxReceiveCount` (e.g. 5) moves poison messages to a DLQ (same type as source queue); alarm on `ApproximateNumberOfMessagesVisible` of DLQ; **DLQ redrive** back to source once fixed. DLQ retention should exceed source retention (message keeps its original enqueue time).
- Retention default 4 days (max 14). Message size max 256 KiB historically (recently raised to 1 MiB - check docs; larger payloads => Extended Client / S3 pointer). Delay queues/message timers up to 15 min. Batch up to 10.
- **At-least-once => consumers must be idempotent**: dedupe table (`INSERT ... ON CONFLICT DO NOTHING` or DynamoDB conditional put on messageId/business key), or naturally idempotent updates.
- Queue encryption SSE-SQS default; policy needed for SNS/S3/EventBridge to send.
- Spring Cloud AWS 3: `@SqsListener("orders-queue")`, `SqsTemplate.send(...)`; acknowledgement mode ON_SUCCESS by default (delete after listener returns without exception).

### 10.2 SNS and fan-out
```
Order service --publish--> SNS topic "order-events"
                             |--> SQS "email-queue"      (filter: type=ORDER_PLACED)  --> email worker
                             |--> SQS "inventory-queue"                               --> inventory worker
                             |--> SQS "analytics-queue"                               --> analytics worker
                             |--> Lambda / HTTPS / SMS / email
```
- SNS pub/sub push; **SNS -> SQS fan-out** gives each consumer its own durable buffer, retries and DLQ; slow consumer does not affect others. **Subscription filter policies** (attribute or payload) avoid useless deliveries. Raw message delivery flag removes SNS envelope. SNS **FIFO** topics can fan out to FIFO queues.
- SQS queue policy must allow `sns:` topic to `sqs:SendMessage` with `aws:SourceArn` condition:
```json
{ "Effect": "Allow", "Principal": { "Service": "sns.amazonaws.com" }, "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:ap-south-1:111122223333:inventory-queue",
  "Condition": { "ArnEquals": { "aws:SourceArn": "arn:aws:sns:ap-south-1:111122223333:order-events" } } }
```

### 10.3 EventBridge
Event bus with **rules** (pattern matching on JSON) -> many targets (Lambda, SQS, SNS, Step Functions, API destinations, other buses/accounts). Sources: AWS service events (EC2 state, S3, ECS), custom apps (`PutEvents`), SaaS partners. Features: **schema registry, archive & replay, Scheduler** (cron/one-time, replaces CloudWatch Events schedules), **Pipes** (source -> filter -> enrich -> target). Use for event-driven decoupling with routing rules; SNS for simple high-fanout push; SQS for buffering/work queues.

### 10.4 Kinesis vs MSK vs SQS decision guide
| Need | SQS | Kinesis Data Streams | MSK (managed Kafka) |
|---|---|---|---|
| Model | queue (message consumed & deleted) | ordered **stream** in shards, replayable, retention 24 h up to 365 days | Kafka topics/partitions, replayable, consumer groups |
| Ordering | FIFO per group | per shard/partition key | per partition |
| Consumers | competing consumers | multiple independent readers (shared/enhanced fan-out) | consumer groups |
| Ops | none | low (on-demand or provisioned shards) | higher (brokers, storage; Serverless option) |
| Ecosystem | AWS native | AWS native (Firehose, Lambda, Flink) | Kafka Connect, Streams, existing Kafka skills/clients |
| Pick when | decouple services, task queues, retry/DLQ | real-time analytics/logs/clickstream in AWS | you already run Kafka, need Kafka APIs/ecosystem, event sourcing at scale |
Rule of thumb: **task distribution -> SQS; event log/replay/stream processing -> Kinesis or Kafka(MSK)**. See Kafka notes for consumer group internals. **Amazon MQ** = managed **RabbitMQ / ActiveMQ** with AMQP/JMS/MQTT: choose it for *lift-and-shift* of an app already using RabbitMQ/JMS (no code change); for new AWS-native designs prefer SQS/SNS/EventBridge. Amazon MQ: broker in VPC, Multi-AZ (RabbitMQ cluster of 3 nodes), you still think about queues/exchanges.

---

## 11. Edge networking

### 11.1 Route 53
- Authoritative DNS + domain registration + health checks. **Alias record** (AWS-specific, at zone apex, free, points to ALB/CloudFront/S3/API GW) vs CNAME (not allowed at apex).
- Routing policies:
  | Policy | Behaviour | Use |
  |---|---|---|
  | Simple | one/multi values | single endpoint |
  | **Weighted** | proportion by weight | canary, blue/green (10% to new) |
  | **Latency** | lowest-latency region | multi-region active-active |
  | **Failover** | primary/secondary using health check | active-passive DR |
  | **Geolocation** | by user's country/continent | compliance, localisation, blocked regions |
  | Geoproximity | distance with bias | traffic shifting |
  | Multivalue answer | up to 8 healthy records | poor-man's LB |
- **Health checks**: HTTP/HTTPS/TCP endpoint checks from global checkers, calculated (combine), CloudWatch-alarm-based (for private resources). Unhealthy record is removed from answers. DNS **TTL** limits how fast clients switch: low TTL (30-60 s) for failover records. Private hosted zones for internal names.

### 11.2 CloudFront
- CDN: viewer -> nearest edge -> regional edge cache -> **origin** (S3, ALB, API Gateway, custom). Behaviors by path pattern each with cache policy, origin request policy, allowed methods.
- **Cache key** = URL + selected headers/cookies/query strings (keep it minimal for high hit ratio). TTLs from origin `Cache-Control` or policy min/default/max.
- **Invalidation** removes objects from edge caches (first ~1,000 paths per month free, then charged; check docs) - prefer versioned file names.
- Features: TLS termination (ACM cert **must be in us-east-1** for CloudFront), HTTP/2/3, signed URLs/cookies for private content, Origin Shield, Functions/Lambda@Edge, WAF attachment, geo restriction, OAC to S3, **origin failover groups**. Also speeds up dynamic APIs (persistent connections over AWS backbone) even without caching.
- Restrict ALB origin so only CloudFront can reach it: managed prefix list `com.amazonaws.global.cloudfront.origin-facing` in ALB SG plus a secret custom header check.

### 11.3 WAF, Shield, ACM
- **WAF**: L7 web ACL on CloudFront/ALB/API GW/AppSync: managed rule groups (OWASP-style, IP reputation, bot control), rate-based rules, geo/IP sets, custom rules. Start in **count mode** to check false positives.
- **Shield Standard**: automatic L3/L4 DDoS protection for all (free). **Shield Advanced**: paid, L7 help, DDoS response team, cost protection.
- **ACM**: free public certs with auto-renew when DNS-validated (attach to ALB/CloudFront/API GW; **cannot export** public certs for use on EC2/Fargate directly - terminate TLS at the LB or use private CA). Regional: ALB cert must be in the ALB's region.

---

## 12. Security services

### 12.1 KMS and envelope encryption
- **KMS** stores/uses **KMS keys** (formerly CMKs) in HSMs; keys never leave KMS unencrypted. Types: AWS-owned, **AWS-managed** (`aws/s3`, cannot edit policy), **customer-managed** (your key policy, rotation optional/annual, aliases, grants). Direct `Encrypt` limited to 4 KB.
- **Envelope encryption**:
```
1. App calls KMS GenerateDataKey(keyId) -> returns {plaintext data key, encrypted data key}
2. App encrypts the big payload locally with the plaintext data key (AES-256-GCM), then wipes the key from memory
3. Store: ciphertext + encrypted data key (together)
4. To decrypt: KMS Decrypt(encrypted data key) -> plaintext data key -> decrypt payload locally
```
  Why: KMS is only called for small keys (fast, cheap, audit-able), data never travels to KMS, per-object keys limit blast radius. S3 SSE-KMS, EBS, RDS use this internally.
- **Key policy is the primary authority**: IAM policies work only if the key policy allows IAM control (default policy delegates to account root). Cross-account use needs key policy + IAM allow in the caller account. Grants for AWS services. KMS API throttling (per-region shared quota) can surface as `ThrottlingException` with heavy SSE-KMS traffic - use S3 Bucket Keys / data key caching.

### 12.2 Secrets Manager vs Parameter Store
| | Secrets Manager | SSM Parameter Store |
|---|---|---|
| Purpose | secrets (DB credentials, API keys) | config + (SecureString) secrets |
| Rotation | **built-in rotation** (Lambda; native for RDS/Aurora) | none built-in (DIY) |
| Cost | per secret per month + per API calls | Standard tier free; advanced tier paid |
| Size | up to 64 KB | 4 KB standard / 8 KB advanced |
| Cross-account/replication | resource policy, multi-region replication | limited |
| Versioning | staging labels `AWSCURRENT/AWSPREVIOUS` | versions |
Use Secrets Manager for DB creds needing rotation; Parameter Store for plain config/feature flags/cheap secrets. Both encrypt with KMS; access via task/pod role. Cache secrets client-side (AWS Secrets Manager caching library) to avoid latency and throttling; on rotation, the app must refetch or reconnect (or use dual-user rotation).

### 12.3 Cognito
- **User pools** = user directory + sign-up/sign-in + JWTs (ID/access/refresh), MFA, social/SAML/OIDC federation, hosted UI; API GW / ALB can validate directly. Spring Boot resource server: validate JWT via issuer URI (`spring.security.oauth2.resourceserver.jwt.issuer-uri=https://cognito-idp.<region>.amazonaws.com/<poolId>`).
- **Identity pools (federated identities)** = exchange a token for temporary **AWS credentials** (e.g. browser uploads directly to S3 under an IAM role scoped per user). Do not confuse the two.

### 12.4 Detection and governance overview
| Service | One-liner |
|---|---|
| **CloudTrail** | API audit log (who did what, when, from where); management events on by default (90-day Event history); create an org **trail to S3** with log file validation for long-term; data events (S3 objects, Lambda) opt-in |
| **Config** | Records resource configuration history + evaluates compliance rules ("no public S3", "RDS encrypted") + remediation |
| **GuardDuty** | Threat detection from CloudTrail, VPC flow, DNS logs (compromised creds, crypto mining, odd API calls) |
| **Inspector** | Vulnerability scanning of EC2, ECR images, Lambda |
| **Security Hub** | Aggregates findings + benchmark checks (CIS/FSBP) across accounts |
| **Macie** | Finds sensitive data (PII) in S3 |
| **Access Analyzer** | Finds resources shared externally; generates least-privilege policies; validates policies |
| **Detective** | Investigation graph |

---

## 13. Observability

- **CloudWatch Metrics**: namespace + name + dimensions; standard 1-min for most services (EC2 basic 5-min, detailed 1-min); custom metrics via `PutMetricData` (or Micrometer CloudWatch registry). High-resolution 1-second custom metrics possible. **Alarms**: threshold/anomaly/composite, states OK/ALARM/INSUFFICIENT_DATA, `datapointsToAlarm` (M of N) to reduce flapping, actions -> SNS, ASG, EC2, Systems Manager. Choose statistic carefully (p99 vs average) and `treatMissingData`.
- **Logs**: log groups/streams, retention (set it! default is *never expire* = cost), metric filters, subscription filters (to Lambda/Kinesis/OpenSearch), **Logs Insights** query language:
```
fields @timestamp, level, traceId, message
| filter level = "ERROR" and message like /timeout/
| stats count() as errors by bin(5m)
| sort errors desc
```
- **Embedded Metric Format (EMF)**: emit structured JSON log lines containing a `_aws` metadata block; CloudWatch extracts **custom metrics** asynchronously from logs - no PutMetricData API calls, cheap, high-cardinality context preserved (Lambda and ECS via agent). Example:
```json
{"_aws":{"Timestamp":1735689600000,"CloudWatchMetrics":[{"Namespace":"Shop/Orders","Dimensions":[["Service"]],
  "Metrics":[{"Name":"OrderLatencyMs","Unit":"Milliseconds"}]}]},
 "Service":"orders","OrderLatencyMs":182,"orderId":"o-987"}
```
- **Tracing**: **X-Ray** (segments/subsegments, service map, sampling) and **AWS Distro for OpenTelemetry (ADOT)** collector as the recommended instrumentation (OpenTelemetry Java agent -> ADOT collector -> X-Ray/CloudWatch). Propagate trace ids across ALB (`X-Amzn-Trace-Id`), SQS, Lambda. Correlate logs by putting `traceId` into MDC.
- **Container Insights**, **Application Signals** (APM/SLOs), **Synthetics canaries**, **RUM** for the React front end, **dashboards**.
- **CloudTrail** is audit (control plane), CloudWatch is operations (telemetry). Golden signals: latency, traffic, errors, saturation. Health endpoint metrics: ALB `HTTPCode_Target_5XX_Count`, `TargetResponseTime`, `UnHealthyHostCount`, RDS `CPUUtilization/DatabaseConnections/FreeableMemory/ReplicaLag`, SQS `ApproximateAgeOfOldestMessage`.

---

## 14. IaC and delivery

### 14.1 CloudFormation vs Terraform vs CDK
| | CloudFormation | Terraform | CDK |
|---|---|---|---|
| Language | YAML/JSON | HCL | TypeScript/Java/Python/C#... synthesises to CloudFormation |
| Scope | AWS only | multi-cloud + SaaS | AWS (CDK for Terraform exists) |
| State | **managed by AWS** (stacks) | **you manage** state (S3 backend + DynamoDB/S3 native locking; check docs) | via CloudFormation |
| Preview | change sets | `terraform plan` | `cdk diff` |
| Notes | drift detection, rollback on failure, StackSets | large ecosystem/modules, easier multi-account/provider composition | loops/conditions/abstractions (constructs) in a real language; Java devs can use Java |
Pick: AWS-only team wanting no state file => CloudFormation/CDK; multi-cloud or existing Terraform => Terraform. Never click-ops production; review plans in PRs.

CloudFormation snippet (SQS with DLQ + queue policy):
```yaml
Resources:
  OrdersDlq:
    Type: AWS::SQS::Queue
    Properties: { QueueName: orders-dlq, MessageRetentionPeriod: 1209600 }   # 14 days
  OrdersQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: orders-queue
      VisibilityTimeout: 60
      ReceiveMessageWaitTimeSeconds: 20                 # long polling
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrdersDlq.Arn
        maxReceiveCount: 5
```
Terraform snippet (private S3 bucket for CloudFront, encrypted, versioned):
```hcl
resource "aws_s3_bucket" "site" { bucket = "shop-web-prod-123456" }

resource "aws_s3_bucket_public_access_block" "site" {
  bucket                  = aws_s3_bucket.site.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
resource "aws_s3_bucket_versioning" "site" {
  bucket = aws_s3_bucket.site.id
  versioning_configuration { status = "Enabled" }
}
resource "aws_s3_bucket_server_side_encryption_configuration" "site" {
  bucket = aws_s3_bucket.site.id
  rule { apply_server_side_encryption_by_default { sse_algorithm = "AES256" } }
}
data "aws_iam_policy_document" "oac" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.site.arn}/*"]
    principals { type = "Service", identifiers = ["cloudfront.amazonaws.com"] }
    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.site.arn]
    }
  }
}
resource "aws_s3_bucket_policy" "site" { bucket = aws_s3_bucket.site.id  policy = data.aws_iam_policy_document.oac.json }
```
Terraform ECS task-role/execution-role sketch:
```hcl
data "aws_iam_policy_document" "ecs_trust" {
  statement { actions = ["sts:AssumeRole"]
    principals { type = "Service", identifiers = ["ecs-tasks.amazonaws.com"] } }
}
resource "aws_iam_role" "task"      { name = "orders-task"      assume_role_policy = data.aws_iam_policy_document.ecs_trust.json }
resource "aws_iam_role" "execution" { name = "orders-execution" assume_role_policy = data.aws_iam_policy_document.ecs_trust.json }
resource "aws_iam_role_policy_attachment" "exec" {
  role       = aws_iam_role.execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```
Terraform tips: remote state with locking, small state per environment/component, `prevent_destroy` on data stores, `terraform plan` in PR, never store secrets in state carelessly (state contains them).

### 14.2 CodePipeline / CodeBuild / CodeDeploy
- **CodePipeline**: orchestrates stages (Source -> Build -> Test -> Approval -> Deploy). **CodeBuild**: managed build containers running `buildspec.yml` (Maven build, docker build/push to ECR); pay per build minute, runs in VPC if needed. **CodeDeploy**: deployment to EC2/on-prem (in-place or blue/green), Lambda (traffic shifting), ECS (blue/green with ALB) with hooks and automatic rollback on alarms. Many teams use **GitHub Actions/Jenkins** for CI and only CodeDeploy for the deploy step.

### 14.3 Deployment strategies
| Strategy | How | Rollback | Risk/cost |
|---|---|---|---|
| Rolling | replace tasks gradually (ECS minHealthy/max %) | redeploy previous; circuit breaker auto | mixed versions during rollout (DB compat!) |
| **Blue/green** | new (green) stack alongside old (blue); shift ALB/Route53 traffic | flip back instantly | double capacity temporarily |
| Canary / linear | 10% then rest (CodeDeploy ECS/Lambda, weighted Route 53) | auto rollback on alarm | slow, safest |
Database migrations must be **backward compatible** (expand -> migrate -> contract with Flyway/Liquibase) so blue and green can both run.

### 14.4 Deployment flow (ECS Fargate)
```
git push -> GitHub Actions: mvn verify -> docker build -> (OIDC role) push to ECR :sha
   -> render new task definition revision with image :sha -> ecs update-service (or CodeDeploy blue/green)
   -> new tasks start -> ALB health check passes (after grace period) -> old tasks drained (deregistration delay) -> done
   -> circuit breaker rolls back if new tasks keep failing; CloudWatch alarms watch 5xx/latency
```

### 14.5 GitHub Actions OIDC to AWS (no static keys)
```
GitHub job (id-token: write) --1. request OIDC JWT from GitHub (iss token.actions.githubusercontent.com, sub repo:org/repo:ref:refs/heads/main)
        --2. sts:AssumeRoleWithWebIdentity(JWT, role) --> IAM verifies signature via OIDC provider, checks trust conditions
        <--3. temp credentials (~1h) --> deploy steps
```
IAM OIDC provider URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`. Role trust policy:
```json
{ "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
    "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:my-org/shop-backend:ref:refs/heads/main" } } }
```
Workflow:
```yaml
permissions: { id-token: write, contents: read }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: arn:aws:iam::111122223333:role/gha-deploy-shop, aws-region: ap-south-1 }
      - uses: aws-actions/amazon-ecr-login@v2
```
Always pin `sub` to a specific repo and branch/environment; a wildcard `repo:my-org/*` lets any repo in the org assume the prod role. Separate roles per environment.

---

## 15. Well-Architected, HA/DR, cost

### 15.1 Six pillars applied to concrete decisions
| Pillar | Question | Concrete decision |
|---|---|---|
| Operational excellence | Can we deploy and diagnose safely? | IaC, CI/CD with OIDC, runbooks, structured logs + traces, game days |
| Security | Least privilege, encrypted, auditable? | roles not keys, private subnets, KMS, WAF, CloudTrail org trail, SCPs, multi-account |
| Reliability | Survive failure and recover? | Multi-AZ everywhere, health checks, retries with jitter, timeouts, DLQs, tested restores |
| Performance efficiency | Right tool, right size? | CloudFront, cache-aside, read replicas, Graviton, Lambda memory tuning |
| Cost optimisation | Pay only for value? | Savings Plans, Spot workers, S3 lifecycle, endpoints vs NAT, delete idle, budgets |
| Sustainability | Minimise resources | right-size, Graviton, serverless scale-to-zero, storage tiers |

### 15.2 HA vs DR
- **HA** = keep running through component/AZ failure (Multi-AZ, ASG, stateless tiers). **DR** = recover from region loss/corruption/human error (backups, cross-region).
- **RPO** (Recovery Point Objective) = max tolerable **data loss** (time). **RTO** = max tolerable **downtime**.
- DR strategies (cost and speed increase downward):
| Strategy | RPO / RTO (order of) | What runs in DR region | Cost |
|---|---|---|---|
| **Backup & restore** | hours / hours-24h | nothing; backups (S3 CRR, cross-region snapshots) + IaC to rebuild | lowest |
| **Pilot light** | minutes / tens of minutes | data replicated (Aurora Global/RDS replica, DynamoDB global table); app tier off, AMI/images ready, scale up on disaster | low |
| **Warm standby** | seconds-minutes / minutes | scaled-down full copy running; scale up + DNS failover | medium |
| **Multi-site active-active** | ~0 / ~0 | full capacity in both regions, Route 53 latency/weighted | highest; conflict handling (global tables last-writer-wins) |
- **Multi-AZ vs multi-region**: Multi-AZ = same region, sync replication, minutes-or-less failover, protects against **AZ/hardware** failure; needed by default for prod. Multi-region = protects against regional outage, compliance, latency; async replication (nonzero RPO), complex (data consistency, config, secrets, certificates per region, DNS failover, cost). Most systems: Multi-AZ + cross-region backups; go multi-region only for a stated RTO/RPO or legal requirement.
- Also protect against **human/logical corruption**: replication faithfully copies a `DROP TABLE`; only versioned/PITR backups help. Use separate backup account, S3 Object Lock, AWS Backup vaults with vault lock.
- Untested DR = no DR. Run restore drills and failover tests.

### 15.3 Cost optimisation checklist
1. **NAT Gateway**: per-hour x per-AZ + per-GB. Add S3/DynamoDB gateway endpoints (free), interface endpoints for ECR/logs/SQS where volume justifies, avoid pulling big images/data through NAT, consider IPv6 egress-only IGW, consolidate dev to one NAT.
2. **Data transfer**: inbound free; **outbound to internet charged**; **cross-AZ** charged both ways (chatty microservices across AZs, NLB cross-zone, cross-AZ NAT); same-AZ private traffic free. CloudFront egress is cheaper than direct-from-S3/EC2 and often lower. Keep chatty pairs in same AZ where HA is not compromised (topology-aware routing).
3. **Right-size**: Compute Optimizer, CloudWatch CPU/memory (Java heap vs container memory); Graviton (~ 20% cheaper/better perf commonly).
4. **Commitments**: Compute Savings Plan for steady baseline; RDS reserved instances.
5. **Spot** for stateless workers, CI, batch (interruption-tolerant, SQS-based).
6. **S3**: lifecycle to IA/Glacier, Intelligent-Tiering, expire old versions and incomplete multipart uploads, delete unneeded log data; S3 request costs for tiny objects.
7. **Idle**: unattached EBS volumes, old snapshots, unused Elastic IPs (charged when idle), idle load balancers, stopped-but-billed resources, dev environments at night (Instance Scheduler / scale-to-zero).
8. **Logs**: CloudWatch Logs ingestion is expensive; set retention, lower verbosity, sample, export to S3.
9. **Governance**: **tags** (`env`, `team`, `service`, `cost-center`) activated as cost allocation tags; **Budgets** with alerts and actions; **Cost Explorer** + Cost Anomaly Detection; CUR/Athena; per-account separation.
10. Serverless is cheaper at low/spiky load, containers cheaper at steady high load - compute breakeven, do not assume.

---

## 16. Spring Boot on AWS specifics

### 16.1 Credentials provider chain (state accurately)
Spring Cloud AWS 3.x is built on **AWS SDK for Java v2** and by default uses the SDK's `DefaultCredentialsProvider` (you can override with a `CredentialsProvider` bean or `spring.cloud.aws.credentials.*`). SDK v2 default order (check the SDK docs for your version):
1. **Java system properties** `aws.accessKeyId` / `aws.secretAccessKey` (/`aws.sessionToken`)
2. **Environment variables** `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (/`AWS_SESSION_TOKEN`)
3. **Web identity token** (`AWS_WEB_IDENTITY_TOKEN_FILE` + `AWS_ROLE_ARN`) - this is **IRSA** and also GitHub OIDC-style setups
4. **Shared config/credentials profile** (`~/.aws/credentials`, `~/.aws/config`, `AWS_PROFILE`; includes SSO and assume-role profiles)
5. **Container credentials** (`AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` / `FULL_URI`) - **ECS task role** and **EKS Pod Identity**
6. **EC2 instance profile** via IMDS (IMDSv2)
Note: in SDK v1 the order was env vars first, then system properties - so "env -> system props" is the v1/older ordering; in v2 system properties come first. Newer SDK versions may add other providers (e.g. login/SSO based) - do not memorise beyond this, know the principle: **explicit config overrides; on AWS the role is picked up automatically; locally use the profile/SSO**. Classic bug: stale `AWS_ACCESS_KEY_ID` env vars on a dev machine or CI runner silently override the intended role.

### 16.2 Spring Cloud AWS 3: S3, SQS, SNS
```xml
<dependencyManagement><dependencies><dependency>
  <groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-dependencies</artifactId>
  <version>3.x</version><type>pom</type><scope>import</scope></dependency></dependencies></dependencyManagement>
<!-- starters: spring-cloud-aws-starter-s3, -sqs, -sns, -secrets-manager, -parameter-store, -dynamodb -->
```
```yaml
spring:
  cloud:
    aws:
      region:
        static: ap-south-1
      s3:
        # endpoint override for LocalStack in dev: endpoint: http://localhost:4566
        path-style-access-enabled: false
      sqs:
        listener:
          max-concurrent-messages: 10
          poll-timeout: 10s
```
```java
@Component
class OrderWorker {
  @SqsListener("orders-queue")                     // ACK on success (delete), on exception message reappears after visibility timeout
  void handle(@Payload OrderEvent e, @Header(SqsHeaders.SQS_MESSAGE_ID_HEADER) String messageId) {
    if (!dedupe.firstTime(messageId, e.orderId())) return;   // idempotency
    service.process(e);
  }
}
@Service
class Publisher {
  private final SqsTemplate sqs; private final SnsTemplate sns; private final S3Template s3;
  void send(OrderEvent e) { sqs.send(to -> to.queue("orders-queue").payload(e)); }
  void announce(OrderEvent e) { sns.convertAndSend("order-events", e); }
  void upload(String key, InputStream in) { s3.upload("shop-uploads", key, in); }
}
```
Local dev: **LocalStack** via Testcontainers; same code, only endpoint override.

### 16.3 RDS + Secrets Manager config
Import the secret so Boot properties resolve from it (Spring Cloud AWS 3 config import; JSON secret keys become properties):
```yaml
spring:
  config:
    import: "aws-secretsmanager:/prod/orders/db"      # secret JSON {"username":"..","password":".."}
  datasource:
    url: jdbc:postgresql://orders.cluster-xyz.ap-south-1.rds.amazonaws.com:5432/orders   # use the ENDPOINT (DNS), never IP
    username: ${username}
    password: ${password}
    hikari:
      maximum-pool-size: 10
      max-lifetime: 1500000            # 25 min: less than DB/proxy/idle-kill timeouts
      connection-timeout: 3000
      keepalive-time: 300000
```
- Give the **task/pod role** `secretsmanager:GetSecretValue` (+ `kms:Decrypt` for CMK) on that secret ARN only. With ECS you can alternatively inject through task definition `secrets` (needs the **execution role**) - the secret is read at task start only; rotation needs restart.
- Rotation: secret changes -> existing pool connections stay valid until closed; new logins use new password. Use alternating-users rotation for zero downtime; set Hikari `max-lifetime` short so pools re-login; handle auth failures by refetching the secret.
- Pool sizing: total connections = tasks x pool size must be < DB `max_connections` (mind Lambda/blue-green overlap). RDS Proxy for many clients.
- Failover: after Multi-AZ failover, Hikari connections break; validation (`connectionTestQuery` not needed for JDBC4) + DNS TTL settings (`-Dnetworkaddress.cache.ttl=10`) let the app reconnect within seconds; use retries (`spring-retry`/Resilience4j) on transient errors; consider AWS Advanced JDBC Wrapper for faster Aurora failover detection.

### 16.4 Actuator health behind ALB
```yaml
management:
  endpoints.web.exposure.include: health,info,prometheus
  endpoint.health.probes.enabled: true          # /actuator/health/liveness and /readiness
  endpoint.health.show-details: never
server:
  shutdown: graceful
  tomcat.keep-alive-timeout: 75s                # > ALB idle timeout (60 s default)
spring.lifecycle.timeout-per-shutdown-phase: 30s
```
- **Target group health check path**: `/actuator/health/readiness` (or `/actuator/health`) - **not `/`** (may 404/redirect/require auth => all targets unhealthy => ALB returns 503). Success codes `200`. Keep the endpoint **unauthenticated** in Spring Security (permit it) and on the traffic port. Don't include DB in liveness; include downstream dependencies in readiness carefully (a DB blip should not make ALB drain every task at once; consider `readiness` = app-only and alarms for DB).
- Tuning: interval 15-30 s, healthy threshold 2, unhealthy 3, timeout < interval.
- **Grace period**: ECS service `healthCheckGracePeriodSeconds` (e.g. 120 for slow Spring startup), ASG `HealthCheckGracePeriod`, K8s `startupProbe`. Must exceed boot time or tasks are killed in a restart loop.
- **Graceful shutdown**: on SIGTERM ECS/K8s starts stop timer (`stopTimeout` default 30 s on Fargate; configurable up to 120 s); ALB deregistration delay drains connections; `server.shutdown=graceful` finishes in-flight requests. Align: deregistration delay ~ graceful timeout ~ stopTimeout.
- Behind ALB: `server.forward-headers-strategy=framework` so redirects/HATEOAS use `https` and the client host.

---

## 17. Reference architectures and failure analysis
Prices are **approximate monthly, on-demand, us-east-1/ap-south-1 order of magnitude, small production sizing; vary by region and change - check the pricing calculator**.

### 17.1 (a) React on S3+CloudFront + Spring Boot on ECS Fargate + RDS Multi-AZ + ElastiCache + SQS worker
```
                       Route53 (alias)
                            |
                       CloudFront + WAF  --(OAC)--> S3 (React build, private)
                            | /api/*
                            v
        VPC 10.0.0.0/16    ALB (public subnets AZ-a, AZ-b, HTTPS/ACM)
                            |  target group (ip), health /actuator/health/readiness
              +-------------+--------------+
        AZ-a  v                            v  AZ-b
     [ECS svc "api": task] [task]     [task] [task]        private-app subnets (Fargate, awsvpc)
              |   \                         \
              |    +--> ElastiCache Redis (primary AZ-a, replica AZ-b, auto-failover)
              |    +--> SQS "orders-queue" (+DLQ) --> [ECS svc "worker": tasks (Spot Fargate)] --> S3 / SES / 3rd party
              +--------> RDS PostgreSQL Multi-AZ (writer AZ-a, standby AZ-b)   private-data subnets
     NAT-a / NAT-b + endpoints (S3 gw, ecr.api, ecr.dkr, logs, sqs, secretsmanager)
     Secrets Manager (DB creds) | CloudWatch Logs/Alarms | X-Ray/ADOT | task role: sqs/s3 ; exec role: ecr/logs/secrets
```
Failure analysis
| Event | What happens | Your action / design point |
|---|---|---|
| **AZ-a dies** | ALB stops routing to AZ-a targets (health checks fail); CloudFront/ALB continue via AZ-b; ECS scheduler starts replacement tasks in AZ-b (needs headroom: run >=2 tasks per AZ or autoscale); RDS fails over to AZ-b (60-120 s of write errors); Redis replica promoted; NAT-a loss affects only AZ-a subnets | Capacity for N-1 AZ; retries/timeouts; verify SG/subnet space in AZ-b |
| **RDS failover** | Connections drop; DNS endpoint re-points; app errors for ~1-2 min then recovers if DNS TTL/pool config good | Hikari max-lifetime, JVM DNS TTL, retry idempotent ops, circuit breaker, queue writes via SQS; RDS Proxy shortens |
| **Bad deploy** | New tasks fail health check -> ECS circuit breaker halts and rolls back; if bug passes health checks but errors at runtime -> CloudWatch alarm (5xx/latency) -> CodeDeploy auto rollback (blue/green) or redeploy prior task-def revision | Small canary, readiness check meaningful, backward-compatible DB migrations, alarms tied to deploy |
| Redis down | cache-aside falls back to DB (must survive stampede) | Timeouts on Redis calls, circuit breaker |
| Worker poison message | retries up to maxReceiveCount then DLQ | Alarm on DLQ depth |
Approx. cost (small prod): Fargate 4 API tasks 1 vCPU/2 GB ~ $140-160; worker 2 tasks small ~ $30-70 (less with Spot); ALB ~ $20-40; 2 NAT ~ $65 + data (**watch this**); RDS Multi-AZ db.m6g.large Postgres ~ $200-300 + storage; ElastiCache 2x cache.t4g.small ~ $50-60; CloudFront+S3 ~ $5-30 low traffic; WAF ~ $10-20 + requests; CloudWatch/logs ~ $20-100; SQS pennies. Total roughly **$550-800/month** for a small HA setup; NAT + RDS + compute dominate.

### 17.2 (b) Same on EKS
```
Route53 -> CloudFront(+WAF) -> S3 (React)
                 | /api/*
                 v
           ALB  <-- created by AWS Load Balancer Controller from a K8s Ingress (target-type ip)
                 |
   EKS cluster (control plane managed, multi-AZ)                  Namespaces: shop, workers, monitoring
   +-----------------------------------------------------------------------------+
   | Managed node group / Karpenter nodes across AZ-a, AZ-b (private subnets)     |
   |  Deployment api (HPA) -> Pods (IRSA / Pod Identity role "orders-api")        |
   |  Deployment worker (KEDA scales on SQS depth)                                |
   |  DaemonSets: aws-node (VPC CNI), kube-proxy, fluent-bit, ADOT collector      |
   +-----------------------------------------------------------------------------+
        |--> RDS Multi-AZ | ElastiCache | SQS | Secrets via External Secrets Operator -> K8s Secret
```
Failure analysis
| Event | What happens | Notes |
|---|---|---|
| AZ dies | nodes in that AZ NotReady; pods rescheduled by the control plane (after node-monitor grace ~ minutes; faster if using pod topology spread + PDBs) to other AZ nodes; Karpenter/Cluster Autoscaler adds nodes; ALB drops unhealthy IPs | Use `topologySpreadConstraints` across zones, PodDisruptionBudgets, requests/limits, spare capacity |
| RDS failover | same as ECS | Same client-side fixes |
| Bad deploy | Deployment rolling update stalls when new pods fail readiness (`maxUnavailable`), `kubectl rollout undo` / Argo Rollouts / Flagger canary auto-rollback | `progressDeadlineSeconds`, readiness probe |
| Node/IP exhaustion | pods Pending (`insufficient IPs` with VPC CNI) | large subnets/prefix delegation |
| Control-plane upgrade | managed but add-ons/nodes are yours | schedule upgrades, test in staging |
Approx. cost: everything in 17.1 minus Fargate plus EKS control plane (~$70-75/month per cluster approx. - check) plus EC2 nodes (e.g. 3 x m6g.large ~ $180-220; better packing and Savings Plan/Spot reduce). Roughly **$650-950/month** small; economics improve with many services per cluster, worsen for one small service. Ops effort is the real extra cost.

### 17.3 (c) Serverless API
```
Route53 -> CloudFront (React on S3) ; /api -> API Gateway (HTTP API, JWT authorizer = Cognito)
                                               |
                                    Lambda (Java 21 + SnapStart, 1024-2048 MB)   [in VPC only if needed]
                                        |--> DynamoDB (on-demand)          (or RDS via RDS Proxy)
                                        |--> S3 (presigned URLs, events -> SQS -> Lambda)
                                        |--> SQS -> worker Lambda (batch, ReportBatchItemFailures, DLQ)
                                        |--> Step Functions (saga) ; EventBridge (domain events)
```
Failure analysis
| Event | What happens | Notes |
|---|---|---|
| AZ dies | Lambda, API GW, DynamoDB, SQS, S3 are multi-AZ by design; Lambda in VPC needs subnets in >=2 AZs | Fewest moving parts; nothing to fail over yourself |
| DB failover | DynamoDB: none visible. RDS Aurora via RDS Proxy: brief errors | Proxy essential |
| Bad deploy | Lambda **aliases with weighted traffic shifting** via CodeDeploy (canary 10%/5min) + alarms -> automatic rollback; or repoint alias | Publish versions, use aliases (SnapStart requires them) |
| Traffic spike | Lambda scales out fast but bounded by concurrency limits; downstream (RDS) gets crushed -> reserved concurrency as a brake, SQS as buffer | Throttling 429s |
| Cold starts | first requests slow | SnapStart/provisioned concurrency |
Approx. cost: at low/moderate traffic (millions of requests/month) tens of dollars: API GW HTTP API ~ $1 per million requests approx., Lambda GB-seconds + requests (free tier covers a lot), DynamoDB on-demand per million reads/writes, no idle cost. Becomes more expensive than containers at sustained high throughput. Typically **$10-100/month** for modest load, but cost scales linearly with traffic.

---

## 18. Worked design answers

### 18.1 Design a file-upload service
Requirements: users upload files up to ~2 GB, files private, virus-scanned, listed per user, 1M uploads/day.
```
React -> API (ECS) : POST /uploads {name,size,type}
API: authz, quota check, create DB row status=PENDING (id, key=uploads/{user}/{uuid}), return presigned URL
     (<= 100 MB: presigned PUT ; larger: CreateMultipartUpload + presigned UploadPart URLs per part)
Browser -> S3 direct (parallel parts, retry per part, Content-MD5/checksum) -> CompleteMultipartUpload via API
S3 ObjectCreated -> EventBridge/SQS -> Scan worker (ClamAV in Fargate / GuardDuty Malware Protection for S3)
      clean -> tag/move to "clean/" prefix, DB status=READY ; infected -> quarantine + alert
Download: API returns short-lived presigned GET (or CloudFront signed URL/cookie) after authz
Lifecycle: abort incomplete multipart 1 day; IA after 30 d; Glacier per policy
```
Key points: bytes never pass through the app (cheap, scalable); **validate server-side** (size via `content-length-range` with presigned POST, type by scanning not extension); unguessable keys, per-user prefix; SSE-KMS; block public access; CORS; idempotent event handling (S3 events may duplicate); DB row ties metadata; orphan cleanup job for PENDING > 24 h; CloudFront in front for downloads; rate limit; audit through CloudTrail data events if required. Failure modes: upload abandoned (lifecycle abort), duplicate events (idempotent), scanner down (queue buffers, files remain PENDING), malicious large uploads (quota, WAF).

### 18.2 Design a scalable image-processing pipeline
```
Client -> presigned PUT -> S3 "originals"
S3 event -> SQS "image-jobs" (DLQ)  ---(depth)---> autoscale workers
Workers: ECS Fargate (Spot) or Lambda (memory-heavy, <=15 min) : read original, resize to N sizes (thumbnails/webp), write S3 "derived/"
Update DynamoDB/RDS status ; publish "image.processed" to SNS/EventBridge -> notify UI (WebSocket/poll)
Serving: CloudFront -> S3 derived (immutable cache-control) ; optional on-the-fly resize via Lambda@Edge/CloudFront Function + cache
```
Decisions: **SQS between S3 and workers** absorbs bursts, gives retries and DLQ; workers scale on `ApproximateNumberOfMessagesVisible / running tasks` (backlog per instance) via target tracking (ECS) or KEDA; **idempotent output keys** (`derived/{id}/{size}.webp`) so duplicates just overwrite; visibility timeout > max processing; large images use Fargate (more CPU/RAM, no 15 min cap), small bursty ones Lambda; Spot for cost with graceful handling of reclaim (message reappears). Java: use `libvips` (via native) or ImageIO/Thumbnailator; watch heap for large bitmaps. Poison images -> DLQ + alarm. Cost levers: Spot, WebP/AVIF, lifecycle on originals, CloudFront hit ratio.

### 18.3 Migrate a monolith to AWS
1. **Discover/assess** (dependencies, data size, RPO/RTO, compliance, traffic pattern). Pick the **6 Rs**: Rehost (lift-and-shift via AWS Application Migration Service), Replatform (managed DB, containers), Refactor, Repurchase, Retire, Retain.
2. **Landing zone** first: multi-account (Control Tower/Organizations), SSO, VPC design, logging account, guardrails (SCPs), IaC pipeline.
3. **Phase 1 - rehost/replatform**: containerise the Spring Boot monolith to ECS Fargate (or EC2 ASG) behind ALB; DB to RDS via **AWS DMS** (full load + CDC) with schema conversion tool if engines differ; keep it stateless (sessions to Redis, files to S3, config to Parameter Store/Secrets Manager).
4. **Cutover**: rehearse; sync data with CDC; lower DNS TTL beforehand; **weighted Route 53** (5% -> 100%); keep rollback path (reverse replication) until confidence; freeze window.
5. **Phase 2 - strangler fig**: put ALB/API Gateway in front; peel off modules by domain (notifications, image processing, search) into services/Lambda using events (SQS/SNS/EventBridge); each service owns its data; anti-corruption layer.
6. **Operate**: observability, autoscaling, Savings Plans, DR plan; decommission old datacentre.
Risks: data gravity/downtime (DMS CDC), hard-coded IPs/file paths/local disk state, licence issues, latency to on-prem dependencies (VPN/Direct Connect), hidden batch jobs/cron, session stickiness, network cutover. Answer style: incremental, measurable, reversible.

---

## 19. Production war stories (symptom -> diagnosis -> fix)

### 19.1 NAT Gateway bill shock
- **Symptom**: monthly bill jumps by thousands; Cost Explorer shows "NatGateway-Bytes" / EC2-Other.
- **Diagnosis**: VPC Flow Logs (or CloudWatch `BytesOutToDestination`) show huge traffic from private subnets to S3/ECR/DynamoDB public IPs; a batch job reading TBs from S3 or every Fargate task pulling 1 GB images through NAT; cross-AZ NAT usage adds data charges.
- **Fix**: S3 + DynamoDB **gateway endpoints** (free), ECR/logs interface endpoints, slim images, one NAT per AZ with AZ-local routing, budget alarm + anomaly detection.

### 19.2 S3 public leak
- **Symptom**: security researcher/GuardDuty/Config says customer files are public.
- **Diagnosis**: someone added a bucket policy with `"Principal": "*"` or ACL `public-read`, or an old bucket predating Block Public Access; `aws s3api get-bucket-policy-status`, Access Analyzer finding, Config rule `s3-bucket-public-read-prohibited`.
- **Fix**: enable **Block Public Access at account level**, remove policy/ACLs, disable ACLs (bucket owner enforced), serve public content only via CloudFront OAC, rotate/invalidate anything exposed, review CloudTrail/S3 access logs for exfiltration, notify per policy, add SCP denying `s3:PutBucketPublicAccessBlock` changes and Config auto-remediation.

### 19.3 IAM AccessDenied hunt
- **Symptom**: `AccessDeniedException: User: arn:aws:sts::111:assumed-role/orders-task/abc is not authorized to perform: kms:Decrypt` after a new secret deployed; works in dev.
- **Diagnosis**: read the ARN (right role - yes); identity policy allowed `secretsmanager:GetSecretValue` but the secret uses a **CMK**; KMS key policy in prod didn't allow that role; check with CloudTrail `errorMessage`, policy simulator. (Other real causes: SCP deny on region, permissions boundary, resource ARN typo, missing `/*`.)
- **Fix**: grant `kms:Decrypt` in identity policy **and** allow the role in the key policy (or use `kms:ViaService`); add the case to IaC; add Access Analyzer/policy validation in CI.

### 19.4 ALB 502/504 from timeout mismatch
- **Symptom**: sporadic 502 under low traffic, 504 on long reports.
- **Diagnosis**: 502 appears after idle periods: **Tomcat keep-alive (20-60 s) shorter than or equal to ALB idle timeout 60 s** so app closes connections ALB reuses. 504: report endpoint takes 90 s; ALB idle timeout 60 s (`ELB 504`), also API Gateway ~29 s limit or downstream client timeout. Confirm with ALB access logs (`elb_status_code`, `target_status_code` `-`), `RequestCount`, target reset counters.
- **Fix**: app keep-alive > ALB idle timeout (e.g. 75 s vs 60 s); raise ALB idle timeout only if truly needed; make long jobs async (202 + status endpoint / SQS); keep timeouts monotonically ordered: client < ALB < app < DB.

### 19.5 RDS failover connection storm
- **Symptom**: after a Multi-AZ failover (or reboot) the app stays broken 10+ minutes; DB CPU 100% after recovery; "too many connections".
- **Diagnosis**: JVM cached old IP (DNS TTL), Hikari holds dead connections, all tasks reconnect at once with retries without jitter; 200 tasks x pool 20 exceeds `max_connections`; sudden reconnect + cold cache = load spike.
- **Fix**: `networkaddress.cache.ttl` low, Hikari `max-lifetime`/`keepalive`, sane pool sizes with the budget formula, **RDS Proxy**, exponential backoff + jitter, circuit breaker, warm cache; test failover in staging (`reboot with failover`).

### 19.6 Lambda cold start timeouts
- **Symptom**: first API calls after quiet periods take 6-10 s; API GW returns 504 (29 s limit) or client times out at 3 s; SQS-triggered batches fail intermittently.
- **Diagnosis**: CloudWatch `Init Duration` in REPORT lines; Spring Boot full context + Hibernate init; VPC + RDS connection creation; 512 MB memory = slow CPU.
- **Fix**: SnapStart (with CRaC hooks for connections), raise memory (CPU), provisioned concurrency for user-facing paths, lazy init/less classpath scanning, RDS Proxy, or move steady traffic to Fargate.

### 19.7 ASG flapping (instances launched and killed repeatedly)
- **Symptom**: instance count oscillates, ALB target group shows targets registering/deregistering, CPU never stabilises, deployments never finish.
- **Diagnosis**: ASG uses ELB health checks with **grace period 60 s** but app needs 120 s to start => marked unhealthy and terminated in a loop. Or scaling: alarm with 1 datapoint on CPU without warm-up, scale-out then immediate scale-in; or health check path returns 500 when DB blips (all instances killed at once).
- **Fix**: grace period > boot time (or warm pool/lifecycle hook, AMI baked); target tracking with appropriate instance warm-up; scale-in cooldown/longer evaluation; shallow health checks (no DB dependency in ELB check); check ASG activity history and `Description` of terminations.

---

## 20. Interview questions (55) - graded, with model answers, follow-ups, wrong answers

Format: **Q (level)** - model answer. *Follow-ups* chain deeper. **Wrong:** the answer that loses marks.

### IAM
**Q1 (Easy) Role vs user?** A user has long-term credentials; a role has none and is assumed to get temporary credentials via STS. Workloads and cross-account use roles. *Follow-up:* how does an EC2 get a role? (instance profile -> IMDS.) Why better than keys? (auto-rotation, no secret to leak, audit by session.) **Wrong:** "role is a group of users".

**Q2 (Easy) What is least privilege and how do you apply it?** Grant only needed actions on only needed resources with conditions; start narrow, use Access Analyzer/CloudTrail-based generation, review last-accessed. *Follow-up:* how do you keep it from rotting? (policy validation in CI, periodic access review.) **Wrong:** "attach AdministratorAccess to avoid failures".

**Q3 (Medium) Trust policy vs permission policy?** Trust = who can assume the role; permission = what the role can do. Both needed. *Follow-up:* error "not authorized to perform sts:AssumeRole" - which one? (trust policy and/or caller's own sts:AssumeRole permission.) **Wrong:** "trust policy defines S3 access".

**Q4 (Medium) Explain policy evaluation order.** Default deny; any explicit deny wins; SCPs and boundaries must allow (they only filter); then an Allow must exist in identity policy or (same account) resource policy; cross-account needs both sides. *Follow-up:* SCP allows S3 but user has no policy - result? (denied.) Resource policy names role ARN same account, no identity policy - result? (allowed.) **Wrong:** "allow always wins over deny" / "SCP grants permissions".

**Q5 (Medium) Cross-account access to an S3 bucket - options?** (1) Bucket policy naming the other account principal + their IAM policy allows (both sides), or (2) role in bucket-owner account that the other account assumes. Role assumption keeps ownership/audit cleaner and avoids ACL ownership problems. *Follow-up:* KMS-encrypted objects? (key policy must also allow the external principal.) **Wrong:** "just make bucket public".

**Q6 (Medium) IRSA vs Pod Identity?** IRSA uses cluster OIDC provider + `AssumeRoleWithWebIdentity` with service-account annotation and `sub` condition; Pod Identity uses an agent add-on + association, trust principal `pods.eks.amazonaws.com`, role reusable across clusters, no OIDC provider setup. *Follow-up:* what breaks if the IRSA trust lacks `sub` condition? (any SA in the cluster can assume it.) **Wrong:** "put access keys in a Kubernetes Secret".

**Q7 (Hard) A Spring Boot app on ECS gets AccessDenied on S3 but works on my laptop. Debug.** Laptop uses my profile (admin-ish); in ECS the **task role** applies. Check: which ARN in error, task role permissions on bucket vs object ARN, bucket policy deny, SCP, KMS, VPC endpoint policy, and that the container actually uses the task role (no stale env var keys). CloudTrail for exact denial. *Follow-up:* how would you have caught it earlier? (integration tests against LocalStack with role-like policy, IAM Access Analyzer in CI.) **Wrong:** "add s3:* on *".

**Q8 (Hard) How do you let GitHub Actions deploy without storing keys?** OIDC provider + role with `AssumeRoleWithWebIdentity` trust conditioned on `aud` and `sub` (repo/branch/environment); workflow `id-token: write`; short-lived creds. *Follow-up:* what's the risk with `sub` wildcard? (any repo/branch can deploy prod.) **Wrong:** "store keys in GitHub secrets and rotate yearly".

**Q9 (Hard) What does an SCP do and not do? Does it affect the management account?** Sets max permissions for member accounts' principals (including their root); grants nothing; not applied to the management account; use deny-list style with region lock and protection of security services. *Follow-up:* difference from permissions boundary? (boundary is per-identity in one account; SCP is org/account-wide.) **Wrong:** "SCP gives users permission".

### VPC
**Q10 (Easy) Public vs private subnet?** Public has a route to an Internet Gateway (and public IPs); private doesn't and reaches internet via NAT. It is a routing property. **Wrong:** "public subnets are configured with a checkbox".

**Q11 (Easy) Security group vs NACL?** SG: instance-level, stateful, allow-only; NACL: subnet-level, stateless, ordered allow/deny. *Follow-up:* return traffic on a NACL? (ephemeral ports must be allowed.) **Wrong:** "NACLs are stateful".

**Q12 (Medium) Why put the database in a private subnet and how do you reach it for debugging?** No internet route, SG allows only app-sg; debug through SSM Session Manager port forwarding (no inbound port, IAM authorised, audited) rather than a bastion with SSH. *Follow-up:* what does SSM need? (agent, instance profile, network path/endpoints.) **Wrong:** "make RDS publicly accessible and IP-whitelist myself".

**Q13 (Medium) NAT Gateway: how does it work, cost, HA?** Zonal managed NAT in a public subnet with EIP; private route `0.0.0.0/0 -> nat`; billed hourly + per GB; for HA one per AZ with AZ-local routes. *Follow-up:* how cut the bill? (gateway endpoints for S3/DynamoDB, interface endpoints, avoid cross-AZ.) **Wrong:** "NAT gateway is free like IGW".

**Q14 (Medium) Gateway vs interface endpoints?** Gateway: S3/DynamoDB only, route-table entry, free. Interface: ENI + private DNS via PrivateLink for most services and your own, paid. *Follow-up:* Fargate in private subnet pulling from ECR without NAT? (ecr.api, ecr.dkr, s3 gateway, logs.) **Wrong:** "S3 endpoint is an ENI".

**Q15 (Medium) Peering vs Transit Gateway vs PrivateLink?** Peering point-to-point non-transitive; TGW hub for many VPCs/on-prem with transitive routing; PrivateLink exposes a single service one-way with no CIDR constraints. *Follow-up:* overlapping CIDRs? (peering/TGW fail; PrivateLink works.) **Wrong:** "peering is transitive".

**Q16 (Hard) Design VPC CIDRs for 3 environments + EKS.** Non-overlapping /16 per env, /24 public, larger private subnets (/20-/19) per AZ for pods (VPC CNI IP per pod), separate small data subnets, reserve space; consider secondary CIDR/prefix delegation. Five IPs reserved per subnet. *Follow-up:* what if we later need to connect to on-prem with 10.0.0.0/16? (overlap problem - plan up front, or NAT/PrivateLink workaround.) **Wrong:** "/28 subnets to save IPs".

### Compute / ELB
**Q17 (Easy) On-demand vs Reserved vs Savings Plan vs Spot?** On-demand flexible; RI/SP commit $ for discount; Spot spare capacity up to ~90% off with 2-min interruption. Baseline SP, burst on-demand, stateless batch Spot. **Wrong:** "Spot for the primary database".

**Q18 (Medium) ASG scaling policies?** Target tracking (default; keep metric at target), step (tiered alarms), scheduled (known events), predictive. Health check type ELB + grace period. *Follow-up:* what metric for a web tier? (ALBRequestCountPerTarget or CPU.) **Wrong:** "ASG scales on memory by default" (memory needs custom metrics).

**Q19 (Medium) ALB vs NLB?** ALB L7 routing, WAF, auth, Lambda targets; NLB L4, static IPs, extreme throughput/low latency, TLS pass-through, PrivateLink service. *Follow-up:* cross-zone defaults? (ALB on, NLB off.) When NLB for Spring API? (static IPs, non-HTTP protocol.) **Wrong:** "NLB does path-based routing".

**Q20 (Medium) Explain deregistration delay and connection draining in deploys.** Target enters draining, gets no new requests, in-flight can finish up to delay (default 300 s). Tune to p99 + margin and align with app graceful shutdown. *Follow-up:* symptom if too low? (5xx during deploys.) **Wrong:** "it delays health checks".

**Q21 (Hard) You see intermittent 502s from ALB. Causes?** Keep-alive mismatch (app timeout <= ALB 60 s idle), app crash/OOM restarts, target reset/closed connection, deployment killing tasks without draining, oversized headers. 504: target slower than timeout / SG blocking. Use ALB logs (`elb_status_code` vs `target_status_code`), target 5XX metrics. *Follow-up:* fix? (Tomcat keep-alive 75 s, graceful shutdown, deregistration delay.) **Wrong:** "increase instance size".

**Q22 (Hard) Health checks - what path and what should it check?** Dedicated unauthenticated readiness endpoint (`/actuator/health/readiness`) on traffic port; shallow (app-level), not a deep DB check that could drain all targets during a DB blip; grace period beyond startup. **Wrong:** "use `/` since it returns 200 in dev".

### Containers
**Q23 (Easy) ECS vs Fargate?** ECS is the orchestrator; Fargate/EC2 is where tasks run. Fargate = serverless compute (no instances to manage). **Wrong:** "Fargate is a separate service from containers".

**Q24 (Medium) Task role vs execution role?** Execution role: ECS agent pulls ECR image, writes logs, fetches secrets for injection. Task role: your app's AWS API access. *Follow-up:* which for `CannotPullContainerError`? (execution role/network.) **Wrong:** "same role for both, it doesn't matter" (works but violates least privilege and confuses debugging).

**Q25 (Medium) ECS vs EKS - how do you choose?** ECS: simple AWS-native, low ops; EKS: Kubernetes API/portability/ecosystem, higher ops and cost; Lambda for event-driven; Beanstalk for lift-and-shift PaaS. Decision by team skills, portability, multi-team scale. *Follow-up:* what extra do you own in EKS? (nodes, add-ons, upgrades, CNI IP planning, RBAC mapping.) **Wrong:** "EKS is always better, it's Kubernetes".

**Q26 (Hard) How do ECS rolling deployments avoid downtime and roll back?** minHealthy/max percent start new tasks, register in target group, pass health checks (after grace), drain old; **circuit breaker** rolls back on repeated task failure; alarms-based rollback; blue/green with CodeDeploy for instant switch. DB migrations backward-compatible. *Follow-up:* a bug passing health checks? (alarm-based rollback/canary.) **Wrong:** "restart tasks in place during peak".

**Q27 (Hard) Map an EKS ingress request end to end.** Route53 -> CloudFront/ALB (created by AWS LB Controller from Ingress, target type ip) -> pod IP directly (VPC CNI) -> Service is only for controller discovery; readiness gates; pod uses IRSA/Pod Identity for AWS calls. *Follow-up:* NLB instead? (Service type LoadBalancer annotation.) **Wrong:** "traffic goes through kube-proxy NodePort always" (true only for instance target type).

### Lambda / serverless
**Q28 (Easy) What is a cold start?** Time to create an environment and initialise runtime/code before handler; Java worst due to JVM+framework. **Wrong:** "delay when Lambda is throttled".

**Q29 (Medium) How to mitigate Java cold starts?** SnapStart (with CRaC hooks), provisioned concurrency, GraalVM native, lighter frameworks, more memory, lazy init, reuse clients outside handler. *Follow-up:* SnapStart pitfalls? (uniqueness of secrets/random/connections; only published versions.) **Wrong:** "keep the function warm with a scheduled ping" (fragile, doesn't handle concurrency).

**Q30 (Medium) How do memory, CPU, timeout, concurrency relate?** CPU scales with memory; timeout max 15 min; concurrency = simultaneous environments, account limit ~1000 default (raisable), reserved vs provisioned. Cost = GB-seconds; tune memory. **Wrong:** "CPU is configured separately".

**Q31 (Medium) Lambda with SQS - failure handling?** Event source mapping polls; batch failures return partial batch failures; messages return after visibility timeout; DLQ on the queue via maxReceiveCount; visibility timeout >= function timeout (guidance ~6x); idempotent handlers. *Follow-up:* one bad message in a batch? (`ReportBatchItemFailures`.) **Wrong:** "configure DLQ on the Lambda for SQS triggers".

**Q32 (Hard) Lambda hitting RDS - problems and fixes?** Connection storms (each env holds a connection), VPC/subnet setup, secrets. RDS Proxy, small pool size 1, reserved concurrency cap, IAM auth, or DynamoDB/Data API. *Follow-up:* internet access from that Lambda? (NAT/endpoints needed.) **Wrong:** "put Lambda in public subnet to reach internet".

**Q33 (Medium) API Gateway REST vs HTTP vs ALB vs Function URL?** HTTP API cheaper/simpler with JWT; REST for API keys/usage plans/WAF/caching/transform; ALB good for containers and Lambda at steady traffic; Function URL for simple endpoints. Timeout limits differ. **Wrong:** "REST API is deprecated".

**Q34 (Hard) Implement a saga on AWS.** Step Functions orchestration with retries/Catch and compensating tasks, idempotent steps with idempotency keys; or choreography with EventBridge/SNS + SQS and outbox pattern; discuss Standard vs Express. *Follow-up:* compensation failure? (retry with backoff, manual queue, alarms.) **Wrong:** "use distributed 2PC across services".

### Storage
**Q35 (Easy) S3 consistency model?** Strong read-after-write and list consistency since Dec 2020. **Wrong:** "eventual consistency for overwrites".

**Q36 (Medium) How to let a browser upload big files safely?** Presigned URL/POST (or multipart presigned parts) issued by authenticated API, short expiry, key chosen by server, CORS, size/type constraints, post-upload scan; bytes bypass server. *Follow-up:* expiry limit? (up to 7 days for IAM user keys; with role creds bounded by credential lifetime.) **Wrong:** "stream through Spring controller, it's easier".

**Q37 (Medium) Bucket policy vs ACL vs IAM; Block Public Access?** IAM for identities, bucket policy for resource/cross-account/conditions, ACLs legacy (disabled by default now). BPA overrides public grants and is on by default on new buckets. *Follow-up:* serve website privately? (CloudFront OAC.) **Wrong:** "enable static website hosting publicly for React".

**Q38 (Medium) SSE-S3 vs SSE-KMS vs SSE-C?** S3-managed keys (default), KMS keys with audit/control/cost/throttle (use Bucket Keys), customer-supplied keys per request. *Follow-up:* extra IAM needed for SSE-KMS? (kms:GenerateDataKey/Decrypt + key policy.) **Wrong:** "S3 objects are unencrypted unless configured" (default SSE-S3 since 2023).

**Q39 (Medium) EBS vs EFS vs instance store?** EBS per-AZ block, EFS multi-AZ shared NFS, instance store ephemeral local. **Wrong:** "EBS volume can be attached to instances in other AZ".

**Q40 (Hard) Storage cost reduction for 500 TB of logs/media?** Lifecycle to IA/Glacier, Intelligent-Tiering for unpredictable, compress/columnar (Parquet), expire noncurrent versions, abort multipart, minimise small objects (aggregate), analyse with Storage Lens; consider retrieval costs/minimum durations. **Wrong:** "move everything to Glacier Deep Archive" (retrieval time/cost).

### Databases
**Q41 (Easy) Multi-AZ vs read replica?** Multi-AZ: synchronous standby for HA, automatic DNS failover, no reads. Replica: async for read scaling/DR, lag, manual promotion. **Wrong:** "Multi-AZ improves read throughput".

**Q42 (Medium) What happens in an RDS failover from the app's view?** Primary fails, standby promoted, DNS CNAME flips (~60-120 s), connections dropped; app needs reconnect, DNS TTL low, retry with backoff; RDS Proxy speeds. *Follow-up:* Aurora difference? (readers already share storage; failover typically well under a minute.) **Wrong:** "app connects to primary's IP so no issue" (IP changes).

**Q43 (Medium) Aurora vs RDS?** Aurora: distributed 6-way replicated storage across 3 AZs, up to 15 low-lag readers, fast failover, auto-growing storage, Serverless v2; higher per-hour cost, vendor lock-in; RDS simpler/cheaper for small. **Wrong:** "Aurora is just RDS with SSD".

**Q44 (Medium) DynamoDB vs RDS for an order system?** Orders need relational integrity/reporting -> RDS/Aurora; DynamoDB for known access patterns at scale (carts, sessions, dedupe). *Follow-up:* hot partition? (skewed key; shard keys, better cardinality, caching.) **Wrong:** "DynamoDB is always cheaper and faster".

**Q45 (Hard) Design a DynamoDB table for "orders by customer, and latest orders per status".** `PK=CUSTOMER#id, SK=ORDER#timestamp#orderId` for customer queries; GSI on `status` + `createdAt` - but `status` as PK is hot; use sharded status (`STATUS#PENDING#03`) or query by time bucket; strongly consistent read not available on GSI (eventual); use conditional writes for state transitions; TTL for expiry. **Wrong:** "Scan with a filter on status".

**Q46 (Medium) Cache patterns with ElastiCache and their pitfalls?** Cache-aside w/ TTL and evict on write; write-through; stampede protection (locks, jitter, request coalescing), stale data, cold cache after failover, eviction policy, timeouts with fallback. **Wrong:** "cache is source of truth".

### Messaging / edge / security / observability / delivery
**Q47 (Medium) SQS standard vs FIFO, visibility timeout, DLQ?** Standard: at-least-once, unordered, unlimited throughput. FIFO: ordered per group, dedupe, lower throughput. Visibility timeout hides in-flight messages; too short => duplicates. DLQ after maxReceiveCount. Consumers must be idempotent. *Follow-up:* how do you make consumer idempotent? (dedupe key store, conditional write, natural idempotency.) **Wrong:** "SQS guarantees exactly-once on standard queues".

**Q48 (Medium) Kinesis vs MSK vs SQS?** SQS work queue (competing consumers); Kinesis AWS-native replayable ordered stream; MSK Kafka compatibility/ecosystem/ops. SNS->SQS for fan-out. Amazon MQ for lift-and-shift RabbitMQ/JMS. **Wrong:** "Kinesis and SQS are the same".

**Q49 (Medium) Route 53 routing policies and failover?** Simple, weighted (canary), latency, failover with health checks (active-passive), geolocation; alias records; TTL bounds failover speed; health checks from outside VPC need public endpoint or CloudWatch-alarm-based. **Wrong:** "DNS failover is instantaneous".

**Q50 (Medium) CloudFront in front of S3 for React - details?** Private bucket + OAC, ACM cert in us-east-1, SPA fallback errors -> index.html, hashed assets long cache, short `index.html`, invalidation, `/api/*` behavior to ALB with no caching, WAF. **Wrong:** "make the bucket public and use S3 website endpoint over HTTP".

**Q51 (Medium) Envelope encryption and KMS?** GenerateDataKey -> encrypt data locally with data key -> store encrypted data key alongside; KMS only touches small keys; audit via CloudTrail; key policy is authority. **Wrong:** "send the file to KMS to encrypt" (4 KB limit).

**Q52 (Medium) Secrets Manager vs Parameter Store?** Secrets Manager: rotation, larger, paid per secret, replication; Parameter Store: config + cheap SecureString, no built-in rotation. Access via task role, cache values. **Wrong:** "put passwords in environment variables in the task definition in plain text".

**Q53 (Hard) A deployment causes elevated 5xx; what safe process and tooling?** Alarms (ALB 5xx/latency) as deployment gates, canary/blue-green with auto rollback, circuit breaker, feature flags, backward-compatible DB migrations, observability with traces and logs Insights to diagnose, post-mortem. **Wrong:** "hotfix directly in production".

**Q54 (Hard) HA/DR: RPO 5 min, RTO 30 min across region loss - propose.** Warm standby or pilot light: Aurora Global Database/cross-region replicas, S3 CRR, infra by IaC in DR region, Route 53 failover with health checks, secrets/certs replicated, regular drills. Explain cost tradeoff vs active-active, and that Multi-AZ does not protect region loss. *Follow-up:* logical corruption? (PITR/versioning/backup vault lock; replication copies mistakes.) **Wrong:** "Multi-AZ RDS is our DR".

**Q55 (Hard) Cut a $30k/month AWS bill by 30% - plan.** Cost Explorer by service/tag; check NAT/data transfer, idle resources, oversize instances, logs retention, S3 lifecycle; Savings Plans for baseline, Spot workers, Graviton, right-size RDS, scale dev to zero; budgets and anomaly detection; ownership via tags. Measure before/after. **Wrong:** "switch everything to Spot".

---

## 21. One-page cheat sheet

**Identity**
- Roles > keys. Trust policy = who assumes; permission policy = what it can do.
- Eval: default deny -> explicit deny wins -> SCP/boundary/session must allow -> some Allow (cross-account: both sides).
- S3: `ListBucket` on bucket ARN, `Get/PutObject` on `/*`. KMS key policy must allow.
- Workloads: EC2 instance profile, ECS task role (app) vs execution role (pull/logs/secrets), Lambda execution role, EKS IRSA/Pod Identity, CI = OIDC.
- Creds chain (SDK v2): system props -> env -> web identity -> profile -> container -> instance profile.

**Network**
- Public subnet = route to IGW. NAT in public subnet, one per AZ, costs per hour + GB. Endpoints: gateway (S3, DynamoDB, free), interface (rest, paid).
- SG stateful/allow-only/SG-to-SG; NACL stateless/ordered. 5 IPs reserved/subnet. CIDR: no overlaps.
- Peering non-transitive; TGW hub; PrivateLink one service. SSM instead of bastion.
- Layout: ALB public x2 AZ -> app private x2 AZ -> data private x2 AZ.

**Compute**
- ASG: launch template, target tracking, ELB health + grace period > boot. Spot 2-min notice.
- ALB L7 (path/host, target groups, deregistration delay 300 s default, idle 60 s, app keep-alive > idle). NLB L4 static IP.
- Fargate default; ECS simple, EKS for K8s needs. Task role vs exec role.
- Lambda: 15 min, memory = CPU, cold start fix = SnapStart/provisioned; RDS Proxy; DLQ on queue for SQS.

**Data**
- S3: strong consistency, versioning, lifecycle, presigned URLs, BPA on, SSE-S3 default, multipart >100 MB (req >5 GB), CloudFront OAC.
- RDS Multi-AZ = HA (sync, ~60-120 s, DNS flip); replica = scale (async lag). PITR = new instance. Aurora: 6 copies/3 AZs, <=15 readers, fast failover.
- DynamoDB: PK design, hot partitions, GSI eventual/LSI at creation, on-demand vs provisioned, streams/TTL, conditional writes.
- ElastiCache: cache-aside + TTL, beware stampede.

**Messaging**
- SQS at-least-once => idempotent; visibility timeout > processing; long poll 20 s; DLQ maxReceiveCount. FIFO ordered per group.
- SNS -> SQS fan-out; EventBridge routing; Kinesis/MSK for replayable streams; Amazon MQ for lift-and-shift Rabbit/JMS.

**Edge/Security/Ops**
- Route 53: alias, weighted/latency/failover/geo, TTL. CloudFront: cache key, invalidation, ACM in us-east-1. WAF L7, Shield L3/4.
- KMS envelope: GenerateDataKey. Secrets Manager (rotation) vs Parameter Store (config).
- CloudTrail=audit, Config=compliance, GuardDuty=threats, CloudWatch=metrics/logs/alarms/Insights/EMF, X-Ray/ADOT=traces.

**Delivery / Resilience / Cost**
- IaC: CloudFormation (AWS state), Terraform (own state), CDK (code -> CFN). GitHub Actions OIDC, pin `sub`.
- Deploy: rolling, blue/green, canary; backward-compatible migrations; circuit breaker + alarms.
- DR: backup/restore -> pilot light -> warm standby -> active-active; RPO=data loss, RTO=downtime; Multi-AZ != DR.
- Cost: NAT/data transfer, endpoints, Savings Plans + Spot, lifecycle, log retention, tags + budgets.

**Spring Boot**: health path `/actuator/health/readiness` (unauthenticated), grace period > startup, `server.shutdown=graceful`, keep-alive 75 s, Hikari max-lifetime < DB timeouts, pool x tasks < max_connections, DB endpoint DNS with low JVM TTL, secrets via Secrets Manager import.
