# aks-gitops-production

# Real-World GitOps with AKS, Flux, and Kustomize

This project demonstrates a more realistic **GitOps deployment model** using **Azure Kubernetes Service (AKS)**, **Flux v2**, **GitHub**, and **Kustomize**.

The project manages multiple environments from Git and uses Flux to continuously reconcile the desired configuration with the running AKS cluster.

## Project Goals

The main objectives are to:

- Manage DEV, STAGING, and PROD configurations from Git
- Use Git as the source of truth
- Deploy Kubernetes resources automatically with Flux
- Use Kustomize overlays for environment-specific configuration
- Promote application releases between environments
- Detect and correct configuration drift
- Perform rollbacks using Git history
- Practice troubleshooting Flux and Kubernetes workloads

## Architecture

```text
Developer
    |
    v
GitHub Repository
    |
    v
Flux
    |
    v
AKS Cluster
    |
    +------------------+
    |        |         |
    v        v         v
   DEV    STAGING     PROD
```

Each environment has its own Kubernetes configuration while sharing a common application base.

## Technologies Used

- Azure Kubernetes Service
- Flux v2
- GitHub
- Git
- Kubernetes
- Kustomize
- Azure CLI
- kubectl
- Nginx

## Repository Structure

```text
aks-gitops-production/
|
├── platform/
│   └── namespaces/
│       ├── dev.yaml
│       ├── staging.yaml
│       ├── prod.yaml
│       └── kustomization.yaml
│
└── apps/
    └── web/
        ├── base/
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── kustomization.yaml
        │
        └── overlays/
            ├── dev/
            │   └── kustomization.yaml
            ├── staging/
            │   └── kustomization.yaml
            └── prod/
                └── kustomization.yaml
```

## Environment Design

The project uses a shared application base and separate Kustomize overlays.

```text
Base Kubernetes Configuration
            |
      +-----+-----+
      |     |     |
      v     v     v
     DEV  STAGING PROD
```

Example replica configuration:

```text
DEV       = 1 replica
STAGING   = 2 replicas
PROD      = 3 replicas
```

This allows each environment to use the same core application configuration while applying environment-specific settings.

## 1. Create the Resource Group

```bash
export RG="rg-gitops-production"
export LOCATION="canadacentral"
export AKS="aks-gitops-production"
```

Create the resource group:

```bash
az group create \
  --name "$RG" \
  --location "$LOCATION"
```

## 2. Create the AKS Cluster

```bash
az aks create \
  --resource-group "$RG" \
  --name "$AKS" \
  --location "$LOCATION" \
  --node-count 2 \
  --enable-managed-identity \
  --generate-ssh-keys
```

Get AKS credentials:

```bash
az aks get-credentials \
  --resource-group "$RG" \
  --name "$AKS" \
  --overwrite-existing
```

Verify:

```bash
kubectl get nodes
```

## 3. Validate the Kustomize Configuration

Check the namespace configuration:

```bash
kubectl kustomize platform/namespaces
```

Validate DEV:

```bash
kubectl kustomize apps/web/overlays/dev
```

Validate STAGING:

```bash
kubectl kustomize apps/web/overlays/staging
```

Validate PROD:

```bash
kubectl kustomize apps/web/overlays/prod
```

A dry-run can also be used:

```bash
kubectl apply \
  --dry-run=client \
  -k apps/web/overlays/prod
```

## 4. Push the Configuration to GitHub

```bash
git add .

git commit -m "Create multi-environment GitOps configuration"

git push
```

Git now stores the desired state for all three environments.

## 5. Configure Flux

Set the Git repository URL:

```bash
export GIT_REPO="https://github.com/YOUR-GITHUB-USERNAME/aks-gitops-production"
```

Create the Flux configuration:

```bash
az k8s-configuration flux create \
  --resource-group "$RG" \
  --cluster-name "$AKS" \
  --cluster-type managedClusters \
  --name production-gitops \
  --namespace flux-system \
  --scope cluster \
  --url "$GIT_REPO" \
  --branch main \
  --kustomization \
    name=platform \
    path=./platform/namespaces \
    prune=true \
    sync_interval=1m \
    retry_interval=1m \
  --kustomization \
    name=dev \
    path=./apps/web/overlays/dev \
    prune=true \
    sync_interval=1m \
    retry_interval=1m \
    depends_on='["platform"]' \
  --kustomization \
    name=staging \
    path=./apps/web/overlays/staging \
    prune=true \
    sync_interval=1m \
    retry_interval=1m \
    depends_on='["platform"]' \
  --kustomization \
    name=prod \
    path=./apps/web/overlays/prod \
    prune=true \
    sync_interval=1m \
    retry_interval=1m \
    depends_on='["platform"]'
```

The platform configuration creates the namespaces first.

The environment configurations depend on the platform configuration:

```text
platform
   |
   +--------+--------+
   |        |        |
   v        v        v
  dev    staging    prod
```

## 6. Verify Flux

Check the Flux pods:

```bash
kubectl get pods \
  -n flux-system
```

Check Git sources and Kustomizations:

```bash
kubectl get gitrepositories,kustomizations \
  -A
```

Check the Azure Flux configuration:

```bash
az k8s-configuration flux show \
  --resource-group "$RG" \
  --cluster-name "$AKS" \
  --cluster-type managedClusters \
  --name production-gitops \
  --output table
```

## 7. Verify the Environments

Check namespaces:

```bash
kubectl get namespaces
```

Check DEV:

```bash
kubectl get all -n dev
```

Check STAGING:

```bash
kubectl get all -n staging
```

Check PROD:

```bash
kubectl get all -n prod
```

Verify replica counts:

```bash
kubectl get deployments -n dev
kubectl get deployments -n staging
kubectl get deployments -n prod
```

Expected:

```text
DEV       = 1 replica
STAGING   = 2 replicas
PROD      = 3 replicas
```

## 8. Release Promotion

A new release can first be deployed to DEV.

Example:

```text
DEV       = 1.1.0
STAGING   = 1.0.0
PROD      = 1.0.0
```

Update the DEV Kustomization:

```yaml
RELEASE_VERSION=1.1.0
```

Commit the change:

```bash
git add .

git commit -m "Deploy release 1.1.0 to dev"

git push
```

Flux automatically reconciles DEV.

Once validated, the same release can be promoted to STAGING and later PROD using additional Git commits or pull requests.

```text
DEV
 |
 v
STAGING
 |
 v
PROD
```

## 9. Test Configuration Drift

Manually modify production:

```bash
kubectl scale deployment web \
  -n prod \
  --replicas=10
```

Git still defines:

```text
PROD = 3 replicas
```

Flux detects the difference and restores the deployment to the Git-defined state.

Verify:

```bash
kubectl get deployment web \
  -n prod
```

Expected result:

```text
3 replicas
```

This demonstrates automatic drift correction.

## 10. Test a GitOps Rollback

View Git history:

```bash
git log --oneline
```

Revert a deployment commit:

```bash
git revert <COMMIT-ID>
```

Push the rollback:

```bash
git push
```

Flux detects the reverted Git state and reconciles AKS automatically.

```text
Git Revert
    |
    v
GitHub
    |
    v
Flux
    |
    v
AKS Rollback
```

## Troubleshooting

### Check Flux Reconciliation

```bash
kubectl get kustomizations -A
```

### Check Git Repository Status

```bash
kubectl get gitrepositories -A
```

### Inspect Flux Source Controller

```bash
kubectl logs \
  -n flux-system \
  deployment/source-controller
```

### Inspect Kustomize Controller

```bash
kubectl logs \
  -n flux-system \
  deployment/kustomize-controller
```

### Check Failing Pods

```bash
kubectl get pods -n prod
```

Describe a Pod:

```bash
kubectl describe pod \
  -n prod \
  <POD-NAME>
```

Check logs:

```bash
kubectl logs \
  -n prod \
  <POD-NAME>
```

Common issues include:

```text
ImagePullBackOff
CrashLoopBackOff
Pending
CreateContainerConfigError
DependencyNotReady
Invalid YAML
Incorrect Git path
Repository authentication problems
```

## Important GitOps Principle

Changes should normally be made in Git rather than directly in AKS.

Instead of:

```text
kubectl edit
kubectl scale
kubectl apply
```

the normal workflow becomes:

```text
Edit configuration
      |
      v
Commit
      |
      v
Pull Request
      |
      v
Review / Approval
      |
      v
Merge
      |
      v
Flux
      |
      v
AKS
```

## CI and GitOps

In a larger Azure DevOps environment, a CI pipeline can build and test the application while Flux handles deployment.

```text
Application Code
      |
      v
Azure Pipelines
      |
      +-- Build
      +-- Test
      +-- Security Scan
      +-- Push Image to ACR
      |
      v
GitOps Repository
      |
      v
Flux
      |
      v
AKS
```

The key distinction is:

```text
CI builds the application.

GitOps deploys and reconciles the application.
```

## Key Skills Demonstrated

This project demonstrates experience with:

- GitOps principles
- Flux v2
- AKS
- Kubernetes
- Kustomize
- Multi-environment deployments
- Git-based release promotion
- Configuration drift detection
- Automated reconciliation
- Git-based rollback
- Kubernetes troubleshooting
- Declarative infrastructure management

## Cleanup

Delete all Azure resources created for the project:

```bash
az group delete \
  --name rg-gitops-production \
  --yes \
  --no-wait
```

## Summary

This project demonstrates a production-style GitOps workflow where Kubernetes configuration is stored in Git and continuously reconciled by Flux.

The final workflow is:

```text
Developer
    |
    v
Git Change
    |
    v
Pull Request / Commit
    |
    v
GitHub
    |
    v
Flux
    |
    v
AKS
    |
    +--------+--------+
    |        |        |
    v        v        v
   DEV    STAGING    PROD
```

The project provides practical experience with managing multiple Kubernetes environments through Git while using Flux to automate deployment, reconciliation, drift correction, and rollback.