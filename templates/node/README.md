# Template Node.js

Helm chart para deploy de aplicações Node.js (Express, Fastify, NestJS, etc).

## Uso Rápido

### 1. Criar values para sua aplicação

```yaml
# values-minha-api.yaml
app:
  name: minha-api
  port: 3000

image:
  repository: minha-org/minha-api
  tag: "1.0.0"

env:
  NODE_ENV: production
  DATABASE_URL: postgres://...

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### 2. Deploy via ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minha-api
  namespace: squad-empresa
spec:
  source:
    repoURL: https://github.com/Herograme/argocd-multitenancy-lab.git
    path: templates/node
    helm:
      valueFiles:
        - ../../apps/minha-api/values.yaml
  destination:
    namespace: squad-empresa-dev
```

## Recursos Incluídos

| Recurso | Habilitado por Default |
|---------|------------------------|
| Deployment | Sim |
| Service | Sim |
| HPA | Não (`autoscaling.enabled: true`) |
| Ingress | Não (`ingress.enabled: true`) |
| ConfigMap | Não (`configMap.enabled: true`) |
| PDB | Não (`podDisruptionBudget.enabled: true`) |
| ServiceAccount | Não (`serviceAccount.create: true`) |

## Health Checks

Por default, o template configura health checks no endpoint `/health`:

```yaml
healthCheck:
  path: /health
  livenessProbe:
    enabled: true
    initialDelaySeconds: 10
  readinessProbe:
    enabled: true
    initialDelaySeconds: 5
```

Certifique-se que sua aplicação expõe esse endpoint:

```javascript
// Express
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Fastify
fastify.get('/health', async () => ({ status: 'ok' }));

// NestJS
@Get('health')
health() {
  return { status: 'ok' };
}
```

## Variáveis de Ambiente

### Diretas

```yaml
env:
  NODE_ENV: production
  API_URL: https://api.exemplo.com
  LOG_LEVEL: info
```

### Via Secrets

```yaml
envFromSecrets:
  - secretName: db-credentials
    key: password
    envName: DATABASE_PASSWORD
  - secretName: api-keys
    key: stripe-key
    envName: STRIPE_API_KEY
```

### Via ConfigMaps

```yaml
envFromConfigMaps:
  - configMapName: app-config
    key: feature-flags
    envName: FEATURE_FLAGS
```

## Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

## Ingress

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.meudominio.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.meudominio.com
```

## Segurança

O template já vem com boas práticas de segurança:

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

## Exemplo Completo por Ambiente

### Dev

```yaml
# values-dev.yaml
app:
  name: minha-api
  port: 3000

image:
  repository: minha-org/minha-api
  tag: "latest"

replicaCount: 1

env:
  NODE_ENV: development
  LOG_LEVEL: debug

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### Prod

```yaml
# values-prod.yaml
app:
  name: minha-api
  port: 3000

image:
  repository: minha-org/minha-api
  tag: "1.2.3"

replicaCount: 3

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10

env:
  NODE_ENV: production
  LOG_LEVEL: warn

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi

podDisruptionBudget:
  enabled: true
  minAvailable: 2

ingress:
  enabled: true
  hosts:
    - host: api.meudominio.com
      paths:
        - path: /
          pathType: Prefix
```
