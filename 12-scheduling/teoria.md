# Capítulo 12 — Scheduling

---

## 1. O que é Scheduling?

Scheduling é o processo de decidir **em qual node um Pod vai rodar**. O **Scheduler** faz isso automaticamente, mas o K8s oferece muitas formas de influenciar essa decisão.

### Fases do Scheduling

```
Pod em estado Pending (sem node atribuído)
        │
        ▼
Fase 1: FILTRAGEM (Filtering)
  - Remove nodes que NÃO atendem requisitos obrigatórios
  - Critérios: recursos, Taints, NodeSelector, Affinity, etc.
        │
        ▼
Nodes que passaram na filtragem
        │
        ▼
Fase 2: PONTUAÇÃO (Scoring)
  - Atribui pontuação a cada node viável
  - Critérios: balanceamento de recursos, affinity preferences, etc.
        │
        ▼
Node com maior pontuação
        │
        ▼
Pod recebe nodeName → Kubelet do node cria o container
```

---

## 2. Resource Requests e Limits

### O que são

- **Request**: Quantidade de recurso **garantida** ao container. O Scheduler usa os requests para decidir onde caber o Pod.
- **Limit**: Quantidade **máxima** que o container pode usar.

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        cpu: "250m"       # 250 millicores = 0.25 CPU
        memory: "128Mi"   # 128 Mebibytes
      limits:
        cpu: "500m"       # Máximo 0.5 CPU
        memory: "256Mi"   # Máximo 256 MiB
```

### Unidades de CPU

| Valor | Equivalente |
|---|---|
| `1` | 1 CPU core (ou 1 vCPU, 1 hyperthread) |
| `500m` | 0.5 CPU |
| `100m` | 0.1 CPU |
| `1000m` | 1 CPU |

### Unidades de Memória

| Valor | Equivalente |
|---|---|
| `128Mi` | 128 Mebibytes (recomendado) |
| `256M` | 256 Megabytes |
| `1Gi` | 1 Gibibyte |

### Comportamento quando excede os Limits

| Recurso | O que acontece |
|---|---|
| CPU | Container é **throttled** (limitado), mas continua rodando |
| Memória | Container é **OOMKilled** (terminado) — memória não é compressível |

### QoS Classes

O K8s classifica Pods em classes de Quality of Service com base nos resources:

| QoS Class | Critério | Comportamento sob pressão |
|---|---|---|
| **Guaranteed** | Requests == Limits para todos os containers | Último a ser evicted |
| **Burstable** | Requests < Limits (ou só limit definido) | Evicted após BestEffort |
| **BestEffort** | Sem requests nem limits | Primeiro a ser evicted |

---

## 3. NodeSelector

A forma mais simples de restringir em quais nodes um Pod pode rodar. Usa labels do node.

```bash
# Adicionar label ao node
kubectl label node worker-1 tipo-disco=ssd
kubectl label node worker-2 tipo-disco=hdd
```

```yaml
spec:
  nodeSelector:
    tipo-disco: ssd      # Pod só vai para nodes com este label
  containers:
  - name: app
    image: myapp:1.0
```

> **Limitação**: NodeSelector só suporta igualdade exata. Para regras mais complexas, use Node Affinity.

---

## 4. Node Affinity

Node Affinity é a versão avançada do NodeSelector. Suporta operadores `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`.

### Tipos

| Tipo | Comportamento |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | **Obrigatório** para o scheduling. Se nenhum node atende, Pod fica Pending |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Preferência** (boa prática). Se possível usa, senão usa qualquer node |

> "IgnoredDuringExecution": se o node perder o label depois que o Pod já está rodando, o Pod NÃO é movido.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: [amd64]
          - key: tipo-disco
            operator: In
            values: [ssd, nvme]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80                           # Peso da preferência (1-100)
        preference:
          matchExpressions:
          - key: zona
            operator: In
            values: [zona-a]
      - weight: 20
        preference:
          matchExpressions:
          - key: zona
            operator: In
            values: [zona-b]
```

---

## 5. Pod Affinity e Anti-Affinity

Enquanto Node Affinity atrai Pods para **nodes com certos labels**, Pod Affinity/Anti-Affinity atrai ou repele Pods **baseado em outros Pods que já estão rodando**.

### Pod Affinity (atrair)

"Coloque este Pod no mesmo node (ou zona) onde já existe um Pod com label app=cache"

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache          # Outros Pods que devo ficar próximo
        topologyKey: kubernetes.io/hostname   # "Mesmo node"
        # topologyKey: topology.kubernetes.io/zone  # "Mesma zona cloud"
```

### Pod Anti-Affinity (repelir)

"NÃO coloque este Pod no mesmo node onde já existe outro Pod com app=minha-app" (garante distribuição)

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: minha-app      # Não ficar no mesmo node que outro pod igual
        topologyKey: kubernetes.io/hostname
```

> **Caso de uso clássico da Anti-Affinity**: Deployment com 3 réplicas. Garantir que nunca fiquem 2 réplicas no mesmo node (para tolerância a falha de node).

---

## 6. Taints e Tolerations

**Taints** são marcas em **nodes** que **repelem** Pods. **Tolerations** são propriedades em **Pods** que permitem ignorar um Taint específico.

### Taints nos Nodes

```bash
# Adicionar Taint
kubectl taint node worker-1 gpu=true:NoSchedule
#                           key=value:effect

# Remover Taint
kubectl taint node worker-1 gpu=true:NoSchedule-
```

### Efeitos dos Taints

| Efeito | Comportamento |
|---|---|
| `NoSchedule` | Pods sem toleration **não são agendados** neste node (Pods existentes ficam) |
| `PreferNoSchedule` | K8s tenta não agendar, mas pode se não houver alternativa |
| `NoExecute` | Pods sem toleration são **expulsos** (evicted) se já estiverem no node |

### Tolerations nos Pods

```yaml
spec:
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule        # Tolera o taint gpu=true:NoSchedule
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300    # Aguarda 300s antes de ser evicted por NotReady
```

### Taints do Control Plane

Por padrão, o kubeadm adiciona um taint no control plane para que Pods normais não sejam agendados lá:

```bash
kubectl describe node <control-plane-node> | grep Taints
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

Para permitir que um DaemonSet rode no control plane (ex: coletores de log), adicione a toleration:
```yaml
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule
```

---

## 7. Node Cordon e Drain

### Cordon — Marcar node para não receber novos Pods

```bash
kubectl cordon worker-2
# Node marcado: node/worker-2 cordoned
# Novo Taint adicionado: node.kubernetes.io/unschedulable:NoSchedule
```

O node fica com status `SchedulingDisabled`. Pods existentes continuam rodando, mas nenhum novo Pod é agendado nele.

```bash
kubectl uncordon worker-2    # Remove o cordon
```

### Drain — Remover todos os Pods do node (para manutenção)

```bash
kubectl drain worker-2 \
  --ignore-daemonsets \          # DaemonSet Pods não podem ser movidos (ignore)
  --delete-emptydir-data         # Deleta Pods com emptyDir (dados efêmeros)
```

O drain:
1. Adiciona cordon (para novos Pods)
2. Evicts todos os Pods (exceto Pods do DaemonSet e Static Pods)
3. Os Pods evicted são recriados pelo controller em outros nodes

---

## 8. Agendamento Manual (nodeName)

Você pode forçar um Pod para um node específico:

```yaml
spec:
  nodeName: worker-1        # Ignora o Scheduler — vai direto para este node
  containers:
  - name: app
    image: myapp:1.0
```

> Quando `nodeName` está definido, o Scheduler é completamente ignorado. O kubelet do node especificado criará o Pod diretamente.

---

## 9. Priority e Preemption

Pods podem ter prioridade. Quando o cluster está cheio, Pods de **alta prioridade** podem **preemptar** (despejar) Pods de **baixa prioridade**.

```yaml
# Criar PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: alta-prioridade
value: 1000000              # Maior = mais prioridade
globalDefault: false
description: "Prioridade alta para sistemas críticos"
---
# Usar no Pod
spec:
  priorityClassName: alta-prioridade
  containers:
  - name: app
    image: critical-app:1.0
```

---

## 10. Resumo dos Mecanismos de Scheduling

| Mecanismo | O que faz | Onde configura |
|---|---|---|
| `nodeSelector` | Restrição simples por label | Pod spec |
| `nodeAffinity` | Restrição avançada por label de node | Pod spec |
| `podAffinity` | Atrai Pod para nodes com outros Pods | Pod spec |
| `podAntiAffinity` | Repele Pod de nodes com outros Pods | Pod spec |
| `taints` | Marca node para repelir Pods | Node |
| `tolerations` | Permite Pod ignorar taints | Pod spec |
| `nodeName` | Força node específico (ignora scheduler) | Pod spec |
| `resources.requests` | Garante recursos; usado na filtragem | Container spec |
| `resources.limits` | Teto de uso de recursos | Container spec |
| `cordon` | Desabilita scheduling no node | kubectl comando |
| `drain` | Remove Pods do node + cordon | kubectl comando |
