# Capítulo 04: Arquitetura do Kubernetes

## Conceitos Chave para o CKA
1. **kube-apiserver:** O único componente que fala com o `etcd`. É a porta de entrada da API.
2. **etcd:** Banco de dados chave-valor que guarda o estado do cluster. Backup é vital aqui.
3. **kube-scheduler:** Observa novos Pods e decide em qual Nó eles devem rodar.
4. **kube-controller-manager:** Executa os loops de controle (Node Controller, Job Controller, etc).
5. **kubelet:** O "capitão" do nó. Garante que os containers descritos nos PodSpecs estejam rodando.
6. **kube-proxy:** Mantém as regras de rede nos nós (iptables/IPVS).

## Componentes do Sistema (Static Pods)
No nosso lab, os componentes do Control Plane rodam como Pods no namespace `kube-system`.
- Arquivos de manifesto: `/etc/kubernetes/manifests/`

## Comandos Úteis
```bash
# Verificar saúde dos componentes
kubectl get pods -n kube-system

# Ver logs do API Server para troubleshoot
kubectl logs -n kube-system kube-apiserver-control-plane
