# Capítulo 12: Scheduling

## Conceitos Chave para o CKA
1. **Taints and Tolerations:** Repulsão. "Este nó é especial, só entra quem eu permitir".
2. **Node Affinity:** Atração. "Este Pod prefere (ou exige) rodar em nós com a label X".
3. **Resource Requests:** O mínimo que o Pod precisa para rodar.
4. **Resource Limits:** O máximo que o Pod pode consumir.

## Taint Manual em um Nó
```bash
# Aplicar taint (NoSchedule evita novos pods sem toleration)
kubectl taint nodes node-1 chave=valor:NoSchedule
