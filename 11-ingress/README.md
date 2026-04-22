# Capítulo 11: Ingress

## Conceitos Chave para o CKA
- **Ingress Controller:** O software (Nginx, HAProxy) que realmente faz o roteamento.
- **Ingress Resource:** O arquivo YAML que define as regras de "host" e "path".

## Diferença Fundamental
- **Service:** Camada 4 (TCP/UDP).
- **Ingress:** Camada 7 (HTTP/HTTPS). Permite usar o mesmo IP para vários domínios.
