# IAM / OIDC / OAuth

Identity and Access Management — como sistemas verificam quem você é e o que você pode fazer.

---

## Conceitos Principais

- **Authentication (AuthN)**: provar quem você é ("Eu sou Dylan")
- **Authorization (AuthZ)**: provar o que você pode fazer ("Dylan pode ler o repositório X")
- **Identity Provider (IdP)**: sistema que gerencia identidades (Google, Okta, Auth0, Keycloak)

## OAuth 2.0

OAuth é um framework de **autorização** — ele permite que um app de terceiros acesse recursos em nome de um usuário sem compartilhar senhas.

### Papéis

```
Resource Owner    = o usuário (você)
Client            = o app solicitando acesso (ex: um web app)
Authorization Server = emite tokens (Google, Auth0)
Resource Server   = API que guarda os dados (ex: GitHub API)
```

### Authorization Code Flow (mais comum para web apps)

```
User ──► Client ──► Auth Server (/authorize)
                        │
                        ▼
                    Login page (user authenticates)
                        │
                        ▼
User ◄── redirect with authorization code
                        │
Client ──► Auth Server (/token) with code + client_secret
                        │
                        ▼
                    Access Token + Refresh Token
                        │
Client ──► Resource Server with Access Token
                        ▼
                    Protected data
```

### Outros Flows

- **Client Credentials**: machine-to-machine, sem usuário envolvido
- **PKCE** (Proof Key for Code Exchange): para apps mobile/SPA (sem client_secret)
- **Device Flow**: para dispositivos sem browser (TVs, CLIs)
- **Implicit** (descontinuado): token retornado diretamente no redirect — inseguro

### Tokens

- **Access Token**: vida curta (minutos), enviado com requisições de API
- **Refresh Token**: vida longa, usado para obter novos access tokens
- **Scope**: limita o que o token pode fazer (`read:repos`, `user:email`)

## OpenID Connect (OIDC)

OIDC é uma camada de **autenticação** sobre o OAuth 2.0. Adiciona:

- **ID Token**: JWT que contém informações de identidade do usuário
- **UserInfo endpoint**: API para obter o perfil do usuário
- **Scopes padrão**: `openid`, `profile`, `email`

```
OAuth 2.0  = "este app pode acessar seus repositórios" (autorização)
OIDC       = "este usuário é dylan@email.com" (autenticação)
```

### ID Token (JWT)

```json
{
  "iss": "https://accounts.google.com",
  "sub": "1234567890",
  "aud": "myapp.example.com",
  "email": "dylan@email.com",
  "exp": 1710000000,
  "iat": 1709996400
}
```

Sempre valide: `iss`, `aud`, `exp` e a assinatura.

## JWT (JSON Web Tokens)

```
header.payload.signature

Header:    {"alg": "RS256", "typ": "JWT"}
Payload:   {"sub": "123", "name": "Dylan", "exp": 170999}
Signature: HMAC-SHA256(base64(header) + "." + base64(payload), secret)
```

```bash
# Decode a JWT (base64)
echo "eyJhbG..." | base64 -d
# or use https://jwt.io
```

JWTs são **assinados, não criptografados** — qualquer um pode ler o payload. Não coloque secrets neles.

## IAM na Cloud (AWS)

```
User / Role / Service
       │
       ▼
   IAM Policy (JSON)
       │
       ▼
   Allow/Deny action on resource
```

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "IpAddress": {"aws:SourceIp": "10.0.0.0/8"}
  }
}
```

Princípios:
- **Least privilege**: conceda apenas o necessário
- **Roles em vez de users**: prefira IAM roles (credenciais temporárias) a IAM users (chaves permanentes)
- **MFA**: exija para operações sensíveis
- **Conditions**: restrinja por IP, horário, status de MFA, tags

## OIDC Federation (GitHub Actions → AWS)

Sem mais armazenar chaves AWS em secrets do GitHub:

```yaml
permissions:
  id-token: write

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456:role/github-deploy
      aws-region: us-east-1
```

O GitHub gera um token OIDC → a AWS confia no GitHub como IdP → credenciais temporárias são emitidas.

## Relacionados

- [[Security/CloudSecurity]]
- [[Security/SupplyChainSecurity]]
- [[DevOps/CICD/GitHubActions]]
- [[Linux/Security]]

## Resources

- https://oauth.net/2/
- https://openid.net/developers/how-connect-works/
- https://jwt.io

#### Meus comentários
- 
