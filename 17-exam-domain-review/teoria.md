# Capítulo 17 — Exam Domain Review (CKA)

---

## 1. Estrutura da Prova CKA (2024-2026)

| Domínio | Peso |
|---|---|
| Cluster Architecture, Installation & Configuration | 25% |
| Workloads & Scheduling | 15% |
| Services & Networking | 20% |
| Storage | 10% |
| Troubleshooting | 30% |

> **Troubleshooting vale 30%** — é o domínio mais pesado. Invista mais tempo praticando cenários de cluster quebrado.

### Formato da prova

- **Duração**: 2 horas
- **Formato**: 100% prática no terminal (sem múltipla escolha)
- **Ambiente**: Browser com terminal, acesso a 5-6 clusters diferentes
- **Material permitido**: Kubernetes.io/docs (1 aba de browser adicional)
- **Pontuação mínima para aprovação**: 66%
- **Questões**: ~15-20 tarefas

---

## 2. Domínio 1: Cluster Architecture, Installation & Configuration (25%)

### O que cai

- kubeadm init / join
- Configurar Ingress
- Backup e restore do etcd
- Upgrade do cluster
- RBAC (criar Role, ClusterRole, RoleBinding)
- kubeconfig (adicionar cluster, contexto)

### Comandos essenciais

```bash
# kubeadm
kubeadm init --pod-network-cidr=192.168.0.0/16
kubeadm join <ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
kubeadm upgrade plan
kubeadm upgrade apply v1.29.0
kubeadm token create --print-join-command

# RBAC
kubectl create role <nome> --verb=get,list --resource=pods -n <ns>
kubectl create clusterrole <nome> --verb=get,list,watch --resource=nodes
kubectl create rolebinding <nome> --role=<role> --user=<user> -n <ns>
kubectl create clusterrolebinding <nome> --clusterrole=<crole> --user=<user>
kubectl auth can-i <verb> <resource> --as=<user> -n <ns>

# etcd backup
ETCDCTL_API=3 etcdctl snapshot save <path> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Upgrade node
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# (no node) apt-... && kubeadm upgrade node && systemctl restart kubelet
kubectl uncordon <node>
```

---

## 3. Domínio 2: Workloads & Scheduling (15%)

### O que cai

- Criar e gerenciar Deployments, DaemonSets, StatefulSets
- Rolling updates e rollbacks
- ConfigMaps e Secrets
- Resource requests/limits
- Node Affinity, Taints/Tolerations
- HPA

### Comandos essenciais

```bash
# Deployment
kubectl create deployment <nome> --image=<img> --replicas=3
kubectl set image deployment/<nome> <container>=<nova-imagem>
kubectl rollout status deployment/<nome>
kubectl rollout undo deployment/<nome>
kubectl rollout history deployment/<nome>
kubectl scale deployment <nome> --replicas=5

# Gerar YAML boilerplate (CKA tip: muito mais rápido que escrever do zero)
kubectl create deployment myapp --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml

# ConfigMap
kubectl create configmap app-config --from-literal=LOG_LEVEL=info
kubectl create configmap app-config --from-file=app.properties

# Secret
kubectl create secret generic db-secret --from-literal=password=s3cr3t
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key

# Taints
kubectl taint node <node> key=value:NoSchedule
kubectl taint node <node> key=value:NoSchedule-   # Remover

# Labels
kubectl label node <node> disktype=ssd
```

---

## 4. Domínio 3: Services & Networking (20%)

### O que cai

- Criar e configurar Services (ClusterIP, NodePort, LoadBalancer)
- Ingress e IngressClass
- Network Policies
- CoreDNS (troubleshoot DNS)

### Comandos essenciais

```bash
# Services
kubectl expose deployment <nome> --type=ClusterIP --port=80 --target-port=8080
kubectl expose deployment <nome> --type=NodePort --port=80

# Testar conectividade
kubectl run tmp --image=curlimages/curl --rm -it -- curl http://<service>:<port>
kubectl run tmp --image=busybox --rm -it -- nslookup <service>

# Network Policy
kubectl apply -f netpol.yaml
kubectl get networkpolicy -n <ns>

# Ingress
kubectl get ingress -A
kubectl describe ingress <nome>
```

---

## 5. Domínio 4: Storage (10%)

### O que cai

- PersistentVolumes e PersistentVolumeClaims
- StorageClass
- Usar PVC em Pods

### Comandos essenciais

```bash
kubectl get pv
kubectl get pvc -A
kubectl get storageclass
kubectl describe pvc <nome>

# Verificar binding
kubectl get pvc <nome> -o jsonpath='{.status.phase}'
# "Bound" = OK, "Pending" = problema
```

---

## 6. Domínio 5: Troubleshooting (30%)

### O que cai

- Pods em CrashLoopBackOff, Pending, ImagePullBackOff
- Nodes NotReady
- Services sem tráfego (endpoints vazios)
- Componentes do Control Plane falhando
- Logs via kubectl e journalctl

### Checklist mental para qualquer problema

```bash
# 1. Visão geral
kubectl get nodes
kubectl get pods -A | grep -v Running
kubectl get events --sort-by=.lastTimestamp -A | tail -20

# 2. Diagnóstico do objeto problemático
kubectl describe <tipo> <nome> -n <ns>
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous    # Se já reiniciou

# 3. Componentes do sistema
kubectl get pods -n kube-system
kubectl logs -n kube-system <componente>

# 4. No node (se necessário)
journalctl -u kubelet -n 100
systemctl status kubelet
ls /etc/kubernetes/manifests/
```

---

## 7. Dicas de Performance no Exame

### Use aliases e atalhos

```bash
# Crie no início da prova
alias k=kubectl
alias kn='kubectl -n'
export do='--dry-run=client -o yaml'

# Usar
k get po -A
k create deploy myapp --image=nginx $do > deploy.yaml
```

### Defina o namespace padrão

```bash
kubectl config set-context --current --namespace=<namespace-da-questao>
```

### Use kubectl explain para campos que não lembrar

```bash
kubectl explain pod.spec.securityContext
kubectl explain deployment.spec.strategy
kubectl explain networkpolicy.spec.ingress
```

### Edite recursos existentes em vez de criar do zero

```bash
kubectl edit deployment myapp      # Abre editor com o YAML atual
kubectl patch deployment myapp -p '{"spec":{"replicas":5}}'
```

### Verifique o contexto antes de cada questão

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <context-name>
```

---

## 8. Recursos Permitidos na Prova

Você pode usar **kubernetes.io/docs** durante a prova:

| Seção | Útil para |
|---|---|
| kubernetes.io/docs/concepts/ | Verificar campos de YAML |
| kubernetes.io/docs/reference/kubectl/ | Comandos kubectl |
| kubernetes.io/docs/tasks/ | Howtos passo a passo |
| kubernetes.io/docs/reference/generated/kubectl/kubectl-commands | Referência de comandos |

**Dica**: Antes da prova, memorize onde ficam estas páginas:
- Taints and Tolerations
- Assign Pods to Nodes (Node Affinity)
- Configure Service Accounts
- Network Policies
- Back up and Restore etcd
- Upgrading kubeadm clusters

---

## 9. Resumo Final por Frequência na Prova

### Tópicos quase certos (pratique muito):
1. **etcd backup e restore**
2. **Cluster upgrade** (kubeadm upgrade)
3. **RBAC** (Role + RoleBinding)
4. **Network Policy** (deny all + allow específico)
5. **Troubleshooting de Pod** (CrashLoop, Pending)
6. **Troubleshooting de node** (NotReady, kubelet)
7. **PVC + StorageClass**
8. **Deployment rolling update + rollback**
9. **Taints e Tolerations**
10. **Service (NodePort ou ClusterIP) + Ingress**

### Tópicos frequentes:
- StatefulSet
- DaemonSet
- Node Affinity
- Security Context
- ConfigMap/Secret como volume
- cordon/drain/uncordon

### Tópicos menos frequentes:
- Helm (básico: install/upgrade)
- CRDs (apenas interação, não criação)
- HPA
- Job/CronJob
- Pod Security Standards

---

## 10. Simulação Rápida — Resposta Ideal para Questão Típica

**Questão**: "O pod `app-backend` no namespace `producao` está em CrashLoopBackOff. Identifique e corrija o problema."

```bash
# 1. Verificar status atual
kubectl get pod app-backend -n producao

# 2. Ver os logs (container tendo crash)
kubectl logs app-backend -n producao --previous

# 3. Descrever para ver events e configuração
kubectl describe pod app-backend -n producao

# 4. Dependendo do que encontrou:
# - Imagem errada? Editar deployment
# - ConfigMap não existe? Criar o ConfigMap
# - Secret não existe? Criar o Secret  
# - Resources muito baixos? Aumentar limits
# - Liveness probe muito agressiva? Ajustar initialDelaySeconds

# 5. Verificar que corrigiu
kubectl get pod app-backend -n producao
# Status deve ser Running dentro de alguns segundos/minutos
```
