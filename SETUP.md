# Setup Runbook — Blame the Deploy (AWS EKS)

End-to-end guide to reproduce the AI observability stack: **AWS EKS**, **Elastic Cloud**, **OpenTelemetry**, **ArgoCD**, **GitHub Actions**, and **Elastic Agent Builder**.

**Repo:** [github.com/girikgarg8/ai-observability-agent-kubernetes](https://github.com/girikgarg8/ai-observability-agent-kubernetes)

---

## Overview

A single `git push` triggers two paths:

1. **ArgoCD** deploys manifests to EKS  
2. **GitHub Actions** indexes deploy metadata to Elasticsearch  

OpenTelemetry ships cluster telemetry to the same Elastic deployment. When a pod fails, Agent Builder correlates crash data with deploy history.

```
git push
   ├── GitHub Actions → github-deployments (Elasticsearch)
   └── ArgoCD → EKS → OTel → Elastic Cloud
                              │
                        Agent Builder (on-demand triage)
```

**Setup order:**

| Must be first | Depends on |
|---------------|------------|
| EKS | — |
| Elastic Cloud | — |
| Deploy index + GitHub secrets | Elastic Cloud |
| OTel kube-stack | Elastic Cloud (needs endpoint + API key) |
| ArgoCD + Online Boutique | EKS + manifests on GitHub |
| Agent Builder | Elastic Cloud + both data sources populated |

**Online Boutique does not require OTel.** ArgoCD can deploy the app as soon as EKS is ready. OTel can be installed later — it will start collecting telemetry from pods that are already running. You only need OTel in place **before the demo**, so crash data appears in Elasticsearch.

A valid order: `EKS → Elastic Cloud → ArgoCD (app running) → OTel → deploy index → Agent Builder`

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) | EKS access |
| [eksctl](https://eksctl.io/) | Cluster provisioning |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Cluster operations |
| [helm](https://helm.sh/) | OTel kube-stack |
| [argocd CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) | GitOps |
| [gh](https://cli.github.com/) | GitHub secrets |

```bash
aws sts get-caller-identity    # verify AWS credentials
gh auth status                 # verify GitHub CLI
```

### Environment variables

Set once and reuse across steps:

```bash
export AWS_REGION=ap-south-1
export CLUSTER_NAME=elastic-cloud-test
export GITHUB_REPO=girikgarg8/ai-observability-agent-kubernetes
```

---

## Step 1 — EKS Cluster

- EKS requires **≥2 availability zones**
- Use **`m7i-flex.xlarge`** (4 vCPU, 16 GiB) — `large` is too small for 12 microservices + OTel + ArgoCD
- Cluster creation: **~15–20 minutes**

```bash
eksctl create cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --zones "${AWS_REGION}a,${AWS_REGION}b" \
  --version 1.31 \
  --nodegroup-name standard-pool \
  --node-type m7i-flex.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 3 \
  --managed \
  --with-oidc

aws eks update-kubeconfig --region "$AWS_REGION" --name "$CLUSTER_NAME"
kubectl get nodes
```

**Verify:** 3 nodes, `Ready`.

| Symptom | Fix |
|---------|-----|
| `only 1 zone(s) specified ... 2 are required` | Add `--zones "${AWS_REGION}a,${AWS_REGION}b"` |
| Pods `Pending` / `Insufficient cpu/memory` | Use `m7i-flex.xlarge`, not `large` |

---

## Step 2 — Elastic Cloud

1. [cloud.elastic.co](https://cloud.elastic.co) → **Create deployment**
2. Provider: **AWS** | Region: **`ap-south-1`** (match EKS) | Name: `blame-the-deploy`
3. Solution: **Elastic for Observability**
4. Save the `elastic` password (shown once)

### Collect endpoints

| Variable | Where to find |
|----------|---------------|
| `ES_ENDPOINT` | Deployment → **Elasticsearch** → Copy endpoint |
| `OTLP_ENDPOINT` | **Integrations** → **Manage** → **APM** → OTLP endpoint |
| `ES_API_KEY` | Kibana → **Settings** → **Security** → **API keys** → Create |

```bash
curl -s -w "\nHTTP:%{http_code}" \
  -X GET "$ES_ENDPOINT" \
  -H "Authorization: ApiKey $ES_API_KEY"
# Expect: HTTP:200
```

---

## Step 3 — OpenTelemetry Kube-Stack

Collects pod logs and Kubernetes metrics from EKS → Elastic Cloud.

```bash
helm repo add open-telemetry \
  'https://open-telemetry.github.io/opentelemetry-helm-charts' --force-update

kubectl create namespace opentelemetry-operator-system

kubectl create secret generic elastic-secret-otel \
  --namespace opentelemetry-operator-system \
  --from-literal=elastic_otlp_endpoint="$OTLP_ENDPOINT" \
  --from-literal=elastic_api_key="$ES_API_KEY"

helm upgrade --install opentelemetry-kube-stack \
  open-telemetry/opentelemetry-kube-stack \
  --namespace opentelemetry-operator-system \
  --values 'https://raw.githubusercontent.com/elastic/elastic-agent/refs/tags/v9.3.3/deploy/helm/edot-collector/kube-stack/managed_otlp/values.yaml' \
  --version '0.12.4'
```

**Verify:**

```bash
kubectl get pods -n opentelemetry-operator-system
# Expect: cluster-stats-collector, daemon-collector (×nodes), gateway-collector, operator — all Running
```

> Helm conflict: `helm uninstall opentelemetry-kube-stack -n opentelemetry-operator-system` then reinstall.

---

## Step 4 — Deploy Metadata Index

The workflow `.github/workflows/index-deploy.yml` runs on every push to `main` and POSTs deploy metadata to Elasticsearch.

### 4a. Create the index

```bash
curl -X PUT "$ES_ENDPOINT/github-deployments" \
  -H "Authorization: ApiKey $ES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "mappings": {
      "properties": {
        "timestamp":  { "type": "date" },
        "commit_sha": { "type": "keyword" },
        "author":     { "type": "keyword" },
        "service":    { "type": "keyword" },
        "image_tag":  { "type": "keyword" },
        "change":     { "type": "text" },
        "diff_url":   { "type": "keyword" }
      }
    }
  }'
# Expect: {"acknowledged":true}
```

### 4b. Add GitHub secrets

```bash
gh secret set ES_ENDPOINT --repo "$GITHUB_REPO" --body "$ES_ENDPOINT"
gh secret set ES_API_KEY   --repo "$GITHUB_REPO" --body "$ES_API_KEY"
```

### 4c. Verify indexing

```bash
git commit --allow-empty -m "test: verify ES indexing"
git push origin main

gh run list --repo "$GITHUB_REPO" --limit 3
# Expect: "Index deployment to Elasticsearch" → success

curl -s "$ES_ENDPOINT/github-deployments/_count" \
  -H "Authorization: ApiKey $ES_API_KEY"
# Expect: {"count":1,...}
```

---

## Step 5 — ArgoCD

```bash
kubectl create namespace argocd

kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl rollout status deployment/argocd-server -n argocd --timeout=180s

kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc argocd-server -n argocd -w
# Wait 2–5 min for ELB hostname

kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo

argocd login <ARGOCD-ELB-HOSTNAME> \
  --username admin --password <PASSWORD> --insecure
```

### Create Application

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: online-boutique
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/girikgarg8/ai-observability-agent-kubernetes
    targetRevision: main
    path: release
  destination:
    server: https://kubernetes.default.svc
    namespace: online-boutique
  syncPolicy:
    automated:
      prune: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### Sync and verify

```bash
argocd app sync online-boutique --force --prune

kubectl patch configmap argocd-cm -n argocd --type merge \
  -p '{"data": {"timeout.reconciliation": "30s"}}'
kubectl rollout restart deployment/argocd-repo-server -n argocd

kubectl get pods -n online-boutique
# Expect ~12 pods Running
```

**Frontend URL:**

```bash
kubectl get svc frontend-external -n online-boutique \
  -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}' && echo
```

---

## Step 6 — Agent Builder

Uses Elastic's native **Claude claude-sonnet-4-6**.

### 6a. Create tools

Kibana → ☰ → **Search** → **Tools** → **Create tool**

**Tool 1 — `get_crash_logs`**

- Index: `metrics-k8sclusterreceiver.otel-*`
- ES|QL:

```esql
FROM metrics-k8sclusterreceiver.otel-*
| WHERE k8s.namespace.name == "online-boutique"
| WHERE k8s.container.status.last_terminated_reason == "OOMKilled"
| SORT @timestamp DESC
| LIMIT 20
| KEEP @timestamp, k8s.deployment.name, k8s.pod.name, k8s.container.status.last_terminated_reason
```

**Tool 2 — `get_deploy_history`**

- Index: `github-deployments`
- ES|QL:

```esql
FROM github-deployments
| SORT timestamp DESC
| LIMIT 5
| KEEP timestamp, author, commit_sha, service, change, diff_url
```

### 6b. Create agent

Kibana → ☰ → **Search** → **Agent Builder** → **Create agent**

| Field | Value |
|-------|-------|
| Name | `blame-the-deploy` |
| Model | `Claude claude-sonnet-4-6` |
| Tools | `get_crash_logs`, `get_deploy_history` |

**Instructions:**

```
You are an SRE assistant. When asked why a service is crashing:
1. Use get_crash_logs to find OOMKill or memory events
2. Use get_deploy_history to find the most recent deployment to that service
3. If the deploy happened shortly before the crash, that is the likely cause
4. Report: crash time, commit SHA, author, what changed, and how to fix it
Always cite the commit SHA and author name in your answer.
```

---

## Step 7 — Run the Demo

### Break it

In `release/kubernetes-manifests.yaml`, find `paymentservice` Deployment (~line 631):

```yaml
resources:
  requests:
    memory: 10Mi   # was 64Mi
  limits:
    memory: 10Mi   # was 128Mi
```

```bash
git add release/kubernetes-manifests.yaml
git commit -m "perf: tune paymentservice memory limits for cost optimisation"
git push origin main
```

```bash
kubectl get pods -n online-boutique -w
# paymentservice: Running → OOMKilled → CrashLoopBackOff
```

### Ask the agent

Kibana → **Agent Builder** → `blame-the-deploy`

```
Why is paymentservice crashing? Check the logs and recent deployments.
```

### Fix it

```bash
git revert HEAD --no-edit
git push origin main
```

---

## Quick Reference

### Endpoints

| Service | How to get URL |
|---------|----------------|
| Online Boutique | `kubectl get svc frontend-external -n online-boutique` |
| ArgoCD | `kubectl get svc argocd-server -n argocd` |
| Kibana | Elastic Cloud deployment page |
| Elasticsearch | `$ES_ENDPOINT` |

### Useful commands

```bash
kubectl get pods -n online-boutique -w
kubectl get pods -n opentelemetry-operator-system
argocd app get online-boutique
kubectl describe pod -n online-boutique -l app=paymentservice | grep -A5 'Last State\|OOM'
```

### ES|QL — OOMKilled containers

```esql
FROM metrics-k8sclusterreceiver.otel-*
| WHERE k8s.namespace.name == "online-boutique"
| WHERE k8s.container.status.last_terminated_reason == "OOMKilled"
| SORT @timestamp DESC
| LIMIT 20
| KEEP @timestamp, k8s.deployment.name, k8s.pod.name, k8s.container.status.last_terminated_reason
```

### ES|QL — recent deploys

```esql
FROM github-deployments
| SORT timestamp DESC
| LIMIT 10
| KEEP timestamp, author, commit_sha, service, change
```

---

## Cleanup

Run when finished. **EKS and Elastic Cloud are the main cost drivers.**

### 1. Release LoadBalancers (optional, avoids orphaned ELBs)

```bash
kubectl delete svc frontend-external -n online-boutique --ignore-not-found
kubectl delete svc argocd-server -n argocd --ignore-not-found
# Wait 2–5 minutes
```

### 2. Delete EKS cluster

Removes control plane, node groups, VPC, NAT gateways, and cluster security groups.

```bash
eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --wait
# ~10–15 minutes
```

**Verify in AWS Console:**

- EKS → Clusters — gone
- EC2 → Instances — no worker nodes
- EC2 → Load Balancers — no orphaned ELBs
- EC2 → Volumes — delete unattached EBS volumes if any

### 3. Delete Elastic Cloud deployment

1. [cloud.elastic.co](https://cloud.elastic.co) → deployment **`blame-the-deploy`**
2. **Manage deployment** → **Delete deployment**

### 4. Optional

- Kibana → **API keys** — revoke demo keys
- `gh secret delete ES_ENDPOINT --repo "$GITHUB_REPO"`
- `gh secret delete ES_API_KEY --repo "$GITHUB_REPO"`

### Cleanup order

```
1. eksctl delete cluster      → stops AWS EC2, NAT, ELB charges
2. Delete Elastic deployment  → stops Elastic Cloud charges
3. Revoke API keys / secrets  → optional
```
