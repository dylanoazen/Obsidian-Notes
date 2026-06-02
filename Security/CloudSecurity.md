# Cloud Security (AWS)

Securing infrastructure, data, and workloads in AWS.

Related: [[Security/IAM]]

---

## Shared Responsibility Model

```
AWS manages:                    You manage:
├── Physical security           ├── IAM policies and users
├── Hardware                    ├── Security groups / NACLs
├── Hypervisor                  ├── OS patching (EC2)
├── Network infrastructure      ├── Application code
└── Managed service internals   ├── Data encryption
                                ├── Logging and monitoring
                                └── Compliance
```

The higher up the stack you go (Lambda > ECS > EC2), the less you manage.

## IAM Security

```
Principle of Least Privilege:
  Don't: "Effect": "Allow", "Action": "*", "Resource": "*"
  Do:    "Effect": "Allow", "Action": "s3:GetObject", "Resource": "arn:.../*"
```

- Use **IAM Roles** for services (not access keys)
- Enable **MFA** on all human accounts
- Use **AWS Organizations + SCPs** to set guardrails across accounts
- Rotate credentials regularly
- Use **IAM Access Analyzer** to find overly permissive policies

## Network Security

```
VPC
├── Public Subnet (internet-facing)
│   ├── ALB (Application Load Balancer)
│   └── NAT Gateway
│
├── Private Subnet (no direct internet)
│   ├── EC2 instances
│   ├── RDS databases
│   └── ECS tasks
│
├── Security Groups (stateful firewall, per-resource)
│   └── Allow port 443 from 0.0.0.0/0
│
└── NACLs (stateless firewall, per-subnet)
    └── Allow/Deny rules evaluated in order
```

- Put databases in private subnets
- Use VPC endpoints for AWS services (no internet traversal)
- Enable VPC Flow Logs

## Encryption

**At rest:**
- S3: SSE-S3, SSE-KMS, or SSE-C
- EBS: KMS-encrypted volumes
- RDS: KMS encryption at creation (can't add later)

**In transit:**
- Enforce TLS everywhere (ALB → target, service → service)
- Use ACM (AWS Certificate Manager) for free TLS certs

**KMS (Key Management Service):**
```bash
# Create a key
aws kms create-key --description "app encryption key"

# Encrypt data
aws kms encrypt --key-id alias/mykey --plaintext "secret"

# Key policies control who can use/manage keys
```

## Logging and Monitoring

| Service | Purpose |
|---------|---------|
| CloudTrail | API call audit log (who did what) |
| CloudWatch | Metrics, logs, alarms |
| GuardDuty | Threat detection (ML-based) |
| Security Hub | Central security findings dashboard |
| Config | Track resource configuration changes |
| VPC Flow Logs | Network traffic logs |

```bash
# Enable CloudTrail (should be on in every account)
aws cloudtrail create-trail --name main --s3-bucket-name my-trail-bucket

# GuardDuty — enable and forget
aws guardduty create-detector --enable
```

## Container Security on AWS

- **ECR scanning**: automatic vulnerability scanning on push
- **ECS**: use Fargate (no EC2 to patch), task roles (least privilege)
- **EKS**: pod security standards, IRSA (IAM Roles for Service Accounts)

## Secrets Management

```bash
# AWS Secrets Manager
aws secretsmanager create-secret --name prod/db-password --secret-string "s3cret"

# Retrieve in code
aws secretsmanager get-secret-value --secret-id prod/db-password
```

Never put secrets in: environment variables in console, code, CloudFormation parameters.

## Security Checklist

- Enable CloudTrail in all regions
- Enable GuardDuty
- No IAM users with console access without MFA
- No access keys on root account
- S3 buckets: block public access by default
- Enable default EBS encryption
- Use Security Hub for aggregated findings
- Regular IAM Access Analyzer reviews
- Tag everything for accountability

## Related

- [[Security/IAM]]
- [[Security/ThreatModeling]]
- [[Security/SupplyChainSecurity]]
- [[Linux/Security]]

## Resources

- https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/
- https://d1.awsstatic.com/whitepapers/aws-security-best-practices.pdf

#### My commentaries
- 
