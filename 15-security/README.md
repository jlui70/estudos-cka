# Capítulo 15: Segurança

## Conceitos Chave para o CKA (Altamente Crítico!)
1. **RBAC:**
    - **Role:** Permissões em um Namespace.
    - **ClusterRole:** Permissões no cluster inteiro (ex: listar nós).
    - **RoleBinding:** Liga uma Role a um Usuário/ServiceAccount.
2. **Network Policies:** O firewall do K8s. Define quem fala com quem via labels.
3. **Security Context:** Define se o pod roda como root, privilégios de sistema, etc.



## Comando de Teste (Can-I)
```bash
# Verificar se eu (ou um service account) pode deletar pods
kubectl auth can-i delete pods --as system:serviceaccount:default:pipeline
