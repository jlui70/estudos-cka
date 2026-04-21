# Capítulo 15 — Security

---

## 1. Modelo de Segurança do Kubernetes

A segurança no K8s é organizada em camadas (defense in depth):

```
Cluster
  └── Nodes
        └── Pods
              └── Containers
                    └── Aplicação
```

| Camada | Mecanismo |
|---|---|
| Acesso à API | Autenticação + Autorização (RBAC) + Admission Controllers |
| Redes | Network Policies |
| Pods/Containers | Security Contexts, Pod Security Standards |
| Workload identity | ServiceAccounts |
| Dados sensíveis | Secrets (+ Encryption at Rest) |

---

## 2. RBAC — Role-Based Access Control

### Conceito

RBAC controla **quem** pode fazer **o quê** em **quais recursos**.

```
Sujeito (Subject) + Verbo (Verb) + Recurso (Resource) = Permissão
```

### Sujeitos (Subjects)

| Tipo | Descrição |
|---|---|
| `User` | Usuário humano (não existe como objeto K8s — vem do certificado ou token) |
| `Group` | Grupo de usuários |
| `ServiceAccount` | Identidade de uma aplicação rodando no cluster |

### Verbos (Verbs)

| Verbo | Operação HTTP |
|---|---|
| `get` | GET (um recurso) |
| `list` | GET (coleção) |
| `watch` | GET com long-polling |
| `create` | POST |
| `update` | PUT |
| `patch` | PATCH |
| `delete` | DELETE |
| `deletecollection` | DELETE (coleção) |

### 2.1 Role (escopo: Namespace)

Define permissões dentro de um namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: producao
rules:
- apiGroups: [""]              # "" = core API group (v1)
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
```

### 2.2 ClusterRole (escopo: Cluster)

Define permissões globais ou para recursos não-namespaced:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]          # Nodes são cluster-scoped
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
```

### 2.3 RoleBinding (escopo: Namespace)

Associa um Role (ou ClusterRole) a um sujeito dentro de um namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: joao-pod-reader
  namespace: producao
subjects:
- kind: User
  name: joao          # Nome do usuário (CN do certificado)
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: minha-app
  namespace: producao
roleRef:
  kind: Role           # Role ou ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

> **Importante**: Um RoleBinding pode referir-se a um **ClusterRole** — nesse caso, o ClusterRole é aplicado apenas dentro do namespace do RoleBinding.

### 2.4 ClusterRoleBinding (escopo: Cluster)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-global
subjects:
- kind: User
  name: admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin        # ClusterRole embutido: acesso total
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoles embutidos

| ClusterRole | Permissões |
|---|---|
| `cluster-admin` | Acesso total a tudo |
| `admin` | Acesso total a um namespace |
| `edit` | Leitura e escrita, sem RBAC |
| `view` | Somente leitura |

### Comandos RBAC

```bash
# Criar Role imperativo
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  --namespace=producao

# Criar RoleBinding
kubectl create rolebinding joao-pod-reader \
  --role=pod-reader \
  --user=joao \
  --namespace=producao

# Verificar o que um usuário pode fazer
kubectl auth can-i get pods --as=joao -n producao
kubectl auth can-i create deployments --as=joao -n producao

# Listar permissões do usuário atual
kubectl auth can-i --list
kubectl auth can-i --list --namespace=producao

# Ver roles existentes
kubectl get roles -n producao
kubectl get clusterroles
kubectl get rolebindings -n producao
```

---

## 3. ServiceAccounts

### O que é

Um **ServiceAccount** é a identidade usada por **Pods** para autenticar na API do K8s.

- Todo Pod usa um ServiceAccount (por padrão: `default` do namespace)
- O token do ServiceAccount é montado automaticamente em `/var/run/secrets/kubernetes.io/serviceaccount/`
- Boa prática: criar ServiceAccounts dedicados por aplicação

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: minha-app-sa
  namespace: producao
automountServiceAccountToken: false   # Desabilita mount automático se app não precisa da API
```

```yaml
# Usar o ServiceAccount no Pod
spec:
  serviceAccountName: minha-app-sa
  automountServiceAccountToken: false  # Ou desabilitar por Pod
```

```bash
# Criar ServiceAccount imperativo
kubectl create serviceaccount minha-app-sa -n producao

# Dar permissões ao ServiceAccount
kubectl create rolebinding minha-app-binding \
  --role=pod-reader \
  --serviceaccount=producao:minha-app-sa \
  --namespace=producao
```

---

## 4. Network Policies

### Conceito

Por padrão, todos os Pods podem se comunicar com todos os outros Pods no cluster. **Network Policies** permitem restringir esse tráfego.

> **Importante**: Network Policies só funcionam se o CNI Plugin do cluster suportar (Calico, Cilium, Weave suportam. Flannel simples NÃO suporta).

### Estrutura

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: producao
spec:
  podSelector:           # A quais Pods esta policy se aplica
    matchLabels:
      app: database
  policyTypes:
  - Ingress              # Controlar tráfego de entrada
  - Egress               # Controlar tráfego de saída
  ingress:               # Regras de entrada (ALLOWED)
  - from:
    - podSelector:         # Permitir apenas de Pods com app=backend
        matchLabels:
          app: backend
      namespaceSelector:   # Apenas do namespace producao
        matchLabels:
          name: producao
    ports:
    - protocol: TCP
      port: 5432
  egress:                # Regras de saída
  - to:
    - ipBlock:             # Permitir saída para range de IPs
        cidr: 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 443
```

### Cheat Sheet de Network Policies

```yaml
# Bloquear TODO tráfego de entrada (deny all ingress)
spec:
  podSelector: {}        # Aplica a todos os Pods
  policyTypes: [Ingress]
# Sem regras ingress = nega tudo

# Permitir todo tráfego de entrada (allow all ingress)
spec:
  podSelector: {}
  ingress:
  - {}                   # {} = permite tudo

# Bloquear todo tráfego de saída
spec:
  podSelector: {}
  policyTypes: [Egress]

# Permitir tráfego apenas do mesmo namespace
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}    # Qualquer Pod
```

---

## 5. Security Contexts

**Security Context** define configurações de segurança para um Pod ou container.

### No nível do Pod

```yaml
spec:
  securityContext:
    runAsUser: 1000          # UID do processo no container
    runAsGroup: 3000         # GID do processo
    runAsNonRoot: true       # Falha se container rodar como root
    fsGroup: 2000            # GID para volumes montados
    sysctls:                 # Parâmetros de kernel
    - name: net.core.somaxconn
      value: "1024"
```

### No nível do Container

```yaml
containers:
- name: app
  image: myapp:1.0
  securityContext:
    allowPrivilegeEscalation: false   # Impede ganho de privilégio
    readOnlyRootFilesystem: true      # Filesystem raiz somente leitura
    capabilities:
      drop:
      - ALL               # Remove todas as Linux capabilities
      add:
      - NET_BIND_SERVICE  # Adiciona apenas o necessário (bind em ports < 1024)
    privileged: false
```

### Boa prática: Least Privilege

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]
```

---

## 6. Pod Security Standards (PSS)

O PSS substitui o antigo PodSecurityPolicy (removido no K8s 1.25). Define três perfis:

| Perfil | Restrição | Uso |
|---|---|---|
| `privileged` | Nenhuma restrição | Componentes de sistema, add-ons |
| `baseline` | Restrições mínimas (bloqueia pods privilegiados) | Apps genéricas |
| `restricted` | Restrições fortes (best practices de segurança) | Workloads sensíveis |

### Habilitando por Namespace via Label

```bash
# Enforce (rejeita Pods que violam)
kubectl label namespace producao pod-security.kubernetes.io/enforce=restricted

# Warn (permite mas avisa)
kubectl label namespace staging pod-security.kubernetes.io/warn=baseline

# Audit (registra em audit log mas permite)
kubectl label namespace dev pod-security.kubernetes.io/audit=baseline
```

---

## 7. Encryption at Rest

Por padrão, Secrets são armazenados em **texto plano** (apenas base64) no etcd. Para criptografar:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}    # Fallback para leitura de dados não-criptografados
```

E referenciar no API Server: `--encryption-provider-config=/etc/kubernetes/encryption-config.yaml`

---

## 8. Conceitos-Chave

| Conceito | Definição |
|---|---|
| RBAC | Controle de acesso baseado em funções |
| Role | Permissões dentro de um namespace |
| ClusterRole | Permissões globais ou para recursos não-namespaced |
| RoleBinding | Associa Role a um sujeito (User/SA/Group) em um namespace |
| ClusterRoleBinding | Associa ClusterRole a um sujeito globalmente |
| ServiceAccount | Identidade de uma aplicação (Pod) para acessar a API |
| Network Policy | Regra de firewall para tráfego entre Pods |
| Security Context | Configurações de segurança do processo (user, capabilities) |
| Pod Security Standards | Perfis de segurança (privileged/baseline/restricted) por namespace |
| `kubectl auth can-i` | Verifica se usuário tem permissão para uma ação |
