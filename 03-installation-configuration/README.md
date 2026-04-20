# Capítulo 03: Installation and Configuration

Este capítulo documenta a criação do laboratório prático para a certificação CKA, utilizando virtualização nativa no Linux (KVM/Libvirt) sobre o Zorin OS.

## 🏗️ Arquitetura do Lab
- **OS das VMs:** Ubuntu 24.04 LTS (Noble Numbat)
- **Control Plane (Master):** 1 nó (2 vCPU, 4GB RAM)
- **Workers:** 2 nós (2 vCPU, 2GB RAM cada)
- **Runtime:** Containerd v1.7+
- **Kubernetes:** v1.30.x

---

## 🛠️ Passo a Passo da Instalação

### 1. Preparação dos Nós (Base)
Realizado em todos os nós antes da inicialização do cluster:
- **Desativação de Swap:** Necessário para estabilidade do kubelet.
- **Módulos do Kernel:** Ativação de `overlay` e `br_netfilter`.
- **Sysctl:** Configuração de `net.bridge.bridge-nf-call-iptables` e IP Forwarding.

### 2. Runtime (Containerd)
Instalação do Containerd e configuração do `SystemdCgroup = true` no arquivo `/etc/containerd/config.toml`. Este ajuste é crítico para que o Containerd e o Systemd do Ubuntu conversem corretamente sobre o uso de recursos.

### 3. Binários do K8s
Adição do repositório oficial `pkgs.k8s.io` e instalação de:
- `kubeadm`: Ferramenta para inicializar o cluster.
- `kubelet`: Agente que roda nos nós.
- `kubectl`: Ferramenta de linha de comando para interagir com o cluster.

---

## ⚡ Troubleshooting & Desafios (Importante!)

Durante a montagem deste lab, surgiram desafios reais que enriqueceram o aprendizado:

### Conflito de IPs em VMs Clonadas
Ao clonar a VM `k8s-worker-1` para gerar o `k8s-worker-2`, as máquinas apresentavam o mesmo endereço IP via DHCP.
- **Causa:** O identificador de máquina (`/etc/machine-id`) era idêntico em ambos os clones.
- **Solução:**
  1. Reset do `machine-id` no Worker 2.
  2. Ajuste no Netplan (`/etc/netplan/*.yaml`) definindo `dhcp-identifier: mac`.
  3. Atualização do hostname via `hostnamectl`.

### Erro de Privilégio (Preflight Checks)
- **Problema:** Erro `[ERROR IsPrivilegedUser]` ao tentar o join.
- **Solução:** O `kubeadm` exige privilégios de root para manipular serviços do sistema e certificados. Resolvido com o uso de `sudo`.

---

## 🌐 Rede do Cluster (CNI)
Foi instalado o **Calico** como plugin de rede para permitir a comunicação entre Pods em diferentes nós.
```bash
kubectl apply -f [https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml)
