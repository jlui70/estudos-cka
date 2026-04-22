# Capítulo 01: Basics of Kubernetes

## Conceitos Chave para o CKA
1. **O que é o Kubernetes?** Orquestrador de containers baseado em desejos (Desired State).
2. **Nós (Nodes):** Unidades de computação.
3. **Control Plane:** O cérebro que decide onde as coisas rodam.
4. **Data Plane (Worker Nodes):** Onde os containers realmente trabalham.

## Anatomia de um Objeto (YAML)
Todo manifesto que vamos criar no Lab tem 4 campos obrigatórios:
- `apiVersion`: Versão da API (ex: v1).
- `kind`: Tipo do objeto (ex: Pod, Service).
- `metadata`: Dados sobre o objeto (nome, labels).
- `spec`: O estado desejado (o que eu quero que o K8s faça).
