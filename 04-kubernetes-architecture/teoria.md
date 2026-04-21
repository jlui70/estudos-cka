# Capítulo 4 — Arquitetura do Kubernetes

---

## 1. Visão Geral da Arquitetura

O Kubernetes é composto por dois planos:

- **Control Plane**: Gerencia o cluster inteiro — toma decisões, armazena estado, agenda workloads.
- **Data Plane (Worker Nodes)**: Onde as aplicações (Pods) efetivamente rodam.

```
┌─────────────────────────────────────────────────────────────┐
│                      CONTROL PLANE                          │
│                                                             │
│  ┌─────────────┐  ┌──────┐  ┌────────────┐  ┌───────────┐  │
│  │  API Server │  │ etcd │  │ Scheduler  │  │Controller │  │
│  │  (kube-api) │  │      │  │            │  │ Manager   │  │
│  └──────┬──────┘  └──────┘  └────────────┘  └───────────┘  │
│         │ (todos se comunicam via API Server)               │
└─────────┼───────────────────────────────────────────────────┘
          │
      ────┼─────────────────────────────────────────────
          │        (rede do cluster)
  ┌───────┴────────────────────────────────┐
  │             WORKER NODES               │
  │                                        │
  │  ┌──────────┐  ┌────────────┐          │
  │  │ kubelet  │  │ kube-proxy │  CRI     │
  │  └──────────┘  └────────────┘  (containerd/CRI-O)│
  └────────────────────────────────────────┘
```

---

## 2. API Server (kube-apiserver)

### O que é

O **API Server** é o único componente que se comunica diretamente com o etcd. É o "ponto central" de toda comunicação no cluster. Tudo passa por ele:

- `kubectl` → API Server
- kubelet → API Server
- Controller Manager → API Server
- Scheduler → API Server

### Responsabilidades

1. **Validação** de requests (autenticação, autorização, validação de schema)
2. **Processamento** de recursos (CRUD no etcd)
3. **Notificação** via watch mechanisms (outros componentes observam mudanças)
4. **Proxy** para serviços internos (ex: acesso a logs de pods via `kubectl logs`)

### Fluxo de uma request ao API Server

```
kubectl apply -f pod.yaml
        │
        ▼
1. Autenticação    (quem é você? certificado, token, OIDC)
        │
        ▼
2. Autorização     (você pode fazer isso? RBAC)
        │
        ▼
3. Admission Control  (validações & mutações adicionais)
        │
        ▼
4. Persistir no etcd
        │
        ▼
5. Retorna resposta ao cliente
```

### Alta Disponibilidade

Em clusters HA, existem múltiplas instâncias do API Server (tipicamente 3) atrás de um load balancer. Todas podem processar requests simultaneamente (stateless — o estado fica no etcd).

---

## 3. etcd

### O que é

O **etcd** é um banco de dados chave-valor distribuído, consistente e altamente disponível. É o único lugar onde o estado do cluster é armazenado.

### Características

| Característica | Detalhe |
|---|---|
| Algoritmo de consenso | **Raft** (garante consistência em cluster etcd distribuído) |
| Armazenamento | Binário msgpack, otimizado para leituras |
| Acesso | Somente o API Server acessa diretamente |
| Porta padrão | 2379 (client), 2380 (peer/replicação) |

### O que fica no etcd?

Absolutamente **todo o estado do cluster**:
- Todos os objetos K8s (Pods, Deployments, Services, ConfigMaps, Secrets, etc.)
- Status dos nodes
- Configurações de RBAC
- Certificados (boostrap tokens, etc.)

### Por que o etcd é crítico?

Se o etcd for corrompido ou perdido, o cluster perde todo o seu estado. Por isso:
- **Backup regular do etcd é obrigatório** (exigido na CKA)
- Em HA, o etcd deve ter 3 ou 5 membros (número ímpar para quorum)
- Quorum = `(n/2) + 1` membros precisam estar available

| Membros etcd | Tolerância a falhas |
|---|---|
| 1 | 0 falhas |
| 3 | 1 falha |
| 5 | 2 falhas |
| 7 | 3 falhas |

### Comandos etcd (para CKA)

```bash
# Backup do etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verificar status do backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db

# Restaurar (ver cap16 para detalhes completos)
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore
```

---

## 4. Scheduler (kube-scheduler)

### O que é

O **Scheduler** é responsável por decidir **em qual node** cada Pod novo será executado. Ele não executa o Pod — apenas toma a decisão de *placement*.

### Como funciona

```
1. Pod criado sem node atribuído (nodeName vazio)
        │
        ▼
2. Scheduler detecta o Pod via watch no API Server
        │
        ▼
3. Fase de Filtragem:
   - Remove nodes que NÃO atendem os requisitos do Pod
   - Critérios: recursos, taints/tolerations, nodeSelector, affinity, etc.
        │
        ▼
4. Fase de Pontuação (Scoring):
   - Pontua os nodes que passaram na filtragem
   - Critérios: balanceamento de recursos, affinity preferences, spread, etc.
        │
        ▼
5. Node com maior pontuação é escolhido
        │
        ▼
6. Scheduler atualiza o Pod com nodeName no API Server
        │
        ▼
7. Kubelet no node escolhido detecta o Pod e o inicia
```

### Scheduler plugins

O Scheduler usa um framework de plugins. Cada fase (filter, score, etc.) pode ter plugins customizados. Isso permite criar schedulers customizados ou usar múltiplos schedulers no cluster.

---

## 5. Controller Manager (kube-controller-manager)

### O que é

O **Controller Manager** é um processo que roda múltiplos controllers em um único binário. Cada controller implementa um **control loop**:

```
loop {
    estado_atual = observar_estado_real()
    estado_desejado = ler_do_api_server()
    if estado_atual != estado_desejado {
        tomar_acao_corretiva()
    }
    aguardar()
}
```

### Controllers principais

| Controller | Responsabilidade |
|---|---|
| **Node Controller** | Detecta quando nodes ficam offline; marca como NotReady |
| **Replication Controller** | Mantém o número correto de réplicas de Pods |
| **Endpoints Controller** | Atualiza o objeto Endpoints com IPs dos Pods |
| **Service Account Controller** | Cria ServiceAccounts default em novos Namespaces |
| **Namespace Controller** | Limpa recursos quando um Namespace é deletado |
| **Job Controller** | Monitora Jobs e cria Pods conforme necessário |
| **Deployment Controller** | Gerencia ReplicaSets para Deployments |
| **StatefulSet Controller** | Gerencia Pods com identidade estável |
| **DaemonSet Controller** | Garante que um Pod rode em cada node |

### Cloud Controller Manager (cloud-controller-manager)

Em clouds (AWS, GCP, Azure), existe um componente separado responsável por integrar o K8s com a cloud:

| Cloud Controller | Responsabilidade |
|---|---|
| Node controller | Verifica se node ainda existe na cloud; deleta se não |
| Route controller | Configura rotas na cloud para CIDRs |
| Service controller | Cria/atualiza Load Balancers na cloud |

---

## 6. kubelet

### O que é

O **kubelet** é o agente que roda em **cada node** do cluster (incluindo o control plane). É o responsável por garantir que os containers descritos nos PodSpecs estejam rodando.

### Responsabilidades

1. Registra o node no API Server
2. Observa PodSpecs atribuídos ao node (via API Server)
3. Garante que os containers dos Pods estejam rodando e saudáveis
4. Executa **liveness probes**, **readiness probes** e **startup probes**
5. Reporta status do node e dos Pods ao API Server
6. Gerencia volumes (mount/unmount)

### Como o kubelet recebe Pods

- **Via API Server** (modo normal): Pods criados pelo usuário e agendados pelo Scheduler
- **Via Static Pods**: Arquivos YAML em `/etc/kubernetes/manifests/` (modo direto)

### Probes — Verificações de Saúde

O kubelet executa probes nos containers:

| Probe | O que verifica | Ação se falhar |
|---|---|---|
| **livenessProbe** | Container está vivo? | Reinicia o container |
| **readinessProbe** | Container pronto para receber tráfego? | Remove dos Endpoints (sem tráfego) |
| **startupProbe** | App terminou de inicializar? | Evita que liveness mate o container durante boot longo |

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 3
```

---

## 7. kube-proxy

### O que é

O **kube-proxy** roda em cada node e é responsável por implementar as regras de rede necessárias para os **Services** do Kubernetes.

### Modos de operação

| Modo | Implementação | Observação |
|---|---|---|
| **iptables** | Regras iptables no kernel | Modo padrão, performático |
| **ipvs** | IPVS (IP Virtual Server) | Mais eficiente para muitos Services |
| **userspace** | Proxy no userspace | Legado, não recomendado |

### O que o kube-proxy faz na prática

Quando você cria um Service `ClusterIP: 10.96.0.1`, o kube-proxy cria regras iptables em todos os nodes para que qualquer tráfego para `10.96.0.1` seja redirecionado para um dos Pods do Service.

```
Pod A → 10.96.0.1:80 (ClusterIP do Service)
            │
            ▼ (kube-proxy / iptables)
        Escolhe um dos Pods:
        ├── Pod B (10.244.1.5:8080) — 33%
        ├── Pod C (10.244.2.3:8080) — 33%
        └── Pod D (10.244.3.7:8080) — 33%
```

---

## 8. Container Runtime

### O que é

O Container Runtime é o software que efetivamente cria e gerencia os containers. O K8s se comunica com ele via **CRI (Container Runtime Interface)**.

### CRI — Container Runtime Interface

CRI é uma API gRPC definida pelo K8s. Qualquer runtime que implemente a CRI pode ser usado:

```
kubelet ──(gRPC/CRI)──→ containerd ──→ runc (container)
kubelet ──(gRPC/CRI)──→ CRI-O      ──→ runc (container)
```

### Por que o Docker foi removido?

O Docker não implementava CRI natively. O K8s usava um adaptador chamado **dockershim** para conversar com o Docker. Esse adaptador foi removido no K8s 1.24 porque:
- Adicionava latência e complexidade
- Docker internamente já usa **containerd** — o K8s agora fala direto com o containerd

---

## 9. Fluxo Completo: do `kubectl apply` ao Container Rodando

```
kubectl apply -f deployment.yaml
        │
        ▼
1. API Server: autentica, autoriza, valida, persiste no etcd
        │
        ▼
2. Deployment Controller (no Controller Manager):
   - Detecta novo Deployment
   - Cria um ReplicaSet
        │
        ▼
3. ReplicaSet Controller:
   - Detecta que precisa de N Pods
   - Cria N Pods (status: Pending, sem nodeName)
        │
        ▼
4. Scheduler:
   - Detecta Pods em Pending sem nodeName
   - Faz filtragem e pontuação dos nodes
   - Atualiza cada Pod com o nodeName escolhido
        │
        ▼
5. Kubelet (no node escolhido):
   - Detecta Pod atribuído a ele
   - Chama o Container Runtime (containerd) via CRI
   - Baixa a imagem se necessário
   - Cria o container
   - Executa probes
   - Reporta status Running ao API Server
        │
        ▼
6. kube-proxy:
   - Atualiza regras iptables para incluir o novo Pod nos Services relevantes
```

---

## 10. Resumo dos Componentes

| Componente | Localização | Função Principal |
|---|---|---|
| **kube-apiserver** | Control Plane | Hub central; único acesso ao etcd |
| **etcd** | Control Plane | Banco de dados de estado do cluster |
| **kube-scheduler** | Control Plane | Decide onde cada Pod roda |
| **kube-controller-manager** | Control Plane | Garante que o estado real = desejado |
| **cloud-controller-manager** | Control Plane | Integração com APIs da cloud |
| **kubelet** | Todos os nodes | Agente; garante que containers rodem |
| **kube-proxy** | Todos os nodes | Regras de rede para os Services |
| **Container Runtime** | Todos os nodes | Executa containers (containerd, CRI-O) |
