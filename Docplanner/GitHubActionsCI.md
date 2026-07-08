---
tags: [docplanner-prep, ci]
status: draft
---
# GitHub Actions CI — Pipeline PHP/Symfony

```yaml
jobs:
  lint:
    steps:
      - run: vendor/bin/php-cs-fixer fix --dry-run
      - run: vendor/bin/phpstan analyse -l 8

  test:
    needs: lint
    services:
      mysql: { image: mysql:8, env: { MYSQL_DATABASE: test } }
    steps:
      - run: vendor/bin/phpunit --coverage-text

  build:
    needs: test
    steps:
      - uses: docker/build-push-action@v5
```

Detalhes completos → [[DevOps/CICD/GitHubActions]]

## Links
- [[DevOps/CICD/GitHubActions]] · [[DevOps/CICD/CICD]]
