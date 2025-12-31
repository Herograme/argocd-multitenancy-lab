# ApplicationSet para Gerenciamento de Environments

Este documento explica como os environments (dev, staging, prod) são criados automaticamente para cada tenant.

## Arquitetura

```
ArgoCD Root (namespace: argocd)
│
├── applicationset.yaml (Git Generator)
│   └── Detecta: tenants/*
│   └── Cria: argocd-squad-empresa     → namespace: squad-empresa
│   └── Cria: argocd-squad-faturamento → namespace: squad-faturamento
│
└── applicationset-namespaces.yaml (Matrix Generator: tenants × envs)
    └── Detecta: tenants/*
    └── Combina com: [dev, staging, prod]
    └── Cria: ns-squad-empresa-dev      → namespace: squad-empresa-dev
    └── Cria: ns-squad-empresa-staging  → namespace: squad-empresa-staging
    └── Cria: ns-squad-empresa-prod     → namespace: squad-empresa-prod
    └── Cria: ns-squad-faturamento-dev  → namespace: squad-faturamento-dev
    └── ... etc
```

## Como Funciona

Quando você cria uma nova pasta em `tenants/nova-squad/`, o ArgoCD Root automaticamente:

1. **Cria o ArgoCD da squad** (via `applicationset.yaml`)
   - Namespace: `nova-squad`
   - Application: `argocd-nova-squad`

2. **Cria 3 namespaces de ambiente** (via `applicationset-namespaces.yaml`)
   - `nova-squad-dev` com ResourceQuota de desenvolvimento
   - `nova-squad-staging` com ResourceQuota intermediário
   - `nova-squad-prod` com ResourceQuota de produção

## Estrutura do Bootstrap

```
bootstrap/
├── applicationset.yaml            # Cria ArgoCD por tenant
└── applicationset-namespaces.yaml # Cria namespaces de env por tenant

infrastructure/namespace-template/
├── Chart.yaml
├── values.yaml                    # ResourceQuotas por env
└── templates/
    ├── namespace.yaml             # Namespace com labels
    ├── resourcequota.yaml         # Quotas diferenciadas
    └── limitrange.yaml            # Defaults de containers
```

## ResourceQuotas por Ambiente

| Ambiente | CPU Requests | Memory Requests | CPU Limits | Memory Limits | Pods |
|----------|--------------|-----------------|------------|---------------|------|
| dev      | 2 cores      | 4Gi             | 4 cores    | 8Gi           | 20   |
| staging  | 4 cores      | 8Gi             | 8 cores    | 16Gi          | 30   |
| prod     | 8 cores      | 16Gi            | 16 cores   | 32Gi          | 50   |

## Fluxo de Uso da Squad

Após os namespaces serem criados automaticamente, a squad pode:

### 1. Criar Applications no ArgoCD da Squad

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-dev
  namespace: squad-empresa  # namespace do ArgoCD da squad
spec:
  project: default
  source:
    repoURL: https://github.com/sua-org/api.git
    path: deploy/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: squad-empresa-dev  # namespace de ambiente
```

### 2. Usar ApplicationSet para Multi-Environment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: api-envs
  namespace: squad-empresa
spec:
  generators:
    - list:
        elements:
          - env: dev
          - env: staging
          - env: prod
  template:
    metadata:
      name: 'api-{{env}}'
    spec:
      source:
        repoURL: https://github.com/sua-org/api.git
        path: 'deploy/overlays/{{env}}'
      destination:
        namespace: 'squad-empresa-{{env}}'
```

## Aplicando o Bootstrap

```bash
# Aplicar os dois ApplicationSets
kubectl apply -f bootstrap/applicationset.yaml
kubectl apply -f bootstrap/applicationset-namespaces.yaml
```

## Verificando os Recursos Criados

```bash
# Ver Applications criadas
kubectl get applications -n argocd

# Ver namespaces criados
kubectl get namespaces -l managed-by=argocd-root

# Ver ResourceQuotas
kubectl get resourcequotas -A | grep squad
```

## Personalizando ResourceQuotas

Edite `infrastructure/namespace-template/values.yaml` para ajustar os limites:

```yaml
resourceQuota:
  enabled: true
  dev:
    requests.cpu: "2"
    requests.memory: "4Gi"
    # ...
  staging:
    requests.cpu: "4"
    # ...
  prod:
    requests.cpu: "8"
    # ...
```

## Adicionando Nova Squad

1. Crie a pasta do tenant:
   ```bash
   mkdir -p tenants/nova-squad
   cp tenants/squad-empresa/values.yaml tenants/nova-squad/
   ```

2. Edite o `values.yaml` com as configurações da nova squad

3. Commit e push:
   ```bash
   git add tenants/nova-squad
   git commit -m "feat: add nova-squad tenant"
   git push
   ```

4. O ArgoCD Root criará automaticamente:
   - ArgoCD da squad em `nova-squad`
   - Namespaces: `nova-squad-dev`, `nova-squad-staging`, `nova-squad-prod`
