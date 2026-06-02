# Supply Chain Security

Securing the entire chain from code to production — dependencies, builds, artifacts, and delivery.

---

## What is Supply Chain Security?

Your software isn't just your code. It's:

```
Your code
  + open source dependencies (npm, pip, go modules)
  + base images (Docker)
  + build tools (compilers, bundlers)
  + CI/CD pipeline (GitHub Actions, Jenkins)
  + artifact storage (registries, S3)
  + deployment infrastructure
```

An attacker who compromises ANY link in this chain compromises your software.

## Attack Vectors

| Vector | Example |
|--------|---------|
| Dependency confusion | Malicious package with same name as internal one |
| Typosquatting | `reqeusts` instead of `requests` on PyPI |
| Compromised maintainer | Attacker gains access to popular package |
| Malicious CI action | Trojanized GitHub Action exfiltrates secrets |
| Stolen signing keys | Attacker signs malicious release as legitimate |
| Build server compromise | Inject code during build, not in source |
| Registry poisoning | Push malicious image tag to Docker Hub |

## SLSA Framework (Supply-chain Levels for Software Artifacts)

```
Level 0: No guarantees
Level 1: Build process is documented
Level 2: Build service is authenticated (e.g., GitHub Actions)
Level 3: Build is isolated, reproducible, auditable
Level 4: Hermetic build, two-person review, reproducible
```

## Protecting Dependencies

```bash
# Pin exact versions (no ^ or ~)
npm ci --ignore-scripts        # install from lockfile, skip scripts
pip install --require-hashes   # verify package hashes

# Audit
npm audit
pip-audit
go list -m -json all | nancy sleuth

# Lockfiles are critical
package-lock.json / yarn.lock / go.sum / Pipfile.lock
```

**Dependabot / Renovate**: auto-PR for dependency updates.

## Signing and Verification

### Sigstore / Cosign (containers)

```bash
# Sign a container image
cosign sign ghcr.io/myorg/myapp:v1.0

# Verify
cosign verify ghcr.io/myorg/myapp:v1.0

# Keyless signing (OIDC-based, no keys to manage)
cosign sign --identity-token=$(gcloud auth print-identity-token) image
```

### Git Commit Signing

```bash
git config --global commit.gpgsign true
git log --show-signature
```

## SBOM (Software Bill of Materials)

A complete inventory of every component in your software:

```bash
# Generate SBOM
syft packages myimage:latest -o spdx-json > sbom.json

# Scan SBOM for vulnerabilities
grype sbom:sbom.json
```

Formats: SPDX, CycloneDX

## CI/CD Security

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

- Use OIDC for cloud auth instead of stored secrets
- Require PR reviews before merge
- Run security scans in pipeline (SAST, SCA, container scanning)

## Related

- [[DevOps/CICD/GitHubActions]]
- [[DevOps/Docker/DockerAdvanced]]
- [[Security/IAM]]
- [[Security/CloudSecurity]]

## Resources

- https://slsa.dev
- https://www.sigstore.dev
- https://openssf.org/projects/

#### My commentaries
- 
