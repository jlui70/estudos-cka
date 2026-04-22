# Capítulo 08: Helm e Kustomize

## Conceitos Chave
- **Helm:** O "APT" do Kubernetes. Gerencia aplicações complexas via **Charts**.
- **Kustomize:** Permite gerenciar variações de YAML (Dev, Staging, Prod) sem usar templates, apenas overlays.

## Operações Básicas (Helm)
```bash
# Adicionar um repositório
helm repo add bitnami https://charts.bitnami.com/bitnami

# Instalar um chart (ex: nginx)
helm install meu-site bitnami/nginx
