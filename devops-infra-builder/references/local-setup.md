# Local Setup Reference

Use this file when the target environment is local dev: Docker Desktop, Raspberry Pi, k3d, k3s, or Docker Compose.

## Table of Contents
1. Docker Desktop Kubernetes
2. k3d (k3s in Docker)
3. k3s on Raspberry Pi
4. Docker Compose (no orchestration)
5. Local DNS and /etc/hosts patterns
6. Local TLS with mkcert
7. Environment parity tips

---

## 1. Docker Desktop Kubernetes

Enable in Docker Desktop > Settings > Kubernetes > Enable Kubernetes.

Context switching:
```bash
kubectl config use-context docker-desktop
kubectl get nodes
```

Limitations: single node, no LoadBalancer (use NodePort or port-forward). Use MetalLB or `kubectl port-forward` to expose services.

---

## 2. k3d (k3s in Docker)

k3d runs k3s inside Docker containers — best for local multi-node simulation and CI.

Install:
```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

Create a cluster with port mapping for Traefik/Nginx:
```bash
k3d cluster create dev \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer" \
  --agents 2
```

Delete:
```bash
k3d cluster delete dev
```

**Why k3d over minikube**: k3d boots faster, supports multi-node, and mirrors k3s production behavior more closely.

---

## 3. k3s on Raspberry Pi

### Prerequisites
- Raspberry Pi 4 (4GB+ recommended), Ubuntu 22.04 or Raspberry Pi OS 64-bit
- Static IP or mDNS hostname
- SSH access

### Enable cgroups (required for k3s)
Edit `/boot/firmware/cmdline.txt` (Ubuntu) or `/boot/cmdline.txt` (Pi OS) and append:
```
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```
Reboot after.

### Install k3s (server/master)
```bash
curl -sfL https://get.k3s.io | sh -s - \
  --disable traefik \           # we install our own ingress
  --write-kubeconfig-mode 644 \
  --node-name pi-master
```

Fetch the kubeconfig:
```bash
cat /etc/rancher/k3s/k3s.yaml
# Replace 127.0.0.1 with your Pi's IP for remote access
```

### Add a worker node
On the master, get the token:
```bash
cat /var/lib/rancher/k3s/server/node-token
```

On the worker Pi:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```

### Verify
```bash
kubectl get nodes -o wide
```

### ARM64 gotcha
Some Helm charts default to `amd64` images. Always check `image.repository` tags support `arm64` or use `--set image.tag=...` to an arm64-compatible tag.

---

## 4. Docker Compose (no Kubernetes)

Use when Kubernetes overhead is not justified. Good for single-service local dev or Raspberry Pi with limited RAM.

Minimal dev compose with Traefik as reverse proxy:

```yaml
# compose.yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"   # Traefik dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  app:
    image: node:20-alpine
    working_dir: /app
    volumes:
      - .:/app
    command: sh -c "npm install && npm run dev"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.localhost`)"
      - "traefik.http.services.app.loadbalancer.server.port=3000"
```

Access at `http://app.localhost` (works on macOS and Linux without /etc/hosts edits).

---

## 5. Local DNS and /etc/hosts

For `.local` or custom domains locally, add entries:

```
# /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows)
127.0.0.1  app.local grafana.local prometheus.local
```

On macOS, use `dnsmasq` to wildcard a TLD:
```bash
brew install dnsmasq
echo "address=/.local/127.0.0.1" >> /usr/local/etc/dnsmasq.conf
sudo brew services start dnsmasq
```

---

## 6. Local TLS with mkcert

For HTTPS locally without cert-manager:

```bash
brew install mkcert    # macOS
mkcert -install        # installs local CA into system trust
mkcert app.local grafana.local
# Outputs: app.local+1.pem  app.local+1-key.pem
```

Create a Kubernetes TLS secret:
```bash
kubectl create secret tls local-tls \
  --cert=app.local+1.pem \
  --key=app.local+1-key.pem \
  -n default
```

Reference in Ingress:
```yaml
tls:
  - hosts:
      - app.local
    secretName: local-tls
```

---

## 7. Environment Parity Tips

| Goal | Local approach |
|---|---|
| Mirror production namespaces | Use same namespace names (`platform`, `apps`) locally |
| Test Helm values | Use `values.local.yaml` overrides, merge with `-f` |
| Test resource limits | Set `resources.requests/limits` in local values too |
| Test multi-node | k3d with `--agents 2` |
| Test ARM builds | Use `docker buildx` with `--platform linux/arm64` |
| Avoid secrets in git | Use `.env` files + `kubectl create secret` locally; use Sealed Secrets or External Secrets in prod |
