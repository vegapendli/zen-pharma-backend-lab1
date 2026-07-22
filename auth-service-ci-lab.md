# Lab: auth-service CI/CD + GitOps — End to End

You will wire up a complete delivery pipeline for `auth-service` — from a git push on your laptop to a running pod on Kubernetes, with ArgoCD ensuring the cluster always matches git.

```
git push to develop
      │
      ▼
GitHub Actions full CI
  ├─ Maven tests + CodeQL + Semgrep
  ├─ Docker build → Trivy scan
  ├─ Push to ECR  (sha-xxxxxxx)
  └─ Cosign sign
      │
      ├──► deploy-dev job ──► git commit to zen-gitops-lab1
      │                         envs/dev/values-auth-service.yaml
      │                                  │
      │                                  ▼
      │                         ArgoCD detects commit
      │                         helm template → kubectl apply
      │                         DEV pod rolling to sha-xxxxxxx ✓
      │
      └──► open-qa-pr job ──► PR opened in zen-gitops-lab1
                                envs/qa/values-auth-service.yaml
```

---

## What your instructor has already set up

| Resource | Details |
|---|---|
| EKS cluster | `pharma-dev` — you have `kubectl` access |
| ECR repository | `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/auth-service` |
| IAM role | `pharma-dev-github-actions-role` (GitHub OIDC — no static keys needed) |
| RDS PostgreSQL | `<DB_HOST>` — single shared RDS instance used by dev, qa, and prod |
| Kubernetes Secrets | `db-credentials` and `jwt-secret` pre-created in the `dev`, `qa`, and `prod` namespaces |
| ArgoCD | Installed on EKS, apps `auth-service-dev`, `auth-service-qa`, and `auth-service-prod` created pointing at `https://github.com/<YOUR-USERNAME>/zen-gitops-lab1.git` |
| GitHub Actions `production` environment | Pre-configured with required reviewers for the prod approval gate |

Your instructor will give you:
- `<ACCOUNT_ID>` — 12-digit AWS account number
- `<DB_HOST>` — RDS endpoint (same for dev, qa, and prod)
- kubeconfig file for `kubectl` access
- ArgoCD UI URL and login

---

## Instructor Setup — OIDC and IAM role (`pharma-dev-github-actions-role`)

The role `pharma-dev-github-actions-role` already exists in account `516209541629`. Before each lab, update its trust policy to add each student's fork to the allowed `sub` list.

### Step 1 — Add each student's fork to the trust policy

Replace `<STUDENT-USERNAME>` with the student's GitHub username. Run once per student.

```bash
aws iam update-assume-role-policy \
  --role-name pharma-dev-github-actions-role \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Federated":"arn:aws:iam::516209541629:oidc-provider/token.actions.githubusercontent.com"},"Action":"sts:AssumeRoleWithWebIdentity","Condition":{"StringEquals":{"token.actions.githubusercontent.com:aud":"sts.amazonaws.com"},"StringLike":{"token.actions.githubusercontent.com:sub":["repo:DPP-2026/zen-pharma-frontend:ref:refs/heads/main","repo:DPP-2026/zen-pharma-frontend:ref:refs/heads/develop","repo:DPP-2026/zen-pharma-backend:ref:refs/heads/main","repo:DPP-2026/zen-pharma-backend:ref:refs/heads/develop","repo:<STUDENT-USERNAME>/zen-pharma-backend-lab1:ref:refs/heads/develop"]}}}]}'
```

> Each time you add a new student, include all previous entries in the `sub` list — `update-assume-role-policy` replaces the entire policy, not appends.

### Step 2 — Verify

```bash
aws iam get-role \
  --role-name pharma-dev-github-actions-role \
  --query 'Role.AssumeRolePolicyDocument' \
  --output json
```

Confirm the student's repo appears in the `sub` list. GitHub Actions will assume `arn:aws:iam::516209541629:role/pharma-dev-github-actions-role` via the `AWS_ACCOUNT_ID` repository secret (value: `516209541629`).

---

## Instructor Cluster Setup

Run these steps once on the EKS cluster before students start. All commands are manual — no scripts required.

### Step 1 — Install cluster prerequisites

Installs NGINX Ingress, ArgoCD, External Secrets Operator, and metrics-server via Helm.

Requires `kubectl`, `helm`, and the AWS CLI on your laptop.

#### 1.1 — Configure kubectl

```bash
export CLUSTER_NAME=pharma-dev-cluster
export AWS_REGION=us-east-1

aws eks update-kubeconfig --region "$AWS_REGION" --name "$CLUSTER_NAME"
kubectl config current-context
kubectl get nodes
```

#### 1.2 — Add Helm repositories

```bash
helm repo add ingress-nginx    https://kubernetes.github.io/ingress-nginx --force-update
helm repo add external-secrets https://charts.external-secrets.io         --force-update
helm repo add argo             https://argoproj.github.io/argo-helm       --force-update
helm repo add metrics-server   https://kubernetes-sigs.github.io/metrics-server/ --force-update
helm repo update
```

#### 1.3 — Install NGINX Ingress Controller

Creates an AWS Network Load Balancer that routes external HTTP/S traffic into the cluster.

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb" \
  --set controller.replicaCount=2 \
  --wait --timeout 5m

kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}'
```

Save the NLB hostname printed above — it is the application entry point.

#### 1.4 — Install ArgoCD

```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --wait --timeout 10m

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Save the ArgoCD admin password printed above — you'll need it for the UI.

- Username: `admin`
- UI access (port-forward): `kubectl port-forward svc/argocd-server -n argocd 8080:443` → open `https://localhost:8080`

#### 1.5 — Install External Secrets Operator

```bash
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true \
  --wait --timeout 5m
```

#### 1.6 — Install metrics-server

Required for HPA and `kubectl top pods/nodes`.

```bash
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args="{--kubelet-insecure-tls}" \
  --wait --timeout 5m
```

#### 1.7 — Verify installation

```bash
kubectl get pods -n ingress-nginx
kubectl get pods -n argocd
kubectl get pods -n external-secrets
kubectl get deployment metrics-server -n kube-system

# Metrics API may take up to 60s on first install
kubectl top nodes
```

All four components should show running pods. If `kubectl top nodes` fails immediately, wait a minute and retry.

### Step 2 — Create the pharma AppProject

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: pharma
  namespace: argocd
spec:
  description: Pharma microservices project
  sourceRepos:
    - "https://github.com/DPP-2026/zen-gitops-lab1.git"
  destinations:
    - server: https://kubernetes.default.svc
      namespace: dev
    - server: https://kubernetes.default.svc
      namespace: qa
    - server: https://kubernetes.default.svc
      namespace: prod
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
EOF
```

### Step 3 — Create the auth-service ArgoCD Application

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auth-service-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma
  source:
    repoURL: https://github.com/DPP-2026/zen-gitops-lab1.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/dev/values-auth-service.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
EOF
```

Create QA ARgo application
```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auth-service-qa
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma
  source:
    repoURL: https://github.com/DPP-2026/zen-gitops-lab1.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/qa/values-auth-service.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: qa
  syncPolicy:
    automated:
      prune: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
EOF
```

### Step 4 — Set up External Secrets

Wires up AWS Secrets Manager → Kubernetes Secrets (`db-credentials`, `jwt-secret`) in the `dev` namespace via IRSA. The IAM role `pharma-dev-eso-role` is created by Terraform in Stage 1 — no static AWS keys are stored in the cluster.

**Why External Secrets Operator instead of native Kubernetes Secrets:**

- **Single source of truth** — the real secret value lives in AWS Secrets Manager, not in the cluster or in git. Kubernetes Secrets alone have no upstream source; whoever last ran `kubectl apply` (or whatever's checked into git) is the source of truth, which invites drift and stale values.
- **No static credentials in the cluster** — ESO authenticates to AWS via IRSA (the `pharma-dev-eso-role` IAM role), so no long-lived AWS access keys are stored as Kubernetes Secrets or env vars. Native Kubernetes Secrets have no equivalent mechanism for pulling from an external provider.
- **Automatic rotation** — ESO re-fetches from AWS Secrets Manager on a `refreshInterval` (1h in this lab) and updates the Kubernetes Secret in place. Rotate the value in Secrets Manager and it propagates without a manual `kubectl apply` or a redeploy.
- **Centralized audit trail** — every read of a secret value goes through AWS Secrets Manager, which is logged in CloudTrail. Kubernetes Secrets are only base64-encoded and give no equivalent access log for who read what, when.
- **Consistent across environments** — dev, qa, and prod all sync from the same AWS Secrets Manager paths through the same ClusterSecretStore mechanism, so promoting a change is a Secrets Manager update, not a per-namespace `kubectl` command.
- **Kubernetes-native consumption stays the same** — Pods still mount the result as an ordinary Kubernetes Secret, so no application code changes; ESO only replaces how that Secret's contents get populated and kept fresh.

#### 4.1 — Set variables

```bash
export ENV=dev
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=516209541629
export ESO_ROLE_NAME=pharma-dev-eso-role
export ESO_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ESO_ROLE_NAME}"
```

Secrets will sync from these AWS Secrets Manager paths:

- `/pharma/dev/db-credentials` → Kubernetes Secret `db-credentials`
- `/pharma/dev/jwt-secret` → Kubernetes Secret `jwt-secret`

#### 4.2 — Create the target namespace

```bash
kubectl create namespace "$ENV" --dry-run=client -o yaml | kubectl apply -f -
```

#### 4.3 — Annotate the ESO service account (IRSA)

Tells EKS to inject temporary AWS credentials into External Secrets Operator pods so they can call `secretsmanager:GetSecretValue`.

```bash
kubectl annotate serviceaccount external-secrets \
  --namespace external-secrets \
  "eks.amazonaws.com/role-arn=$ESO_ROLE_ARN" \
  --overwrite

kubectl rollout restart deployment/external-secrets -n external-secrets
kubectl rollout status deployment/external-secrets -n external-secrets --timeout=120s
```

#### 4.4 — Create the ClusterSecretStore

```bash
kubectl wait --for=condition=established \
  crd/clustersecretstores.external-secrets.io \
  crd/externalsecrets.external-secrets.io \
  --timeout=60s

kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ${AWS_REGION}
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
EOF
```

#### 4.5 — Create ExternalSecrets in the dev namespace

```bash
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: ${ENV}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: /pharma/${ENV}/db-credentials
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /pharma/${ENV}/db-credentials
        property: password
    - secretKey: SPRING_DATASOURCE_USERNAME
      remoteRef:
        key: /pharma/${ENV}/db-credentials
        property: username
    - secretKey: SPRING_DATASOURCE_PASSWORD
      remoteRef:
        key: /pharma/${ENV}/db-credentials
        property: password
EOF

kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: jwt-secret
  namespace: ${ENV}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: jwt-secret
    creationPolicy: Owner
  data:
    - secretKey: JWT_SECRET
      remoteRef:
        key: /pharma/${ENV}/jwt-secret
        property: secret
EOF
```

#### 4.6 — Verify secrets synced

```bash
kubectl get externalsecret -n dev
kubectl get secret db-credentials jwt-secret -n dev
```

Both ExternalSecrets should show `Ready` with reason `SecretSynced`. If not, check:

1. Secrets Manager paths exist with the expected JSON keys (`username`/`password` for db, `secret` for jwt)
2. IAM role `pharma-dev-eso-role` has `secretsmanager:GetSecretValue` on those paths
3. EKS OIDC provider is configured for IRSA

```bash
kubectl describe externalsecret db-credentials -n dev
```

### Step 5 — Verify dev

```bash
kubectl get applications -n argocd
kubectl get externalsecret -n dev
```

`auth-service-dev` should show `Synced` once a student's first CI build pushes an image and updates `envs/dev/values-auth-service.yaml`.

---

### Step 6 — Create the QA namespace and External Secrets

> **Do this before creating the ArgoCD application.** ArgoCD will sync as soon as the application exists and the gitops values file is present. If the Kubernetes Secrets are not already in the namespace when the first sync fires, Spring Boot cannot connect to the database and the pod enters `CrashLoopBackOff`.

The DB credentials for QA point to the same Secrets Manager path as dev (`/pharma/dev/db-credentials`). QA shares the dev RDS instance — this is intentional and is reflected in the Terraform in the `zen-infra` repo. No separate `/pharma/qa/...` secret paths exist.

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=516209541629

kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: qa
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: /pharma/dev/db-credentials
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /pharma/dev/db-credentials
        property: password
    - secretKey: SPRING_DATASOURCE_USERNAME
      remoteRef:
        key: /pharma/dev/db-credentials
        property: username
    - secretKey: SPRING_DATASOURCE_PASSWORD
      remoteRef:
        key: /pharma/dev/db-credentials
        property: password
EOF

kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: jwt-secret
  namespace: qa
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: jwt-secret
    creationPolicy: Owner
  data:
    - secretKey: JWT_SECRET
      remoteRef:
        key: /pharma/dev/jwt-secret
        property: secret
EOF
```

Wait for both External Secrets to report `SecretSynced` before proceeding:

```bash
kubectl get externalsecret -n qa
# NAME             STORE                REFRESH INTERVAL   STATUS   READY
# db-credentials   aws-secrets-manager  1h                 Ready    True
# jwt-secret       aws-secrets-manager  1h                 Ready    True

kubectl get secret db-credentials jwt-secret -n qa
```

Both Kubernetes Secrets must exist before you create the ArgoCD application. If an ExternalSecret shows `SecretSyncError`, check:

1. The `/pharma/dev/db-credentials` and `/pharma/dev/jwt-secret` paths exist in AWS Secrets Manager (created by Terraform in `zen-infra`)
2. IAM role `pharma-dev-eso-role` has `secretsmanager:GetSecretValue` on those paths
3. The ESO service account annotation is correct: `kubectl get sa external-secrets -n external-secrets -o yaml | grep role-arn`

### Step 7 — Create the auth-service-qa ArgoCD Application

Only run this after Step 6 confirms both secrets are `Ready`.

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auth-service-qa
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma
  source:
    repoURL: https://github.com/DPP-2026/zen-gitops-lab1.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/qa/values-auth-service.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: qa
  syncPolicy:
    automated:
      prune: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
EOF
```

ArgoCD will attempt an initial sync immediately. Because the QA values file already exists in `zen-gitops-lab1` (committed by the student in Part 3), the app will start syncing at once. The pod will start successfully because `db-credentials` and `jwt-secret` are already present in the `qa` namespace.

### Step 8 — Create the prod namespace and External Secrets

> Same principle as QA: secrets must exist before the ArgoCD application is created. For prod, the ArgoCD app uses manual sync, so there is no risk of an immediate uncontrolled deploy — but the namespace and secrets still need to be ready before the student runs `argocd app sync` in Part 10.

Prod shares the same RDS instance and credentials as dev and qa. The ExternalSecrets reference the same `/pharma/dev/...` paths in Secrets Manager, provisioned by Terraform in `zen-infra`.

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=516209541629

kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: /pharma/dev/db-credentials
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /pharma/dev/db-credentials
        property: password
    - secretKey: SPRING_DATASOURCE_USERNAME
      remoteRef:
        key: /pharma/dev/db-credentials
        property: username
    - secretKey: SPRING_DATASOURCE_PASSWORD
      remoteRef:
        key: /pharma/dev/db-credentials
        property: password
EOF

kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: jwt-secret
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: jwt-secret
    creationPolicy: Owner
  data:
    - secretKey: JWT_SECRET
      remoteRef:
        key: /pharma/dev/jwt-secret
        property: secret
EOF

kubectl get externalsecret -n prod
kubectl get secret db-credentials jwt-secret -n prod
```

Both Kubernetes Secrets must be `Ready` before the student reaches Part 10.

### Step 9 — Create the auth-service-prod ArgoCD Application

Prod uses **manual sync** — ArgoCD detects drift but will not auto-apply. A human must approve via the CLI or UI before any change lands in production.

The `prod` namespace is already in the AppProject destinations (Step 2), so create the application directly:

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auth-service-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma
  source:
    repoURL: https://github.com/DPP-2026/zen-gitops-lab1.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/prod/values-auth-service.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 3
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 5m
EOF
```

Note: no `automated:` block — prod never syncs without an explicit `argocd app sync auth-service-prod`.

### Step 10 — Configure the GitHub Actions `production` environment

The `promote-prod.yml` workflow requires a GitHub environment named `production` with required reviewers. This creates the manual approval gate in CI.

In each student's `zen-pharma-backend-lab1` fork (or in the upstream template repo):

1. Go to **Settings** → **Environments** → **New environment**
2. Name: `production`, click **Configure environment**
3. Enable **Required reviewers** — add yourself (instructor) or a teaching assistant
4. Enable **Prevent self-review** so students cannot approve their own production deployments
5. Click **Save protection rules**

Verify by navigating to **Settings** → **Environments** → `production` and confirming the reviewer list.

### Step 11 — Verify all ArgoCD applications

```bash
kubectl get applications -n argocd
```

Expected output:

```
NAME               SYNC STATUS   HEALTH STATUS
auth-service-dev   Synced        Healthy
auth-service-qa    OutOfSync     Missing
auth-service-prod  OutOfSync     Missing
```

QA and Prod show `OutOfSync / Missing` until students push their values files and trigger their first CI run. That is expected.

---

## Prerequisites

Install these on your laptop before the lab:

- `git` — `git --version`
- `kubectl` — `kubectl version --client`
- `argocd` CLI — `brew install argocd` / [Windows install](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- AWS CLI — `aws --version` (for ECR verification in Part 4)

Configure your kubeconfig (instructor will provide the command):

```bash
aws eks update-kubeconfig --region us-east-1 --name pharma-dev
kubectl get nodes   # should list cluster nodes
```

---

## Part 1 — Fork both repos

Do this first. You need both repos forked before configuring anything else.

### Step 1.1 — Fork zen-pharma-backend-lab1

1. Go to `https://github.com/DPP-2026/zen-pharma-backend-lab1`
2. Click **Fork** → select your account → **Create fork**
3. Clone your fork:

```bash
git clone https://github.com/<YOUR-USERNAME>/zen-pharma-backend-lab1.git
cd zen-pharma-backend-lab1
```

Your fork contains:

```
zen-pharma-backend-lab1/
├── auth-service/
│   ├── src/
│   ├── pom.xml
│   ├── checkstyle.xml
│   └── Dockerfile
└── .github/
    └── workflows/
        ├── _java-build.yml         ← reusable: full build + ECR push
        └── ci-auth-service.yml     ← trigger: develop merges
```

You do not create any workflow files — they are already there.

### Step 1.2 — Fork zen-gitops-lab1

1. Go to `https://github.com/DPP-2026/zen-gitops-lab1`
2. Click **Fork** → select your account → **Create fork**
3. Clone your fork in a separate directory:

```bash
cd ..
git clone https://github.com/<YOUR-USERNAME>/zen-gitops-lab1.git
cd zen-gitops-lab1
```

---

## Part 2 — Configure GitHub Actions

All configuration below is in GitHub Settings — no code changes.

### Step 2.1 — Create a Personal Access Token for GitOps writes

The CI pipeline commits to your `zen-gitops-lab1` repo after every build. It needs a token to do that.

1. GitHub → your profile icon → **Settings**
2. Left sidebar → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
3. Click **Generate new token**
4. Fill in:
   - **Token name**: `zen-pharma-gitops-token`
   - **Expiration**: 90 days
   - **Resource owner**: your account
   - **Repository access**: select only `zen-gitops-lab1` (your fork)
5. Under **Permissions**, set:
   - **Contents**: Read and write
   - **Pull requests**: Read and write
6. Click **Generate token** — **copy it now**, you will not see it again

### Step 2.2 — Add GitHub Secrets to zen-pharma-backend-lab1

Go to your `zen-pharma-backend-lab1` fork → **Settings** → **Secrets and variables** → **Actions** → **Secrets** tab.

Click **New repository secret** and add:

| Secret name | Value |
|---|---|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID (from instructor) |
| `GITOPS_TOKEN` | The PAT you created in Step 2.1 |

### Step 2.3 — Add a GitHub Variable

Still in **Secrets and variables** → click the **Variables** tab → **New repository variable**:

| Variable name | Value |
|---|---|
| `GITOPS_REPO` | `<YOUR-USERNAME>/zen-gitops-lab1` |

---

## Part 3 — Update GitOps values files

The CI pipeline writes the new image tag into these files after every build. ArgoCD reads them to know what to deploy.

Both files already exist in your `zen-gitops-lab1` fork. You only need to fill in the two placeholder values your instructor gave you.

### Step 3.1 — Update envs/dev/values-auth-service.yaml

```bash
cd zen-gitops-lab1
```

Open `envs/dev/values-auth-service.yaml` and replace:

- `<ACCOUNT_ID>` → your 12-digit AWS account ID
- `<DB_HOST>` → the RDS dev endpoint from your instructor

### Step 3.2 — Update envs/qa/values-auth-service.yaml

Open `envs/qa/values-auth-service.yaml` and replace:

- `<ACCOUNT_ID>` → your 12-digit AWS account ID
- `<DB_HOST>` → the same RDS endpoint your instructor gave you for dev — all environments share the same database

### Step 3.3 — Commit and push

```bash
git add envs/dev/values-auth-service.yaml envs/qa/values-auth-service.yaml
git commit -m "feat(auth-service): add dev and qa values files"
git push origin main
```

ArgoCD will now detect these files within ~3 minutes. Since there is no image yet (`tag: latest`), the pod may fail to start — that is expected. The next part fixes it.

---

## Part 4 — Trigger the CI pipeline

### Step 4.1 — Create the develop branch

```bash
cd zen-pharma-backend-lab1
git checkout -b develop
git push -u origin develop
```

### Step 4.2 — Push to develop and watch the full CI

```bash
git checkout develop
echo "" >> auth-service/pom.xml
git add auth-service/pom.xml
git commit -m "test: trigger full CI"
git push origin develop
```

Go to **Actions** → **CI/CD — auth-service**. You will see 3 jobs:

---

**Job 1 — Build & Security Gates** (~12–15 min)

Watch the steps scroll by:

```
✓ Checkout
✓ Set image tag  →  sha-abc1234
✓ Setup Java 17 (Temurin)
✓ Start PostgreSQL container
✓ Initialize CodeQL
✓ Maven verify  (tests pass)
✓ Upload JaCoCo coverage report
✓ CodeQL analyze
✓ Semgrep SAST
✓ Configure AWS credentials (OIDC — no static keys)
✓ Login to Amazon ECR
✓ Build Docker image
✓ Trivy image scan  (no HIGH/CRITICAL CVEs)
✓ Upload Trivy SARIF → GitHub Security tab
✓ Push image to ECR  (sha-abc1234)
✓ Install Cosign
✓ Cosign sign image  (keyless → Rekor transparency log)
```

---

**Job 2 — Deploy DEV** (starts immediately after Job 1)

Watch the logs:

```
Run yq e ".image.tag = \"sha-abc1234\"" -i "_gitops/envs/dev/values-auth-service.yaml"
[main xyz5678] ci(dev): update auth-service → sha-abc1234
```

Then open your `zen-gitops-lab1` fork on GitHub. You will see a new commit on `main`:
```
ci(dev): update auth-service → sha-abc1234
```

---

**Job 3 — Open PR QA Promotion** (starts after Job 2)

Open your `zen-gitops-lab1` fork → **Pull requests**. You will see:
```
promote(qa): auth-service → sha-abc1234
```

Leave this PR open for now — you will merge it in Part 6.

---

### Step 4.3 — Verify the image is in ECR

```bash
aws ecr describe-images \
  --repository-name auth-service \
  --region us-east-1 \
  --query 'sort_by(imageDetails, &imagePushedAt)[-1].imageTags' \
  --output table
```

You should see your `sha-abc1234` tag in the output.

---

## Part 5 — Watch ArgoCD deploy to DEV

The moment Job 2 committed to `zen-gitops-lab1`, ArgoCD began a sync. It polls the repo every ~3 minutes.

### Step 5.1 — Check the ArgoCD app

```bash
argocd login <ARGOCD-URL> --username admin --password <PASSWORD>
argocd app get auth-service-dev
```

Or open the ArgoCD UI in your browser (URL from instructor).

You should see:
```
Sync Status:   Synced
Health Status: Progressing  →  Healthy
```

### Step 5.2 — Watch the pod come up

```bash
kubectl get pods -n dev -w
```

Expected output:
```
NAME                           READY   STATUS              RESTARTS   AGE
auth-service-7d9f8c-xxxx       0/1     ContainerCreating   0          5s
auth-service-7d9f8c-xxxx       0/1     Running             0          20s
auth-service-7d9f8c-xxxx       1/1     Running             0          55s
```

The pod reaches `1/1 Ready` once the readiness probe passes. Spring Boot takes ~30 seconds to start.

Press **Ctrl+C** to exit the watch.

### Step 5.3 — Confirm which image is running

```bash
kubectl get pods -n dev -l app.kubernetes.io/name=auth-service \
  -o jsonpath='{.items[0].spec.containers[0].image}'
```

Expected: `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-abc1234`

### Step 5.4 — Hit the health endpoint

```bash
kubectl port-forward -n dev deploy/auth-service 8081:8081 &
curl http://localhost:8081/actuator/health
```

Expected: `{"status":"UP"}`

```bash
kill %1   # stop the port-forward
```

---

## Part 6 — ArgoCD self-heal demo

This is the key GitOps concept: **the cluster always converges back to what git says**.

If someone makes a manual change on the cluster — deletes a deployment, scales replicas, patches a configmap — ArgoCD detects the drift and reverts it.

### Step 6.1 — Enable selfHeal on the ArgoCD app

The app was created with `selfHeal` disabled. Enable it now:

```bash
argocd app set auth-service-dev --self-heal
```

Verify:
```bash
argocd app get auth-service-dev | grep -A5 "Sync Policy"
# Auto-Sync: enabled (Prune=true, SelfHeal=true)
```

### Step 6.2 — Delete the deployment

```bash
kubectl delete deployment auth-service -n dev
```

Watch what happens:

```bash
kubectl get pods -n dev -w
```

The running pod terminates:
```
auth-service-7d9f8c-xxxx   1/1   Running       0   15m
auth-service-7d9f8c-xxxx   1/1   Terminating   0   15m
auth-service-7d9f8c-xxxx   0/1   Terminating   0   15m
```

### Step 6.3 — Watch ArgoCD detect and fix it

```bash
watch argocd app get auth-service-dev
```

Within ~3 minutes:

1. ArgoCD detects the deployment is missing → marks app `OutOfSync / Missing`
2. ArgoCD applies git state back to cluster → `Syncing`
3. Deployment recreated → pod starts → `Synced / Healthy`

```
auth-service-8e2a1f-yyyy   0/1   ContainerCreating   0   5s
auth-service-8e2a1f-yyyy   1/1   Running             0   50s
```

Press **Ctrl+C** to exit the watch.

**The key insight:** You deleted a Kubernetes resource. ArgoCD brought it back — automatically, without anyone intervening, without any alert firing. Git is the single source of truth. The cluster must match git, always.

### Step 6.4 — Try changing replica count directly

```bash
kubectl scale deployment auth-service -n dev --replicas=3
```

Watch it get reverted:

```bash
watch kubectl get deployment auth-service -n dev -o wide
```

Within 3 minutes, replicas go from `3` → `1` (what the values file says).

---

## Part 7 — Promote to QA

The QA promotion PR was opened automatically by Job 3 in Part 4. Before merging it, confirm the QA secrets are already in the cluster — ArgoCD will fire a sync the moment the merge lands, and if the secrets are missing the pod will enter `CrashLoopBackOff`.

### Step 7.1 — Confirm QA External Secrets are ready

```bash
kubectl get externalsecret -n qa
```

Both rows must show `READY = True` and `STATUS = SecretSynced`:

```
NAME             STORE                REFRESH INTERVAL   STATUS         READY
db-credentials   aws-secrets-manager  1h                 SecretSynced   True
jwt-secret       aws-secrets-manager  1h                 SecretSynced   True
```

```bash
kubectl get secret db-credentials jwt-secret -n qa
```

If either secret is missing, stop and notify your instructor. Do not merge the PR until both secrets exist. The instructor must run the External Secrets setup in Step 6 of the Instructor Cluster Setup section.

### Step 7.2 — Merge the QA promotion PR

1. Open your `zen-gitops-lab1` fork on GitHub
2. Navigate to **Pull requests**
3. Open the PR titled `promote(qa): auth-service → sha-xxxxxxx`
4. Click **Merge pull request** → **Confirm merge**

ArgoCD polls the `main` branch every ~3 minutes and will detect the merge automatically.

### Step 7.3 — Watch ArgoCD sync QA

```bash
argocd app get auth-service-qa
```

The sync status should move from `OutOfSync` → `Syncing` → `Synced / Healthy`.

```bash
kubectl get pods -n qa -w
```

Expected output:

```
NAME                           READY   STATUS              RESTARTS   AGE
auth-service-7d9f8c-xxxx       0/1     ContainerCreating   0          5s
auth-service-7d9f8c-xxxx       0/1     Running             0          20s
auth-service-7d9f8c-xxxx       1/1     Running             0          55s
```

Press **Ctrl+C** to exit the watch.

### Step 7.4 — Confirm the QA image

```bash
kubectl get pods -n qa -l app.kubernetes.io/name=auth-service \
  -o jsonpath='{.items[0].spec.containers[0].image}'
```

Expected: `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-xxxxxxx`

### Step 7.5 — Hit the QA health endpoint

```bash
kubectl port-forward -n qa deploy/auth-service 8082:8081 &
curl http://localhost:8082/actuator/health
kill %1
```

Expected: `{"status":"UP"}`

---

## Part 8 — Set up the Prod CD pipeline

### Step 8.1 — Add the prod values file to your gitops repo

In your `zen-gitops-lab1` fork, open `envs/prod/values-auth-service.yaml` and replace:

- `<ACCOUNT_ID>` → your 12-digit AWS account ID
- `<DB_HOST>` → the same RDS endpoint used for dev and qa — all environments share the same database

```bash
cd zen-gitops-lab1
git add envs/prod/values-auth-service.yaml
git commit -m "feat(auth-service): add prod values file"
git push origin main
```

### Step 8.2 — Create the `production` GitHub Actions environment

The `promote-prod.yml` workflow uses a GitHub environment called `production`. You need to create it in your `zen-pharma-backend-lab1` fork with a required reviewer — this is the manual approval gate before any prod deployment.

1. Go to your `zen-pharma-backend-lab1` fork → **Settings** → **Environments** → **New environment**
2. Name: `production`, click **Configure environment**
3. Enable **Required reviewers** — add a classmate, your instructor, or a teaching assistant
4. Click **Save protection rules**

The prod promotion job will now pause and notify your reviewer before it writes to the gitops repo.

### Step 8.3 — Confirm the `promote-prod.yml` workflow is present

```bash
cd zen-pharma-backend-lab1
cat .github/workflows/promote-prod.yml
```

This workflow triggers on a merge to `main`, verifies the image tag exists in ECR, waits for the `production` environment approval, then commits the new tag to `envs/prod/values-auth-service.yaml` in your gitops repo.

---

## Part 9 — Trigger and approve a prod promotion

### Step 9.1 — Merge develop into main

```bash
cd zen-pharma-backend-lab1
git checkout main
git merge develop --no-ff -m "release: promote auth-service to prod"
git push origin main
```

Go to **Actions** → **Promote to Prod**. The workflow will:

1. Verify the image tag (`sha-xxxxxxx`) exists in ECR
2. Pause at the `production` environment gate — a yellow **Review deployments** banner appears

### Step 9.2 — Approve the deployment

The reviewer you added in Step 8.2 must:

1. Click on the running **Promote to Prod** workflow run
2. Click **Review deployments**
3. Select the `production` environment checkbox
4. Click **Approve and deploy**

Once approved, the workflow completes:

```
✓ Checkout zen-gitops-lab1
✓ Update envs/prod/values-auth-service.yaml  →  sha-xxxxxxx
✓ Commit: ci(prod): update auth-service → sha-xxxxxxx
✓ Push to zen-gitops-lab1 main
```

### Step 9.3 — Review the diff before syncing prod

Prod is configured with manual sync. ArgoCD detects the gitops change but waits for your explicit command.

```bash
argocd app get auth-service-prod
# Sync Status:   OutOfSync
# Health Status: Missing  (no pod yet)
```

Inspect exactly what will be applied before you commit:

```bash
argocd app diff auth-service-prod
```

### Step 9.4 — Sync prod

```bash
argocd app sync auth-service-prod --prune
```

Watch it complete:

```bash
argocd app wait auth-service-prod --health --timeout 300
```

---

## Part 10 — Verify Prod and understand manual sync

### Step 10.1 — Check the prod pod

```bash
kubectl get pods -n prod -w
```

Wait for `1/1 Running`, then press **Ctrl+C**.

### Step 10.2 — Confirm the prod image

```bash
kubectl get pods -n prod -l app.kubernetes.io/name=auth-service \
  -o jsonpath='{.items[0].spec.containers[0].image}'
```

Expected: `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-xxxxxxx`

### Step 10.3 — Hit the prod health endpoint

```bash
kubectl port-forward -n prod deploy/auth-service 8083:8081 &
curl http://localhost:8083/actuator/health
kill %1
```

Expected: `{"status":"UP"}`

### Step 10.4 — Verify prod does NOT self-heal

Unlike DEV, `auth-service-prod` has no `automated:` policy. Manual changes persist until you explicitly sync.

```bash
kubectl scale deployment auth-service -n prod --replicas=3

# ArgoCD shows OutOfSync but does not revert automatically
argocd app get auth-service-prod
# Sync Status:   OutOfSync
# Health Status: Healthy  (3 replicas are healthy)
```

This is intentional for production: an on-call engineer can scale up instantly in an incident without waiting for git. When the incident is resolved, reconcile with git and sync:

```bash
argocd app sync auth-service-prod --prune
```

**The key difference from DEV:** DEV self-heals — no manual changes survive. Prod does not self-heal — humans decide when to sync, giving time to validate or roll back a change before it goes live.

---

## Complete flow — what you just built

```
your laptop
    │
    │  git push → develop
    ▼
GitHub Actions full CI  (ci-auth-service.yml)
    mvn verify → CodeQL → Semgrep
    Docker build → Trivy scan
    ECR push → sha-abc1234
    Cosign sign → Rekor log
    │
    ├──► deploy-dev job
    │         zen-gitops-lab1: envs/dev/values-auth-service.yaml
    │              │
    │              ▼
    │         ArgoCD auto-sync DEV  (selfHeal=true)
    │         helm template → kubectl apply
    │         DEV pod: sha-abc1234 ✓
    │
    └──► open-qa-pr job
              zen-gitops-lab1 PR: envs/qa/values-auth-service.yaml
              │
              │  (QA team reviews and merges PR)
              ▼
         ArgoCD auto-sync QA  (automated, no selfHeal)
         same image → qa namespace ✓

    │  git merge develop → main
    ▼
GitHub Actions  (promote-prod.yml)
    Verify sha-abc1234 exists in ECR
    │
    │  ⏸  production environment gate
    │     reviewer clicks Approve and deploy
    │
    ▼
    zen-gitops-lab1: envs/prod/values-auth-service.yaml
         │
         │  (operator reviews diff, runs argocd app sync)
         ▼
    ArgoCD MANUAL sync PROD  (no automated, no selfHeal)
    same image → prod namespace ✓
```

CI never talks to Kubernetes. CI talks to git. Kubernetes gets its orders from git.

| Environment | ArgoCD sync | Self-heal | Gate |
|---|---|---|---|
| dev | Automated | Yes — reverts manual changes | None (any develop push) |
| qa | Automated | No | GitOps PR review and merge |
| prod | Manual | No | GitHub environment approval + explicit `argocd app sync` |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Job fails: `Error assuming role` | `AWS_ACCOUNT_ID` secret wrong or OIDC trust policy mismatch | Confirm the 12-digit account ID and that the repo name in the trust policy matches your fork exactly |
| Job fails: `denied: Your authorization token has expired` | ECR login failed | Double-check `AWS_ACCOUNT_ID` secret |
| `deploy-dev` fails: `Authentication failed for https://github.com/...` | `GITOPS_TOKEN` expired or scoped to wrong repo | Regenerate PAT with Contents + Pull requests write on your zen-gitops-lab1 fork |
| Maven tests fail: `Connection refused` to DB | PostgreSQL not ready or `needs-database` flag missing | Check `ci-auth-service.yml` has `needs-database: true` passed to the reusable workflow |
| Pod stuck in `CrashLoopBackOff` | Spring Boot cannot connect to RDS | `kubectl logs -n dev deploy/auth-service` — check `DB_HOST` in your values file |
| Pod stuck in `ImagePullBackOff` | ECR pull auth issue | Check the service account's IAM role annotation; verify image tag exists in ECR |
| ArgoCD shows `OutOfSync` but not healing | `selfHeal` not enabled | Run `argocd app set auth-service-dev --self-heal` |
| ArgoCD app still pointing at `DPP-2026/zen-gitops-lab1` | App repoURL not updated to your fork | Notify instructor — the app needs to be re-pointed to `https://github.com/<YOUR-USERNAME>/zen-gitops-lab1.git` |
| `argocd: command not found` | argocd CLI not installed | `brew install argocd` (Mac) or see https://argo-cd.readthedocs.io/en/stable/cli_installation/ |
| QA pod stuck in `CrashLoopBackOff` | Wrong `DB_HOST` in QA values file | `kubectl logs -n qa deploy/auth-service` — confirm `DB_HOST` in `envs/qa/values-auth-service.yaml` matches the dev RDS endpoint (all environments share the same DB). If it's stale, fix it: `cd zen-gitops-lab1 && yq e '.configmap.DB_HOST = "<CURRENT_DEV_RDS_ENDPOINT>"' -i envs/qa/values-auth-service.yaml && git add envs/qa/values-auth-service.yaml && git commit -m "fix(auth-service): correct QA DB_HOST" && git push origin main` — ArgoCD auto-syncs QA within ~3 minutes, or force it with `argocd app sync auth-service-qa` |
| `auth-service-qa` stays `OutOfSync` after merging QA PR | ArgoCD app repoURL still points at DPP-2026 org | Notify instructor — the QA app needs to point at your fork |
| `Promote to Prod` workflow skips the approval gate | `production` environment not configured in your fork | Create the environment under **Settings → Environments** and add a required reviewer (Part 8.2) |
| `Promote to Prod` workflow fails: `image not found in ECR` | The develop CI run did not complete successfully | Check the `ci-auth-service.yml` run on your develop branch; the image must exist before prod promotion |
| `argocd app sync auth-service-prod` hangs | Prod pod cannot pull the image or connect to RDS | `kubectl describe pod -n prod` and `kubectl logs -n prod deploy/auth-service` to identify the failure |
| Prod pod stuck in `CrashLoopBackOff` | Wrong `DB_HOST` in prod values file | Confirm `DB_HOST` in `envs/prod/values-auth-service.yaml` matches the dev RDS endpoint — all environments share the same DB |
| QA External Secrets show `SecretSyncError` | Dev Secrets Manager paths not reachable from ESO | Confirm `/pharma/dev/db-credentials` and `/pharma/dev/jwt-secret` exist in AWS Secrets Manager (created by Terraform in `zen-infra`) and that `pharma-dev-eso-role` has `GetSecretValue` on those paths |
| Prod External Secrets show `SecretSyncError` | Dev Secrets Manager paths not reachable from ESO | Confirm `/pharma/dev/db-credentials` and `/pharma/dev/jwt-secret` exist in AWS Secrets Manager (created by Terraform in `zen-infra`) — prod reuses the same paths as dev and qa |
