# Capítulo 10: Services

## Conceitos Chave para o CKA
1. **ClusterIP:** Acesso interno. IP virtual estável para os Pods.
2. **NodePort:** Abre uma porta (30000-32767) em TODOS os nós do cluster.
3. **LoadBalancer:** Integração com Cloud Providers para gerar um IP externo.
4. **Endpoints/EndpointSlices:** A lista de IPs reais dos Pods que o Service gerencia.



## Comando Útil
```bash
# Expor um deployment como NodePort rapidamente
kubectl expose deployment nginx --port=80 --type=NodePort
