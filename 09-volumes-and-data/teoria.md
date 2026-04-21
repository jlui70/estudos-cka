# Capítulo 9 — Volumes, Persistência e Configuração

---

## 1. O Problema da Persistência em Containers

Containers são efêmeros — quando um container reinicia, todo o filesystem interno é perdido. Para aplicações que precisam persistir dados, o Kubernetes oferece o conceito de **Volumes**.

```
Container sem volume:
  Escreve /data/arquivo.db  →  Container reinicia  →  arquivo.db PERDIDO

Container com volume:
  Escreve /data/arquivo.db  →  Container reinicia  →  arquivo.db PRESERVADO
                                                       (no volume externo)
```

---

## 2. Tipos de Volume (Efêmeros)

Volumes efêmeros têm o mesmo ciclo de vida do Pod — quando o Pod morre, o volume também some.

### 2.1 emptyDir

Um diretório vazio criado quando o Pod é agendado. Útil para compartilhar dados entre containers do mesmo Pod.

```yaml
spec:
  volumes:
  - name: cache
    emptyDir: {}           # Padrão: armazenado em disco
  - name: ramdisk
    emptyDir:
      medium: Memory       # Armazenado em RAM (mais rápido, usa memória do node)
      sizeLimit: 256Mi
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: cache
      mountPath: /tmp/cache
  - name: sidecar
    image: sidecar:1.0
    volumeMounts:
    - name: cache
      mountPath: /shared    # Acessa o mesmo volume
```

### 2.2 hostPath

Monta um diretório do **Node** dentro do container. Perigoso em produção (acesso ao filesystem do host).

```yaml
volumes:
- name: host-log
  hostPath:
    path: /var/log          # Caminho no Node
    type: Directory         # DirectoryOrCreate, File, FileOrCreate, Socket...
```

> **Risco de segurança**: Um Pod com `hostPath: /` pode comprometer o node inteiro. Evite em produção ou restrinja via Network Policies e PSS.

### 2.3 configMap (como volume)

Monta um ConfigMap como arquivos no container (ver seção ConfigMap).

### 2.4 secret (como volume)

Monta um Secret como arquivos no container (ver seção Secret).

---

## 3. Persistent Volumes (PV) e Persistent Volume Claims (PVC)

Para armazenamento que sobrevive além do ciclo de vida de um Pod, o K8s usa o modelo PV/PVC.

### Conceitos

```
PV (PersistentVolume):
  - Recurso de armazenamento provisionado no cluster
  - Abstraí o storage real (NFS, disco local, EBS, Azure Disk...)
  - Criado pelo admin ou dinamicamente pelo StorageClass
  - Cluster-scoped (não tem namespace)

PVC (PersistentVolumeClaim):
  - Requisição de armazenamento feita pelo usuário/app
  - "Preciso de 10Gi, modo ReadWriteOnce"
  - Namespaced
  - Se amarra a um PV que satisfaça os requisitos
```

### Ciclo de vida PV/PVC

```
1. Admin cria PV (ou StorageClass provisiona automaticamente)
2. Dev cria PVC com requisitos
3. K8s faz o "binding" PVC → PV (critérios: tamanho, accessMode, storageClassName)
4. Pod usa o PVC como volume
5. Pod deletado → PVC continua (dados preservados)
6. PVC deletado → PV pode ser:
   - Retain: PV fica com dados, admin decide o que fazer
   - Delete: PV e o storage subjacente são deletados
   - Recycle: Dados apagados, PV fica disponível (deprecated)
```

### 3.1 PersistentVolume (YAML estático)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-dados-1
spec:
  capacity:
    storage: 10Gi              # Tamanho do volume
  accessModes:
  - ReadWriteOnce              # Modo de acesso
  persistentVolumeReclaimPolicy: Retain  # O que fazer quando PVC é deletado
  storageClassName: manual     # Classe de storage (para binding manual)
  hostPath:                    # Tipo de backend (aqui: hostPath para lab)
    path: /data/pv-dados-1
```

### 3.2 Access Modes

| Mode | Abreviação | Descrição |
|---|---|---|
| `ReadWriteOnce` | RWO | Leitura/escrita por UM node por vez |
| `ReadOnlyMany` | ROX | Somente leitura por MÚLTIPLOS nodes |
| `ReadWriteMany` | RWX | Leitura/escrita por MÚLTIPLOS nodes |
| `ReadWriteOncePod` | RWOP | Leitura/escrita por UM Pod (K8s 1.22+) |

> Não todo storage type suporta todos os modes. NFS suporta RWX. EBS/Azure Disk só RWO.

### 3.3 PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: meu-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi             # Pede no mínimo 5Gi
  storageClassName: manual     # Deve bater com o PV
```

### 3.4 Usando PVC em um Pod

```yaml
spec:
  volumes:
  - name: meus-dados
    persistentVolumeClaim:
      claimName: meu-pvc       # Nome do PVC criado acima
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: meus-dados
      mountPath: /app/data
```

---

## 4. StorageClass e Provisionamento Dinâmico

### O que é

O **StorageClass** define um "tipo" de storage. Com ele, quando um PVC é criado, o K8s **provisiona automaticamente** um PV correspondente — sem intervenção do admin.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # StorageClass default
provisioner: kubernetes.io/aws-ebs    # Quem cria o volume
parameters:
  type: gp3                            # Parâmetros do provisioner
  iopsPerGB: "10"
reclaimPolicy: Delete                  # Retain ou Delete
allowVolumeExpansion: true             # Permite expandir PVCs
```

### Fluxo com provisionamento dinâmico

```
1. Dev cria PVC com storageClassName: fast-ssd
2. K8s vê que 'fast-ssd' tem um provisioner
3. Provisioner cria o volume real na cloud (ex: EBS volume)
4. Provisioner cria o PV correspondente
5. K8s faz o binding PVC → PV automático
6. Pod pode usar o PVC
```

### StorageClass default

Se um PVC não especifica `storageClassName`, o K8s usa a StorageClass marcada como default:
```bash
kubectl get storageclass
# A default tem (default) no nome
```

---

## 5. ConfigMap

### O que é

O **ConfigMap** armazena dados de configuração não-confidenciais como pares chave-valor. Permite separar a configuração da imagem do container.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Dados chave-valor simples
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  DATABASE_HOST: "postgres.default.svc.cluster.local"

  # Arquivo de configuração completo
  app.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://db:5432/mydb
    logging.level.root=INFO
```

### Formas de usar um ConfigMap

**1. Como variáveis de ambiente:**
```yaml
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: LOG_LEVEL
```

**2. Todas as chaves como variáveis de ambiente:**
```yaml
envFrom:
- configMapRef:
    name: app-config
```

**3. Como volume (arquivo):**
```yaml
volumes:
- name: config-vol
  configMap:
    name: app-config
    items:
    - key: app.properties
      path: application.properties  # Nome do arquivo no container

volumeMounts:
- name: config-vol
  mountPath: /etc/app/config
```

> Quando montado como volume, cada chave vira um arquivo. A chave é o nome do arquivo, o valor é o conteúdo.

---

## 6. Secret

### O que é

O **Secret** é similar ao ConfigMap, mas para dados **sensíveis** (senhas, tokens, certificados). Os dados são armazenados em **base64** no etcd.

> **Importante**: Base64 não é criptografia — é apenas encoding. Secrets não são seguros por padrão. Para segurança real, use Encryption at Rest no etcd + ferramentas como Vault.

### Tipos de Secret

| Tipo | Uso |
|---|---|
| `Opaque` | Dado arbitrário (padrão) |
| `kubernetes.io/dockerconfigjson` | Credenciais de registry |
| `kubernetes.io/tls` | Certificados TLS (crt + key) |
| `kubernetes.io/service-account-token` | Token de ServiceAccount |
| `kubernetes.io/ssh-auth` | Chave SSH |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=          # "admin" em base64
  password: czNjcjN0         # "s3cr3t" em base64
```

```bash
# Criar secret imperativo
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# Criar secret TLS
kubectl create secret tls meu-tls \
  --cert=server.crt \
  --key=server.key
```

### Usando Secrets em Pods

```yaml
# Como variável de ambiente
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password

# Como volume (arquivo)
volumes:
- name: secret-vol
  secret:
    secretName: db-credentials

volumeMounts:
- name: secret-vol
  mountPath: /etc/secrets
  readOnly: true      # Boa prática: montar secrets como read-only
```

---

## 7. Resumo Comparativo

| Recurso | Escopo | Dados | Ciclo de Vida | Uso típico |
|---|---|---|---|---|
| `emptyDir` | Pod | Efêmero | Com o Pod | Cache, dados temporários entre containers |
| `hostPath` | Node | Persiste no node | Com o Node | Logs do node, labs (evitar em prod) |
| `PV` | Cluster | Persistente | Independente | Storage backend real |
| `PVC` | Namespace | Persistente | Independente do Pod | Requisição de storage por apps |
| `StorageClass` | Cluster | N/A | N/A | Provisionamento dinâmico |
| `ConfigMap` | Namespace | Não-sensível | Independente | Config de app, arquivos de config |
| `Secret` | Namespace | Sensível (base64) | Independente | Senhas, tokens, certificados |
