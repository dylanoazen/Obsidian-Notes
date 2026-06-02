## CI/CD - Continuous Integration / Continuous Delivery

CI/CD is a set of practices that automate the process of integrating code changes, testing them, and deploying to production.

It defines:
- automated build and test pipelines
- consistent deployment processes
- fast feedback loops for developers
- reliable and repeatable releases

## Continuous Integration (CI)

- Developers merge code to a shared branch frequently
- Every merge triggers an automated build and test suite
- Catches bugs early, before they reach production
- Tools: GitHub Actions, Jenkins, GitLab CI, CircleCI

## Continuous Delivery (CD)

- Code that passes CI is automatically prepared for release
- Deployment to staging/production can be triggered with one click
- Reduces risk by deploying small, incremental changes
- Ensures the codebase is always in a deployable state

## Continuous Deployment

- Goes one step further than Continuous Delivery
- Every change that passes all pipeline stages is deployed automatically
- No manual approval needed
- Requires high confidence in test coverage

## Key Concepts

- **Pipeline**: sequence of stages (build → test → deploy)
- **Artifact**: output of a build step (binary, container image, package)
- **Environment**: where the code runs (dev, staging, production)
- **Rollback**: reverting to a previous version if something breaks
- **Blue/Green Deployment**: running two identical environments, switching traffic
- **Canary Release**: deploying to a small subset of users first

## Related

- [[DevOps/Docker/Docker]]
- [[DevOps/Heroku/Heroku]]

## Tools

- GitHub Actions
- Jenkins
- GitLab CI/CD
- CircleCI
- ArgoCD (GitOps for Kubernetes)
- Terraform (Infrastructure as Code)

## Resources

- https://docs.github.com/en/actions
- https://www.redhat.com/en/topics/devops/what-is-ci-cd

#### My commentaries
- 
