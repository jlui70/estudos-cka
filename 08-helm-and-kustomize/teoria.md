# Capítulo 8 — Helm e Kustomize

---

## 1. O Problema que Helm e Kustomize Resolvem

Aplicações Kubernetes reais têm muitos manifestos YAML (Deployments, Services, ConfigMaps, Secrets, Ingress, etc.). Gerenciar esses arquivos manualmente gera problemas:

- **Repetição**: Os mesmos padrões se repetem em dev, staging e prod
- **Versionamento**: Difícil saber qual versão de quais YAMLs está em produção
- **Configuração por ambiente**: Valores diferentes (imagem, réplicas, domínio) por ambiente
- **Dependências**: App A depende de banco B que precisa ser instalado antes

**Helm** e **Kustomize** resolvem esses problemas de formas diferentes:

| | Helm | Kustomize |
|---|---|---|
| Abordagem | Templates + Values | Patches sobre YAMLs base |
| Linguagem | Go templates | YAML puro |
| Gerenciamento de estado | Sim (releases, histórico) | Não (delegado ao kubectl) |
| Curva de aprendizado | Maior | Menor |
| Suporte nativo kubectl | Não (`helm` CLI separado) | Sim (`kubectl apply -k`) |
| Repositórios públicos/pacotes | Sim (Artifact Hub) | Não |

---

## 2. Helm

### O que é

Helm é o **gerenciador de pacotes** do Kubernetes. Similar ao `apt` no Debian ou `pip` no Python.

Conceitos centrais:
- **Chart**: Pacote Kubernetes — conjunto de templates YAML + configurações
- **Repository**: Servidor que hospeda Charts
- **Release**: Uma instância de um Chart instalado no cluster
- **Values**: Variáveis que personalizam o Chart na instalação

### 2.1 Estrutura de um Chart

```
meu-chart/
├── Chart.yaml          # Metadados do chart (nome, versão, descrição)
├── values.yaml         # Valores padrão das variáveis
├── templates/          # Templates YAML com Go template syntax
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl    # Funções reutilizáveis (não gera manifesto)
├── charts/             # Charts de dependências (sub-charts)
└── README.md
```

### 2.2 Chart.yaml

```yaml
apiVersion: v2
name: meu-app
description: Aplicação web de exemplo
type: application
version: 1.2.0          # Versão do próprio chart
appVersion: "2.5.1"     # Versão da aplicação empacotada
```

### 2.3 values.yaml

```yaml
replicaCount: 2

image:
  repository: myregistry/myapp
  tag: "2.5.1"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  host: ""

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### 2.4 Template com Go Template Syntax

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app      # Nome da release
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
spec:
  replicas: {{ .Values.replicaCount }}  # Valor do values.yaml
  selector:
    matchLabels:
      app: {{ .Release.Name }}-app
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-app
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

### 2.5 Comandos Helm essenciais

```bash
# Adicionar repositório
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Buscar charts
helm search repo nginx
helm search hub wordpress      # Busca no Artifact Hub (hub.helm.sh)

# Instalar
helm install meu-nginx bitnami/nginx
helm install meu-nginx bitnami/nginx --namespace producao --create-namespace

# Instalar com valores customizados
helm install meu-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort

# Instalar com arquivo de values
helm install meu-nginx bitnami/nginx -f meus-valores.yaml

# Simular instalação (dry run)
helm install meu-nginx bitnami/nginx --dry-run

# Listar releases
helm list
helm list -A    # Todos os namespaces

# Atualizar uma release
helm upgrade meu-nginx bitnami/nginx --set replicaCount=5

# Instalar ou atualizar (idempotente)
helm upgrade --install meu-nginx bitnami/nginx

# Ver histórico de uma release
helm history meu-nginx

# Rollback
helm rollback meu-nginx 1    # Volta para revisão 1

# Desinstalar
helm uninstall meu-nginx

# Ver os YAMLs que seriam gerados
helm template meu-nginx bitnami/nginx

# Ver valores de uma release
helm get values meu-nginx
```

### 2.6 Helm Templating — Conceitos

| Sintaxe | Descrição |
|---|---|
| `{{ .Values.campo }}` | Acessa values.yaml |
| `{{ .Release.Name }}` | Nome da release |
| `{{ .Chart.Name }}` | Nome do chart |
| `{{ .Chart.Version }}` | Versão do chart |
| `{{ if .Values.ingress.enabled }}` | Condicional |
| `{{ range .Values.lista }}` | Loop |
| `{{ toYaml .Values.obj \| nindent N }}` | Serializa objeto YAML com indentação |
| `{{- }}` | Remove espaço em branco à esquerda |
| `{{ include "helper" . }}` | Chama função de _helpers.tpl |

---

## 3. Kustomize

### O que é

Kustomize permite **personalizar manifestos Kubernetes sem templates**. Você tem YAMLs base e sobre eles aplica **patches** ou **overlays** para cada ambiente.

A filosofia é diferente do Helm:
- **Helm**: Você escreve templates com variáveis
- **Kustomize**: Você escreve YAMLs concretos e os **transforma** por camadas

### 3.1 Estrutura típica

```
k8s/
├── base/                     # YAMLs genéricos (sem valores de ambiente)
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── development/          # Overlay para desenvolvimento
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── production/           # Overlay para produção
        ├── kustomization.yaml
        └── replica-patch.yaml
```

### 3.2 base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
```

### 3.3 base/deployment.yaml (sem valores de env)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minha-app
  template:
    metadata:
      labels:
        app: minha-app
    spec:
      containers:
      - name: app
        image: myapp:latest
```

### 3.4 overlays/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: producao

namePrefix: prod-              # Prefixo em todos os recursos

bases:
- ../../base                   # Inclui a base

images:
- name: myapp                  # Substitui a imagem
  newName: myregistry/myapp
  newTag: "3.2.1"

patchesStrategicMerge:         # Patches merge estratégico
- replica-patch.yaml

commonLabels:                  # Adiciona labels em tudo
  environment: production
  managed-by: kustomize
```

### 3.5 overlays/production/replica-patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
spec:
  replicas: 5               # Sobrescreve o valor da base (era 1)
```

### 3.6 Tipos de Patches no Kustomize

**Strategic Merge Patch** (mais comum): merge inteligente que entende a estrutura K8s
```yaml
# Altera só o campo replicas, mantém o resto
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
spec:
  replicas: 5
```

**JSON Patch** (RFC 6902): operações explícitas
```yaml
patches:
- target:
    kind: Deployment
    name: minha-app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
    - op: add
      path: /spec/template/spec/containers/0/env/-
      value:
        name: LOG_LEVEL
        value: "debug"
```

### 3.7 Comandos Kustomize

```bash
# Ver o YAML final gerado (sem aplicar)
kubectl kustomize overlays/production
# ou
kustomize build overlays/production

# Aplicar
kubectl apply -k overlays/production

# Deletar
kubectl delete -k overlays/production
```

### 3.8 Funcionalidades úteis do Kustomize

```yaml
# Gerar ConfigMap a partir de arquivo
configMapGenerator:
- name: app-config
  files:
  - app.properties
  - database.conf

# Gerar Secret a partir de arquivo
secretGenerator:
- name: db-credentials
  literals:
  - username=admin
  - password=s3cr3t

# Adicionar suffix com hash (garante rollout ao mudar config)
generatorOptions:
  disableNameSuffixHash: false
```

---

## 4. Quando usar Helm vs Kustomize?

| Cenário | Helm | Kustomize |
|---|---|---|
| Instalar software de terceiros (nginx, cert-manager, prometheus) | ✅ Ideal | ❌ Não se aplica |
| Gerenciar suas próprias apps em múltiplos ambientes | ✅ Bom | ✅ Excelente |
| Equipe que não quer aprender Go templates | ❌ Complexo | ✅ Simples |
| Precisa de histórico de versões/rollback | ✅ Nativo | ❌ Depende do git |
| Pipeline CI/CD GitOps (ArgoCD, Flux) | ✅ Suportado | ✅ Suportado |
| Cluster air-gapped (sem internet) | ❌ Complexo | ✅ Fácil |

> Na prática, muitas equipes usam **Helm para apps de terceiros** e **Kustomize para suas próprias apps**. Ou ainda usam **Helm Charts com Kustomize como camada de customização**.

---

## 5. Relevância para a CKA

O capítulo 8 tem menor peso direto na prova CKA. O que é cobrado:

- Saber instalar um Chart Helm básico (`helm install`, `helm upgrade`)
- Entender o conceito de Chart/Release/Values
- Kustomize: entender o conceito de overlay (`kubectl apply -k`)
- Em 2024+, a prova inclui tarefas básicas com Helm

```bash
# Comandos que podem aparecer na CKA
helm install <release> <chart> --namespace <ns>
helm upgrade <release> <chart> --set key=value
helm list -n <namespace>
kubectl apply -k <diretorio>
```
