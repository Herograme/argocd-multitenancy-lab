# Argo CD Multi-Tenancy Lab

Repositório de configuração para ambiente Argo CD multi-tenancy.

## Estrutura

```
.
├── bootstrap/
│   └── applicationset.yaml    # ApplicationSet que cria Argos automaticamente
├── tenants/
│   ├── squad-empresa/         # Configurações da Squad Empresa
│   │   └── values.yaml
│   ├── squad-faturamento/     # Configurações da Squad Faturamento
│   │   └── values.yaml
│   └── squad-devops/          # Configurações da Squad DevOps
│       └── values.yaml
└── README.md
```

## Como funciona

1. O **Argo CD Root** monitora este repositório
2. O **ApplicationSet** detecta pastas em `/tenants/`
3. Para cada pasta, cria automaticamente um Argo CD isolado no namespace correspondente

## Adicionar nova squad

1. Criar pasta em `tenants/<nome-da-squad>/`
2. Adicionar `values.yaml` com configurações
3. Commit e push
4. Argo CD Root cria automaticamente o tenant
