# Capítulo 17: Exam Domain Review

## Resumo de Pesos da Prova
- **Troubleshooting (30%):** Onde a maioria foca. Logs, eventos, rede e componentes.
- **Serviços e Rede (20%):** Services, Ingress, DNS, CNI.
- **Instalação e Configuração (15%):** Kubeadm, RBAC, etcd backup.
- **Carga de Trabalho (15%):** Deployments, Pods, ConfigMaps, Secrets.
- **Armazenamento (10%):** PV, PVC, StorageClasses.

## Estratégia de Prova
1. Use o `kubectl explain` se esquecer algum campo do YAML.
2. Abuse dos comandos imperativos (`--dry-run=client -o yaml`).
3. Verifique sempre se está no contexto de cluster correto.
