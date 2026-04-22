# Capítulo 09: Volumes e Persistência de Dados

## Conceitos Chave para o CKA
- **Persistent Volume (PV):** Recurso de armazenamento provisionado pelo Admin.
- **Persistent Volume Claim (PVC):** Requisição de armazenamento feita pelo usuário/Pod.
- **StorageClass:** Permite o provisionamento dinâmico (sob demanda).

## Ciclo de Vida do Volume
1. **Provisioning:** Criação do volume (Estático ou Dinâmico).
2. **Binding:** O PVC encontra um PV compatível e "se casa" com ele.
3. **Using:** O Pod monta o PVC como um volume.
4. **Reclaiming:** O que acontece quando o PVC é deletado (Retain, Delete ou Recycle).

## Dica para a Prova
Sempre verifique o `accessModes` (ReadWriteOnce, ReadOnlyMany, ReadWriteMany) para garantir compatibilidade entre PV e PVC.
