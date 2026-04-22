# Capítulo 07: Deployments

## Conceitos Chave para o CKA
1. **ReplicaSet:** Garante o número exato de réplicas rodando.
2. **RollingUpdate:** Estratégia padrão. Substitui Pods antigos por novos gradualmente.
3. **Rollback:** Capacidade de desfazer uma atualização defeituosa.

## Anatomia do Deployment (Camadas)
- **Deployment** -> Gerencia o **ReplicaSet** -> Gerencia os **Pods**.

## Comandos Essenciais para o Exame
```bash
# Criar um deployment de forma imperativa (rápido para a prova)
kubectl create deployment nginx-app --image=nginx --replicas=3

# Verificar o histórico de versões
kubectl rollout history deployment/nginx-app

# Desfazer uma atualização (Rollback)
kubectl rollout undo deployment/nginx-app --to-revision=1
