# Blame the Deploy (AWS / EKS)

> Push bad code → ArgoCD deploys it → pod OOMKills → ask the AI agent → it tells you who broke it, what changed, and how to fix it.

AWS-adapted runbook based on [piyushsachdeva/AI-observability SETUP.md](https://github.com/piyushsachdeva/AI-observability/blob/main/SETUP.md).  
**Living document** — AWS-specific steps are added/updated here as we go.

| Instructor (GCP) | This guide (AWS) |
|------------------|------------------|
| GKE zonal cluster | **Amazon EKS** (regional, 2+ AZs required) |
| `e2-standard-4` × 3 | **`m7i-flex.xlarge` × 3** (4 vCPU, 16 GiB each) |
| GCP Marketplace → Elastic | Elastic Cloud hosted on **AWS** (or [elastic.co](https://cloud.elastic.co) direct) |
| `gcloud` | **`aws` CLI + `eksctl`** |

---

## How It Works

```
git push bad commit
       │
       ├──► GitHub Actions → indexes commit to github-deployments (ES)
       │
       └──► ArgoCD (polls every 30s) → deploys to EKS cluster
                                              │
                                        paymentservice pod
                                              │
                                        OOMKilled (10Mi limit)
                                              │
                                        OTel DaemonSet → ships crash to logs-* (ES)
                                              │
                                   ┌──────────┴──────────┐
                                logs-*          github-deployments
                                   └──────────┬──────────┘
                                              │
                                        Agent Builder
                                              │
                              "commit a3f92b by @author — that's the cause"
```

| Component | What it does |
|-----------|--------------|
| **EKS Cluster** | Runs 12 microservices + ArgoCD + OTel |
| **Online Boutique** | Google's demo e-commerce app — the thing we break |
| **ArgoCD** | Watches GitHub repo, auto-deploys every push |
| **OTel kube-stack** | Collects all pod logs + metrics → ships to Elastic Cloud |
| **`github-deployments`** | Custom ES index — stores who deployed what and when |
| **`logs-*`** | Pod logs in Elasticsearch — OOMKill events live here |
| **Agent Builder** | AI agent — correlates crash logs with deploy history |

**Why the order matters:**

1. Manifest must be pushed to GitHub **before** ArgoCD is created — ArgoCD deploys whatever is in the repo at first sync
2. Elastic Cloud must exist **before** OTel — OTel needs the endpoint + API key to send logs
3. `github-deployments` index must exist **before** GitHub secrets are added — first push writes to it
4. Everything must be running **before** the demo — agent needs data in both indices

---

## Prerequisites

```bash
# AWS CLI
brew install awscli                             # macOS
# Linux: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
aws configure                                   # or: aws sso login
aws sts get-caller-identity                     # verify credentials

# eksctl
brew install eksctl                             # macOS
# Linux:
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# kubectl
brew install kubectl                            # macOS
# Linux:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# helm
brew install helm                               # macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash  # Linux

# argocd CLI
brew install argocd                             # macOS
# Linux:
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# gh CLI
brew install gh && gh auth login                # macOS
sudo apt install gh -y && gh auth login         # Linux

# git
git config --global user.name "your-username"
git config --global user.email "your@email.com"
```

---

## Step 1 — EKS Cluster

**GCP equivalent:** `gcloud container clusters create` + `e2-standard-4` node pool × 3 in `us-central1-a`.

**AWS notes:**

- EKS control plane requires **at least 2 availability zones** (unlike GKE zonal clusters).
- `m7i-flex.xlarge` ≈ `e2-standard-4` (4 vCPU, 16 GiB) — do **not** use `m7i-flex.large` (2 vCPU); Online Boutique + OTel + ArgoCD need ~6+ vCPU total.
- Cluster creation takes **~15–20 minutes**.

```bash
export AWS_REGION=ap-south-1          # change if needed; keep Elastic region aligned
export CLUSTER_NAME=elastic-cloud-test

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

# Connect kubectl (GCP equivalent: gcloud container clusters get-credentials)
aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"

# Verify — expect 3 nodes, Status: Ready
kubectl get nodes -o wide
```

**Optional:** pin all worker nodes to a single AZ (closer to instructor's zonal GKE setup):

```bash
# Add this flag to eksctl create cluster above:
#   --node-zones "${AWS_REGION}a"
```

**Troubleshooting**

| Symptom | Cause | Fix |
|---------|-------|-----|
| `only 1 zone(s) specified ... 2 are required` | EKS needs 2+ AZs for control plane | Use `--zones "${AWS_REGION}a,${AWS_REGION}b"` |
| Pods stuck `Pending` | Nodes too small | Use `m7i-flex.xlarge`, not `large` |
| `Insufficient cpu/memory` on describe pod | Same | Scale up instance type or add nodes |

**Cleanup**

See **[Step 9 — Cleanup](#step-9--cleanup-stop-billable-resources)** for full teardown (EKS + Elastic Cloud).

```bash
eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
```

---

## Step 2 — This Repo (standalone, matches instructor layout)

Same structure as [piyushsachdeva/AI-observability](https://github.com/piyushsachdeva/AI-observability) — Online Boutique manifests + workflow + docs at **repo root**, not nested inside a monorepo.

```
ai-observability-agent-kubernetes/
├── SETUP.md
├── README.md
├── release/
│   └── kubernetes-manifests.yaml    ← ArgoCD deploys this (path: release)
└── .github/workflows/
    └── index-deploy.yml             ← GitHub Actions (GitHub-hosted runner)
```

**Pre-configured in `release/kubernetes-manifests.yaml` (from instructor):**

- All 12 Online Boutique services (public container images — work on EKS)
- `strategy: type: Recreate` on cartservice, paymentservice, frontend, recommendationservice
- No Istio manifests (not needed)

**Only workflow needed:** `index-deploy.yml` — no upstream `ci-main.yaml`, `ci-pr.yaml`, etc.

### 2a. Publish as its own GitHub repo

This folder currently lives inside `DevOps-Projects/`. Publish it as a **separate repo** so GitHub Actions and ArgoCD can use standard paths.

```bash
cd /Users/ggarg1/Personal/DevOps-Projects/ai-observability-agent-kubernetes

# Initialize git in this folder (separate from DevOps-Projects monorepo)
git init
git branch -M main
git add SETUP.md README.md .gitignore release/ .github/
git commit -m "Initial commit: Blame the Deploy on AWS EKS"

# Create GitHub repo and push (requires: gh auth login)
gh repo create ai-observability-agent-kubernetes \
  --public \
  --source=. \
  --remote=origin \
  --description "AI observability agent: correlate K8s crashes with Git commits (AWS EKS + Elastic)" \
  --push
```

Result: **`https://github.com/girikgarg8/ai-observability-agent-kubernetes`**

Use this repo URL everywhere below (ArgoCD, `gh secret set`, etc.).

### 2b. Optional — stop tracking in DevOps-Projects monorepo

If you don't want two copies, add to `DevOps-Projects/.gitignore`:

```
ai-observability-agent-kubernetes/
```

Then remove from the monorepo index (keeps files on disk):

```bash
cd /Users/ggarg1/Personal/DevOps-Projects
git rm -r --cached ai-observability-agent-kubernetes
git commit -m "Move ai-observability to standalone repo"
```

> **Do not edit manifests yet.** Come back in Step 8 (demo loop) to break `paymentservice` memory limits.

**EKS notes:**

1. **LoadBalancer delay** — `frontend-external` stays `<pending>` for **2–5 min** on EKS while AWS provisions the ELB.
2. **Images are public** — `us-central1-docker.pkg.dev/google-samples/...` pulls work on EKS without changes.

---

## Step 3 — Elastic Cloud

> _AWS-specific provider/region selection TBD._

### 3a. Create deployment

1. [cloud.elastic.co](https://cloud.elastic.co) → **Create deployment**
2. Provider: **AWS** | Region: **`ap-south-1`** (match EKS region) | Name: `blame-the-deploy`
3. Solution: **Elastic for Observability**
4. Click **Create** → **save the `elastic` password immediately** (shown once only)

### 3b. Get your endpoints

From `cloud.elastic.co` → click your deployment:

| What | Where to find it | Used for |
|------|------------------|----------|
| Elasticsearch endpoint | Under **Elasticsearch** → **Copy endpoint** | GitHub secret `ES_ENDPOINT`, curl commands |
| OTLP ingest endpoint | **Integrations** → **Manage** → **APM** → OTLP endpoint | OTel secret `elastic_otlp_endpoint` |

### 3c. Create API key

Kibana → **Settings** (bottom left) → **Security** → **API keys** → **Create API key**

Verify:

```bash
curl -s -w "\nHTTP:%{http_code}" \
  -X GET "<YOUR-ES-ENDPOINT>" \
  -H "Authorization: ApiKey <YOUR-API-KEY>"
# Expect: HTTP:200
```

---

## Step 4 — OpenTelemetry Kube-Stack

Ships all pod logs + metrics from EKS to Elastic Cloud. Install once, never touch again.

```bash
helm repo add open-telemetry 'https://open-telemetry.github.io/opentelemetry-helm-charts' --force-update

kubectl create namespace opentelemetry-operator-system

kubectl create secret generic elastic-secret-otel \
  --namespace opentelemetry-operator-system \
  --from-literal=elastic_otlp_endpoint='<YOUR-OTLP-INGEST-ENDPOINT>' \
  --from-literal=elastic_api_key='<YOUR-API-KEY>'

helm upgrade --install opentelemetry-kube-stack open-telemetry/opentelemetry-kube-stack \
  --namespace opentelemetry-operator-system \
  --values 'https://raw.githubusercontent.com/elastic/elastic-agent/refs/tags/v9.3.3/deploy/helm/edot-collector/kube-stack/managed_otlp/values.yaml' \
  --version '0.12.4'
```

> If Helm fails with conflict: `helm uninstall opentelemetry-kube-stack -n opentelemetry-operator-system` then reinstall.

Verify:

```bash
kubectl get pods -n opentelemetry-operator-system
# Expect all Running:
# cluster-stats-collector   1/1 Running
# daemon-collector          1/1 Running  (×3, one per node)
# gateway-collector         1/1 Running  (×2)
# opentelemetry-operator    2/2 Running
```

---

## Step 5 — GitHub Deployments Index

Stores who deployed what and when — the bridge between crash logs and commit history.

### GitHub Actions runner — hosted by GitHub (not AWS, not self-hosted)

The deploy-index workflow uses a **GitHub-hosted runner**:

```yaml
jobs:
  index:
    runs-on: ubuntu-latest   # ← GitHub-provided VM, not your EKS cluster
```

| Question | Answer |
|----------|--------|
| **Who owns the runner?** | **GitHub.** It is a temporary `ubuntu-latest` VM in GitHub's cloud, spun up for each workflow run and destroyed after. |
| **Does it run on EKS?** | **No.** The runner never touches your cluster. It only runs `git diff` + `curl` to POST metadata to Elastic Cloud. |
| **Do I need to install a runner?** | **No.** GitHub-hosted runners work out of the box on any public/private repo with Actions enabled. No EC2, no EKS, no self-hosted agent. |
| **What about the author's other workflows?** | Google's upstream Online Boutique uses **self-hosted GCE runners** for build/deploy CI. We **do not** use those — only `index-deploy.yml`. |

```
push to main
     │
     ▼
GitHub-hosted runner (ubuntu-latest)     ← GitHub's infrastructure
     │  checkout repo
     │  detect changed service
     │  curl POST → Elastic Cloud
     ▼
github-deployments index

(separately, on EKS)

ArgoCD polls same repo → deploys manifests to cluster
```

> **Important:** ArgoCD runs **inside EKS**. GitHub Actions runs **on GitHub's runners**. They are two independent pipelines triggered by the same `git push`.

### 5a. Workflow (already in this repo)

`.github/workflows/index-deploy.yml` is included at repo root — triggers on push to `main`, runs on **GitHub-hosted** `ubuntu-latest`. No changes needed for AWS/EKS.

After Step 2a push, verify it appears on GitHub: **Actions** tab → workflow **"Index deployment to Elasticsearch"**.

### 5b. Create the index

```bash
curl -X PUT "<YOUR-ES-ENDPOINT>/github-deployments" \
  -H "Authorization: ApiKey <YOUR-API-KEY>" \
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

### 5c. Add GitHub secrets

Secrets go on **this repo** (`ai-observability-agent-kubernetes`), not `DevOps-Projects`.

```bash
export REPO=girikgarg8/ai-observability-agent-kubernetes

gh secret set ES_ENDPOINT \
  --repo "$REPO" \
  --body "https://<your-es-endpoint>:443"

gh secret set ES_API_KEY \
  --repo "$REPO" \
  --body "<YOUR-API-KEY>"
```

### 5d. Test it

```bash
cd /Users/ggarg1/Personal/DevOps-Projects/ai-observability-agent-kubernetes

git commit --allow-empty -m "test: verify ES indexing"
git push origin main

gh run list --repo girikgarg8/ai-observability-agent-kubernetes --limit 3
# Expect: "Index deployment to Elasticsearch" → completed success

curl -s "<YOUR-ES-ENDPOINT>/github-deployments/_count" \
  -H "Authorization: ApiKey <YOUR-API-KEY>"
# Expect: {"count":1,...}
```

---

## Step 6 — ArgoCD

Watches the repo and auto-deploys every push to the cluster.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=180s

# Expose publicly (EKS provisions AWS ELB — wait 2-5 min for EXTERNAL-IP/hostname)
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc argocd-server -n argocd -w

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo ""

# Log in — use EXTERNAL-IP or ELB hostname from get svc above
argocd login <EXTERNAL-IP-OR-HOSTNAME> --username admin --password <PASSWORD> --insecure
```

### Create the Application

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

### First sync + reduce poll interval

```bash
argocd app sync online-boutique --force --prune

kubectl patch configmap argocd-cm -n argocd --type merge \
  -p '{"data": {"timeout.reconciliation": "30s"}}'
kubectl rollout restart deployment/argocd-repo-server -n argocd

kubectl get pods -n online-boutique
# Expect ~12 pods Running (wait 2-5 min if LoadBalancer / image pull delays)
```

---

## Step 7 — Agent Builder

Uses Elastic's native **Claude claude-sonnet-4-6** — no external LLM connector needed.

### 7a. Create the tools

Kibana → ☰ → **Search** → **Tools** → **Create tool**

**Tool 1 — `get_crash_logs`**

- Name: `get_crash_logs`
- Type: **Elasticsearch query**
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

> **EKS / OTel note:** use flat `k8s.namespace.name` fields and the `metrics-k8sclusterreceiver.otel-*` index — OOMKilled is a kubelet termination reason, not application stdout.

**Tool 2 — `get_deploy_history`**

- Name: `get_deploy_history`
- Type: **Elasticsearch query**
- Index: `github-deployments`
- ES|QL:

```esql
FROM github-deployments
| SORT timestamp DESC
| LIMIT 5
| KEEP timestamp, author, commit_sha, service, change, diff_url
```

### 7b. Create the agent

Kibana → ☰ → **Search** → **Agent Builder** → **Create agent**

- **Name:** `blame-the-deploy`
- **Model:** `Claude claude-sonnet-4-6`
- **Instructions:**

```
You are an SRE assistant. When asked why a service is crashing:
1. Use get_crash_logs to find OOMKill or memory events in logs-*
2. Use get_deploy_history to find the most recent GitHub deployment to that service
3. If the deploy happened shortly before the crash, that is the likely cause
4. Report: crash time, commit SHA, author, what changed, and how to fix it
Always cite the commit SHA and author name in your answer.
```

Add tools: `get_crash_logs`, `get_deploy_history` → **Save**.

---

## Step 8 — Run the Demo

### Break it

In `release/kubernetes-manifests.yaml`, find `paymentservice` Deployment (~line 631). Change memory:

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

Watch:

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
# Pod recovers in ~30 seconds (ArgoCD poll interval)
```

```
Is paymentservice healthy now?
```

---

## Step 9 — Cleanup (Stop Billable Resources)

Run when you are done with the demo or no longer need the stack. **EKS and Elastic Cloud are the main cost drivers.**

### 9a. Delete EKS cluster (AWS)

Removes worker nodes (EC2), control plane charges, and cluster LoadBalancers (ArgoCD, frontend ELB).

```bash
export AWS_REGION=ap-south-1          # match your setup
export CLUSTER_NAME=elastic-cloud-test

# Deletes cluster, node groups, and associated AWS resources (~10–15 min)
eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
```

**Verify in AWS Console:**

- **EKS → Clusters** — cluster gone
- **EC2 → Instances** — no leftover worker nodes
- **EC2 → Load Balancers** — no orphaned ELBs from `frontend-external` or `argocd-server`
- **EC2 → Volumes** — delete any unattached EBS volumes if present

### 9b. Delete Elastic Cloud deployment

Hosted Elasticsearch/Kibana is billed separately from AWS.

1. Go to [cloud.elastic.co](https://cloud.elastic.co)
2. Open deployment **`blame-the-deploy`** (or your deployment name)
3. **Manage deployment** → **Delete deployment**
4. Confirm deletion

**Also clean up (optional but recommended):**

- Kibana → **Settings** → **Security** → **API keys** — delete demo API keys
- Remove GitHub repo secrets if decommissioning entirely:

```bash
export REPO=girikgarg8/ai-observability-agent-kubernetes

gh secret delete ES_ENDPOINT --repo "$REPO"
gh secret delete ES_API_KEY --repo "$REPO"
```

### 9c. What costs nothing to leave running

| Resource | Cost |
|----------|------|
| GitHub repo + Actions secrets | Free (within GitHub limits) |
| Agent Builder config | Gone with Elastic deployment |
| Local kubeconfig / CLI tools | Free |

### Cleanup order

```
1. eksctl delete cluster     ← stops AWS EC2 + ELB charges
2. Delete Elastic deployment ← stops Elastic Cloud charges
3. Revoke API keys + GitHub secrets (optional)
```

> **Tip:** Managed node groups cannot scale to zero. To pause between demo takes without full teardown, either leave the cluster running (costs accrue) or delete and recreate from Step 1.

---

## Quick Reference

### Endpoints (fill in after setup)

| Service | URL |
|---------|-----|
| Online Boutique | `http://<frontend-external ELB hostname>` |
| ArgoCD | `https://<argocd-server ELB hostname>` → `admin` / `<password>` |
| Kibana | `https://<deployment-id>.kb.<region>.aws.cloud.es.io` |
| Elasticsearch | `https://<deployment-id>.<region>.aws.cloud.es.io:443` |

```bash
# Get LoadBalancer hostnames on EKS
kubectl get svc frontend-external -n online-boutique \
  -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}' && echo
kubectl get svc argocd-server -n argocd \
  -o jsonpath='https://{.status.loadBalancer.ingress[0].hostname}' && echo
```

### Useful commands

```bash
kubectl get pods -n online-boutique
kubectl get pods -n online-boutique -w
kubectl get pods -n opentelemetry-operator-system
argocd app get online-boutique
argocd app sync online-boutique --force
kubectl describe pod -n online-boutique <pod> | grep -A5 'Last State\|OOM'
```

### ES|QL — find OOMKilled containers

```esql
FROM metrics-k8sclusterreceiver.otel-*
| WHERE k8s.namespace.name == "online-boutique"
| WHERE k8s.container.status.last_terminated_reason == "OOMKilled"
| SORT @timestamp DESC
| LIMIT 20
| KEEP @timestamp, k8s.deployment.name, k8s.pod.name, k8s.container.status.last_terminated_reason
```

### ES|QL — find recent deploys

```esql
FROM github-deployments
| SORT timestamp DESC
| LIMIT 10
| KEEP timestamp, author, commit_sha, service, change
```

---

## AWS vs GCP cheat sheet

| Instructor (GCP) | This guide (AWS) |
|------------------|------------------|
| `gcloud container clusters create` | `eksctl create cluster` |
| `--zone us-central1-a` | `--zones ap-south-1a,ap-south-1b` (min 2 AZs) |
| `e2-standard-4` × 3 | `m7i-flex.xlarge` × 3 |
| `gcloud ... get-credentials` | `aws eks update-kubeconfig` |
| GCP Marketplace → Elastic | Elastic Cloud on **AWS** at elastic.co |
| GKE LoadBalancer | EKS → AWS ELB (2–5 min to provision) |

---

## Changelog

| Date | Change |
|------|--------|
| 2026-06-15 | Initial AWS runbook; Step 1 EKS with `m7i-flex.xlarge` × 3, 2-AZ fix |
| 2026-06-15 | Step 5: GitHub-hosted runner docs + `index-deploy.yml` setup; skip upstream CI workflows |
| 2026-06-15 | Option C: standalone repo layout — `release/` + workflow at root (matches instructor) |
| 2026-06-15 | Step 9: full cleanup guide (EKS + Elastic Cloud + secrets) |
