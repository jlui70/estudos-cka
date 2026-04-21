# Capítulo 13 — Logging e Troubleshooting

> Este é o capítulo mais importante para a CKA. O exame é repleto de cenários de troubleshooting onde o cluster está quebrado e você precisa identificar e corrigir o problema.

---

## 1. Estratégia Geral de Troubleshooting

Sempre siga uma abordagem top-down:

```
1. Verificar o estado geral do cluster
   kubectl get nodes
   kubectl get pods -A

2. Identificar o objeto com problema
   kubectl get pod <nome> -o wide
   kubectl describe pod <nome>

3. Ver os logs
   kubectl logs <pod>

4. Olhar os Events
   kubectl get events --sort-by=.metadata.creationTimestamp

5. Se necessário, entrar no container
   kubectl exec -it <pod> -- /bin/sh

6. Verificar componentes do Control Plane (se o problema for no cluster)
   kubectl get pods -n kube-system
```

---

## 2. kubectl logs

```bash
# Logs de um Pod (container único)
kubectl logs meu-pod

# Logs de um container específico (Pod com múltiplos containers)
kubectl logs meu-pod -c meu-container

# Streaming de logs (--follow)
kubectl logs meu-pod -f

# Últimas N linhas
kubectl logs meu-pod --tail=50

# Logs das últimas 1 hora
kubectl logs meu-pod --since=1h

# Logs de um Pod que reiniciou (container anterior)
kubectl logs meu-pod --previous

# Logs de todos os Pods de um label
kubectl logs -l app=frontend --all-containers=true
```

---

## 3. kubectl describe

O `describe` é sua principal ferramenta de diagnóstico. Ele mostra:
- Configuração atual do objeto
- Status detalhado
- **Events** (seção mais importante para troubleshoot)

```bash
kubectl describe pod meu-pod
kubectl describe node worker-1
kubectl describe deployment minha-app
kubectl describe service meu-service
kubectl describe pvc minha-pvc
```

### Lendo os Events

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Normal   Scheduled         5m    default-scheduler  Successfully assigned default/meu-pod to worker-1
  Normal   Pulling           5m    kubelet            Pulling image "myapp:1.0"
  Warning  Failed            4m    kubelet            Failed to pull image "myapp:999": not found
  Warning  BackOff           3m    kubelet            Back-off pulling image "myapp:999"
```

> `Warning` events são sinais de problema. O campo `Message` normalmente explica o que há de errado.

---

## 4. Status de Pods e o que significam

```bash
kubectl get pods
# NAME        READY   STATUS             RESTARTS   AGE
# app-1       1/1     Running            0          5m     ← Saudável
# app-2       0/1     Pending            0          10m    ← Sem node disponível
# app-3       0/1     CrashLoopBackOff   5          8m     ← Container crashando
# app-4       0/1     ImagePullBackOff   0          3m     ← Imagem não encontrada
# app-5       0/1     OOMKilled          2          6m     ← Out of Memory
# app-6       1/1     Terminating        0          1m     ← Sendo deletado (stuck?)
# app-7       0/1     Init:0/1           0          2m     ← Init container falhando
# app-8       0/1     ContainerCreating  0          1m     ← Criando (aguarda ou stuck)
```

### Status e causas comuns

| Status | Causa Provável | Como investigar |
|---|---|---|
| `Pending` | Sem recursos, node cordoned, PVC pendente, taints, affinity | `describe pod` → Events |
| `CrashLoopBackOff` | App crashando, configuração errada, falta de dependência | `logs --previous` |
| `ImagePullBackOff` / `ErrImagePull` | Imagem errada, registry privado sem secret, sem acesso | `describe pod` → Events |
| `OOMKilled` | Container excedeu memory limit | `describe pod`, aumentar limits |
| `ContainerCreating` stuck | Volume não montado, CNI problema, secret/configmap não existe | `describe pod` → Events |
| `Terminating` stuck | Finalizer travado, PVC em uso | `delete --force --grace-period=0` |
| `Error` | Container saiu com código de erro | `logs`, `logs --previous` |

---

## 5. Troubleshooting de Nodes

### Node NotReady

```bash
kubectl get nodes
# worker-2   NotReady   <none>   1d   v1.28.0

kubectl describe node worker-2
# Conditions:
#   Type    Status  Reason
#   Ready   False   KubeletNotReady / MemoryPressure / DiskPressure / NetworkUnavailable
```

**Na máquina do node:**
```bash
# Verificar o kubelet
systemctl status kubelet
journalctl -u kubelet -f     # Logs do kubelet em tempo real
journalctl -u kubelet -n 50  # Últimas 50 linhas

# Causas comuns:
# - kubelet parou (systemctl restart kubelet)
# - Disco cheio
# - Certificado expirado
# - CNI plugin com problema
# - Swap ativado
```

### Verificar componentes do Control Plane

```bash
# Componentes do Control Plane rodam como Static Pods no kube-system
kubectl get pods -n kube-system

# Se API Server estiver fora, você não consegue usar kubectl
# Nesse caso: verificar diretamente nos logs do container
crictl ps -a    # Listar containers (containerd)
crictl logs <container-id>

# Ou ver os Static Pod YAMLs
ls /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

### Logs dos Componentes do Control Plane

```bash
# Via kubectl (se API Server estiver funcionando)
kubectl logs -n kube-system kube-apiserver-<nome>
kubectl logs -n kube-system etcd-<nome>
kubectl logs -n kube-system kube-scheduler-<nome>
kubectl logs -n kube-system kube-controller-manager-<nome>

# Via journalctl (no node do control plane)
journalctl -u kubelet

# Via logs diretos dos containers
/var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log
/var/log/pods/kube-system_etcd-*/etcd/*.log
```

---

## 6. Troubleshooting de Redes

### Pod não consegue se comunicar com Service

```bash
# 1. Verificar se o Service existe e tem endpoints
kubectl get service meu-service
kubectl get endpoints meu-service
# Se endpoints estiver vazio → selector não bate com labels dos Pods

# 2. Verificar labels dos Pods
kubectl get pods --show-labels
kubectl get pods -l app=meu-service    # Testar o selector

# 3. Testar DNS de dentro do cluster
kubectl run debug --image=curlimages/curl --rm -it -- sh
# Dentro do debug pod:
nslookup meu-service
nslookup meu-service.default.svc.cluster.local
curl http://meu-service:80

# 4. Verificar CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system deployment/coredns
```

### Pod não consegue se conectar à internet

```bash
# Testar de dentro do Pod
kubectl exec -it meu-pod -- curl https://kubernetes.io

# Verificar CNI plugin
kubectl get pods -n kube-system | grep -E "calico|flannel|weave|cilium"

# Verificar kube-proxy
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-<nome>
```

---

## 7. kubectl exec — Entrar no Container

```bash
# Shell interativo
kubectl exec -it meu-pod -- /bin/bash
kubectl exec -it meu-pod -- /bin/sh    # Para imagens sem bash

# Container específico (Pod multi-container)
kubectl exec -it meu-pod -c meu-container -- /bin/sh

# Executar comando único
kubectl exec meu-pod -- cat /etc/config/app.conf
kubectl exec meu-pod -- env          # Ver variáveis de ambiente
kubectl exec meu-pod -- ps aux       # Processos rodando
```

---

## 8. kubectl get events

Events são registros de o que aconteceu no cluster. Extremamente úteis:

```bash
# Events do namespace atual, ordenados por tempo
kubectl get events --sort-by=.metadata.creationTimestamp

# Só events de Warning
kubectl get events --field-selector type=Warning

# Events de um namespace específico
kubectl get events -n producao

# Filtrar por objeto
kubectl get events --field-selector involvedObject.name=meu-pod
```

---

## 9. Debugging com Pods Efêmeros (K8s 1.23+)

Para imagens que não têm shell (distroless, scratch), use debug containers:

```bash
# Adicionar um container de debug ao Pod em execução
kubectl debug -it meu-pod --image=busybox --target=meu-container

# Copiar o Pod com configurações extras para debug
kubectl debug meu-pod -it --copy-to=pod-debug --image=ubuntu
```

---

## 10. Troubleshooting de Volumes/PVC

```bash
# PVC em estado Pending
kubectl describe pvc minha-pvc
# Eventos mostrarão: ProvisioningFailed, WaitForFirstConsumer, etc.

# Causas comuns:
# - StorageClass não existe (kubectl get storageclass)
# - Provisioner não disponível
# - PV com tamanho insuficiente
# - AccessMode incompatível entre PVC e PV disponível
```

---

## 11. Checklist de Troubleshooting para a CKA

**Cluster não funciona (API Server inacessível):**
```
□ SSH para o control plane node
□ systemctl status kubelet
□ ls /etc/kubernetes/manifests/ — arquivos dos Static Pods existem?
□ crictl ps -a — containers estão rodando?
□ journalctl -u kubelet -n 100 — erro no kubelet?
□ Verificar /etc/kubernetes/manifests/kube-apiserver.yaml — porta, certs, flags corretas?
```

**Pod em CrashLoopBackOff:**
```
□ kubectl logs <pod> --previous — o que o app printou antes de crashar?
□ kubectl describe pod <pod> — Events, limites de memória, configuração
□ A imagem existe? kubectl describe pod → Events: pull error?
□ ConfigMap/Secret que o Pod usa existem?
□ A aplicação tem dependências não disponíveis?
```

**Pod em Pending:**
```
□ kubectl describe pod → Events: "0/3 nodes available: 3 Insufficient cpu"?
□ Nodes têm recursos suficientes? kubectl describe node
□ Taints bloqueando? kubectl describe node | grep Taint
□ Node Affinity muito restritiva?
□ PVC pendente? kubectl get pvc
```

**Service sem tráfego:**
```
□ kubectl get endpoints <service> — tem endpoints?
□ Labels dos Pods batem com selector do Service?
□ Porta do Service bate com containerPort?
□ kubectl exec em outro Pod: curl http://<service>:<port>
□ Network Policy bloqueando tráfego?
```
