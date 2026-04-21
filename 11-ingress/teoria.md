# Capítulo 11 — Ingress

---

## 1. O Problema que Ingress Resolve

Com Services do tipo **LoadBalancer**, cada Service precisa de um IP externo próprio (e consequentemente um Load Balancer na cloud). Para 10 aplicações, isso significa 10 Load Balancers — caro e ineficiente.

O **Ingress** permite:
- **Um único ponto de entrada** (um único LB/IP) para múltiplas aplicações
- Roteamento baseado em **hostname** e **path** HTTP/HTTPS
- Terminação de **TLS/HTTPS** centralizada

```
Sem Ingress:                      Com Ingress:
frontend-lb → IP1:443             
backend-lb  → IP2:443             IP único:443
api-lb      → IP3:443                 └── Ingress Controller
                                           ├── app.com/        → frontend-svc
                                           ├── app.com/api     → backend-svc
                                           └── admin.app.com   → admin-svc
```

---

## 2. Componentes do Ingress

O Ingress é composto por dois elementos:

### 2.1 Ingress Resource

O objeto Kubernetes que **define as regras de roteamento** (YAML). Não faz nada sozinho.

### 2.2 Ingress Controller

O software que **implementa as regras** do Ingress Resource. É um Pod que roda no cluster e age como proxy reverso.

> **Importante**: O K8s **não inclui** um Ingress Controller por padrão. Você precisa instalar um.

| Ingress Controller | Mantenedor | Notas |
|---|---|---|
| **NGINX Ingress Controller** | Kubernetes Community | Mais popular, padrão em labs CKA |
| ingress-nginx | NGINX Inc. | Versão oficial do NGINX (diferente do acima) |
| Traefik | Containous | Configuração automática via annotations |
| HAProxy | HAProxy Tech | Alta performance |
| Istio Gateway | Istio | Para service mesh |
| AWS ALB Controller | AWS | Cria ALB nativo na AWS |

---

## 3. Instalando o NGINX Ingress Controller

```bash
# Via Helm (forma recomendada)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Verificar
kubectl get pods -n ingress-nginx
kubectl get service ingress-nginx-controller -n ingress-nginx
```

---

## 4. Ingress Resource — Estrutura

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: meu-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /    # Annotation específica do NGINX
spec:
  ingressClassName: nginx          # Qual IngressClass usar
  rules:
  - host: app.meudominio.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 8080
  - host: admin.meudominio.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 80
```

### Path Types

| PathType | Comportamento |
|---|---|
| `Exact` | Path deve ser exatamente igual (`/foo` não bate com `/foo/`) |
| `Prefix` | Path é prefixo (case-insensitive, `/foo` bate com `/foo/bar`) |
| `ImplementationSpecific` | Depende do controller |

---

## 5. Ingress com TLS (HTTPS)

### Grau 1: TLS com Secret manual

```bash
# Criar certificado autoassinado (lab)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=app.meudominio.com"

# Criar Secret TLS
kubectl create secret tls meu-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

```yaml
spec:
  tls:
  - hosts:
    - app.meudominio.com
    secretName: meu-tls-secret    # Secret com tls.crt e tls.key
  rules:
  - host: app.meudominio.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

### Grau 2: TLS automático com cert-manager

O **cert-manager** é um add-on que automatiza a obtenção de certificados Let's Encrypt:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod    # Annotation para cert-manager
spec:
  tls:
  - hosts:
    - app.meudominio.com
    secretName: app-tls-secret    # cert-manager criará este Secret automaticamente
```

---

## 6. IngressClass

A partir do K8s 1.18, o campo `ingressClassName` substitui a annotation `kubernetes.io/ingress.class`:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # Default class
spec:
  controller: k8s.io/ingress-nginx
```

---

## 7. Annotations Comuns do NGINX Controller

```yaml
annotations:
  # Redirecionar HTTP para HTTPS
  nginx.ingress.kubernetes.io/ssl-redirect: "true"

  # Reescrever path (ex: /api/users → /users no backend)
  nginx.ingress.kubernetes.io/rewrite-target: /$2

  # Rate limiting
  nginx.ingress.kubernetes.io/limit-rps: "10"

  # Tamanho máximo do body
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"

  # CORS
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://trusted.com"

  # Autenticação básica
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth
```

---

## 8. Ingress vs Service LoadBalancer — Quando usar cada um?

| Critério | Service LoadBalancer | Ingress |
|---|---|---|
| Protocolo | Qualquer (TCP/UDP/HTTP) | HTTP/HTTPS |
| Roteamento | Só por porta | Por hostname e path |
| Custo | Um LB por Service | Um LB para todos os Services |
| TLS | Manual por Service | Centralizado |
| Uso | DB, gRPC, protocolos não-HTTP | APIs REST, apps web |

---

## 9. Fluxo completo de uma request via Ingress

```
Cliente
  │
  ▼ DNS: app.meudominio.com → IP do LB
LoadBalancer (ex: AWS NLB)
  │
  ▼ TCP port 443
NGINX Ingress Controller Pod
  │ TLS termination (decodifica HTTPS)
  ▼ lookup: host=app.meudominio.com path=/api
  backend-svc:8080
  │
  ▼ (kube-proxy / ClusterIP)
  backend Pod
```

---

## 10. Comandos Úteis — Ingress

```bash
# Listar Ingresses
kubectl get ingress
kubectl get ingress -A

# Detalhes
kubectl describe ingress meu-ingress

# Ver events (útil para troubleshoot)
kubectl describe ingress meu-ingress | grep Events -A 20

# Ver logs do Ingress Controller
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Verificar IngressClasses disponíveis
kubectl get ingressclass
```

---

## 11. Conceitos-Chave

| Conceito | Definição |
|---|---|
| Ingress Resource | Objeto K8s com regras de roteamento HTTP(S) |
| Ingress Controller | Pod que implementa as regras (NGINX, Traefik...) |
| IngressClass | Associa um Ingress Resource a um Controller específico |
| TLS Termination | Decodifica HTTPS no Ingress Controller; tráfego interno pode ser HTTP |
| Path-based routing | Rotas por prefixo de URL (/api, /web, /static) |
| Host-based routing | Rotas por domínio (api.com, admin.com) |
| cert-manager | Add-on que automatiza certificados TLS via Let's Encrypt |
| Annotation | Metadados que configuram comportamentos específicos do controller |
