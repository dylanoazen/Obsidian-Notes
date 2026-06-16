# Kubernetes

Kubernetes (K8s) é uma plataforma de orquestração de containers — ela gerencia onde e como seus containers rodam em múltiplas máquinas.

---

## O Problema que Resolve

Docker permite rodar um container em uma máquina. Mas em produção você tem perguntas que o Docker sozinho não consegue responder:

- E se essa máquina morrer?
- Como rodo 10 cópias do meu backend?
- Como atualizo minha aplicação sem downtime?
- Como os containers se comunicam entre si?

Kubernetes responde a tudo isso.

**Analogia:** Docker é um container de carga. Kubernetes é o porto que decide qual navio carrega qual container, lida com navios danificados e mantém tudo funcionando.

---

## Conceitos Fundamentais

### Cluster

Um **cluster** é o ambiente Kubernetes completo — um conjunto de máquinas trabalhando juntas.

```
Cluster
├── Control Plane (o cérebro)
└── Nodes (os trabalhadores)
```

### Node

Um **node** é uma máquina (física ou virtual) que roda seus containers. Você pode ter muitos nodes em um cluster.

### Pod

Um **pod** é a menor unidade implantável no Kubernetes — um wrapper em torno de um ou mais containers.

```
Pod
└── Container (sua aplicação)
```

- Pods são efêmeros — podem morrer e ser substituídos a qualquer momento
- Cada pod recebe seu próprio endereço IP dentro do cluster
- Se um pod morre, o Kubernetes cria um novo automaticamente

### Deployment

Um **deployment** diz ao Kubernetes quantas cópias (réplicas) do seu pod manter em execução o tempo todo.

```yaml
replicas: 3  ← mantenha sempre 3 pods rodando
```

Se um pod morre, o deployment percebe e cria um novo. Você declara o estado desejado — o Kubernetes descobre como chegar lá.

### Service

Um **service** é um endpoint de rede estável que fica na frente dos seus pods.

Como pods vêm e vão (novos IPs a cada vez), um service fornece um endereço fixo que sempre roteia para pods saudáveis.

```
Client → Service (IP fixo) → Pod 1
                           → Pod 2
                           → Pod 3
```

**Analogia:** o service é a recepcionista — você sempre liga para o mesmo número, e ela te direciona para quem está disponível.

### Ingress

Um **ingress** expõe seus services para o mundo externo (internet).

```
Internet → Ingress → Service → Pods
```

Ele cuida de roteamento, terminação TLS e roteamento baseado em domínio (ex.: `api.myapp.com` → backend service).

---

## Como Se Conecta ao que Você Já Conhece

Você estudou horizontal scaling — adicionar mais instâncias de backend sob carga. Kubernetes é o que gerencia isso em produção:

```
Alto tráfego detectado
→ Kubernetes escala o Deployment de 3 para 10 pods automaticamente
→ Service roteia o tráfego entre os 10
→ Tráfego cai → escala de volta para 3
```

Isso se chama **autoscaling**.

---

## Estado Desejado vs Estado Real

Esta é a filosofia central do Kubernetes:

Você descreve **o que quer** (estado desejado), o Kubernetes descobre **como fazer acontecer** (estado real).

```
Você diz:   "Quero 3 réplicas do meu backend rodando"
K8s faz:    cria pods, monitora, substitui os que morrem
```

Você nunca diz "inicie este container naquela máquina" — você declara a intenção e o Kubernetes age.

---

## Resumo dos Objetos Principais

| Objeto | O que faz |
|---|---|
| Pod | Roda seu container |
| Deployment | Gerencia quantos pods rodam e os mantém saudáveis |
| Service | Acesso de rede estável aos pods |
| Ingress | Expõe services para a internet |
| ConfigMap | Armazena configurações (não sensíveis) |
| Secret | Armazena dados sensíveis (senhas, tokens) |
| Namespace | Isolamento lógico dentro de um cluster |

---

## Rolling Updates — Deploys sem Downtime

Quando você faz deploy de uma nova versão da sua aplicação, o Kubernetes atualiza os pods um por vez:

```
v1 v1 v1 v1
→ substitui um por vez
v2 v1 v1 v1
v2 v2 v1 v1
v2 v2 v2 v1
v2 v2 v2 v2
```

Em nenhum momento a aplicação fica completamente fora do ar. Se a nova versão estiver quebrada, o Kubernetes faz rollback automaticamente.

---

## Kubernetes e Redis

No contexto do MiniRedisGo:

```
[Backend Pods x3]  →  Redis Service  →  Redis Pod (primary)
                                      →  Redis Pod (replica)
```

- Seu backend escala horizontalmente entre os pods
- Redis roda como uma carga de trabalho stateful com replicação
- O Service garante que os backends sempre encontrem o Redis no mesmo endereço

Redis no Kubernetes tipicamente usa um **StatefulSet** em vez de um Deployment — porque os pods do Redis precisam de identidades estáveis e armazenamento persistente, ao contrário de backends stateless.

---

## Notas de Design

- Kubernetes não constrói seus containers — esse é o trabalho do Docker
- K8s gerencia o estado no nível de infraestrutura, não no nível de aplicação
- `kubectl` é a CLI para interagir com um cluster
- Kubernetes é complexo — versões gerenciadas (GKE, EKS, AKS) escondem a maior parte do peso operacional

---

## Comandos kubectl Comuns

```bash
kubectl get pods                    # list running pods
kubectl get deployments             # list deployments
kubectl describe pod <name>         # inspect a pod
kubectl logs <pod-name>             # view pod logs
kubectl scale deployment app --replicas=5  # scale manually
kubectl apply -f deployment.yaml    # apply a config file
```

---

## Notas Relacionadas

- [[Replication]]
- [[SecurityAuth]]
- [[PubSub]]
