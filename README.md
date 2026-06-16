# k8s-production-lab

Lab de produção local para estudo e validação de práticas de engenharia de plataforma com Kubernetes, GitOps e Service Mesh.

## Stack

| Componente | Papel |
|---|---|
| **Minikube** (Docker driver) | Cluster Kubernetes local no WSL2 |
| **ArgoCD** | GitOps controller — sincroniza este repo com o cluster |
| **Istio** | Service Mesh injetado no namespace `default` |
| **Metrics Server** | Coleta de métricas de CPU/memória para o HPA |

## Estrutura

```
apps/
  payment-api.yaml   # Deployment + Service da payment-api (stress de CPU)
  hpa.yaml           # HorizontalPodAutoscaler da payment-api
```

O ArgoCD monitora o path `apps/` via a Application `fintech-production-core` com `selfHeal` e `autoPrune` habilitados. Qualquer push para `main` é aplicado automaticamente no cluster.

## Módulo: Auto-scaling com HPA

A `payment-api` é um Deployment de stress de CPU usado para validar o comportamento do HPA em condições de carga real.

**Configuração:**
- Imagem: `polinux/stress-ng` — executa `stress-ng --cpu 1` em loop infinito
- Resource request: `100m` CPU / limit: `200m` CPU
- Probes: `exec pgrep stress-ng` (readiness e liveness)
- HPA: mínimo 2 réplicas, máximo 5, alvo de 50% de utilização de CPU

**Resultado observado:** com 2 pods consumindo ~200m CPU cada (200% do request de 100m), o HPA escalou para 5 réplicas em menos de 2 minutos.

## Fluxo GitOps

```
git push origin main
       │
       ▼
  GitHub (main)
       │
       ▼  refresh automático ou hard refresh
    ArgoCD
       │
       ▼
  kubectl apply
       │
       ▼
   Cluster k8s
```

Para forçar sincronização imediata sem esperar o ciclo automático:

```bash
kubectl annotate application fintech-production-core \
  -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

## Comandos úteis

```bash
# Estado dos pods da payment-api
kubectl get pods -l app=payment-api

# Métricas de CPU em tempo real
kubectl top pods -l app=payment-api

# Acompanhar o HPA
kubectl get hpa payment-api-hpa

# Status de sync do ArgoCD
kubectl get application fintech-production-core -n argocd

# Iniciar o cluster (se parado)
minikube start
```

## Ciclo de vida do lab

Para iniciar um ciclo de testes, sete `replicas: 2` em `apps/payment-api.yaml` e faça push.  
Para encerrar e poupar recursos, sete `replicas: 0` e faça push.  
O HPA, o Service e o Deployment permanecem definidos no cluster entre os ciclos.
