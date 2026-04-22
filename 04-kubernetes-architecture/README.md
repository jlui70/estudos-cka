Capítulo 04: Arquitetura do Kubernetes
Conceitos Chave para o CKA

Control Plane (Cérebro):

kube-apiserver: Único ponto de entrada para gerenciar o cluster.

etcd: Banco de dados chave-valor (estado do cluster). Fundamental para backup.

kube-scheduler: Decide em qual nó o Pod vai rodar baseado em recursos.

kube-controller-manager: Garante que o estado desejado seja mantido (Replication, Endpoint, etc).

Worker Nodes (Operários):

kubelet: O agente que garante que os containers estejam rodando no nó.

kube-proxy: Gerencia as regras de rede e comunicação.

Comandos de Diagnóstico

Bash
# Verificar componentes do sistema
kubectl get pods -n kube-system

# Ver logs de um componente específico (ex: api-server)
kubectl logs -n kube-system kube-apiserver-control-plane
