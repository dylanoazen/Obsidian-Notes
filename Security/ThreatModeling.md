# Threat Modeling

Uma abordagem estruturada para identificar e endereçar ameaças de segurança antes que se tornem vulnerabilidades.

---

## O Que é Threat Modeling?

Fazer quatro perguntas sobre o seu sistema:

1. **O que estamos construindo?** (escopo, arquitetura)
2. **O que pode dar errado?** (ameaças)
3. **O que vamos fazer a respeito?** (mitigações)
4. **Fizemos um bom trabalho?** (validação)

Faça cedo — projetar a segurança desde o início é mais barato do que corrigir depois.

## STRIDE

O modelo de classificação de ameaças da Microsoft:

| Ameaça | Definição | Exemplo |
|--------|-----------|---------|
| **S**poofing | Fingir ser outra pessoa | JWT roubado, e-mail forjado |
| **T**ampering | Modificar dados | Man-in-the-middle, SQL injection |
| **R**epudiation | Negar uma ação | Deletar logs, transações não assinadas |
| **I**nformation Disclosure | Vazar dados | Bucket S3 exposto, erro verboso |
| **D**enial of Service | Tornar o serviço indisponível | DDoS, esgotamento de recursos |
| **E**levation of Privilege | Obter acesso não autorizado | Container escape, IDOR |

## O Processo

### Passo 1: Diagrame o Sistema

```
User ──HTTPS──► Load Balancer ──► API Server ──► Database
                                      │
                                      ├──► Redis Cache
                                      ├──► S3 (file storage)
                                      └──► External Payment API
```

Desenhe:
- Fronteiras de confiança (onde o contexto de segurança muda)
- Fluxos de dados (o que se move para onde)
- Armazenamentos de dados (onde dados sensíveis ficam)
- Entidades externas (usuários, serviços de terceiros)

### Passo 2: Identifique Ameaças (por componente)

Para cada componente e fluxo de dados, aplique STRIDE:

```
API Server:
  [S] Spoofing   → atacante usa API key roubada
  [T] Tampering  → modificar o corpo da requisição sem validação
  [I] Info Disc  → stack trace na resposta de erro
  [E] Elevation  → IDOR: alterar user_id na requisição para acessar outras contas

Database:
  [T] Tampering  → SQL injection
  [I] Info Disc  → backups não criptografados
  [D] DoS        → query lenta esgota conexões
```

### Passo 3: Priorize (Risco = Probabilidade × Impacto)

| Ameaça | Probabilidade | Impacto | Risco | Prioridade |
|--------|-----------|--------|------|----------|
| SQL injection | Média | Crítico | Alto | P1 |
| DDoS na API | Alta | Alto | Alto | P1 |
| API key roubada | Média | Alto | Alto | P2 |
| Erros verbosos | Alta | Baixo | Médio | P3 |

### Passo 4: Mitigue

Para cada ameaça, decida: **mitigar, aceitar, transferir ou evitar**.

```
SQL injection     → parameterized queries (mitigate)
DDoS              → rate limiting + WAF (mitigate)
Stolen API key    → short-lived tokens + rotation (mitigate)
Verbose errors    → generic error messages in production (mitigate)
Earthquake at DC  → multi-region deployment (transfer to AWS)
```

## DREAD (Pontuação de Risco)

Alternativa ao simples probabilidade × impacto:

- **D**amage potential (0-10)
- **R**eproducibility (0-10)
- **E**xploitability (0-10)
- **A**ffected users (0-10)
- **D**iscoverability (0-10)

Pontuação = média → priorize os mais altos.

## Attack Trees

Representação visual de como um atacante poderia atingir um objetivo:

```
Goal: Access admin panel
├── Steal admin credentials
│   ├── Phishing email
│   ├── Brute force login
│   └── Find credentials in public repo
├── Exploit vulnerability
│   ├── SQL injection → escalate
│   └── SSRF → access internal endpoints
└── Social engineering
    └── Call support, impersonate admin
```

## Quando Fazer Threat Modeling

- Design de nova feature ou serviço
- Mudanças de arquitetura
- Antes de revisão de segurança / auditoria
- Após um incidente de segurança (o que deixamos passar?)
- Regularmente (revisão trimestral)

## Relacionados

- [[Security/IAM]]
- [[Security/CloudSecurity]]
- [[Security/SupplyChainSecurity]]

## Resources

- https://owasp.org/www-community/Threat_Modeling
- Threat Modeling: Designing for Security — Adam Shostack (livro)
- https://github.com/OWASP/threat-dragon (ferramenta open source)

#### Meus comentários
- 
