# Capítulo 16 — High Availability, etcd Backup/Restore e Cluster Upgrade

---

## 1. High Availability (HA) no Kubernetes

### O que é HA

Um cluster de **Alta Disponibilidade** garante que o cluster continue funcionando mesmo se um ou mais componentes falharem. O foco é no **Control Plane** — se o Control Plane cair, você não consegue deploy novos Pods, mas os Pods existentes **continuam rodando**.

### Topologia HA

```
┌──────────────────────────────────────────────────────────┐
│                    Load Balancer                         │
│              (HAProxy, AWS NLB, etc.)                    │
└──────┬────────────────┬────────────────┬─────────────────┘
       │                │                │
┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
│  Master 1   │  │  Master 2   │  │  Master 3   │
│             │  │             │  │             │
│ API Server  │  │ API Server  │  │ API Server  │
│ Scheduler   │  │ Scheduler   │  │ Scheduler   │
│ Ctrl Mgr    │  │ Ctrl Mgr    │  │ Ctrl Mgr    │
│             │  │             │  │             │
│  etcd       │  │  etcd       │  │  etcd       │
└─────────────┘  └─────────────┘  └─────────────┘
(etcd cluster com 3 membros — tolerância a 1 falha)
```

### Por que 3 masters (e não 2)?

O etcd usa o algoritmo **Raft** para consenso, que exige **quorum** (maioria):
- 2 nodes: quorum = 2/2 → falha de 1 node = cluster parado
- **3 nodes: quorum = 2/3 → tolera 1 falha**
- 5 nodes: quorum = 3/5 → tolera 2 falhas

> Sempre use número ímpar de membros no etcd: 1, 3, 5, 7.

---

## 2. Backup do etcd

### Por que fazer backup

O etcd armazena **todo o estado do cluster**. Se corrompido ou perdido, você perde todos os seus deployments, services, secrets, configmaps, etc.

**Backup regular do etcd é mandatório em produção** e é tópico quase certo na CKA.

### Onde fica o etcd

```bash
# Verificar configuração do etcd
kubectl describe pod -n kube-system etcd-<control-plane>
# ou
cat /etc/kubernetes/manifests/etcd.yaml
```

Campos importantes:
- `--data-dir=/var/lib/etcd` — Onde os dados são armazenados
- `--cert-file`, `--key-file`, `--trusted-ca-file` — Certificados para acesso

### Fazendo o Backup (snapshot)

```bash
# Definir versão da API do etcdctl
export ETCDCTL_API=3

# Backup
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verificar o backup
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | a1b2c3d4 |    12345 |        800 |     3.5 MB |
```

---

## 3. Restore do etcd

### Cenário: etcd corrompido, precisa restaurar

**Passo 1**: Parar o etcd

Como o etcd é um Static Pod, remova temporariamente o manifesto:
```bash
mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml
# Aguardar o kubelet parar o etcd (~30s)
```

**Passo 2**: Restaurar o snapshot

```bash
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master-1 \
  --initial-cluster=master-1=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380
```

**Passo 3**: Atualizar o manifesto do etcd para usar o novo data-dir

```bash
# Editar /tmp/etcd.yaml
# Alterar: --data-dir=/var/lib/etcd  →  --data-dir=/var/lib/etcd-restored
# Alterar o hostPath do volume correspondente
```

**Passo 4**: Restaurar o manifesto

```bash
mv /tmp/etcd.yaml /etc/kubernetes/manifests/etcd.yaml
# kubelet detecta o arquivo e reinicia o etcd
```

**Passo 5**: Verificar

```bash
kubectl get nodes
kubectl get pods -A
```

---

## 4. Cluster Upgrade com kubeadm

### Política de versões

O K8s suporta **3 versões minor** simultaneamente. A versão dos componentes deve seguir:

```
API Server: X.Y.Z (mais recente)
Controller Manager, Scheduler: X.Y ou X.Y-1
kubelet, kube-proxy: X.Y, X.Y-1, ou X.Y-2
kubectl: X.Y-1, X.Y, ou X.Y+1
```

### Ordem de upgrade

```
1. Control Plane (kubeadm upgrade)
2. Workers (drain → upgrade → uncordon)
```

> Nunca faça upgrade de workers antes do control plane.

### Upgrades do Control Plane

**Passo 1**: Atualizar kubeadm

```bash
# Ver versões disponíveis
apt-cache madison kubeadm

# Desbloquear e atualizar
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# Verificar
kubeadm version
```

**Passo 2**: Planejar e aplicar o upgrade

```bash
# Ver plano de upgrade
kubeadm upgrade plan

# Executar upgrade
kubeadm upgrade apply v1.29.0
```

**Passo 3**: Atualizar kubelet e kubectl do control plane

```bash
# Drenar o control plane (se possível)
kubectl drain <control-plane-node> --ignore-daemonsets

# Atualizar
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl

# Reiniciar kubelet
systemctl daemon-reload
systemctl restart kubelet

# Verificar
kubectl get nodes

# Remover o cordon
kubectl uncordon <control-plane-node>
```

### Upgrade dos Worker Nodes

Para cada worker node, **um de cada vez**:

**No control plane:**
```bash
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
```

**SSH para o worker-1:**
```bash
apt-mark unhold kubeadm kubelet kubectl
apt-get install -y kubeadm=1.29.0-00

kubeadm upgrade node         # Workers usam 'upgrade node', não 'upgrade apply'

apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubeadm kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet
```

**No control plane:**
```bash
kubectl uncordon worker-1
kubectl get nodes    # Verificar versão atualizada
```

Repetir para cada worker (worker-2, worker-3, etc.).

---

## 5. Node Maintenance — Padrão Completo

```bash
# 1. Cordon: novo Pod não vai para este node
kubectl cordon worker-2

# 2. (Opcional) Aguardar Pods terminarem naturalmente

# 3. Drain: remove todos os Pods
kubectl drain worker-2 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# 4. Fazer manutenção (SO update, hardware, etc.)

# 5. Uncordon: node volta a aceitar novos Pods
kubectl uncordon worker-2
```

---

## 6. PodDisruptionBudget (PDB)

O PDB garante que durante um `drain`, um mínimo de Pods continue disponível:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  selector:
    matchLabels:
      app: minha-app
  minAvailable: 2        # Mínimo de 2 Pods disponíveis durante disrupção
  # ou
  # maxUnavailable: 1    # Máximo de 1 Pod indisponível durante disrupção
```

> Se um `drain` violaria o PDB, ele aguarda até que os Pods sejam recriados em outros nodes antes de continuar.

---

## 7. Checklist HA / Backup / Upgrade para a CKA

**Backup do etcd:**
```
□ ETCDCTL_API=3
□ etcdctl snapshot save <caminho>
□ --endpoints, --cacert, --cert, --key corretos
□ Verificar com: etcdctl snapshot status
```

**Restore do etcd:**
```
□ Parar etcd (mover YAML do /etc/kubernetes/manifests/)
□ etcdctl snapshot restore com --data-dir novo
□ Atualizar manifesto YAML para apontar para novo data-dir
□ Restaurar YAML no /etc/kubernetes/manifests/
□ Aguardar etcd subir e verificar
```

**Upgrade:**
```
□ Atualizar kubeadm PRIMEIRO
□ kubeadm upgrade plan (ver versões disponíveis)
□ kubeadm upgrade apply vX.Y.Z (control plane)
□ Atualizar kubelet + kubectl (control plane)
□ Para cada worker: drain → upgrade node → kubelet → uncordon
□ Verificar versões: kubectl get nodes
```

---

## 8. Conceitos-Chave

| Conceito | Definição |
|---|---|
| HA Control Plane | Múltiplos masters atrás de LB — tolera falha de 1 master |
| etcd quorum | Maioria dos membros deve estar disponível |
| `etcdctl snapshot save` | Cria backup do etcd |
| `etcdctl snapshot restore` | Restaura etcd a partir de um backup |
| `kubeadm upgrade apply` | Faz upgrade do Control Plane |
| `kubeadm upgrade node` | Faz upgrade de um Worker Node |
| `kubectl drain` | Remove Pods de um node (para manutenção/upgrade) |
| `kubectl uncordon` | Habilita node a receber novos Pods novamente |
| PodDisruptionBudget | Garante disponibilidade mínima durante drains |
