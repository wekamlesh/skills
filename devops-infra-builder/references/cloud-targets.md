# Cloud & Production Targets Reference

Use this file when the target is a production or cloud environment: bare-metal k3s, AWS EKS, or DigitalOcean.

## Table of Contents
1. Bare-metal k3s (multi-node)
2. DigitalOcean (Droplets + DOKS)
3. AWS EKS
4. MetalLB — LoadBalancer for bare-metal
5. External DNS — auto DNS from Ingress
6. Sealed Secrets & External Secrets
7. Target comparison

---

## 1. Bare-Metal k3s (Multi-Node)

Use when you own physical or dedicated servers (homelab, colocation, Raspberry Pi cluster).

### Prerequisites
- Ubuntu 22.04 or Debian 12 on each node
- Static IPs or stable DHCP reservations
- Firewall: open ports 6443 (API), 10250 (kubelet), 8472/UDP (Flannel VXLAN), 51820/UDP (WireGuard if enabled)

### Master node install
```bash
curl -sfL https://get.k3s.io | sh -s - \
  --disable traefik \
  --disable servicelb \       # we use MetalLB
  --write-kubeconfig-mode 644 \
  --cluster-init \            # enables embedded etcd for HA (3+ masters)
  --tls-san <MASTER_IP> \
  --tls-san <DOMAIN>
```

Get the join token and kubeconfig:
```bash
cat /var/lib/rancher/k3s/server/node-token
cat /etc/rancher/k3s/k3s.yaml
```

### Worker node join
```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<MASTER_IP>:6443 \
  K3S_TOKEN=<TOKEN> sh -
```

### Upgrade k3s
```bash
# On each node (master first, then workers)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.30.0+k3s1 sh -
```

### Verify cluster
```bash
kubectl get nodes -o wide
kubectl get pods -A
```

---

## 2. DigitalOcean

Two options: managed DOKS (hands-off k8s) or self-managed k3s on Droplets.

### Option A: DOKS (Managed Kubernetes)
```bash
# Install doctl
brew install doctl
doctl auth init

# Create cluster
doctl kubernetes cluster create my-cluster \
  --region nyc3 \
  --node-pool "name=default;size=s-2vcpu-4gb;count=3" \
  --version 1.30

# Get kubeconfig
doctl kubernetes cluster kubeconfig save my-cluster
kubectl get nodes
```

DOKS gives you a managed control plane + LoadBalancer integration (DigitalOcean LB = `type: LoadBalancer`).

### Option B: k3s on Droplets (cheaper, more control)

Use Terraform (see iac-gitops.md) to provision Droplets, then install k3s with Ansible.

### DigitalOcean DNS for External DNS
```bash
# Create a DO API token with write access to DNS
kubectl create secret generic do-dns-token \
  --from-literal=access-token=<TOKEN> \
  -n external-dns
```

### DO Spaces (S3-compatible object storage)
Good for Loki, Tempo, Terraform state, and Velero backups:
```bash
# Create a Space via doctl or UI
doctl storage space create my-bucket --region nyc3

# Access keys: API > Spaces Keys
```

---

## 3. AWS EKS

Use EKS for larger teams or when you need AWS ecosystem (IAM, RDS, SES, etc.).

### Prerequisites
```bash
brew install awscli eksctl helm
aws configure   # set Access Key, Secret Key, region
```

### Create cluster with eksctl
```bash
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name default \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed
```

Or with a config file (preferred for GitOps):
```yaml
# infra/iac/eksctl-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-east-1
  version: "1.30"

managedNodeGroups:
  - name: default
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 4
    amiFamily: AmazonLinux2023
    iam:
      withAddonPolicies:
        autoScaler: true
        externalDNS: true
        certManager: true

addons:
  - name: aws-ebs-csi-driver
  - name: coredns
  - name: kube-proxy
  - name: vpc-cni
```

```bash
eksctl create cluster -f infra/iac/eksctl-cluster.yaml
```

### EKS with ALB Ingress (alternative to Nginx/Traefik)
```bash
helm repo add eks https://aws.github.io/eks-charts
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<ACCOUNT>:role/AWSLoadBalancerControllerRole
```

### EKS cost tips
- Use Spot instances for non-critical workloads (add a spot node group)
- Enable Cluster Autoscaler
- Use S3 for Loki/Tempo instead of EBS

---

## 4. MetalLB — LoadBalancer for Bare-Metal

Bare-metal clusters have no cloud LoadBalancer. MetalLB implements `type: LoadBalancer` using IP pools.

**Experimental note**: MetalLB L2 mode is stable. BGP mode is advanced but works well with real networking hardware.

### Install
```bash
helm repo add metallb https://metallb.github.io/metallb
helm upgrade --install metallb metallb/metallb \
  --namespace metallb-system --create-namespace
```

### Configure IP pool (L2 mode)
Pick an IP range on your LAN not used by DHCP:
```yaml
# infra/platform/metallb/ip-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lan-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.200-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
```

```bash
kubectl apply -f infra/platform/metallb/ip-pool.yaml
```

Now `type: LoadBalancer` services get a real LAN IP. Point your DNS A records there.

---

## 5. External DNS — Auto DNS from Ingress

External DNS watches Ingress objects and automatically creates/updates DNS records.

```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm upgrade --install external-dns external-dns/external-dns \
  --namespace external-dns --create-namespace \
  -f infra/platform/external-dns-values.yaml
```

### DigitalOcean provider
`infra/platform/external-dns-values.yaml`:
```yaml
provider: digitalocean
env:
  - name: DO_TOKEN
    valueFrom:
      secretKeyRef:
        name: do-dns-token
        key: access-token
sources:
  - ingress
  - service
domainFilters:
  - example.com
policy: sync   # 'upsert-only' is safer to start
```

### AWS Route53 provider
```yaml
provider: aws
aws:
  region: us-east-1
  zoneType: public
sources:
  - ingress
domainFilters:
  - example.com
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT>:role/ExternalDNSRole
```

---

## 6. Sealed Secrets & External Secrets

Never commit plain Kubernetes Secrets to Git.

### Sealed Secrets (encrypt in Git)
```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Install kubeseal CLI
brew install kubeseal

# Seal a secret
kubectl create secret generic my-secret --dry-run=client \
  --from-literal=api-key=supersecret -o yaml | \
  kubeseal --format yaml > infra/apps/my-app/sealed-secret.yaml

# Commit sealed-secret.yaml safely to Git
```

### External Secrets Operator (pull from AWS SSM / DO / Vault)
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
```

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-ssm
    kind: ClusterSecretStore
  target:
    name: my-app-secret
    creationPolicy: Owner
  data:
    - secretKey: api-key
      remoteRef:
        key: /my-app/api-key
```

---

## 7. Target Comparison

| | Bare-metal k3s | DigitalOcean Droplets + k3s | DOKS | AWS EKS |
|---|---|---|---|---|
| Cost | Hardware only | ~$12–48/mo/node | ~$48/mo (3 nodes) | ~$100+/mo |
| Control | Full | Full | Managed control plane | Managed control plane |
| LoadBalancer | MetalLB | MetalLB or DO LB | DO LB (auto) | AWS ALB / NLB |
| Storage | Local / NFS | DO Block Storage | DO Block Storage | EBS / EFS / S3 |
| Best for | Homelab, Raspberry Pi | Small teams, low budget | Medium teams | AWS ecosystem, scale |
| Upgrade effort | Manual | Manual | One click | Managed |
