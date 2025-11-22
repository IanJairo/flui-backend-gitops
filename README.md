# Flui Backend - GitOps Repository

Repositório GitOps para deploy da aplicação Flui Backend usando ArgoCD com padrão **App-of-Apps**.

## 📂 Estrutura

```
flui-backend-gitops/
├── app-of-apps.yaml              # Application raiz (App-of-Apps)
├── applications/                 # Applications ArgoCD gerenciadas
│   ├── postgres-operator.yaml    # CloudNative-PG Operator (sync-wave: 0)
│   ├── postgres-cluster-dev.yaml # PostgreSQL DEV (sync-wave: 1)
│   ├── postgres-cluster-prod.yaml# PostgreSQL PROD (sync-wave: 1)
│   ├── flui-backend-dev.yaml     # Aplicação DEV (sync-wave: 2)
│   └── flui-backend-prod.yaml    # Aplicação PROD (sync-wave: 2)
└── manifests/                    # Manifestos Kubernetes
    ├── postgres-dev/
    │   └── postgres-cluster.yaml
    └── postgres-prod/
        └── postgres-cluster.yaml
```

## 🚀 Deploy Inicial (App-of-Apps)

### Pré-requisitos

1. Kubernetes cluster rodando (Docker Desktop)
2. ArgoCD instalado
3. Secrets criados nos namespaces

### Passo 1: Criar Secrets

#### DEV (namespace: flui-back-dev)

```bash
kubectl create namespace flui-back-dev

kubectl create secret generic postgres-superuser \
  -n flui-back-dev \
  --from-literal=username=postgres \
  --from-literal=password=PostgresDev123!

kubectl create secret generic flui-db-secret \
  -n flui-back-dev \
  --from-literal=DB_USER=flui \
  --from-literal=DB_PASSWORD=SenhaDevForte@2025

kubectl create secret docker-registry ghcr-secret \
  -n flui-back-dev \
  --docker-server=ghcr.io \
  --docker-username=ianjairo \
  --docker-password=<SEU_GITHUB_TOKEN> \
  --docker-email=ian.jairo.silta@gmail.com
```

#### PROD (namespace: flui-prod)

```bash
kubectl create namespace flui-prod

kubectl create secret generic postgres-superuser-prod \
  -n flui-prod \
  --from-literal=username=postgres \
  --from-literal=password=PostgresProd123!

kubectl create secret generic flui-db-prod-secret \
  -n flui-prod \
  --from-literal=DB_USER=flui \
  --from-literal=DB_PASSWORD=SenhaProdForte@2025

kubectl create secret docker-registry ghcr-secret \
  -n flui-prod \
  --docker-server=ghcr.io \
  --docker-username=ianjairo \
  --docker-password=<SEU_GITHUB_TOKEN> \
  --docker-email=ian.jairo.silta@gmail.com
```

### Passo 2: Aplicar App-of-Apps

**ATENÇÃO:** Este comando cria TODAS as applications automaticamente!

```bash
kubectl apply -f app-of-apps.yaml
```

### Passo 3: Verificar Deploy

```bash
# Ver a application raiz
kubectl get application flui-root -n argocd

# Ver todas as applications criadas
kubectl get applications -n argocd

# Deve mostrar:
# - flui-root (App-of-Apps)
# - postgres-operator
# - postgres-cluster-dev
# - postgres-cluster-prod
# - flui-backend-dev
# - flui-backend-prod
```

### Passo 4: Acessar ArgoCD UI

```bash
# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Abrir: https://localhost:8080
# Usuário: admin
# Senha:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 🔄 Ordem de Deploy (Sync Waves)

O App-of-Apps garante a ordem correta:

```
0. postgres-operator       (Instala CloudNative-PG)
   ↓
1. postgres-cluster-dev    (Cria PostgreSQL DEV)
   postgres-cluster-prod   (Cria PostgreSQL PROD)
   ↓
2. flui-backend-dev        (Deploy aplicação DEV)
   flui-backend-prod       (Deploy aplicação PROD)
```

## 🎯 Como Funciona (App-of-Apps)

1. Você aplica **apenas** `app-of-apps.yaml`
2. `flui-root` monitora a pasta `applications/`
3. `flui-root` cria automaticamente todas as Applications
4. Cada Application gerencia seus recursos

## ➕ Adicionar Novo Ambiente

Para adicionar um novo ambiente (ex: staging):

1. Criar Application:
```yaml
# applications/flui-backend-staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flui-backend-staging
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  # ... configuração
```

2. Commit e push
3. ArgoCD detecta e cria automaticamente!

## 🔧 Comandos Úteis

```bash
# Ver status da App-of-Apps
kubectl get application flui-root -n argocd -o yaml

# Forçar sync de tudo
kubectl patch application flui-root -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Deletar tudo (cuidado!)
kubectl delete application flui-root -n argocd

# Ver logs de sync
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100
```

## 📊 Ambientes

### DEV
- **Namespace**: `flui-back-dev`
- **Branch**: `dev`
- **PostgreSQL**: 1 instância
- **Replicas**: 1
- **Autoscaling**: Desabilitado

### PROD
- **Namespace**: `flui-prod`
- **Branch**: `main`
- **PostgreSQL**: 3 instâncias (HA)
- **Replicas**: 3-10 (HPA)
- **Autoscaling**: Habilitado

## 🔗 Repositórios Relacionados

- **Aplicação**: https://github.com/IanJairo/Flui-BackEnd
- **GitOps**: https://github.com/IanJairo/flui-backend-gitops (este)

## 📚 Referências

- [ArgoCD App-of-Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [CloudNative-PG](https://cloudnative-pg.io/)
- [Helm](https://helm.sh/)
