# Capítulo 10 — Services e Networking

---

## 1. O Problema sem Services

Pods têm IPs dinâmicos — cada vez que um Pod é recriado, ele recebe um novo IP. Além disso, num Deployment com 3 réplicas, como um cliente sabe para qual dos 3 IPs enviar tráfego?

O **Service** resolve os dois problemas:
1. **IP estável**: Fornece um IP virtual fixo (ClusterIP) que não muda
2. **Load balancing**: Distribui tráfego entre todos os Pods do Selector

```
Antes (sem Service):              Depois (com Service):
Pod A → IP 10.244.1.5?           Pod A → Service ClusterIP 10.96.0.100
       → IP 10.244.2.3?                  ├── Pod B (10.244.1.5)
       → IP 10.244.3.7?                  ├── Pod C (10.244.2.3)
                                         └── Pod D (10.244.3.7)
```

---

## 2. Como Services Funcionam

### Endpoints

Quando você cria um Service com um `selector`, o K8s automaticamente cria um objeto **Endpoints** com os IPs de todos os Pods que casam com o selector:

```bash
kubectl get endpoints meu-service
# NAME          ENDPOINTS                                    AGE
# meu-service   10.244.1.5:8080,10.244.2.3:8080,...         5m
```

O **Endpoints Controller** mantém esse objeto atualizado: quando um Pod sobe ou cai, os endpoints são atualizados automaticamente. O **kube-proxy** usa esses endpoints para configurar as regras iptables/ipvs.

### Fluxo de tráfego

```
Pod A → ClusterIP:80
         │
         ▼ (kube-proxy regras iptables)
    Escolhe aleatoriamente:
    ├── Pod B endpoint 10.244.1.5:8080
    ├── Pod C endpoint 10.244.2.3:8080
    └── Pod D endpoint 10.244.3.7:8080
```

---

## 3. Tipos de Service

### 3.1 ClusterIP (padrão)

Expõe o Service com um IP interno ao cluster. Acessível **apenas de dentro do cluster**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP          # Padrão, pode omitir
  selector:
    app: backend
  ports:
  - port: 80               # Porta do Service
    targetPort: 8080       # Porta do container
    protocol: TCP
```

```
Acesso: backend:80  ou  backend.default.svc.cluster.local:80
Quem pode acessar: qualquer Pod no cluster
Quem NÃO pode acessar: tráfego externo
```

### 3.2 NodePort

Expõe o Service em uma porta de **cada Node** do cluster. Acessível externamente via `<IP-do-Node>:<NodePort>`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80           # Porta interna do Service (ClusterIP)
    targetPort: 3000   # Porta do container
    nodePort: 30080    # Porta no Node (30000-32767, ou auto-atribuída)
```

```
Acesso externo: http://<IP-do-qualquer-Node>:30080
Acesso interno: frontend:80
```

> **NodePort é usado em labs e para testes.** Em produção, prefira LoadBalancer ou Ingress. NodePort expõe em TODOS os nodes, mesmo que o Pod não esteja naquele node.

### 3.3 LoadBalancer

Cria um **Load Balancer externo** (em cloud providers). É o NodePort + um LB real na frente.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
```

```
Resultado:
  - ClusterIP criado internamente
  - NodePort criado em todos os nodes
  - Cloud provider cria um Load Balancer apontando para os NodePorts
  - External IP atribuído automaticamente
```

```bash
kubectl get service frontend-lb
# NAME         TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
# frontend-lb  LoadBalancer   10.96.1.50    203.0.113.50     80:30080/TCP
```

> Em **bare metal** (sem cloud), o EXTERNAL-IP ficará em `<pending>`. Use MetalLB para resolver isso em labs on-premises.

### 3.4 ExternalName

Cria um alias DNS para um serviço externo. Não faz proxy de tráfego — apenas retorna um CNAME.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: banco-externo
spec:
  type: ExternalName
  externalName: db.empresa.com    # DNS externo
```

```
Dentro do cluster: "banco-externo.default.svc.cluster.local"
CoreDNS retorna CNAME: "db.empresa.com"
```

> Útil para migrar apps para o cluster gradualmente, mantendo referência DNS estável.

---

## 4. Comparação dos Tipos de Service

| Tipo | IP externo | Acesso | Caso de uso |
|---|---|---|---|
| ClusterIP | Não | Só interno | Comunicação interna entre serviços |
| NodePort | Via Node IP | Externo (Node:porta) | Labs, testes, bare metal simples |
| LoadBalancer | Sim (cloud) | Externo (IP dedicado) | Produção em cloud |
| ExternalName | N/A | N/A | Alias para serviços externos |

---

## 5. Headless Service

Um Service sem ClusterIP — usado principalmente com **StatefulSets** para DNS direto aos Pods.

```yaml
spec:
  clusterIP: None      # Headless: sem ClusterIP
  selector:
    app: postgres
```

Com headless, o DNS resolve diretamente para os IPs dos Pods:
```
postgres-headless.default.svc.cluster.local → [10.244.1.5, 10.244.2.3, ...]
postgres-0.postgres-headless.default.svc.cluster.local → 10.244.1.5  (Pod específico)
```

---

## 6. DNS no Kubernetes (CoreDNS)

O **CoreDNS** é o servidor DNS padrão do K8s (roda como Deployment no `kube-system`).

### Padrão de DNS

```
# Service
<service-name>.<namespace>.svc.<cluster-domain>
# Exemplo:
backend.producao.svc.cluster.local

# Pod (menos comum)
<pod-ip-com-hifens>.<namespace>.pod.<cluster-domain>
# Exemplo:
10-244-1-5.default.pod.cluster.local
```

### Resolução entre namespaces

```bash
# Dentro do mesmo namespace: nome curto funciona
curl http://backend

# Outro namespace: precisa do namespace
curl http://backend.producao

# FQDN sempre funciona
curl http://backend.producao.svc.cluster.local
```

### Por que o DNS funciona nos Pods?

O kubelet configura o `/etc/resolv.conf` de cada Pod:
```
nameserver 10.96.0.10        # IP do CoreDNS Service
search default.svc.cluster.local svc.cluster.local cluster.local
```

---

## 7. Endpoints e EndpointSlices

### Endpoints

Objeto que lista os IPs e portas dos Pods que compõem um Service:

```bash
kubectl describe endpoints backend
# Subsets:
#   Addresses: 10.244.1.5,10.244.2.3,10.244.3.7
#   Ports: 8080/TCP
```

### Service sem Selector (Endpoints manuais)

Você pode criar um Service sem selector e adicionar Endpoints manualmente. Útil para: banco de dados externo, IP fixo de infraestrutura.

```yaml
# Service sem selector
apiVersion: v1
kind: Service
metadata:
  name: db-externo
spec:
  ports:
  - port: 5432
---
# Endpoints manuais
apiVersion: v1
kind: Endpoints
metadata:
  name: db-externo   # DEVE ter o mesmo nome do Service
subsets:
- addresses:
  - ip: 192.168.1.100    # IP do banco externo
  ports:
  - port: 5432
```

---

## 8. Network Policies (Introdução)

Por padrão, **todo Pod pode se comunicar com todo Pod** no cluster. Network Policies permitem restringir esse tráfego (ver Cap. 15 para detalhes).

```yaml
# Exemplo: bloquear todo tráfego de entrada para o namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: producao
spec:
  podSelector: {}      # Aplica a todos os Pods
  policyTypes:
  - Ingress             # Bloqueia ingress (sem regras = nega tudo)
```

---

## 9. Comandos Úteis — Services

```bash
# Listar services
kubectl get services
kubectl get svc -A

# Detalhes e endpoints
kubectl describe service backend
kubectl get endpoints backend

# Criar service via imperativo
kubectl expose deployment minha-app --type=ClusterIP --port=80 --target-port=8080

# Expor como NodePort
kubectl expose deployment minha-app --type=NodePort --port=80 --target-port=8080

# Port-forward para testar localmente (sem criar Service)
kubectl port-forward pod/meu-pod 8080:80
kubectl port-forward service/backend 8080:80

# Testar DNS de dentro do cluster
kubectl run test --image=busybox --rm -it -- nslookup backend.default.svc.cluster.local
kubectl run test --image=curlimages/curl --rm -it -- curl http://backend
```

---

## 10. Conceitos-Chave

| Conceito | Definição |
|---|---|
| Service | Abstração de rede estável sobre um conjunto de Pods |
| ClusterIP | IP virtual interno do Service (só acessível no cluster) |
| NodePort | Porta exposta em todos os Nodes para acesso externo |
| LoadBalancer | NodePort + Load Balancer externo (cloud) |
| ExternalName | Alias DNS para serviço fora do cluster |
| Endpoints | Lista de IPs/portas dos Pods de um Service |
| CoreDNS | Servidor DNS do K8s (resolve nomes de Services) |
| Headless Service | Service sem ClusterIP — DNS resolve para IPs dos Pods |
| kube-proxy | Agente que implementa regras iptables/ipvs para Services |
