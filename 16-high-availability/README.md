# Capítulo 16: Alta Disponibilidade e Manutenção

## Tarefas Críticas de Administração

### Manutenção de Nós
```bash
# Esvaziar o nó para manutenção (remove os pods)
kubectl drain <node-name> --ignore-daemonsets

# Marcar o nó como disponível novamente
kubectl uncordon <node-name>
