---
tags: [docplanner-prep, kubernetes]
status: draft
---
# Kubernetes Essencial

**Pod**: menor unidade. Seus containers.
**Deployment**: gerencia pods, réplicas, rolling updates.
**Service**: DNS + load balancer interno.
**ConfigMap/Secret**: config e credenciais injetadas.
**Liveness Probe**: tá vivo? Restart se não.
**Readiness Probe**: tá pronto? Tira do LB se não.

```yaml
spec:
  replicas: 3
  containers:
    - name: app
      image: docplanner/booking:v1.2.3
      livenessProbe:
        httpGet: { path: /healthz, port: 8080 }
      readinessProbe:
        httpGet: { path: /ready, port: 8080 }
```

## Links
- [[DevOps/Docker/DockerAdvanced]] · [[DistributedSystems/Kubernetes]]
