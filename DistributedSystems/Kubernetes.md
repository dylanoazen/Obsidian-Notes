# Kubernetes

Kubernetes (K8s) is a container orchestration platform — it manages where and how your containers run across multiple machines.

---

## The Problem It Solves

Docker lets you run a container on one machine. But in production you have questions Docker alone cannot answer:

- What if that machine dies?
- How do I run 10 copies of my backend?
- How do I update my app without downtime?
- How do containers talk to each other?

Kubernetes answers all of these.

**Analogy:** Docker is a shipping container. Kubernetes is the port that decides which ship carries which container, handles damaged ships, and keeps everything moving.

---

## Core Concepts

### Cluster

A **cluster** is the full Kubernetes environment — a set of machines working together.

```
Cluster
├── Control Plane (the brain)
└── Nodes (the workers)
```

### Node

A **node** is a machine (physical or virtual) that runs your containers. You can have many nodes in a cluster.

### Pod

A **pod** is the smallest deployable unit in Kubernetes — a wrapper around one or more containers.

```
Pod
└── Container (your app)
```

- Pods are ephemeral — they can die and be replaced at any time
- Each pod gets its own IP address inside the cluster
- If a pod dies, Kubernetes creates a new one automatically

### Deployment

A **deployment** tells Kubernetes how many copies (replicas) of your pod to keep running at all times.

```yaml
replicas: 3  ← always keep 3 pods running
```

If one pod dies, the deployment notices and creates a new one. You declare the desired state — Kubernetes figures out how to get there.

### Service

A **service** is a stable network endpoint that sits in front of your pods.

Since pods come and go (new IPs each time), a service gives you a fixed address that always routes to healthy pods.

```
Client → Service (fixed IP) → Pod 1
                            → Pod 2
                            → Pod 3
```

**Analogy:** the service is the receptionist — you always call the same number, and she routes you to whoever is available.

### Ingress

An **ingress** exposes your services to the outside world (internet).

```
Internet → Ingress → Service → Pods
```

It handles routing, TLS termination, and domain-based routing (e.g. `api.myapp.com` → backend service).

---

## How It Connects to What You Already Know

You studied horizontal scaling — adding more backend instances when under load. Kubernetes is what manages that in production:

```
High traffic detected
→ Kubernetes scales Deployment from 3 to 10 pods automatically
→ Service routes traffic across all 10
→ Traffic drops → scales back to 3
```

This is called **autoscaling**.

---

## Desired State vs Actual State

This is the core philosophy of Kubernetes:

You describe **what you want** (desired state), Kubernetes figures out **how to make it happen** (actual state).

```
You say:   "I want 3 replicas of my backend running"
K8s does:  creates pods, monitors them, replaces dead ones
```

You never say "start this container on that machine" — you declare intent and Kubernetes acts.

---

## Key Objects Summary

| Object | What it does |
|---|---|
| Pod | Runs your container |
| Deployment | Manages how many pods run and keeps them healthy |
| Service | Stable network access to pods |
| Ingress | Exposes services to the internet |
| ConfigMap | Stores configuration (non-sensitive) |
| Secret | Stores sensitive data (passwords, tokens) |
| Namespace | Logical isolation within a cluster |

---

## Rolling Updates — Zero Downtime Deploys

When you deploy a new version of your app, Kubernetes updates pods one at a time:

```
v1 v1 v1 v1
→ replace one at a time
v2 v1 v1 v1
v2 v2 v1 v1
v2 v2 v2 v1
v2 v2 v2 v2
```

At no point is the app fully down. If the new version is broken, Kubernetes rolls back automatically.

---

## Kubernetes and Redis

In your MiniRedisGo context:

```
[Backend Pods x3]  →  Redis Service  →  Redis Pod (primary)
                                      →  Redis Pod (replica)
```

- Your backend scales horizontally across pods
- Redis runs as a stateful workload with replication
- The Service ensures backends always find Redis at the same address

Redis in Kubernetes typically uses a **StatefulSet** instead of a Deployment — because Redis pods need stable identities and persistent storage, unlike stateless backends.

---

## Design Notes

- Kubernetes does not build your containers — that is Docker's job
- K8s manages state at the infrastructure level, not the application level
- `kubectl` is the CLI to interact with a cluster
- Kubernetes is complex — managed versions (GKE, EKS, AKS) hide most of the operational burden

---

## Common kubectl Commands

```bash
kubectl get pods                    # list running pods
kubectl get deployments             # list deployments
kubectl describe pod <name>         # inspect a pod
kubectl logs <pod-name>             # view pod logs
kubectl scale deployment app --replicas=5  # scale manually
kubectl apply -f deployment.yaml    # apply a config file
```

---

## Related Notes

- [[Replication]]
- [[SecurityAuth]]
- [[PubSub]]
