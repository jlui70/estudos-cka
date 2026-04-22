# Capítulo 2 — Basics do Kubernetes

---

## 1. O que é Kubernetes?

Kubernetes (K8s) é uma plataforma open-source de **orquestração de containers**. Ele automatiza o deployment, o scaling e o gerenciamento de aplicações containerizadas.

O nome vem do grego: "timoneiro" ou "piloto de navio". A abreviação **K8s** vem de "K" + 8 letras + "s".

---

## 2. Containers vs VMs — Comparação Teórica

Entender por que containers (e não VMs) são a base do K8s é fundamental.

```
VM                               Container
┌─────────────────────┐          ┌─────────────────────┐
│ App A               │          │ App A               │
├─────────────────────┤          ├─────────────────────┤
│ Guest OS (Linux)    │          │ Libs / Dependências  │
├─────────────────────┤          ├─────────────────────┤
│ Hypervisor          │          │ Container Runtime    │
│ (VMware, KVM, etc.) │          │ (containerd, CRI-O)  │
├─────────────────────┤          ├─────────────────────┤
│ Hardware / Host OS  │          │ Host OS (Linux Kernel│
└─────────────────────┘          └─────────────────────┘
```

| Critério | VM | Container |
|---|---|---|
| Isolamento | Total (kernel próprio) | Parcial (namespaces/cgroups) |
| Tamanho | GB (inclui SO) | MB (só app + libs) |
| Boot time | Minutos | Segundos/milissegundos |
| Portabilidade | Limitada (depende do hypervisor) | Alta (qualquer runtime compatível) |
| Overhead | Alto | Baixo |

> **Importante**: Containers compartilham o kernel do host. O isolamento é feito por **Linux namespaces** (pid, net, mnt, uts, ipc, user) e **cgroups** (limitação de CPU/memória).

---

## 3. Estrutura de um Cluster Kubernetes

Um cluster K8s tem dois tipos principais de nós (máquinas):

```
                    ┌─────────────────────────────────┐
                    │         CONTROL PLANE           │
                    │  (antes chamado de "Master")    │
                    │                                 │
                    │  ┌──────────┐  ┌─────────────┐  │
                    │  │ API      │  │ etcd        │  │
                    │  │ Server   │  │ (banco KV)  │  │
                    │  └──────────┘  └─────────────┘  │
                    │  ┌──────────┐  ┌─────────────┐  │
                    │  │Scheduler │  │ Controller  │  │
                    │  │          │  │ Manager     │  │
                    │  └──────────┘  └─────────────┘  │
                    └────────────┬────────────────────┘
                                 │ API calls
              ┌──────────────────┼──────────────────┐
              │                  │                  │
    ┌─────────┴──────┐  ┌────────┴───────┐  ┌──────┴──────────┐
    │   Worker Node  │  │   Worker Node  │  │   Worker Node   │
    │                │  │                │  │                 │
    │ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐  │
    │ │  Kubelet   │ │  │ │  Kubelet   │ │  │ │  Kubelet   │  │
    │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘  │
    │ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐  │
    │ │ Kube-proxy │ │  │ │ Kube-proxy │ │  │ │ Kube-proxy │  │
    │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘  │
    │ ┌──┐ ┌──┐ ┌──┐ │  │ ┌──┐ ┌──┐     │  │ ┌──┐ ┌──┐       │
    │ │P │ │P │ │P │ │  │ │P │ │P │     │  │ │P │ │P │       │
    │ └──┘ └──┘ └──┘ │  │ └──┘ └──┘     │  │ └──┘ └──┘       │
    │   (Pods)       │  │   (Pods)       │  │   (Pods)        │
    └────────────────┘  └────────────────┘  └─────────────────┘
```

### Control Plane (Master)

É o "cérebro" do cluster. Toma todas as decisões de orquestração. Em produção, recomenda-se ter 3 ou 5 nós de control plane para alta disponibilidade.

### Worker Nodes

São as máquinas onde as aplicações (Pods) efetivamente rodam. Podem ser físicas ou VMs. Quanto mais workers, mais capacidade de carga o cluster tem.

> **Nota de terminologia**: O termo "Master" foi depreciado em favor de "Control Plane" a partir do K8s 1.20, em resposta a iniciativas de linguagem inclusiva.

---

## 4. Componentes Básicos

### 4.1 Pod

O **Pod** é a menor unidade do Kubernetes — não é o container em si, mas o **wrapper** ao redor de um ou mais containers.

```
Pod
├── Container 1 (app principal)
├── Container 2 (sidecar: ex. log shipper)
└── Volumes compartilhados entre os containers
```

**Características do Pod:**
- Todos os containers de um Pod compartilham o mesmo **IP**, o mesmo **hostname** e os mesmos **volumes**.
- Containers dentro de um Pod comunicam-se via `localhost`.
- Pods são efêmeros — se morrem, não ressuscitam. Um controller (ex. Deployment) cria um novo Pod substituto.
- Cada Pod recebe um IP único dentro do cluster.

**Quando usar múltiplos containers num Pod (padrões):**
| Padrão | Descrição | Exemplo |
|---|---|---|
| Sidecar | Container auxiliar ao principal | Log shipper, proxy |
| Ambassador | Proxy que intermedia comunicação externa | Autenticação, TLS termination |
| Adapter | Transforma a saída do app principal | Formata métricas para Prometheus |

**Pod básico em YAML:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: meu-pod
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

### 4.2 Node

Um **Node** é uma máquina (física ou virtual) que faz parte do cluster. Pode ser um Worker Node ou o próprio Control Plane.

**Informações de um node:**
```bash
kubectl get nodes
kubectl describe node <nome-do-node>
```

**Status de um node:**
- `Ready` — Node saudável, aceitando Pods
- `NotReady` — Node com problemas (kubelet parou, sem recursos, etc.)
- `SchedulingDisabled` — Node em manutenção (cordon ativo)

**Componentes que rodam em cada Worker Node:**
- **kubelet**: Agente que garante que os containers estejam rodando conforme os PodSpecs
- **kube-proxy**: Mantém as regras de rede (iptables/ipvs) para os Services
- **Container Runtime**: Executa os containers (containerd, CRI-O)

### 4.3 Namespace

Namespaces são **partições lógicas** do cluster. Permitem isolar recursos entre times, projetos ou ambientes (dev/staging/prod).

**Namespaces padrão:**
| Namespace | Uso |
|---|---|
| `default` | Onde recursos vão quando namespace não é especificado |
| `kube-system` | Componentes do próprio K8s (CoreDNS, kube-proxy, etc.) |
| `kube-public` | Legível por qualquer usuário (incluindo não autenticados) |
| `kube-node-lease` | Objetos de "heartbeat" dos nodes |

```bash
kubectl get pods --namespace kube-system
kubectl get all -n default
```

### 4.4 Cluster

O **Cluster** é o conjunto de todos os nodes (control plane + workers) mais a configuração e o estado armazenado no etcd. É a unidade máxima de gerenciamento do K8s.

---

## 5. O Modelo Declarativo vs Imperativo

### Imperativo
Você diz **o que fazer**: "Crie um pod com nginx agora."
```bash
kubectl run nginx --image=nginx
```

### Declarativo
Você diz **qual deve ser o estado**: "O cluster deve ter um pod com nginx rodando."
```bash
kubectl apply -f pod.yaml
```

O Kubernetes trabalha fundamentalmente de forma **declarativa**. Você descreve o estado desejado e o K8s (através de seus controllers) trabalha continuamente para garantir que o estado real corresponda ao estado desejado. Isso é chamado de **reconciliation loop** ou **control loop**.

```
Estado Desejado (YAML) ──→ API Server ──→ etcd (armazena)
                                              │
                                              ↓
                                    Controller Manager
                                    (verifica: real == desejado?)
                                              │
                                    ┌─────────┴──────────┐
                                 Sim│                     │Não
                                    ↓                     ↓
                               Não faz nada       Toma ação corretiva
```

---

## 6. Kubernetes vs Docker Swarm vs Mesos — Comparação

| Feature | Kubernetes | Docker Swarm | Apache Mesos |
|---|---|---|---|
| Complexidade | Alta | Baixa | Muito alta |
| Escalabilidade | Muito alta | Média | Muito alta |
| Auto-healing | Sim | Sim | Sim |
| Rolling updates | Sim | Sim | Sim |
| RBAC nativo | Sim | Limitado | Sim |
| Suporte cloud | GKE, EKS, AKS, etc. | Limitado | Limitado |
| Comunidade | Enorme | Pequena (declining) | Menor |
| Curva de aprendizado | Íngreme | Suave | Muito íngreme |
| Padrão de mercado | **Sim** | Não | Não |

> Docker Swarm foi descontinuado como foco principal do Docker Inc. em 2019. Mesos é usado em nichos específicos. **Kubernetes é o padrão de facto**.

---

## 7. Conceitos-Chave para Memorizar

| Conceito | Definição em uma linha |
|---|---|
| Pod | Menor unidade deployável; encapsula 1+ containers com IP compartilhado |
| Node | Máquina (física ou VM) no cluster |
| Cluster | Conjunto de todos os nodes gerenciados pelo K8s |
| Namespace | Partição lógica do cluster |
| Control Plane | Conjunto de componentes que gerenciam o cluster |
| Worker Node | Node onde as aplicações (Pods) rodam |
| kubelet | Agente do K8s em cada node |
| etcd | Banco de dados chave-valor que armazena todo o estado do cluster |
| Declarativo | Modelo onde você descreve o "o quê", não o "como" |
| Controller | Loop que reconcilia estado desejado com estado real |

---

## 8. Resumo do Fluxo Básico

```
Developer               Kubernetes
    │                       │
    │── kubectl apply ──────→│ API Server valida e armazena no etcd
    │                       │
    │                       │ Scheduler escolhe o node
    │                       │
    │                       │ Kubelet no node cria o container
    │                       │
    │                       │ App rodando!
    │←── kubectl get pods ──│ Dev verifica o status
```
