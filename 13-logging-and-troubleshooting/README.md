# Capítulo 13: Logging & Troubleshooting

## Fluxo de Resolução de Problemas (Troubleshooting)

### 1. Problemas no Pod
- `kubectl logs <pod-name>`: Ver a saída da aplicação.
- `kubectl describe pod <pod-name>`: Ver eventos (falhas de imagem, scheduling).
- `kubectl exec -it <pod-name> -- bin/bash`: Entrar no container para testes.

### 2. Problemas no Nó (Node)
- `kubectl describe node <node-name>`: Ver pressão de memória ou disco.
- `ssh <node-ip>` + `journalctl -u kubelet`: Ver logs do agente do Kubernetes.

### 3. Problemas no Cluster
- Verificar se o serviço `containerd` está ativo.
- Checar certificados expirados em `/etc/kubernetes/pki`.
