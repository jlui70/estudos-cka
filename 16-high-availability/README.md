# Capítulo 16: Alta Disponibilidade e Manutenção

## Tarefas Críticas de Administração

### Manutenção de Nós

## Esvaziar o nó para manutenção (remove os pods)
kubectl drain <node-name> --ignore-daemonsets

## Marcar o nó como disponível novamente
kubectl uncordon <node-name>

## Backup do etcd

# Exemplo de comando de snapshot
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup.db
