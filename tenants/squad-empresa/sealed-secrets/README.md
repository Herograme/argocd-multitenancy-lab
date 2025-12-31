# Sealed Secrets - Squad Empresa

## O que e Sealed Secrets?

Sealed Secrets permite armazenar secrets encriptados no Git de forma segura.
Apenas o controller no cluster consegue decriptar.

## Pre-requisitos

1. Instalar o controller no cluster:
```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system
```

2. Instalar o CLI kubeseal:
```bash
# Linux AMD64
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.5/kubeseal-0.24.5-linux-amd64.tar.gz

# Linux ARM64 (Oracle ARM, Raspberry Pi, etc)
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.5/kubeseal-0.24.5-linux-arm64.tar.gz

tar -xvf kubeseal-*.tar.gz
sudo mv kubeseal /usr/local/bin/
```

## Como gerar um SealedSecret

### Passo 1: Criar o Secret normal (dry-run)
```bash
kubectl create secret generic oidc-secret \
  --namespace=squad-empresa \
  --from-literal=clientSecret=SEU_SECRET_AQUI \
  --dry-run=client -o yaml > /tmp/secret.yaml
```

### Passo 2: Encriptar com kubeseal
```bash
kubeseal --format yaml < /tmp/secret.yaml > oidc-secret.yaml
```

### Passo 3: Commit do SealedSecret (seguro!)
```bash
git add oidc-secret.yaml
git commit -m "Add sealed secret for OIDC"
git push
```

## Como funciona no ArgoCD

O ArgoCD aplica o SealedSecret no cluster:
```
SealedSecret (Git) --> ArgoCD --> Cluster --> Controller decripta --> Secret real
```

O values.yaml referencia o secret:
```yaml
configs:
  cm:
    oidc.config: |
      clientSecret: $oidc-secret:clientSecret
```

## Arquivos nesta pasta

- `oidc-secret.yaml` - SealedSecret para credenciais OIDC (Cloudflare/GitHub)
- Adicione outros secrets conforme necessario
