# Supply Chain Security

Protegendo toda a cadeia desde o código até a produção — dependências, builds, artifacts e entrega.

---

## O Que é Supply Chain Security?

Seu software não é apenas o seu código. É:

```
Seu código
  + dependências open source (npm, pip, go modules)
  + imagens base (Docker)
  + ferramentas de build (compiladores, bundlers)
  + pipeline CI/CD (GitHub Actions, Jenkins)
  + armazenamento de artifacts (registries, S3)
  + infraestrutura de deploy
```

Um atacante que comprometa QUALQUER elo dessa cadeia compromete o seu software.

## Vetores de Ataque

| Vetor | Exemplo |
|--------|---------|
| Dependency confusion | Pacote malicioso com o mesmo nome de um interno |
| Typosquatting | `reqeusts` em vez de `requests` no PyPI |
| Maintainer comprometido | Atacante obtém acesso a um pacote popular |
| CI action maliciosa | GitHub Action trojanizada exfiltra secrets |
| Chaves de assinatura roubadas | Atacante assina release maliciosa como legítima |
| Servidor de build comprometido | Injeção de código durante o build, não no source |
| Registry poisoning | Push de tag de imagem maliciosa para Docker Hub |

## Framework SLSA (Supply-chain Levels for Software Artifacts)

```
Level 0: Sem garantias
Level 1: Processo de build documentado
Level 2: Serviço de build autenticado (ex: GitHub Actions)
Level 3: Build isolado, reproduzível e auditável
Level 4: Build hermético, revisão por duas pessoas, reproduzível
```

## Protegendo Dependências

```bash
# Pin exact versions (no ^ or ~)
npm ci --ignore-scripts        # install from lockfile, skip scripts
pip install --require-hashes   # verify package hashes

# Audit
npm audit
pip-audit
go list -m -json all | nancy sleuth

# Lockfiles são críticos
package-lock.json / yarn.lock / go.sum / Pipfile.lock
```

**Dependabot / Renovate**: PR automático para atualizações de dependências.

## Assinatura e Verificação

### Sigstore / Cosign (containers)

```bash
# Sign a container image
cosign sign ghcr.io/myorg/myapp:v1.0

# Verify
cosign verify ghcr.io/myorg/myapp:v1.0

# Keyless signing (OIDC-based, no keys to manage)
cosign sign --identity-token=$(gcloud auth print-identity-token) image
```

### Assinatura de Commits Git

```bash
git config --global commit.gpgsign true
git log --show-signature
```

## SBOM (Software Bill of Materials)

Um inventário completo de cada componente do seu software:

```bash
# Generate SBOM
syft packages myimage:latest -o spdx-json > sbom.json

# Scan SBOM for vulnerabilities
grype sbom:sbom.json
```

Formatos: SPDX, CycloneDX

## Segurança de CI/CD

```yaml
# Pin actions to SHA (not tags — tags can be moved)
- uses: actions/checkout@abc123def456

# Minimal permissions
permissions:
  contents: read

# Don't echo secrets
- run: deploy --token=$TOKEN
  env:
    TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

- Use OIDC para autenticação na cloud em vez de secrets armazenados
- Exija revisões de PR antes do merge
- Execute varreduras de segurança no pipeline (SAST, SCA, container scanning)

## Relacionados

- [[DevOps/CICD/GitHubActions]]
- [[DevOps/Docker/DockerAdvanced]]
- [[Security/IAM]]
- [[Security/CloudSecurity]]

## Resources

- https://slsa.dev
- https://www.sigstore.dev
- https://openssf.org/projects/

#### Meus comentários
- 
