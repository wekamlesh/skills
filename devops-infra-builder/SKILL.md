---
name: devops-infra-builder
description: >-
  Step-by-step DevOps infrastructure implementation guide for local dev and production environments.
  Use when setting up or extending Kubernetes/k3s clusters, Docker Compose stacks, ingress controllers
  (Traefik, Nginx Ingress, Istio), TLS with cert-manager and Let's Encrypt, observability stacks
  (Prometheus, VictoriaMetrics, Grafana, Loki, Jaeger, Tempo, OpenTelemetry), alerting
  (Alertmanager, PagerDuty), IaC (Terraform, Pulumi, Helm, Kustomize, Ansible), and GitOps
  (ArgoCD, Flux). Covers local runtimes (Docker Desktop, Raspberry Pi, k3s, k3d, Docker Compose)
  and production targets (bare-metal k3s, AWS EKS, DigitalOcean). Educates while implementing —
  explains concepts, shows real configs, uses experimental and cutting-edge tools when appropriate.
  Triggered by requests like "set up k3s", "add Traefik ingress", "install Prometheus and Grafana",
  "configure cert-manager", "set up ArgoCD", "deploy to DigitalOcean", "implement observability",
  "DevOps stack", "monitoring stack", "infrastructure as code".
---

# DevOps Infrastructure Builder

Step-by-step DevOps implementation guide for local and production environments. Educate while building — explain concepts, show real working configs, and embrace experimental or cutting-edge tools when they fit the context.

## Core Principles

- **Learn by doing**: Each implementation step explains the why, not just the what.
- **Stack-flexible**: Never assume a single tool. Match ingress, observability, and IaC choices to what the user specifies in the prompt or ask if ambiguous.
- **Living on the edge**: Experimental tools and new approaches are valid choices. Flag them as such and explain tradeoffs.
- **Runtime-agnostic**: Infra configs work for any app runtime; use Node or Go as concrete examples only when needed.
- **Both worlds**: Cover local dev parity and production promotion paths equally well.

## Classify The Request

Decide the path before generating anything:

- **Full greenfield stack**: new cluster or compose project from scratch — produce project layout, all configs, bring-up steps, and a runbook.
- **Component addition**: adding ingress, observability, IaC, or GitOps to an existing cluster — scope narrowly, preserve existing conventions.
- **Environment-specific task**: local-only (Pi, Docker Desktop) or cloud-only (AWS, DO, bare-metal) — load the relevant reference.
- **Educational walkthrough**: explain a concept, compare tools, or guide a decision — write structured explanations with concrete examples.

If the request is ambiguous, ask: environment target, desired components, and whether this is greenfield or extension.

## Default Architecture Choices

These are defaults only — override freely from the prompt:

| Concern | Local default | Production default |
|---|---|---|
| Orchestration | k3s (single node) or Docker Compose | k3s multi-node or managed k8s |
| Ingress | Traefik (k3s built-in) | Traefik or Nginx Ingress |
| TLS | cert-manager + Let's Encrypt staging | cert-manager + Let's Encrypt prod |
| Metrics | Prometheus | Prometheus or VictoriaMetrics |
| Dashboards | Grafana | Grafana |
| Logs | Loki | Loki |
| Traces | Tempo + OTel Collector | Jaeger or Tempo + OTel Collector |
| Alerting | Alertmanager + Slack/ntfy | Alertmanager + PagerDuty |
| IaC | Helm + Kustomize | Terraform or Pulumi + Helm |
| GitOps | ArgoCD | ArgoCD or Flux |

## Workflow

1. Classify the request (see above).
2. Identify the target environment(s): local, bare-metal, AWS, DigitalOcean, or mixed.
3. Choose or confirm the stack components from the prompt.
4. Load the relevant reference files (see below).
5. Generate numbered, step-by-step implementation with real configs — not pseudocode.
6. After each major component, add a **Verify** section with commands to confirm it works.
7. Add a **What you just built** summary explaining the concepts installed.
8. If experimental tools are used, add an **Experimental note** with caveats and alternatives.

## Project Layout Convention

For Kubernetes-based projects, prefer:

```text
infra/
  cluster/          # cluster bootstrap (k3s install, kubeadm, etc.)
  apps/             # per-app Helm values or Kustomize overlays
  platform/         # shared platform services
    ingress/
    cert-manager/
    observability/
    alerting/
    gitops/
  iac/              # Terraform or Pulumi root modules
  scripts/          # helper scripts: bootstrap, teardown, port-forward
  docs/             # ADRs and runbooks
```

For Docker Compose projects, use `/opt/<project>/` with `compose.yaml`, `.env`, `config/`, `ops/`.

## Output Expectations

Every implementation must produce:

- Numbered, step-by-step instructions a developer can follow from zero
- Complete, copy-paste-ready config files (YAML, HCL, shell scripts)
- A **Verify** block after each major step (kubectl, curl, or CLI commands)
- A brief concept explanation where a tool or pattern may be unfamiliar
- A **Next steps** section pointing to the next logical component to add

Do not produce partial snippets. If a config file is referenced, write the whole file.

## Reference Files

Load only what the request needs — do not load all at once:

- [references/local-setup.md](./references/local-setup.md) — Docker Desktop, Raspberry Pi, k3d, k3s local, Docker Compose dev environments
- [references/ingress-tls.md](./references/ingress-tls.md) — Traefik, Nginx Ingress, Istio, cert-manager, Let's Encrypt patterns
- [references/observability.md](./references/observability.md) — Prometheus, VictoriaMetrics, Grafana, Loki, Jaeger, Tempo, OpenTelemetry Collector
- [references/alerting.md](./references/alerting.md) — Alertmanager, PagerDuty, Grafana OnCall, ntfy, free alerting options
- [references/iac-gitops.md](./references/iac-gitops.md) — Terraform, Pulumi, Helm, Kustomize, Ansible, ArgoCD, Flux
- [references/cloud-targets.md](./references/cloud-targets.md) — AWS EKS, DigitalOcean DOKS, bare-metal k3s provisioning
