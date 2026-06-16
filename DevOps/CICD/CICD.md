## CI/CD - Continuous Integration / Continuous Delivery

CI/CD é um conjunto de práticas que automatizam o processo de integrar mudanças de código, testá-las e fazer deploy em produção.

Ele define:
- pipelines automatizados de build e teste
- processos de deploy consistentes
- ciclos de feedback rápidos para desenvolvedores
- releases confiáveis e reproduzíveis

## Continuous Integration (CI)

- Desenvolvedores fazem merge de código em um branch compartilhado com frequência
- Cada merge dispara um build automatizado e uma suite de testes
- Captura bugs cedo, antes que cheguem à produção
- Ferramentas: GitHub Actions, Jenkins, GitLab CI, CircleCI

## Continuous Delivery (CD)

- Código que passa no CI é automaticamente preparado para release
- O deploy para staging/produção pode ser disparado com um clique
- Reduz riscos ao fazer deploy de mudanças pequenas e incrementais
- Garante que a codebase esteja sempre em um estado deployável

## Continuous Deployment

- Vai um passo além do Continuous Delivery
- Cada mudança que passa por todos os estágios do pipeline é deployada automaticamente
- Sem aprovação manual necessária
- Requer alta confiança na cobertura de testes

## Conceitos Chave

- **Pipeline**: sequência de estágios (build → test → deploy)
- **Artifact**: saída de uma etapa de build (binário, container image, pacote)
- **Environment**: onde o código roda (dev, staging, produção)
- **Rollback**: reverter para uma versão anterior se algo quebrar
- **Blue/Green Deployment**: rodar dois ambientes idênticos e alternar o tráfego
- **Canary Release**: fazer deploy para um pequeno subconjunto de usuários primeiro

## Relacionados

- [[DevOps/Docker/Docker]]
- [[DevOps/Heroku/Heroku]]

## Ferramentas

- GitHub Actions
- Jenkins
- GitLab CI/CD
- CircleCI
- ArgoCD (GitOps for Kubernetes)
- Terraform (Infrastructure as Code)

## Resources

- https://docs.github.com/en/actions
- https://www.redhat.com/en/topics/devops/what-is-ci-cd

#### Meus comentários
- 
