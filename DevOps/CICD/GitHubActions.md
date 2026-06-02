# GitHub Actions

GitHub's built-in CI/CD platform — automate builds, tests, and deployments directly from your repo.

Related: [[DevOps/CICD/CICD]], [[Git/Git]]

---

## Core Concepts

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

- **Workflow**: YAML file that defines automation
- **Event**: what triggers the workflow
- **Job**: set of steps that run on the same runner
- **Step**: individual command or action
- **Runner**: VM that executes the job (GitHub-hosted or self-hosted)

## Basic Workflow

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

## Secrets and Variables

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

Never hardcode secrets — use Settings → Secrets and Variables.

## Matrix Strategy

Run tests across multiple versions/OS:

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

Share data between jobs:

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

## Docker Build and Push

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

## Security Best Practices

- Pin actions to commit SHA, not tags: `actions/checkout@abc123`
- Use `permissions` to limit GITHUB_TOKEN scope
- Never echo secrets in logs
- Use environments with required reviewers for production deploys
- Enable branch protection rules

```yaml
permissions:
  contents: read
  packages: write
```

## Related

- [[DevOps/CICD/CICD]]
- [[DevOps/Docker/DockerAdvanced]]
- [[Git/Git]]
- [[Security/SupplyChainSecurity]]

## Resources

- https://docs.github.com/en/actions
- https://github.com/sdras/awesome-actions

#### My commentaries
- 
