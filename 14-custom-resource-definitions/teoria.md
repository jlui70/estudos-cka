# Capítulo 14 — Custom Resource Definitions (CRDs)

---

## 1. O que são CRDs?

O Kubernetes possui recursos nativos: Pod, Service, Deployment, etc. Mas e se você precisar de um tipo de recurso que não existe nativamente?

**CRDs (Custom Resource Definitions)** permitem que você **estenda a API do Kubernetes** com seus próprios tipos de recursos. Um CRD é como registrar um novo "tipo de objeto" no K8s.

```
API nativa:
  /api/v1/pods
  /apis/apps/v1/deployments

Depois de criar um CRD para "Database":
  /apis/mycompany.com/v1/databases   ← Novo endpoint registrado
```

---

## 2. Por que CRDs existem?

CRDs foram criados para permitir que:
- **Projetos open-source** estendam o K8s (cert-manager, Prometheus Operator, ArgoCD)
- **Equipes internas** criem abstrações específicas do negócio
- O **padrão Operator** seja implementado

### Exemplos de CRDs populares

| Projeto | CRD criado | Descrição |
|---|---|---|
| cert-manager | `Certificate`, `Issuer`, `ClusterIssuer` | Gerencia certificados TLS |
| Prometheus Operator | `PrometheusRule`, `ServiceMonitor` | Configura monitoramento |
| Argo CD | `Application`, `AppProject` | GitOps delivery |
| Istio | `VirtualService`, `DestinationRule` | Service Mesh |
| Crossplane | `Composition`, `XRD` | Infrastructure as Code |
| KEDA | `ScaledObject` | Autoscaling por métricas externas |

---

## 3. Anatomia de um CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.com     # formato: <plural>.<group>
spec:
  group: mycompany.com              # API group
  scope: Namespaced                 # Namespaced ou Cluster
  names:
    plural: databases               # Usado na URL: /apis/mycompany.com/v1/databases
    singular: database              # Nome singular
    kind: Database                  # CamelCase para o YAML
    shortNames:
    - db                            # kubectl get db
  versions:
  - name: v1
    served: true          # Esta versão está disponível
    storage: true         # Esta versão é usada para armazenamento no etcd
    schema:
      openAPIV3Schema:    # Validação dos campos (schema)
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
                enum: [postgres, mysql, mongodb]
              version:
                type: string
              storage:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 5
            required: [engine, version, storage]
```

---

## 4. Custom Resource (CR) — Usando o CRD

Após criar o CRD, você pode criar instâncias:

```yaml
apiVersion: mycompany.com/v1
kind: Database
metadata:
  name: meu-postgres
  namespace: producao
spec:
  engine: postgres
  version: "14.5"
  storage: "50Gi"
  replicas: 3
```

```bash
# Usar o CRD como qualquer recurso nativo
kubectl get databases
kubectl get db              # Usando shortName
kubectl describe database meu-postgres
kubectl delete database meu-postgres
```

---

## 5. O Padrão Operator

Somente o CRD não faz nada automaticamente. Ele apenas registra o tipo e permite armazenar objetos no etcd.

Para que algo **aconteça** quando um CR é criado/modificado/deletado, você precisa de um **Controller** que observa esses CRs e age.

A combinação de **CRD + Controller** é chamada de **Operator Pattern**.

```
Operador = CRD + Controller (lógica de negócio)

CRD:        Define o tipo "Database"
Controller: Quando um objeto "Database" é criado:
            → Cria um StatefulSet com o banco de dados
            → Cria um Service para expô-lo
            → Cria um PVC com o storage solicitado
            → Configura backups automáticos
            → Monitora a saúde e faz failover
```

### Níveis de maturidade de um Operator

| Nível | Capacidade |
|---|---|
| 1 — Basic Install | Instala e configura o app |
| 2 — Seamless Upgrades | Atualiza o app sem downtime |
| 3 — Full Lifecycle | Backup, restore, failover |
| 4 — Deep Insights | Métricas, alertas, análise de logs |
| 5 — Auto Pilot | Tuning automático, scaling horizontal e vertical |

### Exemplos de Operators famosos

- **Prometheus Operator**: Gerencia stack de monitoramento completa
- **PostgreSQL Operator (Zalando)**: Gerencia clusters PostgreSQL com HA, backup, failover
- **Strimzi**: Operator para Apache Kafka no K8s
- **Argo CD**: Operator de GitOps
- **Rook-Ceph**: Operator para storage distribuído

---

## 6. Controlador vs Operator — Distinção

| | Controller nativo | Operator |
|---|---|---|
| Quem cria | Core K8s team | Comunidade / equipes internas |
| Gerencia | Recursos K8s nativos (Pods, etc.) | Custom Resources + recursos K8s |
| Exemplos | Deployment Controller | PostgreSQL Operator |
| Conhecimento de domínio | Genérico | Específico da aplicação (DB, broker...) |

---

## 7. Subresources: Status e Scale

CRDs podem ter subresources que adicionam funcionalidades:

### Status Subresource

Separa o `spec` (escrito pelo usuário) do `status` (escrito pelo controller):

```yaml
versions:
- name: v1
  subresources:
    status: {}       # Habilita o subresource /status
```

Isso permite que o controller atualize o status sem sobrescrever o spec:
```bash
kubectl get database meu-postgres -o jsonpath='{.status}'
```

### Scale Subresource

Permite usar `kubectl scale` com o CRD:

```yaml
subresources:
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.readyReplicas
```

```bash
kubectl scale database meu-postgres --replicas=5
```

---

## 8. kubectl para CRDs

```bash
# Listar todos os CRDs instalados no cluster
kubectl get crd
kubectl get customresourcedefinitions

# Detalhes de um CRD específico
kubectl describe crd databases.mycompany.com

# Listar instâncias do CRD
kubectl get databases
kubectl get databases -A

# Documentação inline do CRD (se o schema foi definido)
kubectl explain database.spec
kubectl explain database.spec.engine

# Ver o YAML de uma instância
kubectl get database meu-postgres -o yaml
```

---

## 9. Relevância para a CKA

Para a CKA, CRDs são cobertos **conceitualmente**. Você não vai criar CRDs complexos na prova, mas precisa:

- Entender o que é um CRD e por que ele existe
- Saber que um Operator usa CRD + Controller
- Conseguir interagir com CRs já existentes no cluster:
  ```bash
  kubectl get <crd-resource>
  kubectl describe <crd-resource> <nome>
  kubectl delete <crd-resource> <nome>
  ```
- Entender que HPA, NetworkPolicy, StorageClass são recursos K8s estendidos (alguns via CRD, alguns nativos)

---

## 10. Resumo

| Conceito | Definição |
|---|---|
| CRD | Estende a API K8s com novos tipos de recursos |
| CR (Custom Resource) | Instância de um tipo definido por um CRD |
| Controller | Loop que reage a mudanças em recursos e toma ações |
| Operator Pattern | CRD + Controller com conhecimento de domínio específico |
| Operator Hub | Repositório de Operators prontos (operatorhub.io) |
| `served: true` | Versão disponível na API |
| `storage: true` | Versão usada para persistir no etcd |
| openAPIV3Schema | Validação de schema dos CRs (campos obrigatórios, tipos, etc.) |
