# Capítulo 06: API Objects

## Conceitos Chave para o CKA
1. **Namespaces:** Divisões lógicas (multi-tenancy). O `default` é onde tudo cai se não especificado.
2. **Labels:** Metadados para identificar e organizar objetos (`app=frontend`).
3. **Selectors:** A "cola" que une objetos. Ex: Um Service usa um selector para achar seus Pods.
4. **Annotations:** Metadados para ferramentas externas (não usados para seleção).

## Comandos de Organização
```bash
# Filtrar pods por label
kubectl get pods -l env=production

# Criar um namespace rápido
kubectl create namespace desenvolvimento
