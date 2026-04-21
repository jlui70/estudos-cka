# Capítulo 6 — API Objects: Namespaces, Labels, Selectors e Annotations

---

## 1. Namespaces

### O que são

Namespaces são **divisões lógicas** dentro de um cluster Kubernetes. Permitem que múltiplos times, projetos ou ambientes compartilhem o mesmo cluster fisicamente, mas com isolamento lógico.

```
Cluster Kubernetes
├── Namespace: default         (workloads sem namespace explícito)
├── Namespace: kube-system     (componentes internos do K8s)
├── Namespace: producao        (apps de produção)
├── Namespace: staging         (apps de staging)
└── Namespace: time-backend    (time específico)
```

### Recursos com e sem Namespace

| Com Namespace (namespaced) | Sem Namespace (cluster-scoped) |
|---|---|
| Pod, Deployment, Service | Node |
| ConfigMap, Secret | PersistentVolume |
| PVC | StorageClass |
| ServiceAccount | ClusterRole |
| Role, RoleBinding | ClusterRoleBinding |

```bash
# Ver quais recursos são namespaced
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

### Operações com Namespaces

```bash
# Listar namespaces
kubectl get namespaces

# Criar namespace
kubectl create namespace producao

# Operações em namespace específico
kubectl get pods -n producao
kubectl get all -n producao

# Definir namespace padrão para o contexto
kubectl config set-context --current --namespace=producao

# Ver recursos em todos os namespaces
kubectl get pods --all-namespaces
kubectl get pods -A
```

### Namespace em YAML

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: producao
  labels:
    environment: production
    team: platform
```

### Isolamento de DNS por Namespace

Dentro do cluster, o DNS de um Service segue o padrão:
```
<service-name>.<namespace>.svc.cluster.local
```

- Dentro do mesmo namespace: só `<service-name>` basta
- Outro namespace: `<service-name>.<namespace>`
- FQDN completo: `<service-name>.<namespace>.svc.cluster.local`

### Resource Quotas por Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cota-producao
  namespace: producao
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    services: "10"
    persistentvolumeclaims: "5"
```

---

## 2. Labels

### O que são

Labels são **pares chave-valor** que você coloca em qualquer objeto Kubernetes. São a principal forma de organizar, filtrar e selecionar recursos.

```yaml
metadata:
  labels:
    app: frontend           # Qual aplicação
    version: "2.1"          # Versão
    environment: production # Ambiente
    tier: web               # Camada da arquitetura
    team: core-infra        # Time responsável
```

### Regras de nomenclatura

- **Chave**: `[prefixo/]nome`
  - Prefixo opcional: deve ser DNS válido (ex: `app.kubernetes.io/`, `mycompany.com/`)
  - Nome: máx 63 caracteres, alfanumérico, `-`, `_`, `.`
- **Valor**: máx 63 caracteres, pode ser vazio

### Labels recomendados (padrão Kubernetes)

| Label | Exemplo | Descrição |
|---|---|---|
| `app.kubernetes.io/name` | `nginx` | Nome da aplicação |
| `app.kubernetes.io/version` | `1.25.0` | Versão |
| `app.kubernetes.io/component` | `database` | Componente da arquitetura |
| `app.kubernetes.io/part-of` | `ecommerce` | App maior da qual faz parte |
| `app.kubernetes.io/managed-by` | `helm` | Ferramenta que gerencia |

### Adicionar/Remover Labels

```bash
# Adicionar label
kubectl label pod meu-pod ambiente=producao

# Atualizar label
kubectl label pod meu-pod ambiente=staging --overwrite

# Remover label (sufixo -)
kubectl label pod meu-pod ambiente-

# Listar pods com seus labels
kubectl get pods --show-labels
```

---

## 3. Selectors

Selectors permitem **filtrar objetos** pelos seus labels. São usados em Services, Deployments, Jobs, e nos comandos kubectl.

### 3.1 Equality-based Selectors

Operadores: `=`, `==`, `!=`

```bash
kubectl get pods -l app=frontend
kubectl get pods -l app=frontend,environment=production   # AND
kubectl get pods -l environment!=staging
```

### 3.2 Set-based Selectors

Operadores: `in`, `notin`, `exists`

```bash
kubectl get pods -l 'environment in (production, staging)'
kubectl get pods -l 'tier notin (frontend, backend)'
kubectl get pods -l 'app'          # Existe a chave 'app'
kubectl get pods -l '!app'         # Não existe a chave 'app'
```

### 3.3 Selectors em recursos

**Service → seleciona Pods:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend        # Seleciona pods com app=frontend
    tier: web
  ports:
  - port: 80
    targetPort: 8080
```

**Deployment → seleciona seus Pods:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: frontend     # deve bater com template.metadata.labels
  template:
    metadata:
      labels:
        app: frontend   # label nos Pods criados pelo Deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

> **Importante**: O `selector.matchLabels` de um Deployment **deve** corresponder aos `labels` do `template`. Caso contrário o Deployment não saberá quais Pods são seus.

### 3.4 matchExpressions (mais expressivo que matchLabels)

```yaml
selector:
  matchExpressions:
  - key: environment
    operator: In
    values: [production, staging]
  - key: tier
    operator: NotIn
    values: [cache]
  - key: app
    operator: Exists
```

---

## 4. Annotations

### O que são

Annotations são também **pares chave-valor**, mas com propósito diferente dos Labels:

| | Labels | Annotations |
|---|---|---|
| Usado para seleção | Sim | **Não** |
| Tamanho do valor | Pequeno | Pode ser grande |
| Consulta por API | Sim (selector) | Não |
| Propósito | Identificar e selecionar | Armazenar metadados arbitrários |

### Quando usar Annotations

- Informações para ferramentas externas (Helm, Ingress Controller, Prometheus)
- Timestamps de build/deploy
- Contato do time responsável
- Links para documentação ou tickets
- Dados de auditoria

```yaml
metadata:
  annotations:
    # Para o Ingress NGINX
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Metadados de build
    git-commit: "a1b2c3d4"
    build-date: "2024-01-15T10:30:00Z"
    ci-pipeline: "https://jenkins.mycompany.com/job/123"

    # Documentação
    description: "Backend API do sistema de pagamentos"
    team-contact: "squad-payments@mycompany.com"

    # Para Prometheus
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

### Diferença prática entre Labels e Annotations

```yaml
metadata:
  name: payment-api
  labels:
    # LABELS: usados por Services, Deployments, etc. para SELECIONAR este Pod
    app: payment-api
    version: "3.2"
    environment: production
  annotations:
    # ANNOTATIONS: metadados para HUMANOS e FERRAMENTAS, não para seleção
    description: "Processador de pagamentos com cartão"
    owner: "squad-payments@company.com"
    last-reviewed: "2024-01-10"
```

---

## 5. Field Selectors

Além de labels, você pode filtrar recursos por campos:

```bash
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector spec.nodeName=worker-1
kubectl get pods --field-selector metadata.namespace=default
kubectl get events --field-selector reason=Failed
```

---

## 6. Resumo Comparativo

| Conceito | Propósito | Exemplo de uso |
|---|---|---|
| **Namespace** | Isolamento lógico de recursos | Separar dev/staging/prod no mesmo cluster |
| **Label** | Identificar e selecionar recursos | Service selecionar Pods, filtrar com kubectl |
| **Selector** | Filtrar recursos por labels | matchLabels em Deployment, -l em kubectl |
| **Annotation** | Metadados não-selecionáveis | Info de build, configuração de Ingress controller |
| **ResourceQuota** | Limitar recursos por namespace | Max 20 pods, 4 CPUs no namespace `staging` |
| **LimitRange** | Definir defaults de CPU/mem | Pod sem requests/limits recebe valores padrão |

---

## 7. LimitRange

O `LimitRange` define valores padrão e limites para containers em um namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limites-default
  namespace: staging
spec:
  limits:
  - type: Container
    default:          # Limit default (se não especificado no container)
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # Request default
      cpu: "100m"
      memory: "128Mi"
    max:              # Máximo permitido
      cpu: "2"
      memory: "1Gi"
    min:              # Mínimo exigido
      cpu: "50m"
      memory: "64Mi"
```

> Se um container não especifica `resources`, o Admission Controller `LimitRanger` aplica os valores `default`/`defaultRequest` automaticamente.
