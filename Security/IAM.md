# IAM / OIDC / OAuth

Identity and Access Management — how systems verify who you are and what you can do.

---

## Core Concepts

- **Authentication (AuthN)**: proving who you are ("I am Dylan")
- **Authorization (AuthZ)**: proving what you can do ("Dylan can read repo X")
- **Identity Provider (IdP)**: system that manages identities (Google, Okta, Auth0, Keycloak)

## OAuth 2.0

OAuth is an **authorization** framework — it lets a third-party app access resources on behalf of a user without sharing passwords.

### Roles

```
Resource Owner    = the user (you)
Client            = the app requesting access (e.g., a web app)
Authorization Server = issues tokens (Google, Auth0)
Resource Server   = API that holds the data (e.g., GitHub API)
```

### Authorization Code Flow (most common for web apps)

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

### Other Flows

- **Client Credentials**: machine-to-machine, no user involved
- **PKCE** (Proof Key for Code Exchange): for mobile/SPA apps (no client_secret)
- **Device Flow**: for devices with no browser (TVs, CLIs)
- **Implicit** (deprecated): token returned directly in redirect — insecure

### Tokens

- **Access Token**: short-lived (minutes), sent with API requests
- **Refresh Token**: long-lived, used to get new access tokens
- **Scope**: limits what the token can do (`read:repos`, `user:email`)

## OpenID Connect (OIDC)

OIDC is an **authentication** layer on top of OAuth 2.0. Adds:

- **ID Token**: JWT that contains user identity info
- **UserInfo endpoint**: API to get user profile
- **Standard scopes**: `openid`, `profile`, `email`

```
OAuth 2.0  = "this app can access your repos" (authorization)
OIDC       = "this user is dylan@email.com" (authentication)
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

Always validate: `iss`, `aud`, `exp`, and signature.

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

JWTs are **signed, not encrypted** — anyone can read the payload. Don't put secrets in them.

## IAM in Cloud (AWS)

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

Principles:
- **Least privilege**: only grant what's needed
- **Roles over users**: prefer IAM roles (temporary credentials) over IAM users (permanent keys)
- **MFA**: require for sensitive operations
- **Conditions**: restrict by IP, time, MFA status, tags

## OIDC Federation (GitHub Actions → AWS)

No more storing AWS keys in GitHub secrets:

```yaml
permissions:
  id-token: write

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456:role/github-deploy
      aws-region: us-east-1
```

GitHub generates an OIDC token → AWS trusts GitHub as IdP → temporary credentials issued.

## Related

- [[Security/CloudSecurity]]
- [[Security/SupplyChainSecurity]]
- [[DevOps/CICD/GitHubActions]]
- [[Linux/Security]]

## Resources

- https://oauth.net/2/
- https://openid.net/developers/how-connect-works/
- https://jwt.io

#### My commentaries
- 
