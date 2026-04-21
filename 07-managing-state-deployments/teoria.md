# Capítulo 7 — Deployments e Ciclo de Vida de Aplicações

---

## 1. Por que não usar Pods diretamente?

Pods são efêmeros — se morrerem, não ressuscitam sozinhos. Para aplicações reais, você precisa de controllers que garantam o estado desejado:

| Necesidade | Controller |
|---|---|
| Manter N réplicas de um Pod rodando | ReplicaSet / Deployment |
| Um Pod por node | DaemonSet |
| Pod com identidade estável | StatefulSet |
| Tarefa que roda até completar | Job |
| Tarefa agendada (cron) | CronJob |

---

## 2. ReplicaSet

O **ReplicaSet** garante que um número especificado de réplicas de um Pod esteja rodando a qualquer momento.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend-rs
spec:
  replicas: 3          # Número desejado de Pods
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend  # DEVE bater com o selector acima
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

### Como o ReplicaSet funciona

```
Controller Loop (ReplicaSet Controller)
│
├── Estado desejado: 3 réplicas (spec.replicas)
├── Estado real: conta Pods com label app=frontend
│
├── Se real < desejado → cria Pods
├── Se real > desejado → deleta Pods
└── Se real == desejado → não faz nada
```

> **Na prática**: Você raramente cria ReplicaSets diretamente. O Deployment gerencia ReplicaSets para você.

---

## 3. Deployment

O **Deployment** é o objeto mais usado para aplicações stateless. Ele gerencia ReplicaSets e adiciona funcionalidades de:
- Rolling updates
- Rollbacks
- Controle de ritmo de atualização

### Estrutura completa de um Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
  namespace: default
  labels:
    app: minha-app
spec:
  replicas: 3                    # Número de Pods
  selector:
    matchLabels:
      app: minha-app
  strategy:                      # Estratégia de atualização
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                # Máx de pods a mais durante update
      maxUnavailable: 0          # Máx de pods indisponíveis durante update
  template:                      # Template do Pod
    metadata:
      labels:
        app: minha-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

### Hierarquia de objetos

```
Deployment
  └── ReplicaSet (nova versão)
        └── Pod (x3)
  └── ReplicaSet (versão anterior — ainda existe para rollback)
```

---

## 4. Scaling

### Manual

```bash
kubectl scale deployment minha-app --replicas=5
```

### Via HPA (Horizontal Pod Autoscaler)

O HPA escala automaticamente baseado em métricas:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: minha-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # Escala quando CPU média > 70%
```

> Para HPA funcionar, o Metrics Server precisa estar instalado no cluster.

---

## 5. Rolling Update

O Rolling Update atualiza os Pods gradualmente, garantindo que parte da aplicação sempre esteja disponível.

### Fluxo

```
Estado inicial: 3 Pods com image:v1

        [v1] [v1] [v1]

Atualização para v2 (maxSurge=1, maxUnavailable=0):

Passo 1: Cria 1 Pod v2 (total: 4 Pods)
        [v1] [v1] [v1] [v2]

Passo 2: Remove 1 Pod v1 (total: 3 Pods)
        [v1] [v1] [v2]

Passo 3: Cria 1 Pod v2
        [v1] [v1] [v2] [v2]

Passo 4: Remove 1 Pod v1
        [v1] [v2] [v2]

... e assim por diante até:
        [v2] [v2] [v2]
```

### Configuração da estratégia

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Pods a mais permitidos durante update (ex: 3+1=4)
    maxUnavailable: 0    # Pods indisponíveis durante update (0 = sem downtime)
```

- `maxSurge`: pode ser número absoluto ou porcentagem (ex: 25%)
- `maxUnavailable`: pode ser número absoluto ou porcentagem

### Recreate Strategy

Alternativa ao Rolling Update — deleta todos os Pods antigos antes de criar os novos:

```yaml
strategy:
  type: Recreate    # Downtime proposital, mas garante que nunca coexistem versões
```

> Use `Recreate` quando a aplicação não suporta múltiplas versões rodando ao mesmo tempo (ex: migrations de banco de dados que quebram compatibilidade).

### Comandos de atualização

```bash
# Atualizar imagem
kubectl set image deployment/minha-app app=myapp:2.0

# Verificar status do rollout
kubectl rollout status deployment/minha-app

# Verificar histórico
kubectl rollout history deployment/minha-app

# Pausar rollout
kubectl rollout pause deployment/minha-app

# Retomar rollout
kubectl rollout resume deployment/minha-app
```

---

## 6. Rollback

Se uma atualização falhou, você pode reverter para a versão anterior:

```bash
# Rollback para a versão anterior
kubectl rollout undo deployment/minha-app

# Rollback para uma revisão específica
kubectl rollout undo deployment/minha-app --to-revision=2

# Ver histórico de revisões
kubectl rollout history deployment/minha-app

# Ver detalhes de uma revisão
kubectl rollout history deployment/minha-app --revision=2
```

### Como o rollback funciona

O Deployment mantém os ReplicaSets antigos (com 0 réplicas). O rollback simplesmente aumenta as réplicas do ReplicaSet antigo e zera o novo:

```
Antes: RS-v2 (3 pods), RS-v1 (0 pods)
Depois do rollback: RS-v2 (0 pods), RS-v1 (3 pods)
```

O campo `spec.revisionHistoryLimit` controla quantos ReplicaSets antigos são mantidos (padrão: 10).

---

## 7. DaemonSet

O **DaemonSet** garante que **um Pod rode em cada node** (ou em nodes selecionados).

### Casos de uso típicos

- Coleta de logs (Fluentd, Filebeat)
- Monitoramento (Node Exporter do Prometheus)
- Agentes de segurança
- CNI plugins de rede

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane  # Rodar também no control plane
        effect: NoSchedule
        operator: Exists
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

---

## 8. StatefulSet

O **StatefulSet** é para aplicações **stateful** que precisam de:
- Identidade de rede estável (nome do Pod não muda)
- Armazenamento persistente por Pod
- Ordem de criação/deleção garantida

### Diferenças vs Deployment

| | Deployment | StatefulSet |
|---|---|---|
| Nome dos Pods | Aleatório (deploy-abc123) | Sequencial (web-0, web-1, web-2) |
| Ordem de criação | Paralela | Sequencial (web-0 antes de web-1) |
| Storage | Compartilhado ou efêmero | PVC próprio por Pod |
| Identidade de rede | IP muda ao recriar | Hostname estável via Headless Service |
| Uso | Apps stateless | Bancos de dados, message queues |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"     # Headless Service obrigatório
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # PVC criado para cada Pod
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

Resultado: Pods `postgres-0`, `postgres-1`, `postgres-2`, cada um com seu próprio PVC `data-postgres-0`, `data-postgres-1`, `data-postgres-2`.

---

## 9. Job

O **Job** executa um Pod até que ele complete com sucesso.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1      # Número de runs bem-sucedidos desejados
  parallelism: 1      # Pods rodando em paralelo
  backoffLimit: 4     # Tentativas antes de marcar como Failed
  template:
    spec:
      restartPolicy: Never     # Never ou OnFailure (nunca Always)
      containers:
      - name: migration
        image: myapp:1.0
        command: ["python", "manage.py", "migrate"]
```

---

## 10. CronJob

O **CronJob** cria Jobs em uma agenda (sintaxe cron):

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-db
spec:
  schedule: "0 2 * * *"           # Todo dia às 2h (formato cron)
  concurrencyPolicy: Forbid        # Não roda se o anterior ainda estiver rodando
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:1.0
            command: ["/backup.sh"]
```

**Sintaxe cron:**
```
┌──────── minuto (0-59)
│ ┌────── hora (0-23)
│ │ ┌──── dia do mês (1-31)
│ │ │ ┌── mês (1-12)
│ │ │ │ ┌ dia da semana (0-7, 0 e 7 = domingo)
│ │ │ │ │
* * * * *
```

---

## 11. Resumo Comparativo dos Controllers

| Controller | Garante | Identidade | Storage | Casos de Uso |
|---|---|---|---|---|
| **Deployment** | N réplicas | Efêmera | Sem persistência própria | Apps web stateless, APIs |
| **ReplicaSet** | N réplicas | Efêmera | Sem persistência própria | Gerenciado pelo Deployment |
| **DaemonSet** | 1 Por node | Efêmera | Geralmente hostPath | Agents, coletores de logs |
| **StatefulSet** | N réplicas estáveis | Estável (nome fixo) | PVC por Pod | DB, Kafka, ElasticSearch |
| **Job** | 1 Execução completa | Efêmera | Sem persistência própria | Migrations, processos batch |
| **CronJob** | Job agendado | Efêmera | Sem persistência própria | Backups, tarefas periódicas |
