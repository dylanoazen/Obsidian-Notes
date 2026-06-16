# GitHub Actions

A plataforma CI/CD nativa do GitHub — automatize builds, testes e deploys diretamente do seu repositório.

Related: [[DevOps/CICD/CICD]], [[Git/Git]]

---

## Conceitos Principais

```
Workflow (.github/workflows/*.yml)
  │
  ├── Event (trigger: push, PR, schedule, manual)
  │
  ├── Job 1 (runs on a runner)
  │   ├── Step 1: actions/checkout
  │   ├── Step 2: run tests
  │   └── Step 3: build
  │
  └── Job 2 (can depend on Job 1)
      └── Step 1: deploy
```

- **Workflow**: arquivo YAML que define a automação
- **Event**: o que dispara o workflow
- **Job**: conjunto de steps que rodam no mesmo runner
- **Step**: comando ou action individual
- **Runner**: VM que executa o job (hospedada pelo GitHub ou self-hosted)

## Workflow Básico

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - run: go test ./...

      - run: go build -o myapp
```

## Triggers

```yaml
on:
  push:
    branches: [main, 'release/**']
    paths: ['src/**']              # only run if src changed

  pull_request:
    types: [opened, synchronize]

  schedule:
    - cron: '0 6 * * 1'           # every Monday at 6 AM

  workflow_dispatch:                # manual trigger
    inputs:
      environment:
        type: choice
        options: [staging, production]
```

## Secrets e Variáveis

```yaml
env:
  APP_ENV: production

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ vars.DEPLOY_TARGET }}"
        env:
          API_KEY: ${{ secrets.API_KEY }}
```

Nunca coloque secrets no código — use Settings → Secrets and Variables.

## Matrix Strategy

Execute testes em múltiplas versões/SO:

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        go-version: ['1.21', '1.22']
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
      - run: go test ./...
```

## Caching

```yaml
- uses: actions/cache@v4
  with:
    path: ~/go/pkg/mod
    key: go-mod-${{ hashFiles('go.sum') }}
    restore-keys: go-mod-
```

## Artifacts

Compartilhe dados entre jobs:

```yaml
jobs:
  build:
    steps:
      - run: go build -o myapp
      - uses: actions/upload-artifact@v4
        with:
          name: binary
          path: myapp

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: binary
```

## Docker Build e Push

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
- uses: docker/build-push-action@v5
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy_key:
        required: true

# Caller workflow
jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
    secrets:
      deploy_key: ${{ secrets.DEPLOY_KEY }}
```

## Boas Práticas de Segurança

- Fixe actions em um commit SHA, não em tags: `actions/checkout@abc123`
- Use `permissions` para limitar o escopo do GITHUB_TOKEN
- Nunca exiba secrets nos logs
- Use environments com aprovadores obrigatórios para deploys em produção
- Habilite regras de proteção de branch

```yaml
permissions:
  contents: read
  packages: write
```

## Relacionados

- [[DevOps/CICD/CICD]]
- [[DevOps/Docker/DockerAdvanced]]
- [[Git/Git]]
- [[Security/SupplyChainSecurity]]

## Resources

- https://docs.github.com/en/actions
- https://github.com/sdras/awesome-actions

#### Meus comentários
- 
