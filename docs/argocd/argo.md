# GitOps Context: Managing Argo CD with Argo CD (Kustomize + Helm)

This document provides context for the declarative, self-managing architecture for Argo CD using a **Kustomize-wrapped Helm** layout. This pattern ensures our GitOps controller is version-controlled, easily upgradable via global base configurations, and safely patchable across multiple clusters without configuration drift.

---

## 1. Directory Architecture

The repository isolates the global shared configuration (`base`) from individual cluster environments (`overlays`), keeping the infrastructure DRY (Don't Repeat Yourself).

```text
crossplane-example/
├── docs/argocd/argo.md      # This context file
├── bootstrap/
│   └── root-app.yaml        # The "App-of-Apps" bootstrap manifest
└── argocd-infra/
    ├── base/
    │   ├── kustomization.yaml   # Fetches Helm chart & tracks core resources
    │   └── shared-values.yaml   # Global Argo CD configurations (90% of settings)
    └── overlays/
        ├── development/
        │   └── kustomization.yaml  # Dev-specific overrides (Lightweight)
        └── production/
            ├── kustomization.yaml  # Prod-specific overrides (HA, RBAC, Patches)
            └── patches.yaml        # Surgical structural patches (e.g., scale-up)

```

---

## 2. Configuration Files

### 2.1 The Global Base (`argocd-infra/base/`)

The base definition anchors the official Helm chart version and establishes company-wide defaults.

**`argocd-infra/base/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd

helmCharts:
  - name: argo-cd
    repo: [https://argoproj.github.io/argo-helm](https://argoproj.github.io/argo-helm)
    version: 7.1.1 # Define the global chart version here
    releaseName: argocd
    namespace: argocd
    valuesFile: shared-values.yaml

```

**`argocd-infra/base/shared-values.yaml`**

```yaml
global:
  domain: argocd.internal.net

configs:
  params:
    server.insecure: "true" # Offloads SSL handling to ingress/load-balancer

  cm:
    # Company-wide UI customization or standard parameters
    ui.bannercontent: "Managed entirely via GitOps (crossplane-example)"

```

---

### 2.2 Environment Overlays (`argocd-infra/overlays/`)

Overlays inherit the base configuration and apply specific mutations or append missing resources natively.

#### Development Environment

**`argocd-infra/overlays/development/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Inherit everything from base unmodified for Dev
resources:
  - ../../base

```

#### Production Environment (With High Availability Patches)

**`argocd-infra/overlays/production/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patches:
  - path: patches.yaml

```

**`argocd-infra/overlays/production/patches.yaml`**

```yaml
# Surgical patch: Scale up repo server replicas for HA without touching Helm variables
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  replicas: 3
---
# Surgical patch: Inject a production-only corporate configuration key into argocd-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
$patch: delete
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  # Appends or modifies specific strings in the rendered ConfigMap
  admin.enabled: "false" # Disabling local admin account in Prod

```

---

## 3. The Root Bootstrap Application (`bootstrap/`)

This manifest acts as the core engine. When applied, it points Argo CD to the target overlay, closing the loop and allowing Argo CD to take ownership of itself.

**`bootstrap/root-app.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-root-manager
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io # Safeguard delete tracking
spec:
  project: default
  source:
    # Git repository endpoint for crossplane-example
    repoURL: 'https://github.com/letajmal/crossplane-example.git'
    targetRevision: HEAD
    # Target path determines which cluster flavor this cluster runs
    # Change to 'argocd-infra/overlays/production' for production setups
    path: argocd-infra/overlays/development
  destination:
    server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true # Highly recommended to reduce cluster API load

```

---

## 4. Operational Playbook (How to Bootstrap)

To activate this declarative loop on a brand-new cluster, execute the following three commands in sequence:

```bash
# Step 1: Secure the targeted namespace
kubectl create namespace argocd

# Step 2: Spin up a temporary minimal runtime of Argo CD to execute the engine
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

# Step 3: Seed the GitOps Root Application
kubectl apply -f bootstrap/root-app.yaml

```