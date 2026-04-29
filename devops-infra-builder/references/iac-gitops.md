# IaC & GitOps Reference

Use this file when implementing infrastructure as code, Helm chart management, or GitOps workflows.

## Table of Contents
1. Helm — package management
2. Kustomize — overlay-based config
3. Helm + Kustomize together
4. Terraform — cloud infrastructure
5. Pulumi — IaC with real code
6. Ansible — node configuration
7. ArgoCD — GitOps (pull-based)
8. Flux — GitOps (lightweight)
9. IaC tool comparison

---

## 1. Helm

Helm is the Kubernetes package manager. Use it for installing third-party charts and packaging your own apps.

### Install Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Common workflow
```bash
# Add a repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Inspect defaults
helm show values grafana/grafana > grafana-defaults.yaml

# Install with overrides
helm upgrade --install grafana grafana/grafana \
  --namespace monitoring --create-namespace \
  -f infra/platform/observability/grafana-values.yaml

# Check status
helm list -A
helm status grafana -n monitoring
```

### Package your own app
```bash
helm create infra/apps/my-app
# Edit templates/ and values.yaml
helm lint infra/apps/my-app
helm template my-app infra/apps/my-app -f infra/apps/my-app/values.prod.yaml | kubectl apply -f -
```

---

## 2. Kustomize

Kustomize patches Kubernetes manifests without templating. Built into `kubectl` — no install needed.

### Directory layout
```text
infra/apps/my-app/
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
  overlays/
    local/
      kustomization.yaml
      patch-replicas.yaml
    prod/
      kustomization.yaml
      patch-replicas.yaml
      patch-resources.yaml
```

`base/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app: my-app
```

`overlays/prod/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml
images:
  - name: my-app
    newTag: "1.2.3"
```

`overlays/prod/patch-replicas.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
```

Apply:
```bash
kubectl apply -k infra/apps/my-app/overlays/prod
# Dry-run:
kubectl diff -k infra/apps/my-app/overlays/prod
```

---

## 3. Helm + Kustomize Together

Use Helm for third-party charts (install via Helm), Kustomize for your own app manifests and final patching.

A common pattern: render Helm output into a base, then patch with Kustomize:
```bash
helm template grafana grafana/grafana -f values.yaml > infra/platform/observability/base/grafana.yaml
# Then add to a kustomization.yaml and apply patches on top
```

Or use HelmRelease CRD (requires Flux) to manage Helm charts via Git.

---

## 4. Terraform

Terraform provisions cloud infrastructure: clusters, nodes, DNS, load balancers, object storage.

### Project layout
```text
infra/iac/
  main.tf
  variables.tf
  outputs.tf
  providers.tf
  modules/
    k3s-node/
    do-cluster/
    aws-eks/
```

### DigitalOcean k3s node example
`infra/iac/main.tf`:
```hcl
terraform {
  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }
  backend "s3" {
    # Use DO Spaces as S3-compatible backend
    endpoint = "nyc3.digitaloceanspaces.com"
    bucket   = "my-tf-state"
    key      = "infra/terraform.tfstate"
    region   = "us-east-1"   # required but ignored by DO
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true
  }
}

provider "digitalocean" {
  token = var.do_token
}

resource "digitalocean_droplet" "k3s_master" {
  name   = "k3s-master"
  region = "nyc3"
  size   = "s-2vcpu-4gb"
  image  = "ubuntu-22-04-x64"
  ssh_keys = [var.ssh_fingerprint]

  user_data = templatefile("${path.module}/scripts/k3s-install.sh", {
    k3s_token = random_password.k3s_token.result
  })
}

resource "random_password" "k3s_token" {
  length  = 32
  special = false
}

output "master_ip" {
  value = digitalocean_droplet.k3s_master.ipv4_address
}
```

### AWS EKS example (minimal)
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 3
      desired_size   = 2
    }
  }
}
```

### Commands
```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
terraform output master_ip
terraform destroy   # destroys everything — confirm first
```

---

## 5. Pulumi

Pulumi writes infrastructure as real code (TypeScript, Go, Python). Better for complex logic; same providers as Terraform.

**Experimental note**: Pulumi is production-stable but less ubiquitous than Terraform. Choose it when you prefer code over HCL.

### Setup
```bash
curl -fsSL https://get.pulumi.com | sh
pulumi login   # or pulumi login --local for local state
```

### DigitalOcean example (TypeScript)
```typescript
// infra/iac/index.ts
import * as do_ from "@pulumi/digitalocean";
import * as pulumi from "@pulumi/pulumi";

const droplet = new do_.Droplet("k3s-master", {
  region: do_.Region.NYC3,
  size:   do_.DropletSlug.DropletS2VCPU4GB,
  image:  "ubuntu-22-04-x64",
  userData: `#!/bin/bash
curl -sfL https://get.k3s.io | sh -`,
});

export const masterIp = droplet.ipv4Address;
```

```bash
cd infra/iac
npm install
pulumi up
pulumi stack output masterIp
```

---

## 6. Ansible

Ansible configures nodes over SSH — useful for bare-metal k3s setup and day-2 operations.

### Install
```bash
pip install ansible
```

### Inventory
```ini
# infra/iac/ansible/inventory.ini
[master]
192.168.1.100 ansible_user=ubuntu

[workers]
192.168.1.101 ansible_user=ubuntu
192.168.1.102 ansible_user=ubuntu
```

### k3s install playbook
```yaml
# infra/iac/ansible/playbooks/install-k3s.yaml
- name: Install k3s master
  hosts: master
  tasks:
    - name: Install k3s
      shell: |
        curl -sfL https://get.k3s.io | sh -s - \
          --disable traefik \
          --write-kubeconfig-mode 644
      args:
        creates: /usr/local/bin/k3s

    - name: Get node token
      slurp:
        src: /var/lib/rancher/k3s/server/node-token
      register: node_token

- name: Install k3s workers
  hosts: workers
  vars:
    master_ip: "{{ hostvars[groups['master'][0]]['ansible_host'] }}"
    k3s_token: "{{ hostvars[groups['master'][0]]['node_token']['content'] | b64decode | trim }}"
  tasks:
    - name: Join cluster
      shell: |
        curl -sfL https://get.k3s.io | \
          K3S_URL=https://{{ master_ip }}:6443 \
          K3S_TOKEN={{ k3s_token }} sh -
      args:
        creates: /usr/local/bin/k3s
```

```bash
ansible-playbook -i infra/iac/ansible/inventory.ini \
  infra/iac/ansible/playbooks/install-k3s.yaml
```

---

## 7. ArgoCD (GitOps — Pull-based)

ArgoCD watches a Git repo and syncs the cluster state to match. The cluster pulls from Git — no CI push credentials needed.

### Install
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Or via Helm:
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  -f infra/platform/gitops/argocd-values.yaml
```

### Access the UI
```bash
kubectl port-forward svc/argocd-server 8080:443 -n argocd
# Get initial password:
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

### Register an Application
```yaml
# infra/platform/gitops/app-my-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-infra-repo
    targetRevision: main
    path: infra/apps/my-app/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f infra/platform/gitops/app-my-app.yaml
# ArgoCD now auto-syncs on every Git push to main
```

---

## 8. Flux (GitOps — Lightweight)

Flux is lighter than ArgoCD, CLI-first, and integrates tightly with Helm via HelmRelease CRDs.

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux into your cluster (links to GitHub repo)
flux bootstrap github \
  --owner=your-org \
  --repository=your-infra-repo \
  --branch=main \
  --path=infra/clusters/prod \
  --personal
```

### HelmRelease example
```yaml
# infra/clusters/prod/platform/grafana.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: grafana
  namespace: monitoring
spec:
  interval: 10m
  chart:
    spec:
      chart: grafana
      version: "7.x"
      sourceRef:
        kind: HelmRepository
        name: grafana
        namespace: flux-system
  values:
    adminPassword: "changeme"
```

---

## 9. IaC Tool Comparison

| | Helm | Kustomize | Terraform | Pulumi | Ansible | ArgoCD | Flux |
|---|---|---|---|---|---|---|---|
| Layer | k8s packages | k8s config | Cloud infra | Cloud infra | Node config | GitOps | GitOps |
| Language | YAML + Go templates | YAML patches | HCL | TS/Go/Python | YAML | YAML | YAML |
| State | None (k8s is source) | None | Remote state file | Pulumi cloud / local | None | etcd (k8s) | etcd (k8s) |
| Learning curve | Low | Low | Medium | Medium | Low | Medium | Medium |
| Best for | 3rd-party charts | Your app configs | Cloud provisioning | Complex logic | Bare-metal setup | Full GitOps UI | Helm-heavy GitOps |
