# Cloud Security (AWS)

Segurança de infraestrutura, dados e workloads na AWS.

Related: [[Security/IAM]]

---

## Modelo de Responsabilidade Compartilhada

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

Quanto mais alto na pilha você vai (Lambda > ECS > EC2), menos você gerencia.

## IAM Security

```
Principle of Least Privilege:
  Don't: "Effect": "Allow", "Action": "*", "Resource": "*"
  Do:    "Effect": "Allow", "Action": "s3:GetObject", "Resource": "arn:.../*"
```

- Use **IAM Roles** para serviços (não access keys)
- Habilite **MFA** em todas as contas humanas
- Use **AWS Organizations + SCPs** para definir guardrails entre contas
- Rotacione credenciais regularmente
- Use o **IAM Access Analyzer** para encontrar políticas excessivamente permissivas

## Segurança de Rede

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

- Coloque databases em subnets privadas
- Use VPC endpoints para serviços AWS (sem travessia pela internet)
- Habilite VPC Flow Logs

## Criptografia

**Em repouso:**
- S3: SSE-S3, SSE-KMS, ou SSE-C
- EBS: volumes criptografados com KMS
- RDS: criptografia KMS na criação (não pode ser adicionada depois)

**Em trânsito:**
- Enforce TLS em todo lugar (ALB → target, serviço → serviço)
- Use ACM (AWS Certificate Manager) para certificados TLS gratuitos

**KMS (Key Management Service):**
```bash
# Create a key
aws kms create-key --description "app encryption key"

# Encrypt data
aws kms encrypt --key-id alias/mykey --plaintext "secret"

# Key policies control who can use/manage keys
```

## Logging e Monitoramento

| Serviço | Propósito |
|---------|---------|
| CloudTrail | Audit log de chamadas de API (quem fez o quê) |
| CloudWatch | Métricas, logs, alarmes |
| GuardDuty | Detecção de ameaças (baseado em ML) |
| Security Hub | Dashboard central de achados de segurança |
| Config | Rastreamento de mudanças na configuração de recursos |
| VPC Flow Logs | Logs de tráfego de rede |

```bash
# Enable CloudTrail (should be on in every account)
aws cloudtrail create-trail --name main --s3-bucket-name my-trail-bucket

# GuardDuty — enable and forget
aws guardduty create-detector --enable
```

## Segurança de Container na AWS

- **ECR scanning**: varredura automática de vulnerabilidades no push
- **ECS**: use Fargate (sem EC2 para patchar), task roles (menor privilégio)
- **EKS**: pod security standards, IRSA (IAM Roles for Service Accounts)

## Gerenciamento de Secrets

```bash
# AWS Secrets Manager
aws secretsmanager create-secret --name prod/db-password --secret-string "s3cret"

# Retrieve in code
aws secretsmanager get-secret-value --secret-id prod/db-password
```

Nunca coloque secrets em: variáveis de ambiente no console, código, parâmetros de CloudFormation.

## Checklist de Segurança

- Habilite CloudTrail em todas as regiões
- Habilite GuardDuty
- Sem usuários IAM com acesso ao console sem MFA
- Sem access keys na conta root
- Buckets S3: bloqueie acesso público por padrão
- Habilite criptografia padrão de EBS
- Use Security Hub para achados agregados
- Revisões regulares do IAM Access Analyzer
- Tageie tudo para accountability

## Relacionados

- [[Security/IAM]]
- [[Security/ThreatModeling]]
- [[Security/SupplyChainSecurity]]
- [[Linux/Security]]

## Resources

- https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/
- https://d1.awsstatic.com/whitepapers/aws-security-best-practices.pdf

#### Meus comentários
- 
