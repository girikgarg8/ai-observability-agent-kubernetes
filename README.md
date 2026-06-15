# AI Observability Agent on Kubernetes (AWS / EKS)

**Blame the Deploy** — correlate Kubernetes crash logs with Git commit history using Elastic Cloud, OpenTelemetry, ArgoCD, and Elastic Agent Builder.

Based on [piyushsachdeva/AI-observability](https://github.com/piyushsachdeva/AI-observability), adapted for **AWS EKS**.

## Repo layout (matches instructor)

```
ai-observability-agent-kubernetes/     ← this repo (standalone)
├── SETUP.md                           ← AWS runbook
├── README.md
├── release/
│   └── kubernetes-manifests.yaml      ← Online Boutique — ArgoCD deploys this
└── .github/workflows/
    └── index-deploy.yml               ← indexes every push to Elasticsearch
```

## Quick start

1. Follow **[SETUP.md](./SETUP.md)** end-to-end
2. Push this repo to GitHub as **`girikgarg8/ai-observability-agent-kubernetes`** (see Step 2)
3. Set `ES_ENDPOINT` + `ES_API_KEY` secrets on **this repo**
4. Point ArgoCD at **this repo**, `path: release`

## Stack

| Layer | Tech |
|-------|------|
| Cluster | Amazon EKS |
| App | Google Online Boutique (manifests only) |
| GitOps | ArgoCD |
| Observability | Elastic Cloud + OpenTelemetry kube-stack |
| CI metadata | GitHub Actions (`index-deploy.yml`) on **GitHub-hosted runners** |
| AI | Elastic Agent Builder + Claude |
