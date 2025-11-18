# 🎬 Vídeo 4.1 - GitOps com ArgoCD

**Aula**: 4 - GitOps  
**Vídeo**: 4.1  
**Temas**: GitOps; ArgoCD; Continuous Deployment; Sync  

---

## 📚 Parte 1: Conceito GitOps

### Passo 1: O que é GitOps?

#### CI/CD Tradicional vs GitOps

```mermaid
graph TB
    subgraph "❌ Push Model (Tradicional)"
        A1[Code] --> A2[CI Pipeline]
        A2 --> A3[Build & Test]
        A3 --> A4[kubectl apply]
        A4 -->|push direto| A5[Cluster]
    end
    
    subgraph "✅ Pull Model (GitOps)"
        B1[Code] --> B2[CI Pipeline]
        B2 --> B3[Build & Test]
        B3 --> B4[Update Git Repo]
        B4 --> B5[Git Repository]
        B6[GitOps Agent] -->|pull| B5
        B6 -->|sync| B7[Cluster]
    end
```

**Diferenças:**

| Aspecto | Push Model | Pull Model (GitOps) |
|---------|------------|---------------------|
| **Acesso ao Cluster** | ❌ Pipeline precisa credenciais | ✅ Agent no cluster |
| **Auditoria** | ❌ Difícil rastrear | ✅ Git history completo |
| **Estado** | ❌ Pode divergir | ✅ Git = Source of Truth |
| **Self-Healing** | ❌ Manual | ✅ Automático |
| **Segurança** | ❌ Credenciais expostas | ✅ Sem credenciais externas |

---

## ⚙️ Parte 2: Setup ArgoCD

### Passo 2: Criar Cluster EKS

```bash
# Verificar credenciais AWS
aws sts get-caller-identity --profile fiapaws

# Obter Account ID
ACCOUNT_ID=$(aws sts get-caller-identity --profile fiapaws --query Account --output text)

# Discovery automático de subnets (excluindo us-east-1e)
SUBNET_IDS=$(aws ec2 describe-subnets \
  --profile fiapaws \
  --region us-east-1 \
  --filters "Name=map-public-ip-on-launch,Values=true" \
  --query 'Subnets[?AvailabilityZone!=`us-east-1e`].SubnetId' \
  --output text | tr '\t' ',')

echo "Subnets descobertas: $SUBNET_IDS"

# Criar cluster EKS
aws eks create-cluster \
  --name cicd-lab \
  --region us-east-1 \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/LabRole \
  --resources-vpc-config subnetIds=${SUBNET_IDS} \
  --profile fiapaws

# Aguardar cluster ficar ativo (15-20 minutos)
echo "⏳ Aguardando cluster ficar ativo..."
aws eks wait cluster-active \
  --name cicd-lab \
  --region us-east-1 \
  --profile fiapaws

# Discovery de subnets para node group
SUBNET_IDS_SPACE=$(aws ec2 describe-subnets \
  --profile fiapaws \
  --region us-east-1 \
  --filters "Name=map-public-ip-on-launch,Values=true" \
  --query 'Subnets[?AvailabilityZone!=`us-east-1e`].SubnetId' \
  --output text)

# Criar node group
aws eks create-nodegroup \
  --cluster-name cicd-lab \
  --nodegroup-name workers \
  --node-role arn:aws:iam::${ACCOUNT_ID}:role/LabRole \
  --subnets $SUBNET_IDS_SPACE \
  --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=2,desiredSize=2 \
  --region us-east-1 \
  --profile fiapaws

# Aguardar node group ficar ativo
echo "⏳ Aguardando node group ficar ativo..."
aws eks wait nodegroup-active \
  --cluster-name cicd-lab \
  --nodegroup-name workers \
  --region us-east-1 \
  --profile fiapaws

# Configurar kubectl
aws eks update-kubeconfig \
  --name cicd-lab \
  --region us-east-1 \
  --profile fiapaws

# Verificar nodes
kubectl get nodes
```

### Passo 3: Instalar ArgoCD

```bash
# Criar namespace
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Aguardar pods ficarem prontos (2-3 min)
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Ver pods
kubectl get pods -n argocd
```

### Passo 4: Acessar ArgoCD UI

```bash
# Expor ArgoCD via port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Obter senha inicial
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "Password: $ARGOCD_PASSWORD"

# Acessar UI
open https://localhost:8080
# Login: admin / <senha_obtida>
# Aceitar certificado self-signed
```

### Passo 5: Instalar ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Verificar
argocd version
```

### Passo 6: Login via CLI

```bash
# Login
argocd login localhost:8080 \
  --username admin \
  --password $ARGOCD_PASSWORD \
  --insecure

# Mudar senha (opcional)
argocd account update-password
```

---

## 📁 Parte 3: Preparar GitOps Repository

### Passo 7: Entender Estrutura do GitOps Repo

```bash
cd fiap-dclt-aula04

# Ver estrutura completa
tree gitops-repo/
```

**Estrutura Completa:**
```
gitops-repo/
├── README.md                                    # Documentação do repositório
│
├── applications/                                # 📁 Definições de Aplicações
│   ├── fiap-todo-api/                          # Aplicação Todo API
│   │   ├── base/                               # 🔷 Manifests Base (comum a todos ambientes)
│   │   │   ├── deployment.yaml                 #    - Deployment da aplicação
│   │   │   ├── service.yaml                    #    - Service (ClusterIP)
│   │   │   └── kustomization.yaml              #    - Kustomize base config
│   │   │
│   │   └── overlays/                           # 🔶 Overlays (específico por ambiente)
│   │       └── production/                     #    - Ambiente de Produção
│   │           ├── kustomization.yaml          #      - Kustomize overlay config
│   │           └── deployment-patch.yaml       #      - Patches (replicas, resources, etc)
│   │
│   └── fiap-todo-api-app.yaml                  # 🎯 ArgoCD Application (define deploy)
│
└── clusters/                                    # 📁 Configurações FluxCD por Cluster
    └── production/                              # Cluster de Produção
        ├── fiap-todo-api-source.yaml           #    - GitRepository (source)
        └── fiap-todo-api-kustomization.yaml    #    - Kustomization (deploy)
```

**📖 Explicação da Estrutura:**

**1. `applications/fiap-todo-api/base/`** - Manifests Base
   - Contém os recursos Kubernetes **comuns a todos os ambientes**
   - `deployment.yaml`: Define pods, containers, imagem
   - `service.yaml`: Expõe a aplicação internamente
   - `kustomization.yaml`: Lista os recursos base

**2. `applications/fiap-todo-api/overlays/production/`** - Overlay de Produção
   - **Customiza** os manifests base para produção
   - `deployment-patch.yaml`: Altera replicas, resources, labels
   - `kustomization.yaml`: Referencia base + aplica patches
   - **Vantagem**: Mesma base, configurações diferentes por ambiente

**3. `applications/fiap-todo-api-app.yaml`** - ArgoCD Application
   - Define **como o ArgoCD** deve fazer deploy
   - Aponta para: Git repo + path dos manifests
   - Configura: auto-sync, self-healing, namespace destino

**4. `clusters/production/`** - Configurações FluxCD
   - Alternativa ao ArgoCD (mesmo propósito)
   - `*-source.yaml`: Define repositório Git
   - `*-kustomization.yaml`: Define como aplicar manifests

**🎯 Padrão Kustomize:**
```
Base (comum) + Overlay (específico) = Manifests Finais
```

**Exemplo:**
- **Base**: 2 replicas, 128Mi RAM
- **Overlay Production**: 5 replicas, 512Mi RAM
- **Resultado**: Deploy com 5 replicas e 512Mi RAM

### Passo 8: Ver Manifests Base

```bash
# Ver deployment
cat gitops-repo/applications/fiap-todo-api/base/deployment.yaml

# Ver service
cat gitops-repo/applications/fiap-todo-api/base/service.yaml

# Ver kustomization
cat gitops-repo/applications/fiap-todo-api/base/kustomization.yaml
```

---

## 🚀 Parte 4: Deploy com ArgoCD

### Passo 9: Criar Application no ArgoCD

```bash
# Ver Application manifest
cat gitops-repo/applications/fiap-todo-api-app.yaml
```

**fiap-todo-api-app.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fiap-todo-api
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/josenetoo/fiap-dclt-aula04
    targetRevision: HEAD
    path: gitops-repo/applications/fiap-todo-api/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: fiap-todo-prod
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Passo 10: Aplicar Application

```bash
# Criar namespace para a aplicação
kubectl create namespace fiap-todo-prod

# Aplicar Application
kubectl apply -f gitops-repo/applications/fiap-todo-api-app.yaml

# Ver Application
argocd app list

# Ver detalhes
argocd app get fiap-todo-api
```

### Passo 11: Sync Manual (primeira vez)

```bash
# Sync da aplicação
argocd app sync fiap-todo-api

# Aguardar sync completar
argocd app wait fiap-todo-api --health

# Ver status
argocd app get fiap-todo-api
```

---

## 🔄 Parte 5: Testar GitOps Workflow

### Passo 12: Ver Aplicação Deployada

```bash
# Ver pods
kubectl get pods -n fiap-todo-prod

# Ver service
kubectl get service -n fiap-todo-prod

# Ver logs
kubectl logs -l app=fiap-todo-api -n fiap-todo-prod --tail=50
```

### Passo 13: Fazer Mudança no Git

```bash
cd ~/fiap-cicd-handson/aula-04

# Editar deployment (aumentar replicas)
cat > gitops-repo/applications/fiap-todo-api/overlays/production/deployment-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fiap-todo-api
spec:
  replicas: 5  # Era 3, agora 5!
  template:
    spec:
      containers:
      - name: api
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
EOF

# Commit e push
git add gitops-repo/
git commit -m "feat: aumentar replicas para 5"
git push origin main
```

### Passo 14: Ver Auto-Sync

```bash
# ArgoCD vai detectar a mudança automaticamente (3 min)
# Ver status
argocd app get fiap-todo-api

# Acompanhar sync
watch kubectl get pods -n fiap-todo-prod

# Após sync, verificar replicas
kubectl get deployment -n fiap-todo-prod
# Deve mostrar 5 replicas!
```

---

## 🔍 Parte 6: ArgoCD UI

### Passo 15: Explorar UI

**No ArgoCD UI (https://localhost:8080):**

1. **Applications** → Ver `fiap-todo-api`
2. Clicar na aplicação
3. Ver:
   - **Topology**: Visualização gráfica dos recursos
   - **Sync Status**: Synced / OutOfSync
   - **Health Status**: Healthy / Degraded
   - **Last Sync**: Timestamp da última sincronização

4. **App Details**:
   - Source: Git repo e path
   - Destination: Cluster e namespace
   - Sync Policy: Auto-sync habilitado

5. **History**:
   - Ver histórico de syncs
   - Cada commit do Git aparece aqui

### Passo 16: Testar Self-Healing

```bash
# Deletar um pod manualmente
POD=$(kubectl get pods -n fiap-todo-prod -l app=fiap-todo-api -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD -n fiap-todo-prod

# ArgoCD vai recriar automaticamente!
# Ver no UI: Self-healing em ação
kubectl get pods -n fiap-todo-prod -w
```

---

## 🎓 Parte 7: Conceitos Aprendidos

### Passo 17: Arquitetura ArgoCD

```mermaid
graph TB
    A[Developer] -->|git push| B[Git Repository]
    B -->|poll every 3min| C[ArgoCD Server]
    C -->|sync| D[Kubernetes Cluster]
    C -->|UI/CLI| E[Users]
    D -->|status| C
    
    subgraph "ArgoCD Components"
        C
        F[Application Controller]
        G[Repo Server]
        H[Redis]
    end
    
    C --> F
    F --> G
    G --> H
```

**Fluxo GitOps com ArgoCD:**

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Git as Git Repo
    participant Argo as ArgoCD
    participant K8s as Cluster
    
    Dev->>Git: 1. Commit manifests
    Note over Git: Source of Truth
    
    Argo->>Git: 2. Poll (every 3min)
    Git-->>Argo: Detect changes
    
    Argo->>K8s: 3. Apply manifests
    K8s-->>Argo: Report status
    
    Note over Argo,K8s: Self-Healing Active
    
    K8s->>K8s: 4. Pod deleted manually
    Argo->>K8s: 5. Recreate pod
    Note over K8s: Desired state restored
```

**Conceitos-chave:**
- ✅ **Git como source of truth** - Única fonte da verdade
- ✅ **Pull model** - Agent no cluster puxa mudanças
- ✅ **Auto-sync** - Sincronização automática (3 min)
- ✅ **Self-healing** - Recuperação automática de estado
- ✅ **Auditoria** - Git history completo
- ✅ **Rollback** - git revert para voltar versões
- ✅ **UI visual** - Interface gráfica para monitoramento

---

**FIM DO VÍDEO 4.1** ✅
