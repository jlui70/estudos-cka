# Capítulo 05: APIs e Acesso ao Cluster

## Conceitos Chave para o CKA
- **Authentication (Quem é você?):** Certificados X.509, Service Accounts ou Tokens.
- **Authorization (O que você pode fazer?):** RBAC (Role-Based Access Control).
- **Admission Control:** Plugins que modificam ou rejeitam requisições (ex: AlwaysPullImages).

## Gerenciamento de Configuração (Kubeconfig)
O arquivo reside em `~/.kube/config` e é dividido em:
- `clusters`: Endereço do servidor e CA.
- `users`: Credenciais de acesso.
- `contexts`: A combinação de um Usuário + Cluster + Namespace.

## Comandos de Contexto
```bash
# Ver contextos atuais
kubectl config get-contexts

# Alternar entre contextos (muito comum na prova)
kubectl config use-context nome-do-contexto
