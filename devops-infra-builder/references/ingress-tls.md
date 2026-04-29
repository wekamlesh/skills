# Ingress & TLS Reference

Use this file when implementing ingress controllers or TLS certificate management.

## Table of Contents
1. Traefik (k3s default)
2. Nginx Ingress Controller
3. Istio (service mesh + ingress)
4. cert-manager + Let's Encrypt
5. Wildcard certificates
6. Ingress comparison

---

## 1. Traefik

Traefik is k3s's built-in ingress. It uses IngressRoute CRDs or standard Kubernetes Ingress objects.

### Install via Helm (when k3s traefik is disabled)
```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

helm upgrade --install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  -f infra/platform/ingress/traefik-values.yaml
```

`infra/platform/ingress/traefik-values.yaml`:
```yaml
deployment:
  replicas: 1

ports:
  web:
    redirectTo:
      port: websecure
  websecure:
    tls:
      enabled: true

ingressRoute:
  dashboard:
    enabled: true
    entryPoints: ["websecure"]
    middlewares:
      - name: traefik-dashboard-auth

additionalArguments:
  - "--certificatesresolvers.letsencrypt.acme.email=you@example.com"
  - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"
  - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"

persistence:
  enabled: true
  size: 128Mi

service:
  type: LoadBalancer
```

### Standard Ingress object (works with Traefik)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: letsencrypt
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 3000
```

### Traefik Middleware (rate limit example)
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: default
spec:
  rateLimit:
    average: 100
    burst: 50
```

---

## 2. Nginx Ingress Controller

Good choice when you want wide community support and explicit Ingress spec compatibility.

### Install via Helm
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f infra/platform/ingress/nginx-values.yaml
```

`infra/platform/ingress/nginx-values.yaml`:
```yaml
controller:
  replicaCount: 1
  service:
    type: LoadBalancer
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true   # requires Prometheus Operator
  config:
    use-forwarded-headers: "true"
    proxy-body-size: "50m"
```

### Ingress object (Nginx)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 3000
```

### Verify
```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
# Note the EXTERNAL-IP — point your DNS A record here
```

---

## 3. Istio (Service Mesh + Ingress)

Istio is heavier but adds mTLS between services, fine-grained traffic management, and rich observability. Use when you need a service mesh, not just ingress.

**Experimental note**: Istio ambient mode (sidecar-less) is production-stable as of Istio 1.22 but still considered cutting-edge. Prefer it for new installs.

### Install with istioctl
```bash
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/istio-*/bin:$PATH

# Ambient mode (no sidecars — experimental but recommended for new installs)
istioctl install --set profile=ambient --set "components.ingressGateways[0].enabled=true" -y

# Label namespaces to enroll in the mesh
kubectl label namespace default istio.io/dataplane-mode=ambient
```

### Gateway + VirtualService
```yaml
# infra/platform/ingress/gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: main-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: app-tls   # Kubernetes TLS secret
      hosts:
        - "app.example.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
    - "app.example.com"
  gateways:
    - istio-system/main-gateway
  http:
    - route:
        - destination:
            host: my-app
            port:
              number: 3000
```

---

## 4. cert-manager + Let's Encrypt

cert-manager automates TLS certificate issuance and renewal.

### Install
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

### Verify
```bash
kubectl get pods -n cert-manager
# All three pods (controller, cainjector, webhook) should be Running
```

### ClusterIssuer — HTTP-01 challenge (needs public domain)
```yaml
# infra/platform/cert-manager/cluster-issuer-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - http01:
          ingress:
            class: nginx    # or traefik
```

```yaml
# infra/platform/cert-manager/cluster-issuer-prod.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Always test with staging first. Prod has rate limits (5 failures/hour per domain).

### Verify a certificate
```bash
kubectl get certificate -A
kubectl describe certificate app-tls -n default
# STATUS: True = issued successfully
```

---

## 5. Wildcard Certificates (DNS-01 challenge)

Wildcard certs (`*.example.com`) require DNS-01 challenge — cert-manager modifies a DNS TXT record.

### AWS Route53 example
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-dns
    solvers:
      - dns01:
          route53:
            region: us-east-1
            accessKeyIDSecretRef:
              name: route53-credentials
              key: access-key-id
            secretAccessKeySecretRef:
              name: route53-credentials
              key: secret-access-key
```

### DigitalOcean example
```yaml
    solvers:
      - dns01:
          digitalocean:
            tokenSecretRef:
              name: digitalocean-dns
              key: access-token
```

---

## 6. Ingress Comparison

| | Traefik | Nginx Ingress | Istio |
|---|---|---|---|
| Best for | k3s, Docker Compose, simple setups | Standard k8s, wide compatibility | Service mesh, mTLS, advanced traffic |
| Config style | IngressRoute CRDs or standard Ingress | Standard Ingress + annotations | Gateway + VirtualService CRDs |
| Auto TLS | Built-in ACME | Via cert-manager | Via cert-manager |
| Resource usage | Low | Low-Medium | High |
| Dashboard | Built-in | Via ingress-nginx-controller | Kiali |
| Learning curve | Low | Low | High |
| Experimental option | Traefik v3 Gateway API | — | Ambient mode (sidecar-less) |
