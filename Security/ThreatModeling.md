# Threat Modeling

A structured approach to identifying and addressing security threats before they become vulnerabilities.

---

## What is Threat Modeling?

Asking four questions about your system:

1. **What are we building?** (scope, architecture)
2. **What can go wrong?** (threats)
3. **What are we going to do about it?** (mitigations)
4. **Did we do a good job?** (validation)

Do it early — designing security in is cheaper than patching it later.

## STRIDE

Microsoft's threat classification model:

| Threat | Definition | Example |
|--------|-----------|---------|
| **S**poofing | Pretending to be someone else | Stolen JWT, forged email |
| **T**ampering | Modifying data | Man-in-the-middle, SQL injection |
| **R**epudiation | Denying an action | Deleting logs, unsigned transactions |
| **I**nformation Disclosure | Leaking data | Exposed S3 bucket, verbose error |
| **D**enial of Service | Making service unavailable | DDoS, resource exhaustion |
| **E**levation of Privilege | Gaining unauthorized access | Container escape, IDOR |

## The Process

### Step 1: Diagram the System

```
User ──HTTPS──► Load Balancer ──► API Server ──► Database
                                      │
                                      ├──► Redis Cache
                                      ├──► S3 (file storage)
                                      └──► External Payment API
```

Draw:
- Trust boundaries (where security context changes)
- Data flows (what moves where)
- Data stores (where sensitive data lives)
- External entities (users, third-party services)

### Step 2: Identify Threats (per component)

For each component and data flow, apply STRIDE:

```
API Server:
  [S] Spoofing   → attacker uses stolen API key
  [T] Tampering  → modify request body without validation
  [I] Info Disc  → stack trace in error response
  [E] Elevation  → IDOR: change user_id in request to access other accounts

Database:
  [T] Tampering  → SQL injection
  [I] Info Disc  → unencrypted backups
  [D] DoS        → slow query exhausts connections
```

### Step 3: Prioritize (Risk = Likelihood × Impact)

| Threat | Likelihood | Impact | Risk | Priority |
|--------|-----------|--------|------|----------|
| SQL injection | Medium | Critical | High | P1 |
| DDoS on API | High | High | High | P1 |
| Stolen API key | Medium | High | High | P2 |
| Verbose errors | High | Low | Medium | P3 |

### Step 4: Mitigate

For each threat, decide: **mitigate, accept, transfer, or avoid**.

```
SQL injection     → parameterized queries (mitigate)
DDoS              → rate limiting + WAF (mitigate)
Stolen API key    → short-lived tokens + rotation (mitigate)
Verbose errors    → generic error messages in production (mitigate)
Earthquake at DC  → multi-region deployment (transfer to AWS)
```

## DREAD (Risk Scoring)

Alternative to simple likelihood × impact:

- **D**amage potential (0-10)
- **R**eproducibility (0-10)
- **E**xploitability (0-10)
- **A**ffected users (0-10)
- **D**iscoverability (0-10)

Score = average → prioritize highest.

## Attack Trees

Visual representation of how an attacker could achieve a goal:

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

## When to Threat Model

- New feature or service design
- Architecture changes
- Before security review / audit
- After a security incident (what did we miss?)
- Regularly (quarterly review)

## Related

- [[Security/IAM]]
- [[Security/CloudSecurity]]
- [[Security/SupplyChainSecurity]]

## Resources

- https://owasp.org/www-community/Threat_Modeling
- Threat Modeling: Designing for Security — Adam Shostack (book)
- https://github.com/OWASP/threat-dragon (open source tool)

#### My commentaries
- 
