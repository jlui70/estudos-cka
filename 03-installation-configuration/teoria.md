# Capítulo 3 — Instalação e Configuração do Cluster

---

## 1. Visão Geral dos Métodos de Instalação

Existem múltiplas formas de instalar um cluster Kubernetes. Cada uma tem seu propósito:

| Ferramenta | Uso | Ambiente |
|---|---|---|
| **kubeadm** | Cluster real, multi-node | Produção / Lab / CKA |
| **kind** (Kubernetes in Docker) | Cluster dentro de containers Docker | CI/CD, testes locais |
| **minikube** | Cluster local single-node | Aprendizado básico |
| **k3s** | K8s leve | IoT, Edge, Raspberry Pi |
| **GKE / EKS / AKS** | Cluster gerenciado | Cloud produção |

> **Para a CKA**: o exame usa clusters criados com **kubeadm**. Entender o processo do kubeadm é essencial.

---

## 2. kubeadm — Conceito e Fluxo

O `kubeadm` é a ferramenta oficial da Kubernetes para bootstrapar clusters de forma segura e reproduzível. Ele não gerencia a infraestrutura (VMs, redes) — apenas configura o K8s sobre infraestrutura já existente.

### O que o kubeadm faz e o que NÃO faz

| Faz | Não faz |
|---|---|
| Gera certificados TLS | Provisionar VMs |
| Inicializa o Control Plane | Configurar DNS externo |
| Configura o etcd | Instalar o container runtime |
| Gera o kubeconfig para admin | Instalar CNI plugin |
| Facilita join de novos nodes | Monitorar o cluster |

---

## 3. Pré-requisitos do Cluster

Antes de rodar o kubeadm, cada node precisa:

### Sistema Operacional
- Linux (Ubuntu 20.04+, CentOS 7+, Debian 10+)
- Swap **desabilitado** (obrigatório — o kubelet falha com swap ativo)
  ```bash
  swapoff -a
  # Para persistir no reboot, comentar a linha de swap no /etc/fstab
  ```

### Recursos mínimos
- **Control Plane**: 2 CPUs, 2 GB RAM
- **Workers**: 1 CPU, 1 GB RAM (recomendado 2+ CPUs)
- Conectividade de rede entre todos os nodes

### Portas necessárias (firewall)

**Control Plane:**
| Porta | Protocolo | Componente |
|---|---|---|
| 6443 | TCP | API Server |
| 2379-2380 | TCP | etcd |
| 10250 | TCP | Kubelet API |
| 10259 | TCP | Scheduler |
| 10257 | TCP | Controller Manager |

**Worker Nodes:**
| Porta | Protocolo | Componente |
|---|---|---|
| 10250 | TCP | Kubelet API |
| 30000-32767 | TCP | NodePort Services |

### Container Runtime

O K8s não inclui container runtime — ele precisa de um que implemente a **CRI (Container Runtime Interface)**:

| Runtime | Descrição |
|---|---|
| **containerd** | Padrão atual, extraído do Docker |
| **CRI-O** | Lightweight, criado pela Red Hat |
| Docker Engine | Suporte removido no K8s 1.24+ (usa dockershim) |

> Desde o K8s 1.24, o **dockershim** foi removido. Docker como runtime direto não é mais suportado. Usa-se **containerd** ou **CRI-O**.

---

## 4. Instalação Passo a Passo com kubeadm

### 4.1 Em todos os nodes

```bash
# 1. Desabilitar swap
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# 2. Habilitar módulos de kernel necessários
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# 3. Configurar parâmetros sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 4. Instalar containerd
apt install -y containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
# Ativar SystemdCgroup no config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# 5. Instalar kubeadm, kubelet, kubectl
apt-get install -y kubeadm kubelet kubectl
apt-mark hold kubeadm kubelet kubectl  # previne atualização acidental
```

### 4.2 No Control Plane — `kubeadm init`

```bash
kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \   # CIDR para os Pods (depende do CNI)
  --apiserver-advertise-address=<IP>     # IP do control plane
```

**O que acontece internamente durante o `kubeadm init`:**

1. Verifica pré-requisitos (swap, portas, versões)
2. Gera a CA (Certificate Authority) do cluster
3. Gera certificados para todos os componentes (API Server, etcd, kubelet, etc.)
4. Configura o **etcd** local
5. Configura o **API Server**, **Controller Manager**, **Scheduler** como Static Pods
6. Faz bootstrap do kubelet no control plane
7. Configura o kubeconfig `/etc/kubernetes/admin.conf`
8. Gera o token join para os workers

**Após o init, configurar o kubeconfig:**
```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4.3 Instalar o CNI Plugin

O K8s **não inclui** a rede de Pods — você precisa instalar um plugin CNI:

| CNI Plugin | Características |
|---|---|
| **Calico** | Suporta Network Policies, alta performance, mais usado |
| **Flannel** | Simples, sem Network Policies nativas |
| **Weave Net** | Fácil instalação, suporta NetPol |
| **Cilium** | eBPF-based, alta performance, observabilidade |

```bash
# Exemplo com Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

> Sem o CNI, os nodes ficam com status `NotReady`.

### 4.4 Nos Worker Nodes — `kubeadm join`

```bash
kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

O token e o hash são gerados pelo `kubeadm init`. Se o token expirar (24h), gere um novo:
```bash
kubeadm token create --print-join-command
```

---

## 5. Arquivos Importantes Gerados pelo kubeadm

| Arquivo/Diretório | Conteúdo |
|---|---|
| `/etc/kubernetes/admin.conf` | Kubeconfig do administrador |
| `/etc/kubernetes/pki/` | Todos os certificados e chaves do cluster |
| `/etc/kubernetes/manifests/` | Static Pods (api-server, etcd, scheduler, controller-manager) |
| `/var/lib/etcd/` | Dados do etcd |
| `/etc/containerd/config.toml` | Config do container runtime |

---

## 6. Static Pods — Como o Control Plane é Executado

Os componentes do Control Plane são executados como **Static Pods**. Diferente de Pods normais, Static Pods são gerenciados diretamente pelo **kubelet**, não pelo API Server.

```
/etc/kubernetes/manifests/
├── etcd.yaml
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

O kubelet monitora esse diretório. Se você editar um desses arquivos, o kubelet automaticamente mata e recria o pod correspondente. Se deletar o arquivo, o pod é removido.

> **Implicação para troubleshooting**: Se o API Server estiver caído, você ainda pode corrigir os Static Pods editando os arquivos em `/etc/kubernetes/manifests/` diretamente no node.

---

## 7. kubeconfig — Configuração de Acesso

O `kubeconfig` é o arquivo que contém as credenciais e configurações para conectar ao cluster. Localização padrão: `~/.kube/config`.

### Estrutura do kubeconfig

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <base64-ca>  # Certificado da CA do cluster
    server: https://192.168.1.100:6443        # Endereço do API Server
  name: kubernetes
users:
- name: kubernetes-admin
  user:
    client-certificate-data: <base64-cert>   # Certificado do usuário
    client-key-data: <base64-key>            # Chave do usuário
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes  # Contexto ativo
```

### Múltiplos clusters

Você pode ter múltiplos clusters no mesmo kubeconfig e trocar entre eles:

```bash
kubectl config get-contexts          # Lista todos os contextos
kubectl config current-context       # Contexto atual
kubectl config use-context <nome>    # Troca de contexto
kubectl config view                  # Exibe o kubeconfig completo
```

---

## 8. Verificando o Cluster após a Instalação

```bash
kubectl get nodes                    # Todos os nodes devem estar Ready
kubectl get pods -n kube-system      # Componentes do K8s devem estar Running
kubectl cluster-info                 # Endereço do API Server e CoreDNS
kubectl get componentstatuses        # (deprecated, mas ainda útil) Status dos componentes
```

---

## 9. Conceitos-Chave

| Conceito | Explicação |
|---|---|
| kubeadm init | Inicializa o Control Plane |
| kubeadm join | Adiciona um Worker Node ao cluster |
| CNI | Plugin de rede para Pods — obrigatório após o init |
| Static Pod | Pod gerenciado pelo kubelet diretamente (Control Plane) |
| kubeconfig | Arquivo com credenciais e endereço do cluster |
| Contexto | Combinação de cluster + usuário + namespace no kubeconfig |
| CRI | Interface padrão entre K8s e container runtime |
| Swap | Deve estar desabilitado — kubelet falha com swap ativo |
