# Capítulo 5 — APIs e Acesso ao Kubernetes

---

## 1. A API do Kubernetes

Toda interação com o Kubernetes passa pelo **API Server** via API RESTful. Seja pelo `kubectl`, por um controller interno ou por uma aplicação externa — tudo é uma chamada HTTP à API.

### Estrutura da API

A API é organizada em **grupos** e **versões**:

```
/api/v1                          → Core group (Pod, Service, ConfigMap, etc.)
/apis/apps/v1                    → apps group (Deployment, ReplicaSet, DaemonSet)
/apis/batch/v1                   → batch group (Job, CronJob)
/apis/networking.k8s.io/v1       → networking group (Ingress, NetworkPolicy)
/apis/rbac.authorization.k8s.io/v1 → RBAC group
/apis/storage.k8s.io/v1          → storage group (StorageClass, PV, PVC)
```

### Versionamento da API

| Versão | Estabilidade | Exemplo |
|---|---|---|
| `v1` | Estável (GA) | Pod, Service, ConfigMap |
| `v1beta1`, `v1beta2` | Beta — API pode mudar | PodDisruptionBudget (beta) |
| `v1alpha1` | Alpha — pode ser removido | Recursos experimentais |

> Para a CKA, você trabalha quase sempre com `v1` e `apps/v1`.

### Verificar API groups disponíveis

```bash
kubectl api-resources          # Lista todos os recursos e seus API groups
kubectl api-versions           # Lista todas as versões de API disponíveis
kubectl explain pod            # Documentação inline de um recurso
kubectl explain pod.spec       # Documentação de um campo específico
```

---

## 2. kubectl — A CLI do Kubernetes

O `kubectl` é a ferramenta de linha de comando para interagir com o cluster. Ele converte seus comandos em chamadas à API REST.

### Padrão de comandos

```bash
kubectl [verbo] [recurso] [nome] [flags]
```

### Verbos principais

| Verbo | Descrição |
|---|---|
| `get` | Lista recursos |
| `describe` | Detalhes de um recurso |
| `apply` | Cria ou atualiza (declarativo) |
| `create` | Cria (imperativo) |
| `delete` | Remove um recurso |
| `edit` | Abre o recurso no editor para edição ao vivo |
| `exec` | Executa comando dentro de um container |
| `logs` | Exibe logs de um Pod |
| `port-forward` | Redireciona porta local para o Pod |
| `cp` | Copia arquivos para/de um Pod |
| `rollout` | Gerencia rollouts de Deployments |
| `scale` | Altera número de réplicas |
| `patch` | Modifica campos específicos de um recurso |

### Flags úteis

```bash
-n, --namespace    # Especifica o namespace
-A, --all-namespaces  # Todos os namespaces
-o yaml            # Saída em YAML
-o json            # Saída em JSON
-o wide            # Saída com mais colunas
--watch, -w        # Modo watch — atualiza em tempo real
--dry-run=client   # Simula a operação sem aplicar
--force            # Força a operação (delete + recreate)
-l, --selector     # Filtra por label
```

### Dicas para a CKA

```bash
# Gerar YAML de um recurso sem criar
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment myapp --image=nginx --dry-run=client -o yaml > deploy.yaml

# Editar recurso ao vivo
kubectl edit deployment myapp

# Ver eventos (muito útil para troubleshoot)
kubectl get events --sort-by=.metadata.creationTimestamp

# Forçar deleção de pod stuck em Terminating
kubectl delete pod <nome> --force --grace-period=0
```

---

## 3. Autenticação (Authentication)

A autenticação responde: **"Quem é você?"**

O API Server suporta múltiplos métodos de autenticação simultaneamente:

### 3.1 Certificados X.509 (Client Certificate)

O método mais comum em clusters kubeadm. O usuário apresenta um certificado TLS assinado pela CA do cluster.

```
Usuário tem: certificado.crt + chave.key
API Server verifica: o certificado foi assinado pela CA confiável?
```

O `CN` (Common Name) do certificado é o nome do usuário. O `O` (Organization) é o grupo.

```bash
# Ver quem você é baseado no kubeconfig
kubectl config view --minify
# O certificado client-certificate-data contém seu CN (username)
```

### 3.2 Bearer Tokens

Um token string apresentado no header HTTP:
```
Authorization: Bearer <token>
```

Tipos de tokens:
- **ServiceAccount tokens**: Gerados automaticamente para os ServiceAccounts
- **Bootstrap tokens**: Usados durante kubeadm join
- **OIDC tokens**: Tokens JWT de um provedor de identidade externo

### 3.3 ServiceAccount (para aplicações)

Pods usam **ServiceAccounts** para se autenticar na API. O token do ServiceAccount é montado automaticamente em `/var/run/secrets/kubernetes.io/serviceaccount/token`.

```yaml
spec:
  serviceAccountName: meu-serviceaccount
```

### 3.4 OIDC (OpenID Connect)

Integração com provedores de identidade externos (Keycloak, Google, Azure AD). O usuário obtém um JWT do provedor e usa como Bearer token. Muito usado em organizações.

### 3.5 Anonymous Access

Por padrão, requests não autenticados recebem o usuário `system:anonymous` e o grupo `system:unauthenticated`. Geralmente sem permissões úteis.

---

## 4. Autorização (Authorization)

A autorização responde: **"Você pode fazer isso?"**

Após a autenticação, o API Server verifica se o usuário tem permissão para a operação desejada.

### Modos de Autorização

| Modo | Descrição |
|---|---|
| **RBAC** | Role-Based Access Control — padrão moderno |
| **ABAC** | Attribute-Based Access Control — legado, arquivo estático |
| **Node** | Autoriza o kubelet a gerenciar seus próprios Pods/Nodes |
| **Webhook** | Delega decisão para um serviço externo |

> Para a CKA: **RBAC** é o que importa. Os outros são contexto.

### RBAC — Conceitos Fundamentais

O RBAC define **quem** pode fazer **o quê** em **quais recursos**.

```
Sujeito (Subject) + Verbo (Verb) + Recurso (Resource) = Permissão
```

Componentes do RBAC:

| Objeto | Escopo | Descrição |
|---|---|---|
| `Role` | Namespace | Define permissões dentro de um namespace |
| `ClusterRole` | Cluster | Define permissões globais ou em non-namespaced resources |
| `RoleBinding` | Namespace | Associa um Role a um sujeito no namespace |
| `ClusterRoleBinding` | Cluster | Associa um ClusterRole a um sujeito globalmente |

> Detalhe completo de RBAC está no Cap. 15 (Security).

---

## 5. Admission Controllers

Após autenticação e autorização, a request passa pelos **Admission Controllers** antes de ser persistida no etcd.

### O que são

São plugins que interceptam requests ao API Server **após** autenticação/autorização e podem:
- **Validar** a request (bloquear ou aprovar)
- **Mutar** a request (modificar campos automaticamente)

### Tipos

| Tipo | Descrição |
|---|---|
| **Validating** | Só valida, bloqueia se necessário |
| **Mutating** | Modifica o objeto antes de persistir |

### Admission Controllers importantes

| Controller | O que faz |
|---|---|
| `NamespaceLifecycle` | Bloqueia criação em namespaces que estão sendo deletados |
| `LimitRanger` | Aplica LimitRange defaults a Pods sem limits definidos |
| `ResourceQuota` | Garante que quotas de namespace não sejam excedidas |
| `PodSecurity` | Valida Pod Security Standards (substituiu PodSecurityPolicy) |
| `ServiceAccount` | Injeta automaticamente o default ServiceAccount em Pods |
| `DefaultStorageClass` | Adiciona default StorageClass a PVCs sem storageClassName |
| `MutatingAdmissionWebhook` | Permite admission controllers externos via webhook |
| `ValidatingAdmissionWebhook` | Permite validadores externos via webhook |

---

## 6. Acesso à API via REST (sem kubectl)

Você pode acessar a API diretamente via HTTP:

```bash
# Via proxy local (sem precisar de certificados)
kubectl proxy --port=8080 &
curl http://localhost:8080/api/v1/pods

# Via curl com certificados
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
TOKEN=$(kubectl create token default)
curl -X GET "$APISERVER/api/v1/pods" \
  --header "Authorization: Bearer $TOKEN" \
  --insecure
```

---

## 7. Estrutura de um Objeto Kubernetes

Todo objeto K8s segue a mesma estrutura base:

```yaml
apiVersion: apps/v1          # Grupo/Versão da API
kind: Deployment             # Tipo do recurso
metadata:                    # Metadados de identificação
  name: meu-deployment
  namespace: default
  labels:
    app: web
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:                        # Especificação do estado DESEJADO
  replicas: 3
  # ... campos específicos do tipo
status:                      # Estado ATUAL (preenchido pelo K8s, não pelo usuário)
  readyReplicas: 3
  # ... campos de status
```

> **Importante**: Você escreve o `spec`. O K8s preenche o `status`. Nunca edite `status` manualmente.

---

## 8. Conceitos-Chave

| Conceito | Explicação |
|---|---|
| API Group | Organização lógica da API (core, apps, batch, networking...) |
| Autenticação | Verifica identidade (certificados, tokens, OIDC) |
| Autorização | Verifica permissões (RBAC é o padrão) |
| Admission Controller | Plugin que valida/muta requests antes de persistir |
| ServiceAccount | Identidade usada por Pods para acessar a API |
| kubeconfig | Arquivo com credenciais e endpoints do cluster |
| `kubectl explain` | Documentação inline dos campos de qualquer recurso |
| `--dry-run=client` | Simula criação — excelente para gerar YAML boilerplate |
